---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.4.5 e 2.4.6"
---
# Semáforos e Mutexes

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.4.5 e 2.4.6

---

# 🚦 2.4.5 — Semáforos

## De onde viemos — o problema que semáforos resolvem

Em [[Dormir e Despertar]], vimos que `sleep`/`wakeup` são a ideia certa (bloquear em vez de girar), mas têm um defeito fatal: **um wakeup enviado para um processo que ainda não dormiu é simplesmente perdido**. Isso causou deadlock no produtor-consumidor.

A raiz do problema: `sleep` e `wakeup` são eventos binários — ou o processo está dormindo ou não está. Se o wakeup chegar antes do sleep, o sinal desaparece para sempre.

Em 1965, E. W. Dijkstra propôs a solução: em vez de um evento binário (dormindo/acordado), usar um **contador inteiro** que **acumula** os sinais de wakeup. Esse contador é o **semáforo**.

---

## 🧮 O que é um semáforo — a intuição

Pense em um semáforo como um **cofre de créditos de acesso**:

```
Semáforo s = 3

Significa: "ainda há 3 permissões disponíveis para acessar este recurso"

Processo quer acessar → saca 1 crédito → s = 2 → entra
Processo quer acessar → saca 1 crédito → s = 1 → entra
Processo quer acessar → saca 1 crédito → s = 0 → entra
Processo quer acessar → cofre vazio (s = 0) → BLOQUEIA (dorme)

Processo sai → deposita 1 crédito → s = 1 → acorda quem estava dormindo
```

> 💡 **Semáforo:** variável inteira especial que conta o número de "permissões" disponíveis para acessar um recurso. Pode ter valor 0 (nenhuma permissão disponível — próximo processo vai dormir) ou positivo (N permissões disponíveis). Nunca assume valor negativo. É manipulado **apenas** pelas operações `down` e `up`.

Um semáforo com valor 0 indica que nenhum sinal de despertar foi salvo. Se `down` é executado com valor 0, o processo dorme — mas o crédito fica "registrado como devido". Quando alguém faz `up`, o crédito é entregue.

---

## ⬇️⬆️ As operações `down` e `up` — explicação detalhada

Dijkstra definiu duas operações para manipular semáforos. Originalmente chamadas de **P** e **V** (do holandês *proberen* — tentar, e *verhogen* — levantar/erguer), hoje são chamadas universalmente de **`down`** e **`up`**.

### Operação `down` (equivalente ao `sleep` generalizado)

```
down(s):
    se s > 0:
        s = s - 1          ← saca uma permissão, continua executando
    senão (s == 0):
        coloca o processo para dormir  ← sem permissão disponível, bloqueia
        (quando acordar, a operação down será completada automaticamente)
```

> 💡 **`down`:** tenta decrementar o semáforo. Se o valor é maior que zero, decrementa e continua. Se é zero, o processo **bloqueia** (dorme) até que outro processo faça `up`. É a versão generalizada do `sleep`.

**A diferença crucial em relação ao `sleep` puro:** o `down` não apenas dorme — ele primeiro verifica se há crédito disponível. Se houver, consome o crédito e segue sem dormir. O `sleep` antigo dormia incondicionalmente.

### Operação `up` (equivalente ao `wakeup` generalizado)

```
up(s):
    se há processos dormindo neste semáforo:
        acorda um deles   ← o processo acordado completa seu down
    senão:
        s = s + 1         ← ninguém dormindo: deposita o crédito para uso futuro
```

> 💡 **`up`:** incrementa o semáforo ou acorda um processo bloqueado. Se há processos esperando, acorda um (que então completa seu `down`). Se não há ninguém esperando, incrementa o valor — o crédito fica guardado para o próximo `down`. É a versão generalizada do `wakeup`.

**A diferença crucial em relação ao `wakeup` puro:** se ninguém está dormindo quando `up` é chamado, o crédito **não se perde** — fica armazenado no contador. Com `wakeup` antigo, se ninguém estava dormindo, o sinal era simplesmente descartado.

