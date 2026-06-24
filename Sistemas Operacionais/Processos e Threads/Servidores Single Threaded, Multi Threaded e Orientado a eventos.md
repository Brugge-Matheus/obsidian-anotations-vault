---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.3"
---
# Servidores Single Threaded, Multi Threaded e Orientado a Eventos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.3

---

# 🖥️ 2.3 — Servidores Orientados a Eventos

Na seção anterior (2.2.1), vimos o exemplo clássico do servidor web para justificar o uso de threads. Mas Tanenbaum vai além: ele usa exatamente esse servidor web para mostrar que existem **três arquiteturas fundamentais** para lidar com múltiplas requisições simultâneas, cada uma com suas vantagens e trade-offs.

Antes de compará-las, é importante entender o problema que todas tentam resolver:

> 💡 **O problema do servidor:** um servidor web recebe requisições de clientes. Para atender uma requisição, ele frequentemente precisa buscar dados no disco — uma operação lenta (milissegundos vs. nanossegundos da CPU). Como atender muitas requisições ao mesmo tempo sem desperdiçar CPU esperando I/O?

---

# 🧵 Arquitetura 1 — Processo de Thread Única *(Single-Threaded)*

A abordagem mais simples: um único processo com uma única thread atende requisições **sequencialmente**, uma de cada vez.

```
Requisição 1 chega
       │
       ▼
Servidor verifica cache
       │
       ▼  (não está em cache)
Servidor bloqueia esperando disco  ← CPU ociosa aqui
       │
       ▼
Dado chega do disco
       │
       ▼
Responde ao cliente
       │
       ▼
Só agora atende a Requisição 2 (que ficou esperando na fila)
```

**O problema central:** a chamada de sistema de leitura do disco é **bloqueante** — enquanto o servidor espera o disco, nenhuma outra requisição pode ser atendida. Com requisições chegando rapidamente, as filas crescem e a performance despenca.

> 💡 **Chamada de sistema bloqueante:** uma system call que suspende completamente a execução do processo até que a operação seja concluída. Exemplos: `read()` esperando o disco, `recv()` esperando dados da rede. O processo fica no estado *bloqueado* enquanto espera.

**Vantagem:** simplicidade total. Código linear, fácil de escrever e depurar.

**Desvantagem:** péssimo desempenho sob carga. Um único gargalo de I/O paralisa todo o servidor.

---

# 🧵🧵 Arquitetura 2 — Múltiplas Threads *(Multi-Threaded)*

A solução clássica para o problema acima: o servidor é organizado com uma **thread despachante** e várias **threads operárias**.

> 📌 **Figura 2.8 — Servidor web multithreaded (já estudada em Utilização de Threads)**

```
            Processo do servidor web
┌──────────────────────────────────────────────┐
│              Espaço do usuário               │
│                                              │
│  ┌──────────────┐     ┌────────────────────┐ │
│  │   Thread     │────►│  Thread Operária 1 │ │
│  │ Despachante  │     ├────────────────────┤ │
│  │              │────►│  Thread Operária 2 │ │
│  │ (recebe req, │     ├────────────────────┤ │
│  │  distribui)  │────►│  Thread Operária 3 │ │
│  └──────────────┘     └────────────────────┘ │
│         │                      │             │
└─────────┼──────────────────────┼─────────────┘
          ▼                      ▼
    Rede (socket)            Disco (I/O)
```

**Como funciona:**

1. A **thread despachante** fica em loop, aceitando requisições da rede
2. Para cada requisição, ela seleciona uma thread operária disponível e passa o trabalho
3. A **thread operária** verifica o cache; se o dado não estiver lá, faz uma leitura **bloqueante** do disco
4. Enquanto a operária bloqueia esperando o disco, o despachante continua recebendo novas requisições e outras operárias continuam trabalhando

> 💡 **Thread despachante (*dispatcher thread*):** thread cuja única função é receber requisições e distribuí-las para threads operárias disponíveis. É o "gerente" do pool de threads.

> 💡 **Thread operária (*worker thread*):** thread que de fato processa a requisição — verifica cache, acessa disco, monta a resposta e a envia ao cliente.

**Vantagem:** alto desempenho. Múltiplas requisições são processadas genuinamente em paralelo. Chamadas bloqueantes não travam o servidor inteiro — apenas a thread que as faz.

**Desvantagem:** maior complexidade. Variáveis globais compartilhadas entre threads exigem sincronização (o problema do `errno` visto em [[Convertendo Thread em Multithread]]). Gestão de pilhas múltiplas é mais delicada.

---

# ⚡ Arquitetura 3 — Orientada a Eventos *(Event-Driven / Finite State Machine)*

Uma terceira abordagem para quando threads não estejam disponíveis ou sejam consideradas inaceitáveis por perda de desempenho.

## A ideia central

Se as threads múltiplas existem para evitar que uma única thread bloqueie esperando I/O, por que não usar **chamadas de sistema não bloqueantes** e nunca bloquear de jeito nenhum?

