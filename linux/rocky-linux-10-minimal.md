---
title: "Guia — Instalação & Pós-instalação do Rocky Linux 10 Minimal"
parent: Linux
nav_order: 1
---
# 🚀 Guia — Instalação & Pós-instalação do Rocky Linux 10 Minimal (com explicações)

## 📌 Resumo
Procedimento **padrão** para instalar (em **modo texto**) e preparar o **Rocky Linux 10 minimal** para uso profissional. Inclui:
- Repositórios (EPEL/CRB), DNF e atualização segura.
- **Pacotes base** com **explicação do porquê** de cada um.
- Rede/diagnóstico, **toolchain** de compilação, containers (Podman/Buildah/Skopeo).
- SELinux, Firewall (por **serviço/porta/protocolo**), **timezone** e NTP (chrony).
- Virtualização (KVM/libvirt) e **pós-reboot** (tuned, fstrim, sysctl).
- Checklist final e troubleshooting rápido.

> 🎯 Esta série abordará **Linux, containers, Portainer, Kubernetes, WebLogic, JBoss, certificações, ecossistema Red Hat** e **boas práticas**.

---

## 📑 Sumário
- [🧭 Visão geral e objetivos](#-visão-geral-e-objetivos)
- [💿 Instalação (modo texto)](#-instalação-modo-texto)
- [🔄 DNF, CRB e EPEL](#-dnf-crb-e-epel)
- [🛠️ Pacotes base (com explicações)](#️-pacotes-base-com-explicações)
- [🌐 Rede e diagnóstico (com explicações)](#-rede-e-diagnóstico-com-explicações)
- [⚙️ Compilação e toolchain (com explicações)](#️-compilação-e-toolchain-com-explicações)
- [🔒 SELinux: checar, alterar e observações](#-selinux-checar-alterar-e-observações)
- [🔥 Firewalld: por serviço, porta e protocolo](#-firewalld-por-serviço-porta-e-protocolo)
- [⏰ Timezone e NTP (chrony)](#-timezone-e-ntp-chrony)
- [💻 Virtualização (KVM/libvirt)](#-virtualização-kvmlibvirt)
- [🐳 Containers (Podman/Buildah/Skopeo)](#-containers-podmanbuildahskopeo)
- [🧰 Extras úteis do dia a dia](#-extras-úteis-do-dia-a-dia)
- [🔧 Pós-reboot: tuned, fstrim e sysctl](#-pósreboot-tuned-fstrim-e-sysctl)
- [🧪 Checklist final](#-checklist-final)
- [🩺 Troubleshooting rápido](#-troubleshooting-rápido)
- [✅ Conclusão](#-conclusão)

---

## 🧭 Visão geral e objetivos
- **Minimal** = menor superfície de ataque + desempenho + controle.
- Base ideal para **JBoss/WebLogic**, **containers** e **Kubernetes**.
- Alinhado ao ecossistema **RHEL/Red Hat**.

---

## 💿 Instalação (modo texto)
1) Baixe a ISO **Minimal** no site oficial.  
2) Boot pela mídia e selecione **modo texto** (text mode).
3) Durante a instalação:
   - Idioma/teclado conforme preferência.
   - **Particionamento**: LVM recomendado em servidor.
   - Defina senha do `root` e crie usuário admin (que entra no sudo).
   - Selecione **Minimal Install**.
4) Finalize e **reinicie**.

> Dica: em ambientes críticos, separe `/var`, `/home`, `/tmp` e habilite LVM para snapshots/migração.

---

## 🔄 DNF, CRB e EPEL
Habilite plugins e repositórios adicionais essenciais.

```bash
sudo dnf -y update
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --set-enabled crb
sudo dnf -y install epel-release
```

**Por que isso?**  
- `dnf-plugins-core`: habilita `config-manager`, repos extras e recursos de gestão.  
- `crb`: (CodeReady Builder) oferece bibliotecas/headers úteis para builds.  
- `epel-release`: provê muitos utilitários mantidos pela comunidade (qualidade alta).

---

## 🛠️ Pacotes base (com explicações)

```bash
sudo dnf -y install \
  vim-enhanced bash-completion \
  man-db man-pages \
  less tree which \
  tar zip unzip bzip2 xz pigz p7zip p7zip-plugins \
  wget curl rsync \
  openssh-clients ca-certificates
```

**Por que cada um?**
- `vim-enhanced`: editor padrão para **editar configs** (recomendado em servidores).  
- `bash-completion`: autocompletar para comandos/argumentos — reduz erros e agiliza.  
- `man-db` `man-pages`: manuais offline (indispensável em ambiente fechado).  
- `less`: pager eficiente para logs e saídas longas.  
- `tree`: visualiza estrutura de diretórios rapidamente.  
- `which`: localiza binários no `PATH`.  
- `tar` `zip` `unzip` `bzip2` `xz` `pigz` `p7zip*`: **compressão** (inclui paralela via `pigz`) e formatos legados.  
- `wget` `curl`: download e **testes HTTP/HTTPS/APIs**.  
- `rsync`: sincronização eficiente (backups/deploys).  
- `openssh-clients`: `ssh`, `scp`, `sftp` — acesso remoto seguro.  
- `ca-certificates`: cadeia de certificados atualizada (TLS/HTTPS sem dor de cabeça).

> Dica: torne o `vim` seu editor padrão no shell do usuário:
>
> ```bash
> echo 'export EDITOR=vim' | sudo tee -a /etc/profile.d/99-editor.sh
> ```

---

## 🌐 Rede e diagnóstico (com explicações)

```bash
sudo dnf -y install \
  iproute iputils net-tools ethtool \
  nmap-ncat nmap tcpdump traceroute mtr \
  bind-utils socat telnet \
  lsof strace htop iotop perf \
  policycoreutils setools-console
```

**Por que cada um?**
- `iproute` `iputils`: `ip`, `ping` — **base moderna** de rede.  
- `net-tools`: `ifconfig`, `netstat` (legado, ainda útil em troubleshooting e scripts).  
- `ethtool`: estatísticas e tunables de interface (offloads, velocidade, duplex).  
- `nmap-ncat` `nmap`: verificação de portas e serviços; `ncat` para testes de socket.  
- `tcpdump`: captura de pacotes — a “lupa” da rede.  
- `traceroute` `mtr`: rota/latência com visão contínua (MTR é ouro).  
- `bind-utils`: `dig`, `nslookup` — **DNS** sem adivinhação.  
- `socat` `telnet`: testes de conectividade bruta e tunelamentos rápidos.  
- `lsof`: quem abriu qual arquivo/porta.  
- `strace`: rastreia chamadas de sistema (debug fino de processos).  
- `htop` `iotop`: visão clara de CPU/memória e **I/O** por processo.  
- `perf`: profiling de desempenho no kernel/userland.  
- `policycoreutils` `setools-console`: inspecionar/gerir **SELinux** por CLI.

---

## ⚙️ Compilação e toolchain (com explicações)

```bash
sudo dnf -y groupinstall "Development Tools"

sudo dnf -y install \
  gcc gcc-c++ make cmake ninja-build \
  pkgconf pkgconf-pkg-config \
  autoconf automake libtool \
  kernel-headers kernel-devel \
  elfutils-libelf-devel \
  openssl-devel zlib-devel \
  libffi-devel bison flex \
  python3 python3-pip
```

**Por que cada um?**
- **Group “Development Tools”**: metapacote que traz `gcc`, `make`, `gdb`, etc.  
- `gcc` `gcc-c++`: compiladores C/C++.  
- `make` `cmake` `ninja-build`: sistemas de build (clássico e modernos).  
- `pkgconf*`: resolve dependências de compilação (`*.pc`).  
- `autoconf` `automake` `libtool`: toolchain de projetos autoconf.  
- `kernel-headers` `kernel-devel`: headers e fontes mínimos para **módulos/DRM**.  
- `elfutils-libelf-devel`: manipulação ELF (drivers/módulos costumam exigir).  
- `openssl-devel` `zlib-devel` `libffi-devel`: libs comuns para compilar CLIs/daemons.  
- `bison` `flex`: geradores de parser/lexer.  
- `python3` `python3-pip`: automação, CLIs e utilitários escritos em Python.

> Dica: em produção, instale toolchain **só quando necessário** (reduz superfície de ataque).

---

## 🔒 SELinux: checar, alterar e observações

Checar status:
```bash
getenforce
```

Para **desativar permanentemente** (quando **explicitamente necessário**):
```bash
sudo vim /etc/selinux/config
```

Altere:
```bash
SELINUX=disabled
```

Reinicie:
```bash
sudo reboot
```

> ⚠️ **Recomendação**: prefira manter **Enforcing** e corrigir contextos/booleans. Desativar é último recurso, mas está aqui porque muitos ambientes legados exigem.

---

## 🔥 Firewalld: por serviço, porta e protocolo

Instalar/ativar:
```bash
sudo dnf -y install firewalld
sudo systemctl enable --now firewalld
```

**Por serviço** (mapeia múltiplas portas conhecidas):
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --reload
```

**Por porta** (ex.: app em 8080/TCP):
```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

**Por protocolo** (ex.: ICMP):
```bash
sudo firewall-cmd --permanent --add-protocol=icmp
sudo firewall-cmd --reload
```

> Dicas úteis:
> ```bash
> sudo firewall-cmd --get-services         # lista serviços conhecidos
> sudo firewall-cmd --list-all             # o que está liberado no runtime
> sudo firewall-cmd --runtime-to-permanent # salva o runtime no permanente
> ```

---

## ⏰ Timezone e NTP (chrony)

```bash
sudo dnf -y install chrony
sudo systemctl enable --now chronyd
sudo timedatectl set-timezone America/Sao_Paulo
sudo timedatectl set-ntp true
```

Ajustar servidores (opcional):
```bash
sudo vim /etc/chrony.conf
# adicione/ajuste servers pool.ntp.org ou sua NTP interna
```

Ver fontes:
```bash
chronyc sources -v
```

---

## 💻 Virtualização (KVM/libvirt)

```bash
sudo dnf -y install \
  qemu-kvm libvirt virt-install virt-viewer libvirt-client
sudo systemctl enable --now libvirtd
```

Adicionar seu usuário ao grupo (login novo para surtir efeito):
```bash
sudo usermod -aG libvirt $USER
```

Verificar:
```bash
virsh list --all
```

> Dica: para **bridge** e redes avançadas use `nmcli`/`virsh net-*` e valide MTU/VLANs conforme seu ambiente.

---

## 🐳 Containers (Podman/Buildah/Skopeo)

```bash
sudo dnf -y install podman buildah skopeo crun
```

Testes rápidos:
```bash
podman info
podman run --rm registry.access.redhat.com/ubi9/ubi echo OK
```

- **Podman**: runtime sem daemon (root/rootless).  
- **Buildah**: **build** de imagens sem Dockerfile ou com, focado em segurança.  
- **Skopeo**: **copiar/inspecionar** imagens entre registries (sem pull local).  
- **crun**: runtime leve/performático (cgroups v2).

---

## 🧰 Extras úteis do dia a dia

```bash
sudo dnf -y install \
  git jq yq \
  parted lvm2 smartmontools \
  atop ncdu
```

**Por que cada um?**
- `git`: versionar infra/playbooks e clonar tutoriais.  
- `jq` `yq`: manipular **JSON/YAML** (APIs, K8s, Ansible).  
- `parted` `lvm2`: discos/partições e **LVM** (expandir volumes sem downtime).  
- `smartmontools`: saúde do disco (SMART).  
- `atop`: visão de performance por subsistema (útil para RCA).  
- `ncdu`: “du” visual para achar consumo de disco.

---

## 🔧 Pós-reboot: tuned, fstrim e sysctl
**Perfis de desempenho** (servidor):
```bash
sudo dnf -y install tuned
sudo systemctl enable --now tuned
sudo tuned-adm profile throughput-performance
sudo tuned-adm active
```

**TRIM periódico** (SSD/NVMe):
```bash
systemctl status fstrim.timer || sudo systemctl enable --now fstrim.timer
```

**Sysctl básicos e conservadores**:
```bash
sudo vim /etc/sysctl.d/90-server.conf
```

Exemplo:
```bash
# Reduz swapping agressivo em servidor
vm.swappiness = 10

# Melhora filas de rede em cargas paralelas (ajuste conforme testes)
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 4096

# Evita que memória “suja” cresça demais antes de flush
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
```

Aplicar:
```bash
sudo sysctl --system
```

> ⚖️ Sempre **teste** em homologação antes de levar sysctl para produção.

---

## 🧪 Checklist final

```bash
# DNF/Repos
dnf check-update || true
dnf repolist

# Ferramentas essenciais
vim --version | head -n1
bash --version | head -n1
curl --version | head -n1
rsync --version | head -n1

# SELinux / Firewall
getenforce
sudo firewall-cmd --list-all

# Tempo
timedatectl
chronyc tracking

# Virtualização
groups $USER | tr ' ' '\n' | grep libvirt || echo "adicione $USER ao grupo libvirt"
virsh list --all

# Containers
podman info | grep -E 'ociVersion|cgroupVersion' -n
```

---

## 🩺 Troubleshooting rápido
- **Sem rede após instalação minimal**: verifique `nmcli connection show`, ative `NetworkManager` e configure DHCP/IP.  
- **SELinux bloqueando serviço**: cheque `sudo ausearch -m AVC -ts recent` e ajuste contexto/booleans; **desativar** só se necessário.  
- **Firewall liberado mas porta “fechada”**: confirme serviço **ouvindo** (`ss -lntup`) e **zona** aplicada à interface (`nmcli connection show`).  
- **DNS estranho**: valide `/etc/resolv.conf` (evite múltiplos DNS conflitantes).  
- **Hora fora do eixo**: `timedatectl`, `chronyc sources -v` e regras de firewall para NTP (UDP/123).

---

## ✅ Conclusão
Com este procedimento, seu **Rocky Linux 10 minimal** sai do zero para um estado **pronto para produção**: atualizado, seguro, instrumentado e com ferramentas sólidas para **containers**, **virtualização** e **middleware** — terreno fértil para **Portainer, Kubernetes, WebLogic, JBoss** e todo o **ecossistema Red Hat**.

---

Criado por **Jeferson Salles**  
🔗 LinkedIn: https://www.linkedin.com/in/jmsalles/  
📧 E-mail: jefersonmattossalles@gmail.com
````