### Comparação direta: `sleep`/`wakeup` vs. `down`/`up`

```
┌──────────────────────────────────────────────────────────────────┐
│                    sleep/wakeup (antigo)                        │
├──────────────────────────────────────────────────────────────────┤
│ sleep()    → dorme SEMPRE, incondicionalmente                   │
│ wakeup(p)  → acorda p SE ele estiver dormindo                   │
│              SE não estiver → sinal PERDIDO para sempre ❌       │
│                                                                  │
│ Estado: binário (dormindo / acordado)                           │
│ Problema: race condition — wakeup antes do sleep = deadlock     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      down/up (semáforo)                         │
├──────────────────────────────────────────────────────────────────┤
│ down(s)    → SE s > 0: decrementa e continua                    │
│              SE s == 0: dorme (aguarda up)                      │
│ up(s)      → SE há processo dormindo: acorda um                 │
│              SE não há: incrementa s (crédito guardado) ✅       │
│                                                                  │
│ Estado: contador inteiro (0, 1, 2, 3, ...)                      │
│ Solução: up antes do down → crédito acumulado, down não dorme   │
└──────────────────────────────────────────────────────────────────┘
```

### A atomicidade é essencial

As operações `down` e `up` precisam ser **atômicas** — verificar o valor, modificá-lo e possivelmente dormir devem acontecer como uma única operação indivisível.

```
Por que? Imagine down(s) não sendo atômico:

1. Processo A verifica s → vê s = 1 (há permissão!)
   ← INTERRUPÇÃO →
2. Processo B também verifica s → vê s = 1
3. Processo B decrementa → s = 0, entra na região crítica
   ← PROCESSO A RETOMA →
4. Processo A decrementa → s = -1 ← IMPOSSÍVEL!
   Processo A também entra na região crítica
   → DOIS PROCESSOS SIMULTANEAMENTE → race condition
```

É garantido que, **uma vez que uma operação de semáforo tenha começado, nenhum outro processo pode acessar o semáforo até que ela seja concluída ou o processo seja bloqueado**. A implementação usa brevemente a desabilitação de interrupções (no SO) ou TSL/XCHG (em multiprocessadores) para garantir isso.

> ⚠️ **Certificar-se de que você compreendeu:** usar TSL ou XCHG para proteger o acesso ao semáforo em si é bastante diferente do produtor ou consumidor em espera ocupada esperando o buffer esvaziar ou encher. A operação de semáforo levará apenas alguns microssegundos, enquanto o produtor ou consumidor podem levar tempos arbitrariamente longos. O uso de TSL aqui é mínimo e justificado.

---

## 🏭 Resolvendo o produtor-consumidor com semáforos

Agora aplicamos semáforos para resolver definitivamente o problema do produtor-consumidor que falhava com `sleep`/`wakeup`.

A solução usa **três semáforos**:

```
mutex = 1    → controla acesso à região crítica (ao buffer e variáveis)
              Garante que produtor e consumidor não acessem o buffer ao mesmo tempo
              
empty = N    → conta o número de posições VAZIAS no buffer
              Inicialmente N (buffer completamente vazio)
              
full  = 0    → conta o número de posições CHEIAS no buffer
              Inicialmente 0 (buffer vazio, nenhum item disponível)
```

> 📌 **Figura 2.28 — O problema do produtor-consumidor usando semáforos**

