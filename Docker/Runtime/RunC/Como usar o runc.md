---
tags:
  - docker
  - docker/runtime
---

# Como usar o runc

O **runc** é uma ferramenta de linha de comando que permite criar, iniciar, parar e gerenciar containers conforme a especificação OCI. Para usá-lo, você precisa preparar um ambiente básico com uma raiz de sistema de arquivos e um arquivo de configuração.

## Passo 1: Instalar o runc

Em muitas distribuições Linux, você pode instalar o runc via gerenciador de pacotes, ou compilar a partir do código-fonte oficial no GitHub.

Exemplo para Ubuntu/Debian:

```bash
sudo apt-get install runc
```

Ou para compilar:

```bash
git clone https://github.com/opencontainers/runc.git
cd runc
make
sudo make install
```

## Passo 2: Criar um diretório para o container

Você precisa de um diretório que será a raiz do container (root filesystem). Pode ser um sistema de arquivos mínimo, como um Alpine Linux extraído.

Exemplo:

```bash
mkdir -p mycontainer/rootfs
# Extraia uma imagem mínima, por exemplo Alpine, dentro de rootfs
tar -xzf alpine-minirootfs-3.18.2-x86_64.tar.gz -C mycontainer/rootfs
```

## Passo 3: Gerar o arquivo de configuração padrão (config.json)

O runc usa um arquivo `config.json` que descreve o container (namespaces, cgroups, mounts, etc).

Para gerar um arquivo padrão, rode:

```bash
runc spec --rootless
```

Isso cria um `config.json` no diretório atual.

Você pode mover esse arquivo para dentro do diretório do container:

```bash
mv config.json mycontainer/
```

Edite o `config.json` para ajustar o caminho do root filesystem:

```json
"root": {
  "path": "rootfs",
  "readonly": false
}
```

## Passo 4: Criar o container

No diretório que contém o `config.json` e o `rootfs`, rode:

```bash
runc create mycontainer
```

Isso cria o container chamado `mycontainer` (mas não o inicia ainda). Nesta fase o runc já:
- Criou os namespaces com `clone()`
- Configurou os cgroups em `/sys/fs/cgroup/`
- Montou o OverlayFS
- Aplicou o perfil seccomp e as capabilities

O processo principal fica bloqueado esperando o `runc start`.

## Passo 5: Iniciar o container

Para iniciar o container e entrar no processo principal (geralmente um shell):

```bash
runc start mycontainer
```

Você estará dentro do container, isolado pelos namespaces e cgroups configurados. O processo `runc` executa `execve()` para substituir a si mesmo pelo processo da aplicação.

## Passo 6: Parar e deletar o container

Para parar o container:

```bash
runc kill mycontainer
```

Para deletar o container:

```bash
runc delete mycontainer
```

## Comandos úteis do runc

| Comando | Descrição |
| --- | --- |
| `runc spec` | Gera um arquivo `config.json` padrão |
| `runc spec --rootless` | Gera config.json para modo rootless (sem root) |
| `runc create <id>` | Cria o container (não inicia) |
| `runc start <id>` | Inicia o container |
| `runc kill <id>` | Envia sinal para o container (ex: SIGKILL) |
| `runc delete <id>` | Remove o container |
| `runc list` | Lista containers criados |
| `runc state <id>` | Mostra o estado atual do container |
| `runc exec <id> <cmd>` | Executa comando dentro de um container rodando |

## Observações importantes

- O runc exige privilégios de root para criar namespaces e aplicar cgroups, a menos que você use o modo rootless (que tem limitações).
- O root filesystem precisa ser um sistema Linux válido, com binários e bibliotecas para o processo principal rodar.
- O `config.json` pode ser customizado para adicionar volumes, limitar recursos, configurar rede, etc.
- Containers criados com `runc create` ficam no estado `created` — listáveis com `runc list` — até o `runc start` ser chamado.

## Conexão com Sistemas Operacionais

- A estrutura do `config.json` mapeia diretamente para as syscalls que o runc invoca: cada campo `namespaces[]` vira uma flag do `clone()`, cada campo `linux.resources` vira escritas em `/sys/fs/cgroup/` → [[System Calls]], [[Syscalls para Gerenciamento de Processos]]
- `runc spec --rootless` gera uma configuração que usa o **User Namespace** (`CLONE_NEWUSER`) para mapear o UID do usuário atual para root dentro do container — sem precisar de privilégios reais no host → [[Proteção]]
- `runc create` vs `runc start` é uma inicialização em duas fases: `create` configura o ambiente (namespaces, cgroups, mounts, seccomp), `start` executa o processo — isso permite inspecionar e modificar o ambiente antes de iniciar → [[Criação de Processos]]
- `runc kill <id>` envia um sinal SIGKILL (ou outro especificado) para o PID 1 do container — internamente usa a syscall `kill()` com o PID do processo principal do container → [[Término de Processos]]
