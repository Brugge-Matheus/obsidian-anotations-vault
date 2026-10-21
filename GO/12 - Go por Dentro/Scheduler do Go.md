---
tags:
  - go
  - go/runtime
aliases:
  - Escalonador do Go
  - Modelo G-M-P
  - GMP
  - Work stealing
  - sysmon
  - Preempção assíncrona
  - LockOSThread
ordem: 51
modulo: "12 - Go por Dentro"
---
# Scheduler do Go

> **Nota 51 da trilha de Go.** O escalonador é a peça do runtime que mais se parece com um sistema operacional: ele decide **qual goroutine roda em qual thread, e por quanto tempo**, exatamente como o escalonador do kernel decide sobre processos e threads no capítulo 2 do Tanenbaum. Nesta nota veremos o modelo **G-M-P** e por que o "P" existe, o laço do escalonador e a ordem em que ele procura trabalho (com o curioso `runnext`), o **roubo de trabalho**, o que acontece quando uma goroutine bloqueia em channel, rede ou **syscall** (com contagem real de threads), a **preempção** cooperativa e assíncrona, a thread de monitoramento **`sysmon`**, `LockOSThread`, como ler o `schedtrace`, e a comparação com os escalonadores que você estudou.
>
> **Pré-requisitos:** [[Goroutines]], [[Sockets e o Pacote net]], [[Introdução ao Escalonamento]].

---

## 🧭 De onde viemos

Em [[Goroutines]] tivemos uma visão geral do G-M-P, da preempção por `SIGURG` e do `GOMAXPROCS`; em [[Fundamentos de Concorrência]], o mapa SO ↔ Go; em [[Sockets e o Pacote net]], o netpoller visto pelo `strace`; em [[Leitura e Escrita de Arquivos]], a diferença entre E/S de arquivo (que prende threads) e de rede (que não prende). Em [[Profiling e Diagnóstico]] prometemos explicar o `schedtrace`, e em [[Memória e Escape Analysis]] vimos que cada P tem seu próprio cache de memória. No Tanenbaum, você estudou [[Introdução ao Escalonamento]], [[Escalonamento em Sistemas Interativos]], [[Escalonamento de Threads]], [[Política versus Mecanismo]], [[Implementando Threads em User Space]] e [[Implementações Híbridas]]. Esta nota é a aplicação de tudo isso a um escalonador real, escrito em Go.

> 💡 **Escalonador (scheduler):** componente que escolhe, entre as tarefas **prontas** para executar, qual executa em cada processador, e quando interromper a atual para dar a vez a outra.

---

## 🧩 1. G, M e P

```
                         ┌─────────── fila GLOBAL (com trava) ───────────┐
                         │  G  G  G  G                                   │
                         └───────────────────────────────────────────────┘
   P0                                   P1                         (GOMAXPROCS = 2)
 ┌──────────────────────────┐        ┌──────────────────────────┐
 │ runnext: G               │        │ runnext: —               │
 │ fila local: G G G (≤256) │        │ fila local: G (≤256)     │
 │ mcache (memória, sem trava)        │ mcache                   │
 └────────────┬─────────────┘        └────────────┬─────────────┘
              │ "está com"                        │
             M0 (thread do SO) ── roda G         M1 ── roda G
              │                                   │
          núcleo 0                            núcleo 1

   M2 ── presa numa syscall com G (SEM P)          M3 ── ociosa, dormindo (futex)
```

| Letra | O quê | Quantidade |
| --- | --- | --- |
| **G** (*goroutine*) | O trabalho: stack, registradores salvos, estado (ver `struct g` em [[Goroutines]]) | Milhares a milhões |
| **M** (*machine*) | Uma **thread do SO**, que executa código | Cresce conforme necessário (limite padrão: 10 000) |
| **P** (*processor*) | Um "direito de executar código Go": fila local, cache de memória, timers | Exatamente `GOMAXPROCS` |

A regra: **uma M precisa estar com um P para executar Gs**. Uma M sem P (presa numa syscall, ou ociosa) não executa código Go.

### 1.1 Por que existe o P

Até o Go 1.0, o escalonador era só G e M, com **uma fila global protegida por uma trava**: toda operação de escalonamento disputava essa trava, e o desempenho piorava com mais núcleos. No Go 1.1 (2013), Dmitry Vyukov introduziu o **P**: os recursos necessários para rodar Go (fila de goroutines prontas, cache de memória) passaram a ficar **por processador lógico**, acessados sem trava no caso comum. É o mesmo princípio das filas de execução **por CPU** do kernel Linux. E como o número de Ps é fixo, uma M bloqueada numa syscall pode **entregar** o seu P para outra M continuar trabalhando (seção 4).