```c
#define N 100
typedef int semaphore;        /* semáforos são um tipo especial de int */
semaphore mutex = 1;          /* controla o acesso à região crítica */
semaphore empty = N;          /* conta os lugares vazios no buffer */
semaphore full  = 0;          /* conta os lugares preenchidos no buffer */

void producer(void)
{
    int item;
    while (TRUE) {
        item = produce_item();  /* gera algo para colocar no buffer */
        down(&empty);           /* decrementa contador de posições vazias */
        down(&mutex);           /* entra na região crítica */
        insert_item(item);      /* coloca novo item no buffer */
        up(&mutex);             /* sai da região crítica */
        up(&full);              /* incrementa a contagem de posições ocupadas */
    }
}

void consumer(void)
{
    int item;
    while (TRUE) {
        down(&full);            /* decrementa a contagem de posições ocupadas */
        down(&mutex);           /* entra na região crítica */
        item = remove_item();   /* pega item do buffer */
        up(&mutex);             /* sai da região crítica */
        up(&empty);             /* incrementa a contagem de posições vazias */
        consume_item(item);     /* faz algo com o item */
    }
}
```

### Rastreando o que cada semáforo faz — passo a passo

**Cenário: buffer vazio, produtor produz o primeiro item**

```
Estado inicial: mutex=1, empty=N, full=0

PRODUTOR:
  produce_item()        → item gerado
  down(&empty)          → empty era N → empty = N-1 ✅ (havia vaga)
  down(&mutex)          → mutex era 1 → mutex = 0 ✅ (entra na RC)
  insert_item(item)     → insere no buffer
  up(&mutex)            → mutex = 1 ✅ (sai da RC)
  up(&full)             → full era 0, ninguém dormindo → full = 1 ✅

CONSUMIDOR (antes havia tentado):
  down(&full)           → full era 0 → DORME 😴
  
  (produtor fez up(&full) → acorda o consumidor)
  
  down(&full) completa  → full = 0 ✅
  down(&mutex)          → mutex = 1 → mutex = 0 ✅ (entra na RC)
  remove_item()         → retira do buffer
  up(&mutex)            → mutex = 1 ✅
  up(&empty)            → empty = N ✅
  consume_item()        → usa o item
```

**Cenário: buffer cheio, produtor tenta inserir mais**

```
Estado: mutex=1, empty=0, full=N

PRODUTOR:
  produce_item()        → item gerado
  down(&empty)          → empty é 0 → PRODUTOR DORME 😴
  
  (consumidor retira um item e faz up(&empty))
  
  down(&empty) completa → empty = 0 ✅
  continua normalmente...
```

### Por que a ordem dos downs importa — armadilha de deadlock

> ⚠️ **ATENÇÃO — ordem dos downs é crítica:**

```
Se o produtor fizesse down(&mutex) ANTES de down(&empty):

  down(&mutex)   → mutex = 0 (entra na RC)
  down(&empty)   → empty é 0 → PRODUTOR DORME dentro da RC!
  
  Consumidor tenta:
  down(&full)    → full é 0 → CONSUMIDOR DORME
  
  Para continuar, consumidor precisaria de mutex.
  Para liberar mutex, produtor precisaria ser acordado.
  Para acordar produtor, consumidor precisaria completar down(&empty).
  Para completar down(&empty), consumidor precisaria entrar na RC.
  Para entrar na RC, consumidor precisaria de mutex... que o produtor tem.
  
  DEADLOCK ← ambos dormindo para sempre
```

**A regra:** sempre faça `down` no semáforo de recurso (`empty`/`full`) **antes** do `down` no semáforo de exclusão mútua (`mutex`). Nunca tente entrar na região crítica antes de verificar se há recurso disponível.

### Os dois usos dos semáforos — exclusão mútua vs. sincronização

Este exemplo ilustra que semáforos são usados de **duas formas distintas**:

```
┌─────────────────────────────────────────────────────────────────┐
│  USO 1 — EXCLUSÃO MÚTUA (mutex)                                │
│                                                                 │
│  Semáforo inicializado em 1                                    │
│  Garante que apenas 1 processo por vez acessa a RC             │
│  down() antes de entrar, up() ao sair                          │
│  Funcionam como uma "porta com chave única"                    │
│                                                                 │
│  mutex=1: porta aberta → qualquer processo pode entrar         │
│  mutex=0: porta trancada → próximo processo dorme              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  USO 2 — SINCRONIZAÇÃO (full/empty)                            │
│                                                                 │
│  Semáforos inicializados conforme o estado do recurso          │
│  Garantem que determinadas sequências ocorram ou não           │
│  Asseguram que produtor pare quando buffer cheio               │
│  Asseguram que consumidor pare quando buffer vazio             │
│                                                                 │
│  full=0: nenhum item disponível → consumidor dorme             │
│  full=5: 5 itens disponíveis → consumidor pode retirar 5x     │
└─────────────────────────────────────────────────────────────────┘
```

> 💡 **Semáforos binários:** semáforos que são inicializados para 1 e usados por dois ou mais processos para assegurar que apenas um deles consiga entrar em sua região crítica de cada vez. Se cada processo realiza um `down` um pouco antes de entrar em sua região crítica e um `up` logo depois de deixá-la, a exclusão mútua é garantida.

---

## 📚 O problema dos leitores e escritores

Outro problema clássico que semáforos resolvem elegantemente. Modela o acesso a um banco de dados (ex: sistema de reservas de passagens aéreas):

**As regras:**
- Múltiplos leitores podem ler simultaneamente — não há conflito entre leitura e leitura
- Um escritor precisa de **acesso exclusivo** — nenhum outro processo (leitor ou escritor) pode estar ativo enquanto ele escreve

> 📌 **Figura 2.29 — Solução para o problema dos leitores e escritores**

```c
typedef int semaphore;
semaphore mutex = 1;    /* controla acesso ao contador rc */
semaphore db    = 1;    /* controla acesso ao banco de dados */
int rc = 0;             /* número de processos lendo ou querendo ler */

void reader(void)
{
    while (TRUE) {
        down(&mutex);           /* obtem acesso exclusivo a rc */
        rc = rc + 1;            /* um leitor a mais agora */
        if (rc == 1) down(&db); /* se é o primeiro leitor, trava o banco */
        up(&mutex);             /* libera acesso exclusivo a rc */
        read_data_base();       /* acessa os dados */
        down(&mutex);           /* obtem acesso exclusivo a rc */
        rc = rc - 1;            /* um leitor a menos agora */
        if (rc == 0) up(&db);   /* se é o último leitor, libera o banco */
        up(&mutex);             /* libera acesso exclusivo a rc */
        use_data_read();        /* região não crítica */
    }
}

void writer(void)
{
    while (TRUE) {
        think_up_data();        /* região não crítica */
        down(&db);              /* obtém acesso exclusivo */
        write_data_base();      /* atualiza os dados */
        up(&db);                /* libera acesso exclusivo */
    }
}
```

### Como funciona

```
LEITORES:
  rc (read count) conta quantos leitores estão ativos
  
  Primeiro leitor a chegar:
    rc = 0 → 1
    rc == 1? SIM → faz down(&db) → trava o banco para escritores
    Lê normalmente
    
  Leitores subsequentes:
    incrementam rc → não fazem down(&db) de novo (já está travado)
    Todos leem simultaneamente ✅
    
  Último leitor a sair:
    rc = 1 → 0
    rc == 0? SIM → faz up(&db) → libera o banco para escritores

ESCRITORES:
  down(&db) → se rc > 0 (há leitores), dorme
  Quando todos os leitores saírem → up(&db) acorda o escritor
  Escritor tem acesso exclusivo total
  up(&db) ao terminar → próximo leitor ou escritor pode entrar
```

**O problema desta solução — starvation dos escritores:**

```
Escritor quer escrever. Espera todos os leitores saírem.
Novo leitor chega → rc aumenta → escritor não pode entrar.
Outro leitor chega → e outro → e outro...
Com fluxo constante de leitores, escritor NUNCA entra.

Isso é starvation (inanição) — um processo esperando indefinidamente.
```

