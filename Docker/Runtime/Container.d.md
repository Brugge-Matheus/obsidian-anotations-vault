---
tags:
  - docker
  - docker/runtime
---

# Container.d

Se o **runc** é o executor solitário que só entende de "rodar um processo", o **containerd** é o **gerente de operações**.

O **containerd** é um **Runtime de Alto Nível (High-level Runtime)**. Ele não "toca" o Kernel diretamente para criar o container (ele pede para o `runc` fazer isso), mas cuida de todo o resto do ciclo de vida que o `runc` ignora.

## 1. O que o containerd faz (Suas 5 Responsabilidades)

Pense no **containerd** como o orquestrador local de um único host:

1. **Gestão de Imagens:** Sabe como falar com o Docker Hub (ou qualquer Registry), baixar as camadas e armazená-las no disco.
2. **Snapshot Management:** Gerencia o **OverlayFS**. Pega as camadas da imagem e as "monta" em uma pasta pronta para o container usar.
3. **Execution Lifecycle:** Decide quando um container deve iniciar, pausar ou morrer. Chama o `runc` para a execução real.
4. **Metadados:** Guarda o estado de todos os containers rodando na máquina (IDs, imagens em uso, estado atual).
5. **Eventos:** Avisa o Docker (ou o Kubernetes) quando um container morre ou dá erro.

## 2. A Arquitetura do Shim (O Segredo da Estabilidade)

Aqui está o detalhe técnico central do containerd: o **containerd-shim**.

Quando o `containerd` inicia um container, ele não fica "segurando" o processo. Ele cria um processo intermediário minúsculo chamado **shim** para cada container.

```
dockerd / kubelet
      │  (gRPC)
   containerd
      │
      ├─ fork → containerd-shim-runc-v2   ← fica vivo para sempre
                      │
                      ├─ exec → runc (temporário)
                      │            │
                      │            └─ exec → PID 1 do container
                      │                      (runc morre aqui)
                      │
                      ├─ mantém stdin/stdout/stderr abertos
                      ├─ chama waitpid() quando PID 1 termina
                      └─ reporta exit code ao containerd
```

**Por que o Shim existe?**

- **Sobrevivência:** Se o `containerd` ou o `Docker` travar ou precisar ser reiniciado para um update, o **container continua rodando**. O `shim` fica ali cuidando do processo filho. Quando o `containerd` volta, ele se reconecta ao `shim`.
- **Terminal e Logs:** O `shim` mantém abertos os arquivos de log e o terminal (stdin/stdout/stderr) do container, mesmo que o gerente (containerd) suma por um tempo.
- **Coleta de zumbis:** O `shim` chama `waitpid()` quando o PID 1 do container termina, evitando que o processo filho fique como zumbi no sistema.

## 3. Snapshot Management e OverlayFS

O containerd gerencia o ciclo de vida das layers da imagem através do seu subsistema de **snapshots**. O fluxo é:

1. Containerd baixa cada layer da imagem como um arquivo compactado.
2. Descompacta e armazena cada layer como um snapshot imutável (read-only).
3. Antes de iniciar um container, cria um snapshot **read-write** no topo das layers.
4. Chama `mount(2)` com tipo `overlay` para montar o OverlayFS combinando todos os snapshots.
5. Passa o ponto de montagem resultante ao `runc` como o `rootfs` do container.

Quando o container morre, o snapshot read-write é descartado (a não ser que seja commitado como nova layer).

## 4. Onde o containerd vive?

Hoje em dia, o `containerd` é um projeto independente gerenciado pela **CNCF** (Cloud Native Computing Foundation).

- No **Docker**: O Docker usa o `containerd` por baixo dos panos.
- No **Kubernetes**: O Kubernetes parou de usar o Docker e passou a falar diretamente com o `containerd` via a interface **CRI (Container Runtime Interface)**.

Ferramentas úteis para interagir diretamente com o containerd:

```bash
# Listar imagens gerenciadas pelo containerd (independente do Docker)
sudo ctr images list

# Listar containers
sudo ctr containers list

# Ferramenta mais amigável (Docker-like para containerd)
nerdctl ps
nerdctl images
```

## 5. CRI (Container Runtime Interface) e Kubernetes

O CRI é uma interface gRPC que o Kubernetes usa para se comunicar com runtimes de alto nível. Ela define duas interfaces principais:

- **RuntimeService:** Gerencia pods e containers (CreateContainer, StartContainer, StopContainer, RemoveContainer)
- **ImageService:** Gerencia imagens (PullImage, RemoveImage, ListImages)

Isso significa que o Kubernetes nunca fala diretamente com o `runc` — ele fala com o `containerd` via CRI, e o `containerd` delega para o `runc` via OCI Runtime Spec.

```
kubelet
  │  (CRI / gRPC)
containerd
  │  (OCI Runtime Spec)
runc
  │  (syscalls)
Linux Kernel
```

> "containerd é o gerente de logística. Ele cuida do inventário (Imagens), da preparação do terreno (Filesystem/OverlayFS) e da supervisão (Lifecycle). Ele é o elo entre ferramentas de alto nível (Docker/Kubernetes) e os executores de baixo nível (runc). Se o runc é o operário na obra, o containerd é o mestre de obras que garante que todos os materiais estejam lá e que o cronograma seja seguido."

## Conexão com Sistemas Operacionais

- O `containerd-shim` é um processo intermediário entre o containerd e o container — forma uma árvore pai-filho onde o shim é pai do PID 1 do container → [[Hierarquia de Processos]], [[Implementação de Processos]]
- O shim mantém os file descriptors de stdin/stdout/stderr abertos para o container — esses FDs sobrevivem mesmo se o containerd (avô do processo) for reiniciado, porque o relacionamento que importa é shim→container → [[Arquivos]]
- O shim chama `waitpid()` para coletar o exit status do PID 1 do container quando ele termina — prevenção de processos zumbi, igual ao que o init (PID 1 do host) faz para processos órfãos → [[Término de Processos]]
- O containerd usa gRPC para comunicação com o Docker daemon e com o kubelet — gRPC em Go é construído sobre HTTP/2 → [[HTTP (net-http)]]
- O gerenciamento de snapshots (OverlayFS) usa a syscall `mount(2)` para combinar as layers read-only com a layer read-write do container → [[Arquivos]], [[System Calls]]
- A relação containerd/runc é análoga ao modelo M:N de threads: containerd é o scheduler de alto nível que gerencia o ciclo de vida, runc é o executor de baixo nível que interage com o kernel → [[Implementando Threads em User Space]]

## Conexão com Go

- O containerd é escrito inteiramente em Go — o projeto é referência de como construir daemons de sistema em Go → [[Goroutines]]
- Usa goroutines para gerenciar múltiplos containers concorrentemente — cada container tem seu próprio goroutine de lifecycle management
- A comunicação gRPC entre containerd e clientes usa goroutines para servir múltiplas requisições em paralelo → [[HTTP (net-http)]]
- O subsistema de snapshots usa `os.MkdirAll`, `syscall.Mount` e interfaces Go para abstrair diferentes backends (OverlayFS, btrfs, zfs)
