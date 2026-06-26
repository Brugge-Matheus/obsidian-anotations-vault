---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.6"
---
# Mutexes

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.6

---

# 🔒 2.4.6 — Mutexes

## O que é um mutex e por que existe

Em [[Semáforos]], vimos que semáforos têm capacidade de **contagem** — eles acumulam créditos e podem controlar o acesso a N recursos simultâneos. Mas muitas vezes você só precisa de uma coisa: garantir que **apenas um processo por vez** acesse uma região crítica. Nesse caso, a capacidade de contagem do semáforo é desnecessária.

Para esse caso mais simples e comum, existe uma versão simplificada chamada **mutex** (*mutual exclusion*).

> 💡 **Mutex:** variável compartilhada que pode estar em apenas **dois estados**: travado (*locked*) ou destravado (*unlocked*). É basicamente um semáforo binário inicializado em 1, mas com semântica mais simples e implementação mais eficiente. Mutexes são bons somente para gerenciar a exclusão mútua de algum recurso compartilhado ou trecho de código.

```
mutex = 0  → DESTRAVADO — região crítica livre, qualquer thread pode entrar
mutex = 1  → TRAVADO    — região crítica ocupada, próxima thread deve esperar
```

Apenas 1 bit é necessário para representar o estado, mas na prática um inteiro é usado — 0 significa destravado, qualquer outro valor significa travado.

**Por que mutexes são especialmente úteis em user space:** eles são fáceis e eficientes de implementar no espaço do usuário desde que uma instrução TSL ou XCHG esteja disponível. Isso os torna ideais para pacotes de threads implementados inteiramente no espaço do usuário — onde não há acesso direto a syscalls do kernel para sleep/wakeup.

---

## ⚙️ As operações: `mutex_lock` e `mutex_unlock`

### `mutex_lock` — adquirindo a trava

> 📌 **Figura 2.30 — Implementação de mutex_lock e mutex_unlock**

```asm
mutex_lock:
    TSL REGISTER, MUTEX    | copia mutex para registrador e define mutex = 1
    CMP REGISTER, #0       | mutex valia zero (estava destravado)?
    JZE ok                 | se era zero: mutex estava desimpedido → entra na RC
    CALL thread_yield      | mutex está ocupado → escalone outra thread
    JMP mutex_lock         | tente novamente
ok: RET                    | retorna a quem chamou; entrou na região crítica

mutex_unlock:
    MOVE MUTEX, #0         | coloca 0 em mutex
    RET                    | retorna a quem chamou
```

**O que acontece passo a passo:**

```
Thread A quer entrar na RC:
  TSL: lê mutex (=0) e define mutex=1 atomicamente
  CMP: valor lido era 0? SIM
  JZE ok → RET → entra na RC ✅

Thread B quer entrar na RC (enquanto A está dentro):
  TSL: lê mutex (=1) e define mutex=1 (já era 1)
  CMP: valor lido era 0? NÃO
  CALL thread_yield → cede a CPU para outra thread
  JMP mutex_lock → tenta novamente na próxima vez que for escalonada

Thread A termina:
  mutex_unlock: MOVE MUTEX, #0 → mutex = 0

Thread B tenta novamente:
  TSL: lê mutex (=0) e define mutex=1 atomicamente
  CMP: valor lido era 0? SIM
  JZE ok → RET → entra na RC ✅
```

### A diferença crucial: `mutex_lock` vs. `enter_region` (TSL puro)

O código de `mutex_lock` é semelhante ao `enter_region` da [[Exclusão mútua com espera ocupada]] (Figura 2.25), mas com **uma diferença fundamental**:

```
┌──────────────────────────────────────────────────────────────────┐
│  enter_region (TSL puro — espera ocupada)                       │
├──────────────────────────────────────────────────────────────────┤
│  Quando trava está ocupada:                                     │
│    JNE enter_region ← volta imediatamente ao início             │
│    CPU fica 100% ocupada testando a trava em loop               │
│    Thread que tem a trava nunca recebe CPU para terminar        │
│    → laço infinito em threads de usuário                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  mutex_lock (com thread_yield)                                  │
├──────────────────────────────────────────────────────────────────┤
│  Quando trava está ocupada:                                     │
│    CALL thread_yield ← cede a CPU para outra thread             │
│    Outra thread executa, possivelmente a que tem a trava        │
│    Essa thread termina e libera o mutex                         │
│    mutex_lock tenta novamente e consegue entrar                 │
│    → sem espera ocupada, sem laço infinito                      │
└──────────────────────────────────────────────────────────────────┘
```