> 💡 **Starvation (inanição):** situação em que um processo nunca consegue acesso ao recurso que precisa, porque outros processos continuam chegando e sempre têm prioridade. Viola a **condição 4** das regiões críticas (nenhum processo deve esperar eternamente).

Uma solução alternativa dá prioridade aos escritores: quando um escritor chega, nenhum novo leitor é admitido. Leitores já ativos terminam, e o escritor entra. A desvantagem é menor simultaneidade (leitores que chegam enquanto um escritor espera ficam bloqueados mesmo que não haja conflito com outros leitores).

---

# 🔒 2.4.6 — Mutexes

## O que é um mutex

Quando a capacidade de **contagem** do semáforo não é necessária — quando você só quer garantir exclusão mútua simples — uma versão simplificada chamada **mutex** é usada.

> 💡 **Mutex (*mutual exclusion*):** variável compartilhada que pode estar em apenas **dois estados**: travado (*locked*) ou destravado (*unlocked*). Diferente do semáforo, não conta — apenas representa "ocupado" ou "livre". É basicamente um semáforo binário, mas mais simples e eficiente de implementar.

```
mutex = 0  → DESTRAVADO (região crítica livre, pode entrar)
mutex = 1  → TRAVADO    (região crítica ocupada, deve esperar)
```

Apenas 1 bit é necessário para representar um mutex — mas na prática um inteiro é usado, com 0 significando destravado e todos os outros valores significando travado.

## As operações: `mutex_lock` e `mutex_unlock`

Dois procedimentos são usados com mutexes:

**`mutex_lock`** — chamado antes de entrar na região crítica:

```asm
mutex_lock:
    TSL REGISTER, MUTEX    | copia mutex para registrador, define mutex = 1
    CMP REGISTER, #0       | mutex valia zero (estava destravado)?
    JZE ok                 | se sim: estava livre → entra na RC
    CALL thread_yield      | se não: estava travado → CEDE a CPU para outra thread
    JMP mutex_lock         | tenta novamente depois
ok: RET                    | entra na região crítica
```

**`mutex_unlock`** — chamado ao sair da região crítica:

```asm
mutex_unlock:
    MOVE MUTEX, #0         | coloca 0 em mutex (destrava)
    RET
```

> 💡 **`thread_yield`:** chamada que permite que a thread ceda voluntariamente a CPU para outra thread. Diferente do `sleep` (que bloqueia até ser acordado), `yield` apenas reescalona — a thread continua na fila de prontos e tenta novamente logo. Em espaço de usuário, onde não há `sleep` real, é a alternativa à espera ocupada.

## Mutex vs. TSL diretamente — qual a diferença?

A implementação do mutex **também usa TSL/XCHG** internamente. Então por que é diferente das soluções de espera ocupada que criticamos?

```
┌─────────────────────────────────────────────────────────────────┐
│  TSL diretamente (espera ocupada pura)                         │
│                                                                 │
│  while (TSL(lock) != 0) { }  ← gira sem parar                │
│  CPU fica 100% ocupada                                         │
│  Processo H pode bloquear processo L indefinidamente          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  mutex_lock com thread_yield                                   │
│                                                                 │
│  TSL testa → travado → thread_yield() → outra thread executa  │
│  CPU é cedida a quem pode fazer trabalho útil                  │
│  L pode executar, liberar o mutex → H eventualmente entra     │
└─────────────────────────────────────────────────────────────────┘
```

A diferença chave: em vez de girar em falso consumindo CPU, o mutex chama `thread_yield` — **cede a CPU** para que outra thread (possivelmente aquela que detém o mutex) possa executar e liberar o recurso.

## Por que mutexes são especialmente úteis em user space

Mutexes são **fáceis e eficientes de implementar no espaço do usuário**, desde que uma instrução TSL ou XCHG esteja disponível. Isso os torna especialmente úteis em **pacotes de threads implementados inteiramente no espaço do usuário** — onde não há acesso a syscalls do kernel para sleep/wakeup.