> 💡 **Chamada de sistema não bloqueante (*non-blocking system call*):** uma syscall que retorna **imediatamente**, sem esperar que a operação termine. Em vez de bloquear o processo, ela inicia a operação e retorna um indicador de que o resultado ainda não está disponível. O processo pode então fazer outra coisa e verificar mais tarde se o resultado chegou.

> 💡 **Chamada de sistema assíncrona (*asynchronous system call*):** semelhante à não bloqueante — o processo solicita uma operação, continua executando, e é **notificado** (via sinal, interrupção ou callback) quando a operação terminar.

## Como funciona o modelo orientado a eventos

O servidor é implementado como uma **máquina de estado finito** com uma única thread:

```
Servidor inicia, cria socket, entra no loop principal
                     │
                     ▼
            ┌─────────────────┐
            │    select()     │◄────────────────────────────┐
            │  (monitora N    │                             │
            │   descritores   │                             │
            │   simultaneamente│                            │
            └────────┬────────┘                             │
                     │ algum fd ficou pronto                │
                     ▼                                      │
          ┌──────────────────────┐                          │
          │ Qual evento ocorreu? │                          │
          └──┬────────────────┬──┘                          │
             │                │                             │
             ▼                ▼                             │
    Nova conexão         Dados chegaram           ──────────┘
    de cliente           de um fd existente       (volta ao select)
             │                │
             ▼                ▼
    accept() →         receive() → processa
    adiciona fd        → tenta enviar resposta
    ao conjunto        → se não puder enviar tudo:
                         salva estado em tabela,
                         volta ao select()
```

> 💡 **`select()` (ou `epoll`, `kqueue`):** chamada de sistema que recebe um conjunto de descritores de arquivo e **bloqueia até que pelo menos um deles esteja pronto** para leitura ou escrita. É a única operação bloqueante — mas bloqueia esperando *qualquer* evento, não um evento específico. Retorna dizendo quais fds estão prontos. Isso permite que **uma única thread sirva centenas de conexões simultâneas**.

> 💡 **Máquina de estado finito (*finite state machine*):** modelo computacional onde um programa tem um conjunto definido de **estados** e transições entre eles, disparadas por **eventos**. No servidor orientado a eventos, cada requisição em andamento é um objeto com um estado salvo (o que já foi recebido, o que falta enviar), e os eventos são os descritores de arquivo ficando prontos.

## O exemplo do pseudocódigo (Figura 2.19)

Tanenbaum mostra um servidor de agradecimento orientado a eventos que responde "Obrigado!" a cada cliente:

```
/* Variáveis principais: */
svrSock   → socket principal, escuta na porta TCP 12345
toSend    → tabela: para cada fd, armazena bytes pendentes a enviar
inFds     → conjunto de fds para monitorar chegada de dados
outFds    → conjunto de fds prontos para envio
exceptFds → fds com condição de erro

while (TRUE) {
    /* Bloqueia até algum fd ficar pronto */
    rdyFds = select(inFds, outFds, exceptFds, NO_TIMEOUT)

    /* Nova conexão? */
    for (fd in rdyFds) {
        if (fd == svrSock) {
            newSock = accept(svrSock)   /* aceita cliente */
            inFds = inFds ∪ {newSock}  /* monitora o novo fd */
        }
        else {
            /* Dados chegando de cliente existente */
            n = receive(fd, msgBuf, MAX_MSG_SIZE)
            printf("Recebido: %s", msgBuf)
            toSend.put(fd, thankYouMsg) /* agenda envio */
            outFds = outFds ∪ {fd}     /* monitora para escrita */
        }
    }

    /* Enviar respostas para fds prontos para escrita */
    for (fd in rdyOuts) {
        msg = toSend.get(fd)
        n = send(fd, msg, strlen(msg))
        if (n < strlen(thankYouMsg)) {
            /* Não enviou tudo — salva o que falta */
            toSend.put(fd, msg+n)
        } else {
            toSend.destroy(fd)  /* concluído */
            outFds = outFds \ {fd}
        }
    }
}
```

**O ponto chave:** o servidor nunca bloqueia em uma única conexão. Quando não consegue enviar a resposta inteira de uma vez, ele **salva o estado** (quantos bytes faltam) na tabela `toSend` e volta ao `select()`. Na próxima iteração, quando o fd estiver pronto, ele tenta novamente. O estado de cada requisição em andamento vive na tabela — não na pilha de uma thread bloqueada.

## O problema da gestão de pilha no modelo multithreaded

Tanenbaum destaca um problema prático do modelo multithreaded que o modelo orientado a eventos não tem: **gerenciamento de pilha**.

Em sistemas onde o núcleo não conhece todas as threads (implementação em user space), quando a pilha de um thread estoura, o núcleo só consegue fornecer mais pilha para o processo inteiro. Com múltiplas threads, ele não tem como saber que a falta de memória está relacionada ao crescimento da pilha de uma thread específica — o problema passa despercebido até causar corrupção silenciosa.

No modelo orientado a eventos com thread única, há apenas **uma pilha**, e o núcleo consegue gerenciá-la normalmente.

---

# 🌐 Interfaces de Notificação de Eventos nos SOs

Os sistemas operacionais populares oferecem interfaces altamente otimizadas para I/O assíncrona, muito mais eficientes que o `select` básico:

