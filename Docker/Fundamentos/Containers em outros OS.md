---
tags:
  - docker
  - docker/fundamentos
---

# Containers em outros OS

Essa é a "pergunta de um milhão de dólares". Como o **Docker** (que depende de recursos exclusivos do Kernel Linux, como Namespaces e Cgroups) roda tão bem no **Windows** e no **macOS**?

A resposta curta é: **Ele não roda nativamente.** Sempre que você usa Docker no Windows ou Mac, existe uma **máquina virtual Linux minúscula** escondida no meio do caminho.

---

## 1. Docker no Windows (O Salto do WSL2)

Antigamente, o Docker no Windows era lento e problemático porque usava uma VM pesada (Hyper-V). Tudo mudou com o **WSL2 (Windows Subsystem for Linux)**.

- **O que é o WSL2:** O Windows agora vem com um **Kernel Linux real** compilado pela Microsoft que roda em cima de um micro-hypervisor.
- **Como o Docker usa isso:** O Docker Desktop não precisa mais criar uma VM lenta. Ele se "pendura" no Kernel do WSL2.
- **Vantagem:** Performance quase nativa, acesso direto ao sistema de arquivos e suporte total a Namespaces e Cgroups, pois o Kernel Linux está ali, "vivo" dentro do Windows.

---

## 2. Docker no macOS (O HyperKit / Virtualization.framework)

O macOS é baseado em Unix (BSD), mas o Kernel dele (Darwin) **não tem** Namespaces ou Cgroups do Linux. Eles são arquiteturas totalmente diferentes.

- **A Solução:** O Docker Desktop para Mac instala uma máquina virtual Linux extremamente leve (baseada em um Linux customizado chamado *LinuxKit*).
- **Hypervisor:** Ele usa o framework nativo da Apple (`Virtualization.framework` ou antigamente o `HyperKit`) para subir essa VM Linux em milissegundos.
- **O Truque da CLI:** Quando você digita `docker run` no terminal do Mac, o comando atravessa uma "ponte" (socket Unix) e vai ser executado dentro dessa VM Linux invisível.

---

## 3. A Arquitetura Cliente-Servidor

O Docker foi desenhado com uma arquitetura **Client-Server**. Isso é o que permite essa mágica:

1. **Docker Client (CLI):** É o binário que você baixou no Mac ou Windows. Ele é apenas um "controle remoto".
2. **Docker Daemon (Server):** É o motor que realmente gerencia os containers. **Este motor sempre roda em Linux.**

No Windows/Mac, o seu terminal (Client) fala com o Daemon que está morando dentro da VM Linux/WSL2. Para você, desenvolvedor, parece que o container está rodando no seu Mac, mas ele está tecnicamente rodando na VM Linux.

