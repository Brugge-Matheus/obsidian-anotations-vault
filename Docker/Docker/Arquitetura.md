---
tags:
  - docker
  - docker/docker
---

# Arquitetura do Docker

Agora que entendemos que o **Docker** é uma ferramenta de alto nível (uma Engine), vamos detalhar como as peças se encaixam. O Docker funciona em um modelo **Cliente-Servidor**.

Aqui está a anatomia do Docker "por baixo do capô":

---

### A Arquitetura do Docker (Client-Server)

O Docker não é um bloco único de código. Ele é dividido em componentes que se comunicam através de uma **API REST**. Isso permite, por exemplo, que você use o Docker CLI no seu Windows para controlar um Docker Daemon que está rodando em um servidor na nuvem.

### 1. Docker CLI (O Cliente)

É o binário `docker` que você digita no terminal.

- **Função:** Ele é apenas uma interface ("controle remoto"). Ele não processa nada, não cria containers e não baixa imagens.
- **Como ele trabalha:** Quando você digita `docker run`, o CLI transforma esse comando em uma requisição HTTP (API REST) e envia para o **Docker Daemon**.
- **Onde ele vive:** Pode estar no seu Mac, Windows ou Linux.

### 2. Docker Daemon (`dockerd`) (O Cérebro / Servidor)

É o serviço que roda em background no sistema operacional (geralmente como um `systemd` unit no Linux).

- **Função:** Ele é o "porteiro" e coordenador. Ele recebe as ordens do CLI via Unix Socket ou rede.
- **Responsabilidades:**
    - Autenticação em Registries (Docker Hub).
    - Gerenciamento de Redes (criação de Bridges e IPs).
    - Gerenciamento de Volumes (persistência de dados).
    - Gestão de Imagens (build e cache).
- **Nota Importante:** O Daemon não "roda" o processo do container diretamente. Ele delega essa tarefa para o **containerd**.

### 3. containerd (O Gerente de Execução)

O `containerd` foi extraído do Docker para ser um componente independente.

- **Função:** Gerenciar o ciclo de vida completo do container.
- **O que ele faz:** Ele recebe a ordem de "rodar" do Daemon, prepara a pasta da imagem (Snapshot) e chama os executores de baixo nível (como o `runc`).
- **Por que separar?** Isso permite que o Docker Daemon seja atualizado ou reiniciado sem que os containers que estão rodando precisem parar (graças ao `containerd-shim`).

---

### O Fluxo de um Comando (A "Viagem" de um `docker run`)

1. **Usuário:** Digita `docker run nginx`.
2. **Docker CLI:** Faz um POST HTTP para o Docker Daemon: *"Ei, roda o nginx aí!"*.
3. **Docker Daemon:**
    - Procura a imagem localmente.
    - Se não tiver, baixa do Docker Hub.
    - Analisa as redes e volumes.
    - Envia a ordem para o **containerd**.
4. **containerd:**
    - Cria o "Snapshot" (monta as camadas da imagem com OverlayFS).
    - Cria um **shim** (o monitor do processo).
    - Chama o **runc**.
5. **runc:**
    - Fala com o Kernel Linux.
    - Cria os Namespaces e Cgroups.
    - Inicia o processo `nginx`.
    - Morre (sai de cena).
6. **Kernel:** Mantém o processo isolado e rodando.

---

### Diagrama Conceitual

```
[ Docker CLI ]  <-- (REST API) -->  [ Docker Daemon ]
                                          |
                                    [ containerd ]
                                          |
                                    [  runc/shim  ]
                                          |
                                    [ Kernel Linux ]
```

---

### Resumo

> "A arquitetura do Docker é modular e distribuída. O **CLI** é a voz (comando), o **Daemon** é o cérebro (coordenação de recursos) e o **containerd** é o músculo (gestão de execução). Essa separação garante que o Docker seja portável, estável e capaz de gerenciar redes e volumes complexos enquanto mantém os processos rodando de forma eficiente."

---

## Conexões com Sistemas Operacionais

**Docker daemon como processo privilegiado** — O `dockerd` roda com privilégios elevados (exige root ou membro do grupo `docker`) porque precisa criar namespaces, montar filesystems e configurar regras de iptables — operações que exigem modo kernel. Ver [[Proteção]], [[Hardware de Proteção]].

**CLI ↔ Daemon via Unix Socket** — A comunicação `docker CLI → dockerd` é feita sobre `/var/run/docker.sock`, um socket de domínio Unix. Isso é um mecanismo de IPC (Inter-Process Communication) clássico do SO: o CLI chama `connect()` no socket, escreve com `sendmsg()` e lê com `recvmsg()`. Ver [[System Calls]].

**REST API sobre socket Unix** — As requisições HTTP do CLI trafegam sobre o file descriptor do socket local (sem TCP/IP de rede). O kernel entrega os bytes diretamente entre processos na mesma máquina. Ver [[Dispositivos de IO]], [[HTTP (net-http)]].

**`dockerd` como serviço systemd** — No Linux, o Docker daemon é uma unidade systemd (`dockerd.service`): inicia na boot, reinicia em falhas, tem controle de cgroup próprio. Isso é a implementação moderna de "processos daemon". Ver [[Processos]].

**A cadeia completa de chamadas de sistema** — Quando você executa `docker run`:
1. CLI → HTTP POST `/containers/create` via socket
2. `dockerd` delega ao `containerd`
3. `containerd` chama `runc`
4. `runc` invoca `clone()` com flags de namespace (`CLONE_NEWNS`, `CLONE_NEWPID`, `CLONE_NEWNET`, etc.)
5. Kernel cria o processo isolado

Ver [[Syscalls para Gerenciamento de Processos]].

**Conexão com Go** — O Docker daemon (`dockerd`) e o `containerd` são escritos em Go. O daemon gerencia centenas de containers concorrentemente usando goroutines — cada container tem sua goroutine de supervisão. A API REST do daemon é um servidor HTTP em Go. Ver [[Goroutines]] (gerenciamento concorrente de containers), [[HTTP (net-http)]] (servidor REST API).
