---
tags:
  - docker
  - docker/fundamentos
---

# Processo raiz (PID 1)

Entender o **PID 1** é entender como o ciclo de vida de uma aplicação funciona dentro do Linux.

No Linux, o **PID 1** é o primeiro processo que o Kernel inicia. Em um sistema operacional comum (Ubuntu, CentOS), esse processo costuma ser o **systemd** ou o **init**. No mundo dos containers, o **PID 1** é, por padrão, o processo que você definiu no seu `ENTRYPOINT` ou `CMD`.

---

## 1. As duas grandes responsabilidades do PID 1

### 1.1. Colher Processos Zumbis (Reaping Zombie Processes)

No Linux, quando um processo filho termina, ele não desaparece imediatamente. Ele entra em estado "Zombie" (defunct) até que o processo pai "colha" o código de saída dele (usando a chamada de sistema `wait()`).

- **O Problema:** Se o seu container roda um script que abre vários sub-processos e esse script não sabe "limpar" os filhos que morreram, o container vai encher de processos zumbis.
- **A Consequência:** Eventualmente, você atinge o limite máximo de PIDs do Cgroup ou do Host, e o sistema para de aceitar novos processos.

### 1.2. Repassar Sinais (Signal Handling)

Quando você dá um `docker stop`, o Docker envia um sinal **SIGTERM** (solicitação de parada graciosa) para o processo **PID 1**.

- **O Problema:** Muitos executáveis (como scripts de Shell puro ou algumas aplicações Node/Python mal configuradas) **não sabem ouvir sinais**.
- **A Consequência:** O processo ignora o comando de parar. O Docker espera 10 segundos e, frustrado, envia um **SIGKILL** (uma "voadora" no processo que o encerra abruptamente, podendo corromper dados ou não fechar conexões de banco de dados corretamente).

---

## 2. O perigo do Shell Form no Dockerfile

Este é um erro clássico. Existe uma diferença crucial entre como você escreve o comando de execução:

**Shell Form (problemático):**
```dockerfile
CMD node index.js
```

- Aqui, o PID 1 será o `/bin/sh -c`. O seu Node.js será um processo filho do Shell.
- O Shell **não repassa sinais**. Se o Docker pedir para o Shell parar, o Node nunca vai saber. O container vai levar 10 segundos para morrer via SIGKILL.

**Exec Form (correto):**
```dockerfile
CMD ["node", "index.js"]
```

- Aqui, o binário do `node` é executado diretamente como **PID 1**. Ele recebe os sinais diretamente do Kernel/Docker.

### Diferença no nível do processo

```
Shell Form:
PID 1: /bin/sh -c "node index.js"
  PID 2: node index.js          ← nunca recebe SIGTERM

Exec Form:
PID 1: node index.js            ← recebe SIGTERM diretamente
```

---

## 3. A solução: Tini ou dumb-init

Às vezes, sua aplicação é complexa ou você precisa rodar um script Bash como entrada. Para não ter que programar um gerenciador de sinais complexo, a comunidade criou init-systems minúsculos.

O Docker tem isso embutido. Se você usar a flag `--init`:

```bash
docker run --init my-app
```

O Docker insere um binário chamado `tini` como PID 1. O `tini` é um "mordomo" profissional:

1. Ele recebe os sinais e repassa para sua app.
2. Ele limpa automaticamente todos os processos zumbis que sua app deixar para trás.

No Dockerfile também é possível incluir explicitamente:

```dockerfile
FROM node:18
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "index.js"]
```

---

## 4. Como o Kernel trata o PID 1 de forma especial (Under the Hood)

O Kernel Linux trata o PID 1 de forma especial em dois aspectos:

**1. Sinais ignorados por padrão não matam o PID 1:**
Em processos normais, enviar `SIGTERM` sem handler definido causa a morte do processo (disposição padrão). No PID 1, sinais sem handler explícito são silenciosamente ignorados. Isso é um mecanismo de proteção: o init não pode ser derrubado acidentalmente. Mas é uma armadilha para containers — se o PID 1 não tiver um handler explícito para SIGTERM, o `docker stop` vai parecer não funcionar.

**2. Processos órfãos são adotados pelo PID 1:**
Quando um processo pai morre sem fazer `wait()` nos filhos, o Kernel reatribui esses processos ao PID 1. O PID 1 torna-se responsável por chamá-los com `wait()`. Se não o fizer, eles acumulam como zumbis. Em um sistema Linux normal, o `systemd` faz isso. Em um container, é responsabilidade da sua aplicação (ou do `tini`).

```
Processo pai morre
       |
       v
PID 1 herda todos os filhos órfãos
       |
PID 1 deve chamar wait() para colhê-los
       |
       v
Sem wait() → zumbi acumula → tabela de PIDs enche → EAGAIN em fork()
```