---

## 🔄 2. O laço do escalonador

Cada M, quando a goroutine atual para (bloqueou, terminou, foi preemptada), chama `schedule()`, que chama `findRunnable()`. A ordem real de busca (Go 1.27, `runtime/proc.go`):

```
findRunnable():
 1. há trabalho de marcação do GC para este P?        → roda um worker do GC
 2. a cada 61 escalonamentos deste P:                 → pega da fila GLOBAL (justiça)
 3. runnext do P                                      → a goroutine "recém-criada/acordada"
 4. fila LOCAL do P (até 256, sem trava)
 5. fila GLOBAL (com trava)
 6. netpoller, sem bloquear: alguma conexão ficou pronta?
 7. ROUBO: até 4 rodadas, em Ps escolhidos aleatoriamente, roubando METADE da fila de um
    (e timers vencidos de outros Ps)
 8. nada? devolve o P, e a M dorme (futex) — antes, o último a dormir
    pode BLOQUEAR no netpoller (epoll_wait) esperando E/S
```

Dois detalhes desse laço têm efeito visível.

### 2.1 `runnext`: o último criado roda primeiro

Quando uma goroutine cria outra (ou acorda outra via channel), a nova vai para o **`runnext`** do P, à frente da fila; quem estava no `runnext` vai para o **fim** da fila. Saída real, com um único P:

```go
runtime.GOMAXPROCS(1)
for i := range 6 {
	wg.Go(func() { fmt.Print(i, " ") })
}
wg.Wait()
// 5 0 1 2 3 4
```

A goroutine 5, a última criada, ficou no `runnext` e rodou **primeiro**; as outras saíram da fila local em ordem FIFO. O motivo é **localidade**: num padrão produtor-consumidor ("crio uma goroutine/envio num channel e bloqueio esperando a resposta"), a goroutine recém-acordada provavelmente usa dados que ainda estão quentes no cache deste núcleo. Ela herda o restante da fatia de tempo de quem a acordou, para que dois "ping-pongs" não monopolizem o P.

> ⚠️ É mais um motivo para **nunca** depender da ordem de execução de goroutines: ela é um detalhe de implementação.

### 2.2 O 61 e a justiça

Se um P só olhasse a fila local, goroutines na fila **global** (para onde vão, por exemplo, as que excedem as 256 vagas locais) poderiam esperar indefinidamente. Por isso, a cada **61** escalonamentos, o P olha primeiro a global. Por que 61? Um número primo, para não entrar em ressonância com padrões periódicos do programa.

---

## ⚖️ 3. Roubo de trabalho

```
antes:   P0: [G G G G G G G G]        P1: [ ]  ← ficou sem trabalho
                                       │
                         P1 rouba METADE da fila de P0 (sem trava global:
                         operações atômicas na fila circular de P0)
depois:  P0: [G G G G]                P1: [G G G G]
```

Para não gastar CPU à toa, no máximo `GOMAXPROCS/2` Ms ficam **girando** (*spinning*) à procura de trabalho ao mesmo tempo; as outras dormem num futex até serem acordadas. Quando uma goroutine fica pronta e há Ps ociosos, o runtime acorda uma M para pegá-la.

É o **balanceamento de carga** entre processadores que o Tanenbaum discute para multiprocessadores, na variante "puxar" (*pull*): o ocioso vai buscar trabalho, em vez de um coordenador central distribuir.

---

## 🛑 4. Quando uma goroutine bloqueia

Aqui está o coração do modelo, e a razão de goroutines escalarem. Medição real: 100 goroutines esperando de três formas diferentes, contando as **threads do SO** do processo (`/proc/self/status`):

```
início: threads do SO: 4
100 goroutines em time.Sleep(500ms)                threads do SO: 4
100 goroutines esperando um channel                threads do SO: 4
100 goroutines em syscall bloqueante (nanosleep)   threads do SO: 103
```

