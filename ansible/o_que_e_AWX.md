# O que é o **AWX**

## 🎯 Visão geral

**AWX** é a plataforma **upstream** (código aberto) para orquestrar automações **Ansible** com **UI web**, **API REST** e **RBAC**. Ele centraliza inventários, credenciais e projetos Git, padroniza *Job Templates* e *Workflows*, agenda execuções, integra *webhooks* e armazena logs/resultados de forma auditável.

---

## 🧠 Arquitetura em alto nível

* **web**: UI e API REST.
* **task**: agendamento, fila e execução de *jobs*.
* **redis**: mensageria/cache para *web* e *task*.
* **postgres**: persistência transacional (eventos, artefatos, configurações).
* **Execution Environments (EE)**: imagens OCI com ansible + coleções, garantindo reprodutibilidade.
* **Instalação moderna**: **AWX Operator** (CRDs no Kubernetes) — define-se um `AWX` em YAML e o Operator reconcilia Pods/Services/Ingress/PVCs.

---

## 🔧 Fluxo de uso

1. **Projects** apontam para repositórios Git (playbooks/roles/coleções).
2. **Inventories** (estáticos ou dinâmicos) descrevem alvos.
3. **Credentials** (SSH, cloud, Vault, Git…) são vinculadas com escopo.
4. **Job Templates** definem *playbook* + inventário + credenciais + *survey/extra\_vars*.
5. **Workflows** encadeiam *jobs* com decisões (sucesso/falha), aprovações e *fan-out*.
6. **Schedules** executam de forma recorrente; **webhooks** disparam por evento (GitOps).
7. **Logs e artefatos** ficam rastreáveis, prontos para observabilidade.

---

## ✅ Pontos fortes (o que o AWX entrega muito bem)

* **UI + API-first**: tudo que a UI faz a API expõe (integração fácil com pipelines).
* **GitOps** nativo: *webhooks* de GitHub/GitLab/Bitbucket e sincronismo de projetos.
* **RBAC granular**: organizações, times e permissões por objeto.
* **EEs versionados**: execução imutável e reprodutível entre ambientes.
* **Orquestração visual**: *Workflows* com condicionais, aprovações e reutilização de *templates*.
* **Escalabilidade elástica**: concorrência controlada por réplicas e *requests/limits*.
* **Observabilidade**: *stdout* completo, eventos e métricas acessíveis por API.
* **Modelo declarativo**: instalação/upgrade/backup via CRDs (`AWX`, `AWXBackup`, `AWXRestore`).
* **Integrações**: Slack/Email/Teams, inventários dinâmicos de clouds, *credential plugins*, Ansible Collections.

---

## 🔁 Analogia direta **AWX ↔ AAP (Red Hat Ansible Automation Platform)**

* **AWX** é a **origem comunitária** do *controller*, com a mesma filosofia de automação declarativa, UI, API e EEs.
* **AAP** é o **empacotamento enterprise** do ecossistema: inclui **Automation Controller** (baseado no AWX), **Automation Hub** (conteúdos certificados), **Automation Mesh**, **EDA Controller** e serviços de ciclo de vida corporativo.
* Tradução prática: **AWX** acelera equipes a orquestrar Ansible com velocidade e flexibilidade; **AAP** acrescenta camadas e serviços para operações em larga escala e governança corporativa.

---

## 🏭 Quem usa **AWX** em produção (perfis típicos)

* **Times de Plataforma/SRE** que operam Kubernetes e GitOps no dia a dia.
* **Startups e scale-ups** com foco em velocidade, automação e *self-service* para devs.
* **Integradores/MSPs** com catálogos de automações multi-ambiente.
* **Universidades e labs** para ensino, pesquisa e ambientes compartilhados.
* **Operações internas** para padronizar *runbooks*, *day-2 ops* e tarefas recorrentes.

---

## 🛠️ Boas práticas de implantação

* **Versionamento consciente**: *pin* de versões do Operator/AWX e *release notes* no ciclo de mudança.
* **EEs imutáveis**: construa com *ansible-builder* e *lockfiles* de coleções.
* **Fonte da Verdade = Git**: *merge* revisado antes de qualquer mudança operacional.
* **Segurança e segredos**: plugins de credencial, Vault/KMS, *RBAC* por time/objeto.
* **Armazenamento**: PVC **RWX** confiável para *projects* e PostgreSQL bem dimensionado.
* **Observabilidade**: coleta de logs/métricas e alertas de *job failures*; auditoria ativada.
* **Esteira de backup**: `AWXBackup` agendado e teste periódico de `AWXRestore`.

---

## 🚀 Onde o AWX brilha

* Catálogo de automações para **self-service** de times.
* Padronização de *runbooks* operacionais e **redução de toil**.
* Integração CI/CD: do **PR** ao *deploy* com *webhook* e *workflow* de aprovação.
* **Infra como código** com coleções e *EE* dedicados por domínio (rede, banco, cloud).
* Execuções governadas por **RBAC** e trilha de auditoria.

---

## Glossário

* **EE (Execution Environment)**: imagem OCI com ansible + coleções usada para rodar *jobs*.
* **Job Template**: definição parametrizada de execução de um playbook.
* **Workflow Job Template**: orquestra vários *jobs* com regras e dependências.
* **Inventory**: lista/grupos de hosts (estático ou dinâmico).
* **Credential**: identidade/segredo consumido em tempo de execução.
* **Operator/CRD**: controlador Kubernetes que reconcilia recursos a partir de YAML.

---

Criado por **Jeferson Salles**
LinkedIn: [https://www.linkedin.com/in/jmsalles/](https://www.linkedin.com/in/jmsalles/)
E-mail: [jefersonmattossalles@gmail.com](mailto:jefersonmattossalles@gmail.com).
