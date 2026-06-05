---
tags:
  - docker
  - docker/runtime
---

# Runtime e Engine

Esta é a parte que fecha o quebra-cabeça da arquitetura. Imagine isso como uma **hierarquia de responsabilidades**, onde cada camada adiciona uma facilidade para o desenvolvedor.

A confusão existe porque, no início, o Docker era **tudo ao mesmo tempo**. Com a padronização da OCI, o Docker foi "fatiado" em partes menores com responsabilidades bem definidas.

## 1. Container Engine (O Cérebro)

A **Engine** (ex: Docker Engine, Podman) é o software de **mais alto nível**. Ela é feita para seres humanos e ferramentas de automação.

**O que ela faz (que o Runtime não faz):**

- **Interface de Usuário:** Provê o CLI (`docker ps`, `docker build`, `docker run`).
- **API REST:** Permite que outras ferramentas (VS Code, Portainer) controlem containers via rede.
- **Gerenciamento de Redes:** Cria redes virtuais (Bridges, Overlays) e gerencia o IP de cada container.
- **Build de Imagens:** Lê o `Dockerfile` e transforma em camadas OCI.
- **Orquestração Básica:** Gerencia volumes, secrets e configurações.
- **Autenticação:** Lida com login no Docker Hub ou registros privados.

**Exemplos:** Docker Engine, Podman, Mirantis Container Runtime.

## 2. Container Runtime (O Braço Direito)

O **Runtime** é o software focado puramente na **execução**. Ele não se importa com "quem é o usuário" ou "como a imagem foi construída". Ele só quer saber de rodar o processo.

O Runtime se divide em dois níveis:

### High-Level Runtime (Gerente de Logística)

- Recebe ordens da Engine ou do Kubernetes.
- Baixa a imagem, extrai no disco (OverlayFS).
- Prepara o ambiente para o container.
- Gerencia o ciclo de vida completo (create, start, stop, delete).
- **Exemplos:** `containerd`, `CRI-O`.

### Low-Level Runtime (O Executor)

- O único que fala diretamente com o Kernel.
- Cria os Namespaces e Cgroups via syscalls.
- Executa `pivot_root()`, aplica seccomp e capabilities.
- Inicia o processo e "sai de cena" (processo efêmero).
- **Exemplos:** `runc`, `gVisor`, `Kata Containers`.

## 3. A Pilha Completa de Execução

```
Usuário / kubelet / CI-CD
         │
    Docker Engine / Podman         ← Container Engine
         │  (API REST / gRPC)
      containerd / CRI-O           ← High-Level Runtime
         │  (OCI Runtime Spec)
      runc / gVisor / kata         ← Low-Level Runtime
         │  (syscalls)
     Linux Kernel
         │
      Namespaces + Cgroups         ← Isolamento real
```

## 4. Runtimes Alternativos de Baixo Nível

Além do `runc`, existem runtimes que oferecem diferentes trade-offs de segurança e performance:

**gVisor (Google):**
- Intercepta syscalls do container em user space, antes de chegarem ao kernel real.
- Implementa um "kernel em Go" que executa as syscalls num ambiente sandboxed.
- Segurança extra: mesmo que o código do container seja malicioso, ele ataca o gVisor, não o kernel real.
- Trade-off: overhead de performance (cada syscall passa por uma camada extra).

**Kata Containers:**
- Executa cada container dentro de uma micro-VM (QEMU/NEMU com KVM).
- Isolamento de nível VM com velocidade próxima de container.
- Cada container tem seu próprio kernel Linux mínimo.
- Trade-off: startup mais lento, mais memória, mas isolamento muito mais forte.

**runc (padrão):**
- Usa namespaces e cgroups diretamente — compartilha o kernel do host.
- Performance máxima, isolamento suficiente para a maioria dos casos.
- Trade-off: um exploit no kernel afeta todos os containers no host.

## 5. Analogia: O Restaurante

Para tornar a hierarquia visual:

1. **A Engine (Docker) é o Restaurante Inteiro:** Tem o garçom (CLI), o cardápio (API), o sistema de reservas (Network/Volumes) e o marketing (Docker Hub).
2. **O High-Level Runtime (containerd) é a Cozinha:** Recebe o pedido, pega os ingredientes na despensa (Images) e prepara a bancada (Filesystem).
3. **O Low-Level Runtime (runc) é o Fogão/Fogo:** Aplica o calor (Kernel) para transformar o ingrediente bruto em comida pronta (Processo Isolado).

## 6. Por que o Kubernetes "abandonou" a Docker Engine?

O Kubernetes não precisava de todas as funcionalidades da **Engine** (ele já tem seu próprio sistema de rede e build). Ele precisava apenas de alguém que rodasse o container de forma estável.

Por isso, o Kubernetes passou a falar direto com o **Runtime (containerd)** através do protocolo **CRI (Container Runtime Interface)** — interface gRPC definida pelo próprio Kubernetes para padronizar a comunicação com qualquer runtime compatível.

Isso criou um ecosistema onde **qualquer runtime que implemente o CRI** pode ser usado com Kubernetes: containerd, CRI-O, e até gVisor e Kata via adaptadores.

## 7. Resumo Comparativo

| Recurso | Container Engine (Docker) | High-Level Runtime (containerd) | Low-Level Runtime (runc) |
| --- | --- | --- | --- |
| **Foco** | Experiência do Desenvolvedor | Ciclo de vida + imagens | Execução kernel-level |
| **Público** | Humanos e scripts | Engines e Kubernetes | Runtimes de alto nível |
| **Build de Imagem** | Sim (Docker Build) | Não | Não |
| **Gestão de Rede** | Sim (Bridge, Host, Overlay) | Mínimo (via plugins) | Nenhum |
| **Fala com Kernel** | Não (delega) | Não (delega) | Sim (syscalls diretas) |
| **Padrão** | OCI + extras proprietários | OCI + CRI | OCI Runtime Spec |
| **Processo de vida** | Daemon permanente | Daemon permanente | Efêmero (morre após exec) |

> "A Engine é o ecossistema completo que facilita a vida de quem desenvolve. O Runtime é o motor especializado que garante que o container rode conforme as especificações técnicas. Você usa a Engine para trabalhar, mas o seu servidor usa o Runtime para produzir."

## Conexão com Sistemas Operacionais

- A Engine (Docker) é uma abstração de alto nível sobre primitivas do kernel — o mesmo modelo de camadas do SO: programas userspace → chamadas de SO → hardware → [[Processos]], [[System Calls]]
- O CRI (Container Runtime Interface) é uma camada de abstração análoga ao VFS (Virtual Filesystem Switch) do kernel: assim como o VFS define uma interface que qualquer filesystem deve implementar, o CRI define uma interface que qualquer runtime deve implementar → [[Arquivos]]
- gVisor intercepta syscalls em userspace antes que cheguem ao kernel real — mesma técnica usada por emuladores e VMs para interceptar instruções privilegiadas → [[System Calls]], [[Hardware de Proteção]]
- Kata Containers executa cada container dentro de uma micro-VM com KVM — combina isolamento de VM (cada container tem seu próprio kernel) com velocidade de container → [[Máquinas Virtuais Redescobertas]]
- A relação Engine/containerd/runc = camadas de abstração onde cada nível expõe uma interface mais simples para cima e usa uma interface mais complexa para baixo — igual ao modelo de SO: aplicação → syscalls → driver → hardware → [[Processos]]
- A API REST do Docker Engine é um servidor HTTP rodando dentro do `dockerd` que converte chamadas HTTP em operações sobre containers → [[HTTP (net-http)]]

## Conexão com Go

- O Docker Engine (`dockerd`), o containerd e o runc são todos escritos em Go — a linguagem se tornou o padrão da indústria para ferramentas de infraestrutura de containers
- A API REST do Docker Engine é implementada com o pacote `net/http` do Go — servidor HTTP padrão servindo endpoints JSON → [[HTTP (net-http)]]
- O cliente `docker` (CLI) usa `net/http` para se comunicar com o daemon via socket Unix (`/var/run/docker.sock`) ou TCP
- O containerd expõe sua API via gRPC — em Go, o gRPC usa `net/http` internamente e goroutines para multiplexar requisições → [[Goroutines]], [[HTTP (net-http)]]