```
Vantagens do mutex em user space:
  ✅ Não exige syscall para travar/destravar (exceto quando há contenção)
  ✅ Implementável com poucas instruções de assembly
  ✅ Muito eficiente quando a contenção é rara
  ✅ Sem overhead de chamada ao kernel no caso comum (sem contenção)
```

## Mutex vs. Semáforo — quando usar cada um

```
┌──────────────────┬──────────────────────┬──────────────────────────┐
│  Característica  │       Mutex          │       Semáforo           │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Estados          │ 2: travado/destravado│ N: contador inteiro      │
│ Uso principal    │ Exclusão mútua simples│ Exclusão mútua E/OU     │
│                  │                      │ sincronização            │
│ Inicialização    │ 0 (destravado)       │ 0, 1, N conforme o caso  │
│ "Dono"           │ Thread que travou    │ Qualquer processo pode   │
│                  │ deve destravar       │ fazer up/down            │
│ Contagem         │ Não (binário)        │ Sim (conta recursos)     │
│ Típico para      │ Proteger uma variável│ Produt/consumidor,       │
│                  │ ou estrutura         │ leitores/escritores      │
└──────────────────┴──────────────────────┴──────────────────────────┘
```

---

## 🔗 A conexão com interrupções — semáforos no kernel

Os semáforos também são naturais para **esconder interrupções** dentro do SO. A ideia (referenciada na Figura 2.5 do livro):

```
Para cada dispositivo de E/S:
  Semáforo associado inicializado em 0

Processo de gerenciamento inicia operação de E/S:
  down(&sem_dispositivo)  → bloqueia imediatamente (sem = 0)

Interrupção chega (operação concluída):
  Tratador de interrupção faz up(&sem_dispositivo)
  → processo de gerenciamento acorda e continua

No passo 5 da Figura 2.5:
  up() no semáforo do dispositivo
  → no passo 6, escalonador executa o gerenciador do dispositivo
```

Isso permite que o SO modele a espera por hardware de forma elegante, sem polling ou espera ocupada.

---

# ✅ Resumo do Conceito

**Semáforos (`down`/`up`):**
- Variável inteira que conta permissões disponíveis
- `down`: se > 0, decrementa e continua; se = 0, bloqueia
- `up`: se há processo dormindo, acorda um; senão, incrementa
- **Diferença fundamental de `sleep`/`wakeup`:** créditos não são perdidos — `up` antes de `down` incrementa o contador e o processo não dorme
- Operações são **atômicas** — implementadas com desabilitação de interrupções ou TSL/XCHG internamente
- Dois usos: **exclusão mútua** (inicializado em 1, semáforo binário) e **sincronização** (inicializado em 0 ou N)
- No produtor-consumidor: três semáforos (`mutex`, `empty`, `full`) — ordem dos `down` importa para evitar deadlock
- No problema leitores/escritores: `rc` conta leitores ativos; primeiro leitor trava banco, último destrava

**Mutexes:**
- Versão simplificada do semáforo: apenas dois estados (travado/destravado)
- `mutex_lock` usa TSL internamente, mas chama `thread_yield` se travado — cede CPU em vez de girar
- Especialmente úteis em pacotes de threads em user space: eficientes, sem syscall no caso comum
- Diferença do TSL puro: não desperdiça CPU ao chamar `yield` quando há contenção

---

## 🔗 Notas Relacionadas

- [[Dormir e Despertar]] — as primitivas `sleep`/`wakeup` que semáforos substituem e melhoram
- [[Exclusão mútua com espera ocupada]] — TSL/XCHG usados internamente nas operações de semáforo e mutex
- [[Regiões Críticas]] — as 4 condições que semáforos satisfazem (starvation de escritores viola a condição 4)
- [[Race Condition]] — semáforos são a solução robusta para as races vistas no spool e no produtor-consumidor
- [[Estados de Processos]] — `down` com semáforo = 0 move processo para estado bloqueado; `up` move de bloqueado para pronto