```
macOS / Windows
┌────────────────────────────────┐
│  Terminal do usuário           │
│  docker run nginx              │
│         │                      │
│         │ Unix socket / TCP    │
│         ▼                      │
│  [VM Linux (LinuxKit / WSL2)]  │
│  ┌──────────────────────────┐  │
│  │ dockerd (Docker Daemon)  │  │
│  │  - cria namespaces       │  │
│  │  - configura cgroups     │  │
│  │  - monta OverlayFS       │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

---

## 4. O Impacto na Performance (O Gargalo do Disco)

Este ponto causa muita dor de cabeça em desenvolvedores: **I/O de Arquivos (Volume Mounting)**.

- Quando você faz um volume (`-v`) no Linux, é um link direto no HD. Rápido.
- No Windows/Mac, o Docker precisa "traduzir" o sistema de arquivos do Host (NTFS/APFS) para o sistema de arquivos da VM Linux (ext4).
- **Consequência:** Projetos com milhares de arquivos pequenos (ex: `node_modules`, `vendor` do PHP) costumam ser **muito mais lentos** no Windows/Mac do que no Linux nativo devido a essa camada de tradução (gRPC FUSE ou VirtioFS).

### VirtioFS vs gRPC-FUSE

- **gRPC-FUSE (legado):** O filesystem do host é exposto via um servidor FUSE (Filesystem in Userspace) dentro da VM. Toda operação de I/O cruzava a fronteira via gRPC — lento para acesso aleatório com muitos arquivos pequenos.
- **VirtioFS (moderno):** Usa o protocolo virtio para compartilhamento de filesystem entre VM e host com latência muito menor. Ainda assim, há overhead de cruzar a fronteira da VM.

---

## 5. Tabela Comparativa por Sistema Operacional

| Sistema Operacional | Como roda o Docker? | Tecnologia Base | Performance |
| --- | --- | --- | --- |
| **Linux** | Nativo | Kernel do próprio SO | Máxima |
| **Windows** | "Semi-nativo" | WSL2 (Kernel Linux no Windows) | Alta |
| **macOS** | Virtualizado | Micro-VM Linux (LinuxKit) | Média/Alta |

---

## 6. Por que o Linux para Containers? (Under the Hood)

Containers Linux são fundamentalmente recursos do Kernel Linux específicos. Não há como "emular" Namespaces ou Cgroups sem um Kernel Linux real:

- **Namespaces** são estruturas de dados dentro do `task_struct` do Kernel Linux
- **Cgroups** são um virtual filesystem (`cgroupfs`) implementado no Kernel Linux
- **OverlayFS** é um driver de filesystem do Kernel Linux
- **seccomp, capabilities, AppArmor** são mecanismos de segurança do Kernel Linux

Nem o Darwin (macOS) nem o NT Kernel (Windows) implementam esses recursos. Por isso, a única solução é um Kernel Linux real — seja via VM completa, seja via WSL2.

---

## 7. Resumo

> "Containers Linux precisam de um Kernel Linux. Para rodar em Windows ou Mac, o Docker cria uma camada de abstração usando virtualização leve (WSL2 ou Micro-VMs). O comando que você digita no seu terminal viaja até uma pequena ilha Linux escondida no seu sistema, onde os Namespaces e Cgroups realmente residem."

---

## Conexão com Sistemas Operacionais

- **Docker no macOS/Windows sempre roda dentro de uma VM Linux → [[Máquinas Virtuais Redescobertas]], [[A Origem das Máquinas Virtuais]]**
  - A necessidade de uma VM Linux para rodar containers no macOS e Windows é uma demonstração prática de que virtualização e containers não são substitutos completos: containers dependem de recursos de kernel específicos, e sem um kernel Linux real, a virtualização (VM) é o único caminho.

- **WSL2 = Kernel Linux real compilado pela Microsoft rodando em um hypervisor Hyper-V → [[Máquinas Virtuais Redescobertas]]**
  - O WSL2 é uma micro-VM gerenciada pelo Hyper-V da Microsoft. Diferente do WSL1 (que traduzia syscalls Linux para chamadas Windows), o WSL2 executa um Kernel Linux genuíno em modo guest. Isso significa que Namespaces, Cgroups e todas as primitivas de containers funcionam exatamente como em um Linux nativo.

- **LinuxKit = micro-VM Linux mínima (apenas o necessário para containers) → reduzindo o SO ao mínimo → [[Processos]] (serviços essenciais do kernel)**
  - O LinuxKit é construído com a filosofia de incluir apenas os componentes necessários para suportar containers: kernel Linux + containerd + nada mais. É o oposto de uma distribuição Linux completa. Isso demonstra que um SO pode ser reduzido às suas funções essenciais de gerenciamento de processos, memória e I/O.

- **Performance de I/O: mounts de volumes requerem tradução de filesystem (APFS/NTFS → ext4) via VirtioFS/gRPC-FUSE → adiciona round-trip até a VM → [[Dispositivos de IO]], [[Armazenamento não Volátil]]**
  - Cada operação de leitura ou escrita em um volume montado no macOS/Windows atravessa a fronteira da VM. O VirtioFS minimiza isso usando memória compartilhada e o protocolo virtio (projetado para I/O de alta performance em VMs), mas ainda há overhead de contexto de VM. Em Linux nativo, o Docker usa bind mounts diretos sem nenhuma camada de tradução.

- **A comunicação TCP socket entre Docker CLI e Docker Daemon cruza a fronteira da VM → [[System Calls]] (socket, Unix socket)**
  - O Docker CLI se comunica com o dockerd via um socket Unix (`/var/run/docker.sock` dentro da VM) ou TCP. No macOS/Windows, essa comunicação cruza a fronteira do hypervisor. O Docker Desktop cria um túnel que expõe o socket da VM para o host, permitindo que a CLI nativa do macOS/Windows converse com o daemon Linux dentro da VM.