| Tipo de espera | O que o runtime faz | Threads presas |
| --- | --- | --- |
| Channel, mutex, `select`, `WaitGroup` | `gopark`: a G sai do P e fica numa fila de espera do objeto; a M pega outra G | **Nenhuma** |
| `time.Sleep`, timers | A G estaciona; o timer fica no heap de timers do P | **Nenhuma** |
| Rede (sockets) | Socket não bloqueante + netpoller (epoll); a G estaciona em `[IO wait]` | **Nenhuma** |
| **Syscall bloqueante** (arquivo em disco, `nanosleep` cru, DNS via libc) | A M entra no kernel **com** a G; o P é **entregue** a outra M | **Uma por syscall** |
| Chamada **cgo** | Igual a uma syscall: a M fica presa no código C | **Uma por chamada** |

### 4.1 O ciclo de uma syscall

```
G em M0 com P0 chama read(fd_de_arquivo)
  │
  ├─ entersyscall(): marca P0 como "em syscall" (mas ainda ligado a M0)
  │       → caminho rápido: se a syscall voltar logo, M0 continua com P0 (sem custo extra)
  │
  ├─ o sysmon percebe P0 há mais de ~20 µs em syscall
  │       → retake: tira P0 de M0 e entrega a outra M (acordada ou CRIADA)
  │       → as outras goroutines de P0 continuam rodando
  │
  └─ a syscall volta: exitsyscall()
          → tenta pegar um P livre; se não houver, a G vai para a fila global
            e M0 vai dormir (ou morre, se houver threads ociosas demais)
```

É a solução para o defeito clássico das threads de usuário puras ([[Implementando Threads em User Space]]): uma syscall bloqueante **não** trava as outras goroutines, porque o P migra. O custo é criar threads: as 103 do experimento. Com milhares de syscalls bloqueantes simultâneas, chega-se ao limite de 10 000 threads (`fatal error: thread exhaustion`), ajustável com `debug.SetMaxThreads`.

---

## ⏱️ 5. Preempção

### 5.1 Cooperativa

Todo prólogo de função já compara o ponteiro de stack com `g.stackguard0` (para saber se a stack precisa crescer, ver [[Goroutines]]). Para pedir que uma goroutine pare, o runtime coloca um valor especial nesse campo: na próxima **chamada de função**, a verificação "falha", e a goroutine desvia para o escalonador. Isso é barato, mas um laço sem chamadas nunca passaria por esse ponto.

### 5.2 Assíncrona (Go 1.14+)

Para laços apertados, o `sysmon` envia o sinal **`SIGURG`** à thread; o tratador de sinal verifica se o ponto atual é seguro (o compilador sabe, para cada instrução, onde estão os ponteiros) e, se for, salva o estado e devolve o controle ao escalonador. O experimento de [[Goroutines]] (um `for {}` com `GOMAXPROCS(1)` que não trava mais o programa) é exatamente isso.

### 5.3 A fatia de tempo

A constante é `forcePreemptNS = 10 ms`. Mas quem verifica é o `sysmon`, que dorme entre 20 µs e 10 ms. Saída real do `schedtrace` com 10 goroutines de CPU pura em 2 Ps:

```
SCHED 1009ms: gomaxprocs=2 idleprocs=0 threads=4 ... runqueue=5 [ 2 1 ] schedticks=[ 51 55 ]
SCHED 2014ms: gomaxprocs=2 idleprocs=0 threads=4 ... runqueue=5 [ 0 3 ] schedticks=[ 101 105 ]
```

Cerca de **50 trocas por segundo** em cada P: fatias efetivas de ~20 ms, entre os 10 ms da constante e os 10 ms de intervalo do `sysmon`. Compare com o *quantum* de 20–50 ms que o Tanenbaum recomenda em [[Escalonamento em Sistemas Interativos]].

---

## 👁️ 6. `sysmon`: o vigia

O `sysmon` é uma thread especial que roda **sem P** (não executa código Go do usuário), num laço que dorme de **20 µs** (quando há atividade) até **10 ms** (quando o programa está ocioso):

| Tarefa | Quando |
| --- | --- |
| **Retomar Ps** de Ms presas em syscalls | P em syscall por mais de um ciclo do sysmon (≥ 20 µs) |
| **Preemptar** goroutines | Rodando há mais de 10 ms no mesmo P |
| **Consultar o netpoller** | Se ninguém consultou nos últimos 10 ms (evita que goroutines de rede esperem por um P ocupado) |
| **Forçar um GC** | Se não houve nenhum em 2 minutos (ver [[Garbage Collector]]) |
| Acordar o *scavenger* | Para devolver memória ao SO (ver [[Memória e Escape Analysis]]) |

