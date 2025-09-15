---
title: "Tutoriais — Linux, Containers, Middleware & Red Hat"
layout: default
nav_order: 1
---
# 📚 Tutoriais — Linux, Containers, Middleware & Ecossistema Red Hat

Repositório público de guias práticos e diretos sobre **Linux**, **containers** (Podman/Buildah/Skopeo/Portainer), **Kubernetes**, **Middleware** (JBoss EAP, WebLogic), **Redes**, **Segurança**, **Certificações** e boas práticas alinhadas ao **ecossistema Red Hat**.

> Este repo serve como **hub**: cada guia é 100% em **Markdown**, com comandos separados em blocos `bash`, uso de `vim` para edições e foco em **modo texto** para instalações.

---

## 📑 Sumário

- [🔎 Sobre](#-sobre)
- [🧭 Padrões editoriais](#-padrões-editoriais)
- [🗂️ Estrutura de pastas](#️-estrutura-de-pastas)
- [📝 Índice de artigos](#-índice-de-artigos)
  - [Linux](#linux)
    - [Guia — Instalação & Pós-instalação do Rocky Linux 10 Minimal](./linux/rocky-linux-10-minimal.md)

  - [Containers](#containers)
  - [Ansible](#ansible)
    - [O que e AWX](./ansible/o_que_e_AWX.md)
  - [Kubernetesx](#kubernetes)
  - [Middleware](#middleware)
  - [Redes](#redes)
  - [Segurança](#segurança)
  - [Certificações](#certificações)
  - [Extras](#extras)
- [🚀 Fluxo de publicação (GitHub → LinkedIn)](#-fluxo-de-publicação-github--linkedin)
- [🧩 Como contribuir](#-como-contribuir)
- [⚖️ Licença](#️-licença)
- [✍️ Assinatura](#️-assinatura)

---

## 🔎 Sobre
Este repositório organiza **tutoriais práticos** que servem tanto para profissionais de **Infra/DevOps** quanto para quem está se preparando para **certificações** (RHCSA, RHCE, RHCJA, etc.).  
Cada guia traz **contexto, passos claros, comandos prontos**, e um **checklist** final para validação.

---

## 🧭 Padrões editoriais

- **Formato**: Markdown puro (.md), **sem CSS**.
- **Comandos**: sempre em blocos `bash`, com comentários quando útil.
- **Editor**: preferir `vim` ao editar arquivos.
- **Instalação**: priorizar **modo texto** (text-mode) quando aplicável.
- **Estrutura dos guias**: Título → Resumo → Sumário (âncoras limpas) → Seções numeradas/claras → Checklist/FAQ → Referências rápidas → Assinatura.
- **Boas práticas**: segurança por padrão (firewall/SELinux), idempotência, reprodução em HML/DEV antes de PROD.

---