**Verificando dentro do container:**
```bash
# Dentro do container, ver o PID 1
cat /proc/1/status
# Mostra o PID real no host:
cat /proc/1/status | grep NSpid
# NSpid:  1  4582
# (1 = PID dentro do namespace, 4582 = PID real no host)
```

---

## 5. Resumo

> "O PID 1 é o 'pai de todos' dentro do container. No Linux, ser PID 1 traz poderes, mas também obrigações: você deve saber ouvir quando o sistema pede para você parar (Signals) e deve limpar a sujeira que seus processos filhos deixam (Zombies). Se o seu PID 1 falha nessas missões, seu container se torna instável e difícil de gerenciar."

---

## Conexão com Sistemas Operacionais

- **PID 1 = init no Linux normal (systemd/SysV init) → adota processos filhos órfãos → [[Hierarquia de Processos]], [[O Modelo de Processos]]**
  - No modelo de processos Linux, existe sempre uma hierarquia em árvore. Quando um nó da árvore (processo pai) morre antes dos filhos, o Kernel reconecta esses filhos ao PID 1 como novos filhos. É um mecanismo de garantia de que todo processo sempre tem um pai responsável por coletar seu exit status.

- **Processo zumbi: processo termina → entra em estado zumbi → pai deve chamar `wait()` para coletar o exit code → [[Término de Processos]], [[Implementação de Processos]]**
  - Quando um processo chama `exit()`, o Kernel libera a maioria dos recursos (memória, file descriptors), mas mantém a entrada na tabela de processos com o exit code. Essa entrada "fantasma" é o estado zumbi (`Z`). Ela só é removida quando o pai chama `wait()` ou `waitpid()`. Entradas zumbi são inofensivas individualmente, mas ocupam slots da tabela de PIDs.

- **Se o PID 1 não chama `wait()` → zumbis acumulam → tabela de PIDs enche → [[Estados de Processos]]**
  - A tabela de processos do Kernel tem um limite máximo (controlado por `pid_max` e pelo cgroup `pids`). Um vazamento de zumbis é uma forma de resource leak: não consome CPU nem memória, mas consome entradas na tabela. Quando esgotada, chamadas `fork()` falham com `EAGAIN`.

- **Shell form `CMD node app.js` → shell faz fork de node como filho → `/bin/sh -c "node app.js"` é PID 1 → shell não repassa SIGTERM → [[Término de Processos]]**
  - O shell invoca programas externos fazendo `fork()` + `exec()`. Quando recebe um sinal, o shell por padrão não o encaminha para os filhos. O SIGTERM chega ao shell (PID 1), mas o `node` (filho) nunca é notificado. O shell então espera o filho terminar (que nunca termina), e após 10 segundos o Docker envia SIGKILL para toda a árvore.

- **Exec form `CMD ["node", "app.js"]` → node é diretamente exec'd como PID 1 → recebe sinais diretamente → [[Processos]]**
  - O Docker usa `exec()` para substituir o processo inicial pelo binário da aplicação. O processo mantém o mesmo PID (1 no namespace) mas passa a executar o código do `node`. Sinais enviados ao PID 1 chegam diretamente ao processo Node.

- **SIGTERM vs SIGKILL: SIGTERM é catchable (graceful shutdown), SIGKILL não é → [[Término de Processos]]**
  - SIGTERM (sinal 15) é um pedido educado: o processo pode instalar um handler e fazer cleanup (fechar conexões, flush de dados, etc.) antes de terminar. SIGKILL (sinal 9) é enviado diretamente pelo Kernel e não pode ser interceptado ou ignorado — o processo é terminado imediatamente sem chance de cleanup.

- **`docker stop` envia SIGTERM → espera 10s → envia SIGKILL → [[Término de Processos]], [[Processos]]**
  - Esse comportamento segue o padrão Unix de encerramento gracioso: dar ao processo a chance de se auto-finalizar corretamente antes de forçar a terminação. Os 10 segundos são configuráveis via `docker stop --time=<n>`.

- **tini (`--init` flag): um init mínimo que chama `waitpid()` em loop (colhe zumbis) e repassa sinais → [[Hierarquia de Processos]]**
  - O `tini` implementa as duas responsabilidades fundamentais do PID 1: (1) um loop `waitpid(-1, ...)` que colhe qualquer filho que termine, e (2) handlers de sinal que encaminham SIGTERM/SIGINT para o processo filho principal. É a implementação mínima viável do que o `systemd` faz em sistemas completos.

- **`/proc/1/status` dentro do container mostra o PID real do Kernel → [[Implementação de Processos]]**
  - O arquivo `NSpid` em `/proc/<pid>/status` lista o PID do processo em cada namespace da hierarquia. `NSpid: 1  4582` significa: PID 1 no namespace do container, PID 4582 no namespace do host. Isso é a evidência direta de que o "PID 1 do container" é apenas um processo normal do Kernel com uma visão filtrada.