É o equivalente, em espaço de usuário, da **interrupção de relógio** que dá ao kernel a chance de reavaliar quem roda ([[Introdução ao Escalonamento]]).

---

## 🔒 7. `runtime.LockOSThread`

Normalmente, uma goroutine pode trocar de thread a cada vez que é escalonada. `LockOSThread` amarra a goroutine à sua M: só ela roda naquela thread, e ela só roda naquela thread. Saída real (IDs de thread do Linux, lidos com `gettid`, a cada volta de um laço que dorme e trabalha):

```
sem LockOSThread, thread a cada volta: [14416 14416 14419 14416 14419]
com LockOSThread, thread a cada volta: [14418 14418 14418 14418 14418]
```

| Quando é necessário | Por quê |
| --- | --- |
| Bibliotecas C com estado **por thread** (OpenGL, alguns drivers, `errno` em loops cgo) | O estado ficaria na thread errada |
| Syscalls que alteram a **thread** atual: `setns`, `unshare` (namespaces), `setuid` por thread | É assim que runtimes de containers escritos em Go entram em namespaces (ver [[Namespaces]] e [[RunC]]) |
| APIs que exigem a **thread principal** (GUI no macOS) | Trave em `init()` |

> ⚠️ Se uma goroutine travada termina sem `UnlockOSThread`, o runtime **mata a thread** (porque o estado dela pode ter sido alterado de forma irreversível). Use com `defer runtime.UnlockOSThread()`, e só quando necessário.

---

## 🔬 8. Lendo o escalonador

### 8.1 `schedtrace` e `scheddetail`

```bash
GODEBUG=schedtrace=1000 ./app                 # uma linha por segundo
GODEBUG=schedtrace=1000,scheddetail=1 ./app   # cada P, M e G
```

| Campo | Significado |
| --- | --- |
| `gomaxprocs=2` | Número de Ps |
| `idleprocs` | Ps sem trabalho |
| `threads` | Ms existentes |
| `spinningthreads` | Ms procurando trabalho |
| `runqueue=5` | Tamanho da fila **global** |
| `[ 2 1 ]` | Tamanho da fila **local** de cada P |
| `schedticks=[ 51 55 ]` | Quantas vezes cada P escalonou |

Trecho real do `scheddetail=1`:

```
  P0: status=0 schedtick=0 syscalltick=0 m=nil runqsize=0 gfreecnt=0 timerslen=0
  P1: status=0 schedtick=3 syscalltick=0 m=nil runqsize=0 gfreecnt=0 timerslen=0
  M2: p=nil curg=nil ... spinning=true blocked=true lockedg=nil
  M0: p=nil curg=nil ... blocked=true lockedg=1
  G1: status=4(chan receive) m=nil lockedm=0
```

Esse retrato é do instante 0 ms, ainda **dentro da inicialização do runtime**: a goroutine 1 (a futura `main`) está em `chan receive` porque o `runtime.main` espera, por channels, as goroutines de fundo do GC (varredor e scavenger) começarem; e `M0 ... lockedg=1` mostra que, durante a inicialização (inclusive as funções `init()`), a goroutine principal fica **travada** à thread principal, com `LockOSThread`. Para uma visão no tempo (cada G em cada P), use o **trace de execução** (ver [[Profiling e Diagnóstico]]), que mostra latências de escalonamento por goroutine.

### 8.2 Comparando com o Tanenbaum e com o Linux

| Aspecto | Escalonador de Go | Escalonador do Linux (CFS/EEVDF) | Tanenbaum |
| --- | --- | --- | --- |
| Unidade | Goroutine | Thread (tarefa) | Processo/thread |
| Onde roda | Espaço de usuário | Kernel | — |
| Política | FIFO por P + `runnext` + roubo | Justiça por tempo virtual de CPU | Round-robin, prioridades, filas múltiplas... |
| Prioridades | **Nenhuma** (sem `nice` para goroutines) | Sim (`nice`, tempo real) | [[Escalonamento em Sistemas Interativos]] |
| Quantum | ~10–20 ms | Variável (alvo de latência) | 20–50 ms sugerido |
| Preempção | Cooperativa + sinal `SIGURG` | Interrupção de relógio | Interrupção de relógio |
| Multiprocessador | Filas por P + roubo de metade | Filas por CPU + balanceamento | Afinidade, compartilhamento de carga |
| Bloqueio em E/S | Estaciona a G (rede) ou entrega o P (syscall) | Tira a thread da CPU | Estado "bloqueado" ([[Estados de Processos]]) |

