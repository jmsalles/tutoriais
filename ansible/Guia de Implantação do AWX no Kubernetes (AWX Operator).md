# Guia de Implantação do **AWX** no Kubernetes (AWX Operator) — **PVC RWX *default***

> Identidade “Guia” do seu .md: texto limpo, **comandos sempre em blocos `bash`**, edições com **`vim`**, âncoras no Sumário e **assinatura no final**. Sem CSS.

---

## 🎯 Resumo

Instalação do **AWX** via **AWX Operator** em um cluster Kubernetes que **já possui uma StorageClass RWX (ex.: `nfs-client`) marcada como *default***. Exposição da UI via **Ingress NGINX** com **TLS**.
Fluxo: pré-checagens → operator com **tag pinada** → secrets (`admin`/`SECRET_KEY`) → TLS/DNS → CR do AWX (Ingress+RWX) → verificação/ajustes → backup/restore → upgrade → troubleshooting.

---

## 🧭 Sumário

* [Pré-requisitos e premissas](#prérequisitos-e-premissas)
* [Checagens do cluster](#checagens-do-cluster)
* [Namespace e contexto](#namespace-e-contexto)
* [Instalar o AWX Operator (tag fixa)](#instalar-o-awx-operator-tag-fixa)
* [Criar Secrets (admin e SECRET\_KEY)](#criar-secrets-admin-e-secret_key)
* [TLS e DNS para o Ingress](#tls-e-dns-para-o-ingress)
* [Implantar o AWX (Ingress + RWX default)](#implantar-o-awx-ingress--rwx-default)
* [Verificações pós-deploy](#verificações-pósdeploy)
* [Backup & Restore (CRDs)](#backup--restore-crds)
* [Upgrade seguro](#upgrade-seguro)
* [Desinstalação](#desinstalação)
* [Troubleshooting rápido](#troubleshooting-rápido)
* [Checklist final](#checklist-final)

---

## Pré-requisitos e premissas

* Kubernetes saudável (1+ nós) com **Ingress Controller NGINX** acessível (Service `LoadBalancer`/MetalLB **ou** `NodePort`).
* **StorageClass RWX** dinâmica (ex.: `nfs-client`) marcada como ***default*** no cluster.
* `kubectl` apontando para o cluster, usuário com privilégios para criar CRDs e recursos no namespace `awx`.

---

## Checagens do cluster

```bash
# Nós e versões
kubectl get nodes -o wide
kubectl version --short

# StorageClass (★ indica a default) — precisa ser RWX
kubectl get storageclass

# Ingress NGINX (ajuste o namespace se for diferente)
kubectl get ingressclass
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

---

## Namespace e contexto

```bash
kubectl create namespace awx
kubectl config set-context --current --namespace=awx
```

---

## Instalar o AWX Operator (tag fixa)

> **Sempre pinne** a versão. Exemplo abaixo com `2.9.0`.

```bash
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git fetch --tags
git checkout tags/2.9.0
vim kustomization.yaml
```

Cole e salve:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.9.0
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.9.0
namespace: awx
```

Aplicar:

```bash
kubectl apply -k .
kubectl get pods -n awx
# espere: awx-operator-controller-manager em Running
```

---

## Criar Secrets (admin e `SECRET_KEY`)

```bash
# senha do usuário admin
ADMIN_PASS='TroqueEstaSenhaSuperForte!'
kubectl create secret generic awx-admin-password \
  --from-literal=password="$ADMIN_PASS" -n awx

# chave de criptografia do AWX (>= 32 chars)
SECRET_KEY="$(openssl rand -base64 48 | tr -d '\n')"
kubectl create secret generic awx-secret-key \
  --from-literal=secret_key="$SECRET_KEY" -n awx
```

> Sem `SECRET_KEY`, o `awx-web` entra em **CrashLoopBackOff** (Django “SECRET\_KEY must not be empty.”).

---

## TLS e DNS para o Ingress

Exemplo com FQDN `awx.seulab.local` (use o seu domínio).

```bash
# certificado autoassinado (para produção, use cert-manager + ACME)
openssl req -x509 -nodes -days 825 -newkey rsa:4096 \
  -keyout awx.key -out awx.crt -subj "/CN=awx.seulab.local"

kubectl create secret tls awx-tls --key awx.key --cert awx.crt -n awx
```

**DNS:** aponte `awx.seulab.local` para o **EXTERNAL-IP** do Service `ingress-nginx-controller`. Em lab, pode usar `/etc/hosts`.

---

## Implantar o AWX (Ingress + RWX default)

> Para a série do operator usada aqui, use os campos **legados** `hostname`/`ingress_tls_secret` e **`ingress_annotations` como BLOCO DE TEXTO**.

```bash
vim awx.yml
```

Cole e salve:

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: ClusterIP
  ingress_type: ingress
  ingress_class_name: nginx

  # Host e TLS no Ingress
  hostname: awx.seulab.local
  ingress_tls_secret: awx-tls

  # Anotações do NGINX (string multilinha, NÃO mapa)
  ingress_annotations: |
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/enable-websocket: "true"

  # Credenciais e SECRET_KEY
  admin_user: admin
  admin_password_secret: awx-admin-password
  secret_key_secret: awx-secret-key

  # Persistência de Projects (RWX default já supre a SC)
  projects_persistence: true
  projects_storage_access_mode: ReadWriteMany
  projects_storage_size: 20Gi

  # (Recomendado) injeta SECRET_KEY por env também — cinto e suspensório
  extra_env:
    - name: SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: awx-secret-key
          key: secret_key

  # Tuning básico de recursos (ajuste ao seu cluster)
  web_resource_requirements:
    requests: { cpu: 250m, memory: "512Mi" }
    limits:   { cpu: "2",  memory:  "3Gi" }
  task_resource_requirements:
    requests: { cpu: 500m, memory:  "1Gi"  }
    limits:   { cpu: "2",  memory:  "4Gi" }
  ee_resource_requirements:
    requests: { cpu: 250m, memory: "512Mi" }
    limits:   { cpu: "2",  memory:  "2Gi" }
```

Aplicar e acompanhar:

```bash
kubectl apply -f awx.yml
kubectl get pods,svc,ingress,pvc -n awx -w
```

Esperado:

* `awx-postgres-13-0` **Running**
* `awx-task-…` **Running/Ready (4/4)**
* `awx-web-…` **Running/Ready (3/3)**
* `awx-projects-claim` **Bound** (RWX, 20Gi)
* `ingress/awx…` com host `awx.seulab.local`

---

## Verificações pós-deploy

```bash
# Endpoints do service (se vazio/notReady => Ingress 503)
kubectl -n awx get endpoints awx-service -o wide

# Teste interno (sem TLS, direto no Service)
kubectl -n awx run curl --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -lc 'wget -S -O- http://awx-service 2>&1 | head -n1'

# Teste externo via ingress (ajuste EXTERNAL-IP)
curl -skI https://<EXTERNAL-IP> -H 'Host: awx.seulab.local' | head -n1
```

Abra `https://awx.seulab.local/` no navegador (admin / sua senha).
Na UI: **Settings → System → Base URL** = `https://awx.seulab.local`.

---

## Backup & Restore (CRDs)

### PVC de backup

```bash
vim awx-backup-pvc.yml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: awx-backup-pvc
  namespace: awx
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests: { storage: 20Gi }
```

```bash
kubectl apply -f awx-backup-pvc.yml
```

### Backup

```bash
vim awx-backup.yml
```

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWXBackup
metadata:
  name: awx-backup
  namespace: awx
spec:
  deployment_name: awx
  backup_pvc: awx-backup-pvc
  no_log: false
```

```bash
kubectl apply -f awx-backup.yml
kubectl -n awx get jobs
```

### Restore

```bash
vim awx-restore.yml
```

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWXRestore
metadata:
  name: awx-restore
  namespace: awx
spec:
  deployment_name: awx
  backup_pvc: awx-backup-pvc
```

```bash
kubectl apply -f awx-restore.yml
```

---

## Upgrade seguro

1. **Faça backup**.
2. Atualize a **tag** do operator em `kustomization.yaml`.
3. `kubectl apply -k .` para atualizar o operator.
4. Reaplique seu `awx.yml` (o operator reconcilia).
5. Monitore:

```bash
kubectl -n awx logs -f deploy/awx-operator-controller-manager -c awx-manager
kubectl -n awx get pods -w
```

---

## Desinstalação

**Remover apenas o AWX (mantém operator/PVCs):**

```bash
kubectl -n awx delete awx awx
kubectl -n awx get pvc
```

**Remover tudo (AWX, operator e namespace):**

```bash
kubectl -n awx delete awx awx --ignore-not-found
cd ~/awx-operator && kubectl delete -k .
kubectl delete namespace awx
```

> Atenção: dependendo da **reclaimPolicy** da sua SC, PVs podem ser apagados ao deletar o namespace.

---

## Troubleshooting rápido

* **Página 503 no Ingress** → `kubectl -n awx get endpoints awx-service -o wide` (vazio/notReady = `awx-web` não saudável).
* **`SECRET_KEY must not be empty`** → garanta o Secret e o apontamento no CR, e injete `extra_env`:

  ```bash
  kubectl -n awx get secret awx-secret-key -o jsonpath='{.data.secret_key}' | base64 --decode; echo
  kubectl -n awx patch awx awx --type merge -p '{"spec":{"secret_key_secret":"awx-secret-key","extra_env":[{"name":"SECRET_KEY","valueFrom":{"secretKeyRef":{"name":"awx-secret-key","key":"secret_key"}}}]}}'
  kubectl -n awx delete pod -l app.kubernetes.io/name=awx-web
  ```
* **PVC Pending** (raro com RWX default) → `kubectl -n awx describe pvc awx-projects-claim`.
* **OOM / pouca memória** → aumente `*_resource_requirements` e reinicie `awx-web`.
* **Uploads 413** → já coberto com `proxy-body-size: "0"` no Ingress.
* **Base URL/redirects** → ajuste em **Settings → System → Base URL**.

---

## Checklist final

* [ ] SC **RWX** configurada e **default**
* [ ] `ingress-nginx` acessível (LB/NodePort)
* [ ] Namespace `awx` criado e contexto fixado
* [ ] Operator aplicado com **tag pinada**
* [ ] Secrets `awx-admin-password` e `awx-secret-key` criadas
* [ ] `awx.yml` aplicado (Ingress + TLS + RWX)
* [ ] `awx-web` **3/3 Ready**, `awx-task` **4/4 Ready**, `awx-service` com **endpoints**
* [ ] Acesso à UI e **Base URL** ajustada
* [ ] Backup agendado/testado
* [ ] Procedimento de **upgrade** documentado

---

Criado por **Jeferson Salles**
LinkedIn: [https://www.linkedin.com/in/jmsalles/](https://www.linkedin.com/in/jmsalles/)
E-mail: [jefersonmattossalles@gmail.com](mailto:jefersonmattossalles@gmail.com)