**Por que `enter_region` causaria laço infinito em threads de usuário:** com threads de usuário, não há um relógio de hardware que interrompa threads que executaram por tempo demais. Se uma thread entra em espera ocupada, ela **nunca cede a CPU** — e a thread que tem a trava jamais executa para liberá-la. O sistema trava.

> 💡 **`thread_yield`:** chamada que permite à thread ceder voluntariamente a CPU para outra thread. Diferente do `sleep` (que bloqueia até ser acordado por outro processo), `yield` apenas reescalona — a thread continua na fila de prontos e tenta novamente quando for escalonada. Como `thread_yield` e `mutex_unlock` são chamadas para o escalonador de espaço do usuário, **nenhuma chamada de núcleo é necessária** — threads de usuário sincronizam inteiramente no espaço do usuário com apenas um punhado de instruções.

### `mutex_trylock` — tentativa não bloqueante

O sistema mutex mínimo descrito acima pode ser estendido. Uma variação útil é o **`mutex_trylock`**:

> 💡 **`mutex_trylock`:** tenta adquirir a trava e retorna imediatamente com um **código de erro** em vez de bloquear, se o mutex já estiver travado. Dá à thread a flexibilidade de decidir o que fazer em seguida — tentar outro recurso, fazer outro trabalho, ou fazer espera ocupada por um tempo curto — em vez de simplesmente esperar.

---

## 🔗 A questão da memória compartilhada entre processos

Até aqui assumimos que threads compartilham memória — o que é verdade por definição. Mas para a maioria das soluções anteriores (Peterson, semáforos), há uma **suposição velada**: os processos têm acesso a pelo menos um pouco de memória compartilhada (a variável `turn`, o buffer, os semáforos).

Se os processos têm **espaços de endereços disjuntos**, como eles podem compartilhar essa memória?

Há duas respostas:

1. **Semáforos no núcleo:** as estruturas de dados compartilhadas, como os semáforos, podem ser armazenadas no núcleo e acessadas somente por chamadas de sistema. Isso elimina o problema — o núcleo é o intermediário neutro.

2. **Memória compartilhada explícita:** a maioria dos SOs modernos (incluindo UNIX e Windows) oferece uma maneira para os processos compartilharem parte do seu espaço de endereços com outros processos. Buffers e outras estruturas podem ser colocadas nessa área compartilhada. Na pior das hipóteses, um arquivo compartilhado pode ser usado.

> ⚠️ **A distinção processo/thread se torna confusa** quando dois processos compartilham a maior parte ou todo o seu espaço de endereços. Dois processos com espaço de endereços comum ainda têm arquivos abertos diferentes, temporizadores de alarme e outras propriedades por processo — enquanto as threads dentro de um único processo as compartilham.

---

## ⚡ Futexes — o melhor dos dois mundos

Com o paralelismo cada vez maior, a sincronização eficiente é crítica para o desempenho. Há um trade-off claro entre as abordagens que conhecemos:

```
Trava giratória (spin lock):
  ✅ Rápida quando a espera é curta (evita syscall)
  ❌ Desperdiça ciclos de CPU se a espera for longa

Bloquear no núcleo (semáforo/sleep):
  ✅ Eficiente quando há disputa pesada
  ❌ Cara quando a disputa é baixa (troca para núcleo e volta = overhead)
```

A solução que combina o melhor dos dois mundos é o **futex** (*fast userspace mutex* — mutex rápido de espaço do usuário).

