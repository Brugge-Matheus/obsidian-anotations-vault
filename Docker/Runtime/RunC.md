---
tags:
  - docker
  - docker/runtime
---

# RunC

Se os **Namespaces** e **Cgroups** são as peças do LEGO e a **OCI** é o manual de instruções, o **runc** é a mão que monta tudo.

O **runc** é o componente mais "raiz" que existe antes de chegarmos ao Kernel. Ele é o **Runtime de Baixo Nível (Low-level Runtime)**.

## 1. O que é o runc?

O **runc** é uma ferramenta de linha de comando (CLI) escrita em Go para gerar e rodar containers no Linux de acordo com a especificação da OCI.

O fato mais importante: o Docker não "roda" o container diretamente. O Docker pede para o **containerd** pedir para o **runc** rodar o container. Uma vez que o container está de pé, o `runc` pode até sair de cena — ele morre e deixa o processo rodando.

## 2. O que o runc faz na prática?

Quando o `runc` recebe uma ordem, ele segue estes passos exatos no sistema:

1. **Lê o `config.json`:** Recebe o arquivo gerado pela Engine com as regras: namespaces, limites de RAM, caminho do rootfs.
2. **Cria os Namespaces:** Faz as chamadas de sistema (`clone()` com flags como `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC`) para criar as "bolhas" de isolamento.
3. **Configura os Cgroups:** Cria as pastas em `/sys/fs/cgroup/` e escreve os limites de hardware (CPU, memória, I/O).
4. **Faz o Pivot Root:** Chama `pivot_root()` para trocar o diretório raiz do processo para o `rootfs` da imagem, impedindo que o container enxergue o filesystem do host.
5. **Aplica Segurança:** Configura o **Seccomp** (filtro de syscalls permitidas) e as **Capabilities** do kernel (privilégios granulares como `CAP_NET_ADMIN`).
6. **Executa o processo:** Finalmente, chama `exec()` no binário da aplicação — esse processo se torna o **PID 1** do container.

## 3. O runc por dentro: syscalls que ele invoca

Entender o runc é entender o que ele faz no kernel. A sequência de syscalls real é aproximadamente esta:

```
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC)
  → cria processo filho já isolado nos namespaces especificados

mount("overlay", "/var/lib/containerd/.../rootfs", "overlay", ...)
  → monta o OverlayFS com as layers da imagem

pivot_root("/novo-rootfs", "/antigo-root")
  → troca a raiz do processo para o rootfs do container

umount2("/antigo-root", MNT_DETACH)
  → desmonta o caminho para o filesystem real do host

prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &filter)
  → instala o filtro de syscalls (perfil seccomp)

capset(...)
  → define as capabilities do processo (ex: remove CAP_SYS_ADMIN)

execve("/bin/sh", [...], [...])
  → substitui o processo runc pelo processo da aplicação (PID 1)
```

## 4. Onde ele está no sistema?

Se você rodar um container e procurar:

```bash
ps aux | grep runc
```

Dificilmente verá o `runc`. Por que? Porque ele é um **processo de curta duração (transient)**. Ele inicia o container, entrega o processo filho para o kernel cuidar e encerra.

Quem fica monitorando o container depois é o `containerd-shim`.

## 5. Seccomp e Capabilities: segurança de baixo nível

**Seccomp (Secure Computing Mode):** Filtro aplicado em nível de kernel que define quais syscalls o processo pode chamar. O perfil padrão do Docker bloqueia ~300 das ~400 syscalls disponíveis. Qualquer tentativa de fazer uma syscall bloqueada resulta em `EPERM` ou `SIGKILL`.

**Linux Capabilities:** O modelo tradicional Unix é binário — root faz tudo, não-root não faz nada crítico. O modelo de capabilities divide os privilégios de root em ~40 capacidades independentes:

| Capability | O que permite |
| --- | --- |
| `CAP_NET_ADMIN` | Configurar interfaces de rede, rotas, firewall |
| `CAP_SYS_PTRACE` | Fazer `ptrace()` em outros processos |
| `CAP_SYS_ADMIN` | Operações administrativas gerais (muito ampla) |
| `CAP_CHOWN` | Mudar dono de arquivos |
| `CAP_KILL` | Enviar sinais para processos de outros usuários |

O runc configura exatamente quais capabilities o container tem com base no `config.json`.

## 6. O ciclo de vida do runc e o containerd-shim

```
containerd
    │
    ├─ fork → containerd-shim (fica vivo para sempre)
                  │
                  └─ fork → runc (processo temporário)
                                │
                                └─ exec → PID 1 do container (fica vivo)
                                          (runc morre aqui)
```

O `containerd-shim` sobrevive ao `runc` e se torna o **pai adotivo** do PID 1 do container. Ele:
- Mantém os file descriptors de stdin/stdout/stderr abertos
- Chama `waitpid()` quando o container termina para evitar processo zumbi
- Reporta o exit code de volta ao containerd

## 7. Experimento: rodar um container sem Docker

Você pode provar que o Docker é apenas uma interface bonita rodando um container com apenas o `runc`:

```bash
# 1. Criar estrutura de diretórios
mkdir -p mycontainer/rootfs

# 2. Extrair um rootfs mínimo (Alpine Linux)
tar -xzf alpine-minirootfs-3.18.2-x86_64.tar.gz -C mycontainer/rootfs

# 3. Gerar o config.json padrão da OCI
cd mycontainer
runc spec --rootless

# 4. Rodar o container
runc run meu-container
```

Conclusão: o `runc` não sabe o que é "Docker Image", "Registry" ou "Rede". Ele só sabe: "Pegue este diretório e transforme num processo isolado".

> "runc é o executor de baixo nível. Ele é o tradutor que lê as leis da OCI (config.json) e executa as ordens no Kernel Linux. Ele é o responsável por 'puxar o gatilho' dos Namespaces e Cgroups. Se o Kernel é o motor, o runc é a chave de ignição."

Para uso prático passo a passo: [[Como usar o runc]]

## Conexão com Sistemas Operacionais

- `runc` chama `clone()` com flags de namespace (`CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC`) para criar o processo filho já isolado → [[Syscalls para Gerenciamento de Processos]], [[Criação de Processos]]
- `pivot_root()` é uma syscall que troca a raiz do sistema de arquivos do processo — equivale a "prender" o container dentro do seu próprio rootfs, sem acesso ao `/` real do host → [[Arquivos]], [[System Calls]]
- Seccomp (Secure Computing): filtro de syscalls instalado pelo kernel via `prctl()` — o processo tenta uma syscall bloqueada e o kernel mata ele ou retorna EPERM → [[System Calls]], [[Hardware de Proteção]]
- Linux capabilities: modelo granular de privilégios substituindo o binário root/não-root → o runc remove capabilities desnecessárias antes de executar o processo do container → [[Proteção]], [[Hardware de Proteção]]
- O `runc` chama `execve()` para substituir a si mesmo pelo processo da aplicação — o PID 1 do container "herda" o ambiente já configurado (namespaces, cgroups, capabilities) → [[Criação de Processos]]
- O `containerd-shim` sobrevive ao `runc` e adota o processo do container — esse padrão de repaternidade de processos é gerenciado pelo kernel → [[Hierarquia de Processos]], [[Implementação de Processos]]

## Conexão com Go

- O `runc` é escrito inteiramente em Go — usa o pacote `syscall` (e `golang.org/x/sys/unix`) para invocar `clone()`, `pivot_root()`, `mount()` e `prctl()` diretamente → [[System Calls]]
- A inicialização do container usa goroutines para setup concorrente (configurar cgroups, montar filesystems, aplicar seccomp em paralelo) → [[Goroutines]]
- O código-fonte do runc é referência para entender como Go interage com primitivas de baixo nível do kernel Linux
