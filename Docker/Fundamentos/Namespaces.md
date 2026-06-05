---
tags:
  - docker
  - docker/fundamentos
---

# Namespaces

No Linux, um Namespace é um recurso do Kernel que permite isolar recursos globais do sistema para um processo específico.

Imagine que o Kernel é um prédio. Sem Namespaces, todos os moradores (processos) dividem o mesmo corredor, as mesmas tomadas e se veem o tempo todo. Com Namespaces, o Kernel cria uma "realidade paralela" para o processo: ele acha que é o único morador do prédio, que tem seu próprio corredor e sua própria numeração de portas, mas na verdade ele está apenas em uma unidade isolada.

Existem **7 tipos principais** de Namespaces no Linux. Cada um isola um aspecto diferente do sistema.

---

## 1. PID Namespace (Process ID)

Isola a árvore de processos.

- **O que faz:** Dentro do container, o processo principal (sua aplicação) recebe o **PID 1**. Para o container, ele é o "dono" do sistema.
- **Under the Hood:** No Host (sua máquina real), esse mesmo processo tem um PID comum (ex: 4582). O Kernel mapeia o PID 1 do container para o PID 4582 do Host. Se você der um `ps aux` dentro do container, você só verá os processos daquele container.

## 2. NET Namespace (Network)

Isola a pilha de rede.

- **O que faz:** Cada container tem suas próprias interfaces de rede virtuais (`eth0`), sua própria tabela de roteamento e suas próprias portas.
- **Exemplo prático:** Você pode subir 10 containers rodando um servidor na porta `80` ao mesmo tempo no mesmo Host. Eles não conflitam porque cada um está em seu próprio NET Namespace.

## 3. MNT Namespace (Mount)

Isola os pontos de montagem do sistema de arquivos.

- **O que faz:** Permite que o container tenha sua própria estrutura de diretórios (`/`, `/bin`, `/etc`) totalmente independente do Host.
- **Curiosidade:** É aqui que entra o conceito de `chroot` (change root) evoluído. O container acha que o `/` dele é o diretório da imagem, e ele não consegue enxergar o `/home/usuario` do seu Host, por exemplo.

## 4. UTS Namespace (Unix Timesharing System)

Isola o **Hostname** e o nome do domínio.

- **O que faz:** Permite que cada container tenha seu próprio nome de máquina independente. Você pode dar o nome de `meu-servidor-web` para o container, e o seu Host continuará se chamando `notebook-matheus`.

## 5. IPC Namespace (Inter-Process Communication)

Isola a comunicação entre processos.

- **O que faz:** Impede que um processo de um container acesse a memória compartilhada ou filas de mensagens de processos de outro container ou do Host. É uma barreira de segurança vital.

## 6. USER Namespace (User ID)

Isola os IDs de usuário e grupo.

- **O que faz:** Este é incrível para segurança. Ele permite que você seja **root (UID 0)** dentro do container, mas no Host você é mapeado para um usuário comum sem privilégios (ex: UID 1001).
- **Vantagem:** Se um invasor quebrar o isolamento do seu container, ele "cai" no seu PC como um usuário comum, não como root.

## 7. CGROUP Namespace

Isola a visão dos recursos de controle (Cgroups).

- **O que faz:** Faz com que o container não consiga enxergar ou alterar as limitações de recursos que foram impostas a ele mesmo.

---

## 8. Como ver isso na prática

Se você tiver um container rodando, você pode usar o comando `lsns` (list namespaces) no seu Linux para ver todos os namespaces ativos e quais processos pertencem a cada um.

Outro comando interessante no Linux é o `unshare`. Ele permite que você crie um processo isolado manualmente, sem usar Docker:

```bash
unshare --fork --pid --mount-proc /bin/bash
```

> Esse comando cria um novo Namespace de PID e roda um bash. Dentro desse bash, se você der um `ps`, verá que ele se tornou o PID 1. Parabéns, você acabou de criar um "quase-container" na mão!

---

## 9. Como os Namespaces são criados pelo Kernel (Under the Hood)

A criação de namespaces é feita via syscalls. Não há um comando mágico do Docker — tudo desce ao nível do Kernel:

```
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER)
```

O flag `CLONE_NEWPID` cria um novo namespace de PID. O filho do `clone()` será o PID 1 dentro desse novo namespace, enquanto no host ele tem um PID normal.

Você também pode criar namespaces para um processo já existente com `unshare()`, ou juntar um processo a um namespace existente com `setns()`.

### Verificando namespaces no sistema de arquivos