> 💡 **Futex:** recurso do Linux que implementa travamento básico (como um mutex), mas **evita adentrar o núcleo a não ser que realmente precise**. É uma construção com suporte do núcleo para permitir que processos do espaço do usuário sejam sincronizados nos eventos compartilhados. Consiste em duas partes: um serviço de núcleo (fila de espera) e uma biblioteca de usuário (lógica de travamento atômica).

### Como o futex funciona

```
VARIÁVEL DE TRAVA COMPARTILHADA: inicialmente –1 (trava livre)

ADQUIRIR A TRAVA:
  Thread faz "decremento e teste" atômico na variável:
  
  Resultado = –1 → 0?  (era –1, agora 0)
    → Trava adquirida com sucesso ✅
    → ZERO chamadas de sistema — tudo no espaço do usuário!
    
  Resultado = 0 → –1?  (era 0, já estava travada)
    → Outra thread tem a trava
    → Biblioteca futex faz syscall para colocar esta thread
      na FILA DE ESPERA do núcleo (blocking real)
    → Thread dorme no núcleo até ser desbloqueada

LIBERAR A TRAVA:
  Thread faz "incremento e teste" atômico na variável:
  
  Resultado = –1?  (voltou a –1, ninguém estava esperando)
    → Trava liberada ✅
    → ZERO chamadas de sistema!
    
  Resultado != –1?  (havia alguém na fila de espera)
    → Biblioteca futex faz syscall para DESPERTAR um processo da fila
    → Núcleo desbloqueia a thread esperando
```

**O insight chave:** na ausência de disputa (o caso mais comum em código bem escrito), o futex funciona **completamente no espaço do usuário** — zero syscalls, zero overhead de troca para o núcleo. Apenas quando há disputa real o núcleo é envolvido.

> ⚠️ **Futexes são de baixo nível.** A maioria dos usuários nunca os usa diretamente — eles estão embutidos nas bibliotecas padrão, que oferecem primitivas de mais alto nível (como `pthread_mutex_t`). É somente quando levantamos o capô que conseguimos ver o mecanismo futex controlando muitos tipos diferentes de sincronização.

---

## 🧵 Mutexes em Pthreads

Pthreads fornece um conjunto de funções para sincronizar threads. O mecanismo básico usa uma variável **mutex** que pode ser travada ou destravada, para guardar cada região crítica.

No Linux, a implementação de mutex de Pthreads é montada em cima de **futexes**.

### As chamadas principais (Figura 2.31)

> 📌 **Figura 2.31 — Algumas chamadas de Pthreads relacionadas a mutexes**

| Chamada de thread | Descrição |
|---|---|
| `pthread_mutex_init` | Cria um mutex |
| `pthread_mutex_destroy` | Destrói um mutex existente |
| `pthread_mutex_lock` | Obtém uma trava ou é bloqueado |
| `pthread_mutex_trylock` | Obtém uma trava ou falha (não bloqueia) |
| `pthread_mutex_unlock` | Libera uma trava |

**Como funcionam:**

```
Thread quer entrar na RC:
  pthread_mutex_lock(&mutex)
    → se mutex destravado: trava atomicamente, continua ✅
    → se mutex travado: thread é BLOQUEADA até ser destravada

Thread sai da RC:
  pthread_mutex_unlock(&mutex)
    → destrava o mutex
    → se há threads esperando: uma delas é autorizada a continuar
      e trava o mutex novamente
```

> ⚠️ **Travas não são obrigatórias** — cabe ao programador assegurar que as threads as usem corretamente. Se uma thread trapacear e entrar na região crítica sem travar o mutex, a exclusão mútua falha. Mutexes funcionam somente se os processos cooperam.

---

## 🎯 Variáveis de Condição

Além dos mutexes, Pthreads oferece um segundo mecanismo de sincronização: as **variáveis de condição**.

> 💡 **Variável de condição:** mecanismo que permite que threads sejam bloqueadas **não por falta de acesso à região crítica**, mas pelo **não atendimento de alguma condição lógica**. Enquanto mutexes controlam *quem* pode entrar na RC, variáveis de condição controlam *quando* faz sentido entrar.

Mutexes e variáveis de condição são **quase sempre usados juntos**:
- O mutex garante acesso exclusivo aos dados
- A variável de condição permite esperar que os dados atinjam um certo estado