Os dois escalonadores trabalham **juntos**: o do kernel escolhe qual **M** roda em qual núcleo; o de Go escolhe qual **G** roda em cada M. É o modelo de dois níveis de [[Implementações Híbridas]] e de [[Escalonamento de Threads]]. A ausência de prioridades é uma decisão de **política** ([[Política versus Mecanismo]]): Go oferece o mecanismo de concorrência e deixa a priorização para o código (filas separadas, semáforos, `select` com dois níveis como em [[Select]]).

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Depender da ordem de execução | Funciona com `GOMAXPROCS=1`, falha em produção | Sincronização explícita |
| Milhares de syscalls/cgo bloqueantes simultâneas | Explosão de threads; `thread exhaustion` | Limite a concorrência (semáforo, pool) |
| `LockOSThread` sem `Unlock` | Threads mortas e recriadas | `defer runtime.UnlockOSThread()` |
| Esperar prioridade entre goroutines | Não existe | Estruture com filas/semáforos |
| `GOMAXPROCS` muito acima da CPU disponível | Trocas de contexto e *throttling* | Padrão ciente de cgroups (1.25+) |
| `runtime.Gosched()` para "consertar" corridas | Esconde o bug | Sincronização |
| Medir latência sem olhar o escalonador | Causa invisível | Trace de execução, `/sched/latencies:seconds` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| G, M, P | Goroutine, thread do SO, processador lógico (`GOMAXPROCS`); M precisa de P para rodar Go |
| Por que P | Filas e caches por processador, sem trava global (Go 1.1) |
| Busca | GC → global a cada 61 → `runnext` → local (256) → global → netpoll → roubo (4×, metade) → dormir |
| `runnext` | Última criada/acordada roda primeiro (localidade): `5 0 1 2 3 4` |
| Bloqueios | Channel/mutex/sleep/rede: nenhuma thread presa; syscall/cgo: uma thread presa, P migra |
| Experimento | 100 esperas: 4 threads (sleep/channel) vs 103 (syscall) |
| Preempção | Cooperativa no prólogo + `SIGURG` assíncrono; fatia de ~10–20 ms |
| `sysmon` | Sem P; 20 µs a 10 ms; retoma Ps, preempta, consulta netpoll, força GC |
| `LockOSThread` | Goroutine ↔ thread fixas; estado por thread, namespaces |
| Diagnóstico | `schedtrace`, `scheddetail`, trace de execução |
| Política | Sem prioridades; FIFO + roubo; dois níveis com o kernel |

---

## 🔗 Conexões

**Dentro de Go**
- `struct g`, stacks e `GOMAXPROCS` → [[Goroutines]]
- Mapa SO ↔ Go → [[Fundamentos de Concorrência]]
- `gopark` em channels e `select` → [[Channels]], [[Select]]
- Mutex: girar, dormir no semáforo → [[sync.WaitGroup e sync.Mutex]]
- Netpoller e epoll → [[Sockets e o Pacote net]]
- Arquivos prendem threads → [[Leitura e Escrita de Arquivos]]
- Cache de memória por P → [[Memória e Escape Analysis]]
- Workers do GC e o GC forçado → [[Garbage Collector]]
- Chamadas cgo e threads → [[unsafe e cgo]]
- `schedtrace` e trace → [[Profiling e Diagnóstico]]

**Sistemas Operacionais**
- [[Introdução ao Escalonamento]] — preempção, quantum, interrupção de relógio
- [[Escalonamento em Sistemas Interativos]] — round-robin, prioridades, quantum
- [[Escalonamento de Threads]] — escalonamento em dois níveis
- [[Política versus Mecanismo]] — Go sem prioridades
- [[Implementando Threads em User Space]], [[Implementando Threads em Kernel Space]] e [[Implementações Híbridas]] — o modelo N:M e o problema das syscalls bloqueantes
- [[Estados de Processos]] — pronto, executando, bloqueado
- [[Mutexes]] e [[Dormir e Despertar]] — Ms ociosas dormem em futex
- [[Syscalls Diversas]] — sinais (`SIGURG`)

**Docker**
- [[Namespaces]] e [[RunC]] — `LockOSThread` + `setns`/`unshare`
- [[Cgroups]] — o `GOMAXPROCS` padrão

---

⬅️ **Anterior:** [[Garbage Collector]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Modelo de Memória]] ➡️