O Kernel expõe os namespaces como arquivos simbólicos em `/proc`:

```bash
ls -la /proc/<pid>/ns/
# cgroup -> cgroup:[4026531835]
# ipc    -> ipc:[4026531839]
# mnt    -> mnt:[4026531840]
# net    -> net:[4026532001]
# pid    -> pid:[4026531836]
# user   -> user:[4026531837]
# uts    -> uts:[4026531838]
```

Dois processos com o mesmo número de inode em `net` estão no mesmo NET namespace (compartilham a pilha de rede).

### O namespace de PID em detalhe

```
Host (PID global)       Container (PID namespace)
─────────────────       ─────────────────────────
PID 1 (systemd)
PID 2 (kthreadd)
...
PID 4582 (nginx)   ──>  PID 1 (nginx)   ← container "vê" isso
PID 4591 (worker)  ──>  PID 2 (worker)
```

O Kernel mantém uma tabela de mapeamento interna. O processo `4582` existe globalmente, mas dentro do namespace enxerga a si mesmo como `1`.

---

## 10. Resumo

> "Namespaces são a lente de realidade virtual do Linux. Eles não movem os arquivos ou processos de lugar, apenas filtram o que o processo consegue enxergar. Se o Docker diz 'isso é um container', o Kernel diz 'isso é um processo com vários Namespaces aplicados'."

---

## Conexão com Sistemas Operacionais

- **PID namespace: o PID 1 do container mapeia para um PID real no host → [[O Modelo de Processos]], [[Criação de Processos]]**
  - Quando o Docker chama `clone()` com `CLONE_NEWPID`, o processo filho nasce como PID 1 dentro do novo namespace. O Kernel mantém internamente o PID real (ex: 4582) para gerenciamento global, mas o processo só enxerga o namespace local.

- **Syscall `clone()` com flags `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS` cria namespaces → [[Syscalls para Gerenciamento de Processos]]**
  - `clone()` é a syscall central do Linux para criação de processos (é a base de `fork()`). Ao adicionar os flags `CLONE_NEW*`, você instrui o Kernel a criar um novo namespace de cada tipo solicitado e associá-lo ao processo filho.

- **NET namespace: cada container tem sua própria pilha de rede → interfaces virtuais (pares veth) → [[Dispositivos de IO]]**
  - O Kernel cria um par de interfaces virtuais (`veth pair`): uma ponta fica dentro do NET namespace do container (`eth0`) e a outra no host. O tráfego passa por essa "ponte virtual" — é um dispositivo de I/O de rede implementado em software.

- **MNT namespace: syscall `pivot_root()` / `chroot()` muda o filesystem raiz → [[Arquivos]], [[Syscalls para Gerenciamento de Processos]]**
  - O Docker usa `pivot_root()` (mais segura que `chroot()`) para mover a raiz do filesystem do container para o diretório da imagem. Após isso, o processo não consegue acessar caminhos fora do seu MNT namespace.

- **USER namespace: mapeamento de UID (0 no container → 1001 no host) → [[Proteção]], [[Hardware de Proteção]]**
  - O Kernel mantém arquivos `/proc/<pid>/uid_map` e `/proc/<pid>/gid_map` com os mapeamentos. Toda checagem de permissão de arquivo é feita usando o UID efetivo no namespace do processo, mas o controle de acesso ao hardware e recursos do host usa o UID mapeado real.

- **IPC namespace: isola memória compartilhada SysV, filas de mensagens, semáforos → [[Threads POSIX]]**
  - Os mecanismos de IPC do SysV (semáforos, filas de mensagem, memória compartilhada) são usados por threads e processos para sincronização. Sem o IPC namespace, um processo de um container poderia abrir a memória compartilhada de outro container usando o mesmo key.

- **UTS namespace: isola hostname/domainname**
  - Permite que `gethostname()` retorne valores diferentes para processos em namespaces diferentes, sem afetar o hostname do host.

- **`unshare --fork --pid --mount-proc /bin/bash` = criando namespaces manualmente sem Docker → [[Processos]]**
  - Essa é a demonstração direta de que containers não são magia: são processos Linux com namespaces. O `unshare` chama as mesmas syscalls que o Docker usaria.

- **`/proc/[pid]/ns/` symlinks mostram a qual namespace cada processo pertence → [[Implementação de Processos]]**
  - A implementação de processos no Kernel armazena, na `task_struct` (estrutura interna de cada processo), ponteiros para os namespaces de que o processo faz parte. O `/proc` filesystem expõe isso para userspace.