**A analogia:** o mutex é a chave da sala. A variável de condição é a resposta à pergunta "tem algo útil pra mim fazer aqui dentro?". Se não tiver (buffer vazio, buffer cheio), a thread sai da sala, devolve a chave, e dorme esperando ser avisada.

### As chamadas principais (Figura 2.32)

> 📌 **Figura 2.32 — Algumas das chamadas de Pthreads relacionadas com variáveis de condição**

| Chamada de thread | Descrição |
|---|---|
| `pthread_cond_init` | Cria uma variável de condição |
| `pthread_cond_destroy` | Destrói uma variável de condição |
| `pthread_cond_wait` | Bloqueia esperando por um sinal |
| `pthread_cond_signal` | Sinaliza para outra thread e a desperta |
| `pthread_cond_broadcast` | Sinaliza para múltiplas threads e desperta todas |

### Como `pthread_cond_wait` funciona — o detalhe crítico

```
pthread_cond_wait(&cond, &mutex):
  1. DESTRAVA o mutex atomicamente  ← cede o acesso à RC
  2. Coloca a thread para DORMIR    ← bloqueia esperando o sinal
  3. (quando outra thread faz signal)
  4. TRAVA o mutex novamente        ← readquire acesso à RC
  5. Retorna                        ← thread continua de onde parou
```

**Por que destrava o mutex atomicamente com o sleep?** Se destravasse e depois dormisse como duas operações separadas, haveria uma race condition: outra thread poderia fazer `signal` entre o destravamento e o sleep — e o sinal seria perdido (exatamente o problema do [[Dormir e Despertar]]). A operação atômica elimina essa janela.

> ⚠️ **Variáveis de condição não têm memória** — diferente de semáforos. Se um sinal é enviado a uma variável de condição quando não há nenhuma thread esperando, **o sinal é perdido**. Programadores têm de ser cuidadosos para não perder sinais. Sempre verificar a condição em um loop `while` (não `if`) ao retornar de `pthread_cond_wait`, pois a thread pode ter sido despertada por um sinal UNIX ou outra razão espúria.

### `pthread_cond_broadcast` vs. `pthread_cond_signal`

```
pthread_cond_signal(&cond)
  → acorda UMA thread esperando (escolhida pelo sistema)
  → usar quando apenas uma thread pode prosseguir

pthread_cond_broadcast(&cond)
  → acorda TODAS as threads esperando nesta variável de condição
  → usar quando múltiplas threads podem prosseguir após a condição
```

---

## 🏭 Produtor-consumidor com mutex e variável de condição (Figura 2.33)

O exemplo final combina tudo — mutex para exclusão mútua e variável de condição para sincronização de estado.

> 📌 **Figura 2.33 — Problema produtor-consumidor com mutex e variável de condição**

```c
#include <stdio.h>
#include <pthread.h>
#define MAX 1000000000        /* quantos números produzir */

pthread_mutex_t the_mutex;
pthread_cond_t condc, condp;  /* usado para sincronização */
int buffer = 0;               /* buffer usado entre produtor e consumidor */

void *producer(void *ptr)     /* produz dados */
{
    int i;
    for (i = 1; i <= MAX; i++) {
        pthread_mutex_lock(&the_mutex);    /* obtém acesso exclusivo ao buffer */
        while (buffer != 0)
            pthread_cond_wait(&condp, &the_mutex); /* espera buffer esvaziar */
        buffer = i;                        /* coloca item no buffer */
        pthread_cond_signal(&condc);       /* desperta o consumidor */
        pthread_mutex_unlock(&the_mutex);  /* libera acesso ao buffer */
    }
    pthread_exit(0);
}

void *consumer(void *ptr)     /* consome os dados */
{
    int i;
    for (i = 1; i <= MAX; i++) {
        pthread_mutex_lock(&the_mutex);    /* obtém acesso exclusivo ao buffer */
        while (buffer == 0)
            pthread_cond_wait(&condc, &the_mutex); /* espera buffer encher */
        buffer = 0;                        /* retira o item do buffer (não mostrado) */
        pthread_cond_signal(&condp);       /* desperta o produtor */
        pthread_mutex_unlock(&the_mutex);  /* libera acesso ao buffer */
    }
    pthread_exit(0);
}

int main(int argc, char **argv)
{
    pthread_t pro, con;
    pthread_mutex_init(&the_mutex, 0);
    pthread_cond_init(&condc, 0);
    pthread_cond_init(&condp, 0);
    pthread_create(&con, 0, consumer, 0);
    pthread_create(&pro, 0, producer, 0);
    pthread_join(pro, 0);
    pthread_join(con, 0);
    pthread_cond_destroy(&condc);
    pthread_cond_destroy(&condp);
    pthread_mutex_destroy(&the_mutex);
}
```

