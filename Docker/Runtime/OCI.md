---
tags:
  - docker
  - docker/runtime
---

# OCI

## 1. O que é a OCI (Open Container Initiative)?

Antigamente, o Docker fazia tudo: baixava a imagem, gerenciava a rede e rodava o container. O problema é que isso criava um "monopólio técnico". Se alguém quisesse criar outra ferramenta de container, teria que fazer engenharia reversa no Docker.

Em 2015, a Docker (a empresa) e outras gigantes (Google, Red Hat, Microsoft) criaram a **OCI** sob a tutela da Linux Foundation. O objetivo foi criar **padrões abertos** para que qualquer ferramenta pudesse ler e rodar containers de qualquer outra.

Existem **2 padrões principais** (especificações):

## 2. Image Spec (Especificação de Imagem)

Define como uma imagem de container deve ser empacotada.

- O que é o manifesto (o "RG" da imagem)?
- Como as camadas (layers) são compactadas (geralmente `.tar.gz`)?
- Quais metadados ela deve ter (autor, portas, variáveis de ambiente)?

Resultado: Toda "Docker Image" é na verdade uma **OCI Compliant Image**.

## 3. Runtime Spec (Especificação de Runtime)

Define como um container deve ser executado a partir de uma imagem.

- Como o sistema de arquivos deve ser montado?
- Quais Namespaces e Cgroups o Kernel deve criar?
- Quais chamadas de sistema (syscalls) o container pode fazer?

Resultado: Qualquer software que siga essa regra consegue transformar um conjunto de arquivos em um container vivo no Kernel Linux.

O arquivo central da Runtime Spec é o `config.json`, gerado pela engine antes de chamar o runtime. Ele especifica:

- `namespaces`: quais namespaces Linux criar (pid, net, mnt, uts, ipc, user)
- `linux.resources`: limites de CPU, memória e I/O via cgroups
- `linux.capabilities`: quais capabilities Linux o processo tem (ex: `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`)
- `linux.seccomp`: perfil de filtro de syscalls (quais chamadas são permitidas ou bloqueadas)
- `root.path`: caminho do `rootfs`, o diretório que vira `/` dentro do container

## 4. O Fluxo de Execução

Para entender onde a mágica acontece, imagine o caminho que o Docker (ou qualquer engine) percorre:

1. **Download:** O Docker baixa a imagem seguindo a Image Spec.
2. **Unpack:** Ele descompacta as camadas em uma pasta — o **OCI Bundle**.
3. **Configuração:** Ele gera o arquivo `config.json` com as regras de Namespaces, Cgroups, capabilities e seccomp.
4. **Execução:** Ele chama um **Runtime** (como o `runc`) e diz: *"Aqui está o bundle e aqui estão as regras. Cria o container."*

O OCI Bundle tem esta estrutura:

```
meu-container/
├── config.json      ← regras de execução (Runtime Spec)
└── rootfs/          ← sistema de arquivos raiz (resultado das layers)
    ├── bin/
    ├── etc/
    └── ...
```

## 5. As Camadas da Image Spec em Profundidade

Cada camada de uma imagem OCI é um arquivo `.tar.gz` contendo apenas o **diff** em relação à camada anterior — arquivos adicionados, modificados ou removidos. Isso é análogo a snapshots incrementais de sistema de arquivos.

Quando o runtime precisa usar a imagem, as camadas são aplicadas em ordem e montadas via **OverlayFS**:

```
Camada base (Ubuntu)   → layer-0.tar.gz  (read-only)
Camada apt install     → layer-1.tar.gz  (read-only)
Camada COPY app        → layer-2.tar.gz  (read-only)
Container layer        →                  (read-write, efêmera)
```

O resultado final — `rootfs` — é o que se torna o `/` do container após o `pivot_root()`.

## 6. Por que a OCI é importante?

No dia a dia, isso garante **interoperabilidade total**:

- Você builda uma imagem no seu MacBook (Docker).
- Sobe essa imagem para o Google Cloud (Artifact Registry).
- Roda essa imagem no Kubernetes (que usa `containerd`).
- Nada quebra, porque todos respeitam o mesmo "contrato" da OCI.

> "A OCI é o cartório dos containers. Ela não faz o código, ela define as leis. Sem ela, o mundo dos containers seria como se cada marca de carro tivesse seu próprio tipo de combustível e suas próprias estradas. A OCI padronizou o combustível (Imagem) e a largura das estradas (Runtime)."

## Conexão com Sistemas Operacionais

- A Runtime Spec define exatamente quais syscalls, capabilities, namespaces e cgroups um container deve ter — é uma especificação de interface do kernel → [[System Calls]], [[Processos]]
- O `config.json` mapeia diretamente para primitivas do kernel: namespaces a criar, limites de cgroup, capabilities Linux (CAP_NET_ADMIN, CAP_SYS_PTRACE), perfil seccomp → [[Syscalls para Gerenciamento de Processos]], [[Hardware de Proteção]]
- A Image Spec define layers como arquivos `.tar.gz` — cada layer é um diff de sistema de arquivos, análogo a snapshots incrementais de storage → [[Arquivos]]
- A OCI é para containers o que a POSIX é para sistemas operacionais: uma interface padronizada que permite portabilidade — "build once, run anywhere" → [[System Calls]]
- O `rootfs` no OCI bundle é o resultado de aplicar todas as layers em ordem. Esse diretório se torna o `/` do container após a chamada `pivot_root()` — troca da raiz do sistema de arquivos → [[Arquivos]]
- Interoperabilidade OCI = mesmo princípio da API POSIX: programas escritos contra uma interface padrão rodam em qualquer implementação compatível → [[System Calls]]

## Conexão com Go

- A maioria das ferramentas que implementam a OCI (runc, containerd, Docker) são escritas em Go — a linguagem foi escolhida por seu modelo de concorrência e por compilar para binários estáticos sem dependências → [[Goroutines]]
- O parsing do `config.json` em Go usa `encoding/json` com structs; a especificação OCI disponibiliza os tipos Go oficiais no pacote `github.com/opencontainers/runtime-spec/specs-go`
- Ferramentas CLI que seguem a OCI (como `runc`) usam o padrão de `os.Exec` e `syscall.Clone` do Go para invocar as primitivas do kernel → [[System Calls]]