| Sistema             | Interface                   | Característica                                                  |
| ------------------- | --------------------------- | --------------------------------------------------------------- |
| **Linux**           | `epoll`                     | Escalonável para milhares de fds; retorna apenas os fds prontos |
| **FreeBSD / macOS** | `kqueue`                    | Similar ao epoll, altamente eficiente                           |
| **Windows**         | IOCP (I/O Completion Ports) | Modelo diferente: notifica quando operação *completou*          |
| **Solaris**         | `/dev/poll`                 | Similar ao epoll                                                |

> 💡 **Problema C10k:** o desafio de fazer um servidor web atender **10.000 conexões simultâneas** de forma eficiente. Servidores como o **nginx** usam o modelo orientado a eventos com `epoll`/`kqueue` e conseguem lidar confortavelmente com 10k conexões em um único processo — algo inviável com o modelo multithreaded tradicional para cada conexão.

---

# ⚖️ Comparação Final — Figura 2.20

> 📌 **Figura 2.20 — Três maneiras de criar um servidor**

```
┌──────────────────────────────────┬────────────────────────────────────────────┐
│            Modelo                │              Características               │
├──────────────────────────────────┼────────────────────────────────────────────┤
│ Threads (multithreaded)          │ Paralelismo, chamadas de sistema           │
│                                  │ bloqueantes                                │
├──────────────────────────────────┼────────────────────────────────────────────┤
│ Processo de thread única         │ Sem paralelismo, chamadas de sistema       │
│ (single-threaded)                │ bloqueantes                                │
├──────────────────────────────────┼────────────────────────────────────────────┤
│ Máquina de estado finito /       │ Paralelismo, chamadas de sistema não       │
│ Orientada a eventos              │ bloqueantes + interrupções / sinais        │
└──────────────────────────────────┴────────────────────────────────────────────┘
```

**Analisando cada trade-off:**

**Threads vs. Single-Threaded:**
- Threads trazem paralelismo real — múltiplas requisições avançam ao mesmo tempo
- Single-threaded é simples mas desperdiça CPU esperando I/O

**Threads vs. Orientada a Eventos:**
- Threads permitem usar chamadas **bloqueantes** (mais simples de escrever — `read()` e pronto)
- Orientada a eventos alcança paralelismo **sem threads** — via multiplexação de I/O
- Orientada a eventos é **mais difícil de programar**: o estado de cada requisição em andamento precisa ser salvo e restaurado manualmente (o que threads fazem automaticamente com a pilha e os registradores)

> ⚠️ **O modelo de "processo sequencial" se perde** no modelo orientado a eventos: o estado da computação precisa ser salvo e restaurado explicitamente na tabela toda vez que o servidor muda de uma requisição para outra. Estamos, na prática, simulando threads e suas pilhas da maneira mais difícil.

---

# 🏗️ Essas arquiteturas se aplicam ao próprio SO

Tanenbaum faz uma observação importante: essas três abordagens não são apenas para servidores de usuário — elas se aplicam igualmente ao **núcleo do SO**, onde a concorrência é igualmente crítica para o desempenho.

Exemplos reais:
- O **núcleo do Linux** nas CPUs Intel modernas é um núcleo de **múltiplas threads**
- O **MINIX 3** consiste em muitos servidores implementados seguindo o modelo de **máquina de estado finito orientada a eventos**

---

# ✅ Resumo do Conceito

- Existem **três arquiteturas** fundamentais para servidores (e para o próprio kernel): single-threaded, multithreaded e orientada a eventos
- **Single-threaded** é simples mas bloqueia em I/O — inaceitável sob carga
- **Multithreaded** resolve o bloqueio com paralelismo real, mas exige sincronização e gestão cuidadosa de pilhas e variáveis globais compartilhadas
- **Orientada a eventos** usa `select()`/`epoll` para multiplexar N conexões em uma única thread, com chamadas não bloqueantes — alto desempenho sem threads, mas código mais complexo
- O modelo orientado a eventos implementa uma **máquina de estado finito**: o estado de cada requisição é salvo explicitamente em uma tabela, pois não há pilha de thread para guardar esse contexto automaticamente
- **C10k** é o nome do desafio de 10.000 conexões simultâneas — resolvido eficientemente com o modelo orientado a eventos + `epoll`/`kqueue`
- A escolha entre os modelos envolve o clássico trade-off: **simplicidade de código** (bloqueante) vs. **desempenho** (não bloqueante / orientado a eventos)
- Tanto o Linux (multithreaded) quanto o MINIX 3 (orientado a eventos) usam essas arquiteturas internamente no próprio núcleo

---

## 🔗 Notas Relacionadas

- [[Utilização de Threads]] — introdução ao servidor web multithreaded (Figura 2.8) e as razões para usar threads
- [[Convertendo Thread em Multithread]] — problemas com variáveis globais compartilhadas entre threads (ex: `errno`)
- [[O Modelo Clássico de Thread]] — o que cada thread tem de privado vs. compartilhado
- [[Implementando Threads em User Space]] — implementação de threads no espaço do usuário e o problema do gerenciamento de pilha