### Rastreando a lógica

```
PRODUTOR:
  lock(mutex)               → entra na RC
  while (buffer != 0)       → buffer ainda cheio?
    cond_wait(&condp, mutex)→ dorme E destrava mutex atomicamente
                              (consumidor pode entrar agora)
                            → acorda quando consumidor sinaliza condp
  buffer = i                → coloca o item
  signal(&condc)            → acorda consumidor que pode estar esperando
  unlock(mutex)             → sai da RC

CONSUMIDOR (simétrico):
  lock(mutex)               → entra na RC
  while (buffer == 0)       → buffer ainda vazio?
    cond_wait(&condc, mutex)→ dorme E destrava mutex atomicamente
                              (produtor pode entrar agora)
  buffer = 0                → retira o item (reinicializa)
  signal(&condp)            → acorda produtor que pode estar esperando
  unlock(mutex)             → sai da RC
```

**Por que `while` e não `if`?** Ao retornar de `pthread_cond_wait`, a thread deve verificar a condição novamente. Ela pode ter sido despertada por um sinal UNIX espúrio, ou outra thread pode ter consumido o recurso entre o sinal e a execução desta thread. O `while` garante que a condição é realmente satisfeita antes de continuar.

---

# ✅ Resumo do Conceito

- **Mutex** é uma variável binária (travado/destravado) para exclusão mútua simples — mais leve que um semáforo quando contagem não é necessária
- `mutex_lock` usa TSL internamente, mas chama **`thread_yield`** se travado — cede a CPU em vez de girar, evitando o laço infinito que `enter_region` causaria em threads de usuário
- `mutex_trylock` tenta adquirir sem bloquear — retorna código de erro se ocupado
- **Futex** (*fast userspace mutex*) combina o melhor dos dois mundos: opera inteiramente no espaço do usuário quando não há disputa (zero syscalls), e só envolve o núcleo quando realmente há contenção
- **Mutexes em Pthreads:** `pthread_mutex_lock/unlock/trylock` — implementados sobre futexes no Linux; travas não são obrigatórias, exigem cooperação
- **Variáveis de condição** complementam mutexes — permitem bloquear não por falta de acesso, mas por não atendimento de uma condição lógica; `pthread_cond_wait` destrava o mutex e dorme **atomicamente** para evitar perda de sinal
- `pthread_cond_signal` acorda uma thread; `pthread_cond_broadcast` acorda todas
- Variáveis de condição **não têm memória** — sinal enviado sem ninguém esperando é perdido; sempre use `while` ao retornar de `wait`, nunca `if`
- O padrão canônico é: **lock → while(!condição) → wait → trabalho → signal → unlock**

---

## 🔗 Notas Relacionadas

- [[Semáforos]] — semáforos são mais poderosos (contagem), mas mutexes são mais simples e eficientes para exclusão mútua pura
- [[Exclusão mútua com espera ocupada]] — `enter_region` com TSL que `mutex_lock` melhora com `thread_yield`
- [[Dormir e Despertar]] — `pthread_cond_wait` resolve o mesmo problema de sinal perdido que motivou os semáforos, mas para threads Pthreads
- [[Threads POSIX]] — contexto completo da API Pthreads onde mutexes e variáveis de condição vivem
- [[Implementando Threads em User Space]] — por que `thread_yield` é necessário em user space e por que `enter_region` causaria deadlock
