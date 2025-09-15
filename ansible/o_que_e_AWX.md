---
title: "O que é o AWX?"
parent: "Ansible (AAP e AWX)"
nav_order: 1
---

# O que é o **AWX?**

## 🎯 Visão geral
**AWX** é a plataforma **upstream** (código aberto) para orquestrar automações **Ansible** com **UI web**, **API REST** e **RBAC**. Ele centraliza inventários, credenciais e projetos Git, padroniza *Job Templates* e *Workflows*, agenda execuções, integra *webhooks* e armazena logs/resultados de forma auditável.

---

## 🧠 Arquitetura em alto nível

- **web**: UI e API REST.  
- **task**: agendamento, fila e execução de *jobs*.  
- **redis**: mensageria/cache para *web* e *task*.  
- **postgres**: persistência transacional (eventos, artefatos, configurações).  
- **Execution Environments (EE)**: imagens OCI com Ansible + coleções, garantindo reprodutibilidade.  
- **Instalação moderna**: **AWX Operator** (CRDs no Kubernetes) — você define um recurso `AWX` em YAML e o Operator reconcilia Pods/Services/Ingress/PVCs.

---

## 🔧 Fluxo de uso

1. **Projects** apontam para repositórios Git (playbooks/roles/coleções).  
2. **Inventories** (estáticos ou dinâmicos) descrevem alvos.  
3. **Credentials** (SSH, cloud, Vault, Git…) são vinculadas com escopo.  
4. **Job Templates** definem *playbook* + inventário + credenciais + *survey/extra_vars*.  
5. **Workflows** encadeiam *jobs* com decisões (sucesso/falha), aprovações e *fan-out*.  
6. **Schedules** executam de forma recorrente; **webhooks** disparam por evento (GitOps).  
7. **Logs e artefatos** ficam rastreáveis, prontos para observabilidade.

---

## ✅ Pontos fortes

- **UI + API-first**: tudo que a UI faz, a API expõe (integra fácil em pipelines).  
- **GitOps nativo**: *webhooks* de GitHub/GitLab/Bitbucket e sincronismo de projetos.  
- **RBAC granular**: organizações, times e permissões por objeto.  
- **EEs versionados**: execução imutável e reprodutível entre ambientes.  
- **Workflows visuais**: condicionais, aprovações e reutilização de *templates*.  
- **Escalabilidade**: concorrência por réplicas e *requests/limits*.  
- **Observabilidade**: *stdout* completo, eventos e métricas via API.  
- **Modelo declarativo**: CRDs (`AWX`, `AWXBackup`, `AWXRestore`).  
- **Integrações**: Slack/Email/Teams; inventários dinâmicos; *credential plugins*.

---

## 🔁 AWX ↔ AAP (Red Hat Ansible Automation Platform)

- **AWX**: origem comunitária do *Controller* — mesma filosofia de automação, UI, API e EEs.  
- **AAP**: empacotamento **enterprise**: **Automation Controller** (baseado no AWX), **Automation Hub**, **Automation Mesh**, **EDA Controller**, suporte e ciclo de vida corporativo.

Tradução prática: **AWX** acelera e dá flexibilidade; **AAP** adiciona governança, catálogo certificado e suporte para escala corporativa.

---

## 🏭 Perfis que usam AWX em produção

- **Times de Plataforma/SRE** com K8s e GitOps.  
- **Startups/scale-ups** focadas em velocidade e *self-service*.  
- **Integradores/MSPs** com catálogos multi-ambiente.  
- **Universidades/labs** para ensino e pesquisa.  
- **Ops internas** para padronizar *runbooks* e **reduzir toil**.

---

## 🛠️ Boas práticas de implantação

- **Versionamento consciente**: *pin* de Operator/AWX e leitura de *release notes*.  
- **EEs imutáveis**: `ansible-builder` + *lockfiles* de coleções.  
- **Fonte da verdade = Git**: *merge* aprovado antes de mudanças operacionais.  
- **Segurança**: plugins de credencial, Vault/KMS, **RBAC** por time/objeto.  
- **Armazenamento**: PVC **RWX** confiável; PostgreSQL dimensionado.  
- **Observabilidade**: coleta de logs/métricas e alertas de *job failures*; auditoria ativa.  
- **Backups**: `AWXBackup` agendado e teste periódico de `AWXRestore`.

---

## 🚀 Onde o AWX brilha

- Catálogo de automações para **self-service**.  
- Padronização de **runbooks** e operações *day-2*.  
- Integração CI/CD: do **PR** ao *deploy* (webhook + workflow).  
- **Infra como código** com coleções e EEs dedicados por domínio.  
- Execuções governadas por **RBAC** e trilha de auditoria.

---

## Glossário

- **EE (Execution Environment)**: imagem OCI com Ansible + coleções usada para rodar *jobs*.  
- **Job Template**: definição parametrizada de execução de um playbook.  
- **Workflow Job Template**: orquestra *jobs* com regras e dependências.  
- **Inventory**: lista/grupos de hosts (estático ou dinâmico).  
- **Credential**: identidade/segredo consumido em runtime.  
- **Operator/CRD**: controlador K8s que reconcilia recursos a partir de YAML.

---

Criado por **Jeferson Salles**  
LinkedIn: [https://www.linkedin.com/in/jmsalles/](https://www.linkedin.com/in/jmsalles/)  
E-mail: [jefersonmattossalles@gmail.com](mailto:jefersonmattossalles@gmail.com)
