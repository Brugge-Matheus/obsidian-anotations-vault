---
tags:
  - go
  - go/concorrencia
aliases:
  - Goroutine
  - Instrução go
  - GOMAXPROCS
  - Vazamento de goroutines
  - Goroutine leak
ordem: 37
modulo: "09 - Concorrência"
---
# Goroutines

> **Nota 37 da trilha de Go.** A **goroutine** é a unidade de concorrência de Go: uma função que executa de forma independente, com sua própria stack, compartilhando a memória do processo. É criada com uma palavra só (`go`) e custa tão pouco que um programa pode ter **centenas de milhares** delas. Nesta nota veremos a instrução `go`, o ciclo de vida de uma goroutine (e por que o `main` não espera ninguém), a armadilha das closures em laços, por que goroutines são tão baratas, a stack que cresce, uma visão geral do escalonador e da preempção, o `GOMAXPROCS` (inclusive em containers), os **vazamentos de goroutines** e o novo profile que os detecta, e o que o runtime faz quando você escreve `go f()`.
>
> **Pré-requisitos:** [[Fundamentos de Concorrência]], [[Closures e Funções Anônimas]], [[Defer, Panic e Recover]].

---

## 🧭 De onde viemos

Em [[Fundamentos de Concorrência]] usamos goroutines sem explicá-las, para mostrar a diferença entre concorrência e paralelismo. Em [[Closures e Funções Anônimas]] vimos que closures capturam **variáveis**, não valores, e em [[Defer, Panic e Recover]] vimos que um panic numa goroutine derruba o programa inteiro. Do lado do SO, [[O Modelo Clássico de Thread]] definiu o que uma thread tem de próprio (registradores, stack, estado) e o que compartilha (espaço de endereçamento, arquivos abertos). Uma goroutine segue exatamente esse modelo.

> 💡 **Goroutine:** função executando de forma concorrente, gerenciada pelo **runtime de Go** (não diretamente pelo kernel). Tem stack própria e pequena, que cresce sob demanda, e compartilha o espaço de endereçamento com todas as outras goroutines do processo. O nome é um trocadilho com *coroutine* (corrotina).

---

## 🚀 1. A instrução `go`

Colocar `go` antes de uma **chamada de função** faz essa chamada rodar numa nova goroutine. A goroutine que executou o `go` **não espera**: continua imediatamente na linha seguinte.

```go
func trabalhar(id int) {
	fmt.Println("worker", id)
}

go trabalhar(1)                       // função nomeada
go func() { fmt.Println("anônima") }() // função anônima: note os () no final
go srv.Atender(conn)                  // método (o receptor também é avaliado agora)
```

Três regras que vêm de graça porque `go` é parecido com `defer`:

| Regra | Consequência |
| --- | --- |
| A função e os **argumentos são avaliados na hora** do `go`, na goroutine atual | `go f(x)` usa o valor que `x` tinha naquele instante |
| Os **valores de retorno são descartados** | Para devolver um resultado, use um channel ou escreva numa variável sincronizada |
| É preciso uma **chamada** | `go f` (sem parênteses) não compila |

```go
resultado := go calcular()           // ❌ não compila: go não devolve nada

ch := make(chan int, 1)
go func() { ch <- calcular() }()     // ✅ o resultado volta por um channel
fmt.Println(<-ch)
```

---

## ⏱️ 2. Ciclo de vida: ninguém espera ninguém

### 2.1 O `main` não espera as goroutines

Quando a função `main` retorna, o **processo termina**, e todas as goroutines morrem junto, no meio do que estiverem fazendo, sem rodar seus `defer`. Saída real:

```go
func main() {
	go func() {
		time.Sleep(100 * time.Millisecond)
		fmt.Println("goroutine terminou")
	}()
	fmt.Println("main terminou")
}
```

```
main terminou
```

A linha da goroutine nunca aparece. Para esperar, é preciso **sincronizar**: um `sync.WaitGroup`, um channel ou um `errgroup` (ver [[sync.WaitGroup e sync.Mutex]] e [[Padrões de Concorrência]]).

```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
	defer wg.Done()
	trabalhar(1)
}()
wg.Wait()                 // bloqueia até o Done
```

### 2.2 Goroutines não têm "alça"

Diferente de `pthread_create`, que devolve um `pthread_t` para você esperar (`pthread_join`) ou cancelar (`pthread_cancel`), o `go` **não devolve nada**:

| Você não consegue... | Por quê | O que fazer |
| --- | --- | --- |
| Obter o ID de uma goroutine | Não há API pública, de propósito, para evitar "armazenamento local de thread" | Passe o que precisa por parâmetros ou `context` |
| Esperar uma goroutine específica | Não existe `join` | WaitGroup, channel de "terminei" |
| Matar uma goroutine de fora | Não existe `kill` | **Cancelamento cooperativo**: a goroutine verifica um sinal (`ctx.Done()`) e retorna sozinha |

> 💡 **Cancelamento cooperativo:** a tarefa não é interrompida à força; ela é **avisada** e encerra a si mesma num ponto seguro, liberando recursos. É o modelo de [[context.Context]]. Matar à força deixaria locks travados e estruturas pela metade (o mesmo motivo pelo qual `pthread_cancel` é tão perigoso em C).

### 2.3 Como uma goroutine termina

| Forma | Defers rodam? | Afeta as outras? |
| --- | --- | --- |
| A função retorna | ✅ | Não |
| `runtime.Goexit()` | ✅ | Não (encerra só esta goroutine) |
| Panic recuperado dentro dela | ✅ | Não |
| Panic **não** recuperado | Só os desta goroutine | ❌ **Derruba o processo inteiro** |
| `main` retorna ou `os.Exit` | ❌ | Todas morrem |

> ⚠️ Um panic não recuperado em **qualquer** goroutine termina o programa, e um `recover` em outra goroutine não ajuda (vimos em [[Defer, Panic e Recover]]). Goroutines de longa duração que rodam código arriscado devem ter seu próprio `defer func() { recover() ... }()`.

---

## 🔁 3. Closures em laços: o bug que o Go 1.22 resolveu

Uma closure captura a **variável**, não o valor (ver [[Closures e Funções Anônimas]]). Até o Go 1.21, a variável de um `for` era **uma só para todas as voltas**, então goroutines disparadas no laço viam o valor **final**. Saída real do **mesmo código**, compilado com `go 1.21` e com `go 1.22` (ou mais) no `go.mod`:

```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Print(i, " ")
	}()
}
wg.Wait()
```

```
go 1.21 no go.mod:  3 3 3
go 1.22+ no go.mod: 2 0 1
```

Desde o **Go 1.22**, cada volta do laço tem **sua própria** variável `i`, e o bug desapareceu. A ordem continua imprevisível (as goroutines rodam em qualquer ordem), mas cada uma vê o seu valor. A mudança vale por módulo: depende da linha `go` do `go.mod` (ver [[Sistema de Módulos (go mod)]]).

> 💡 Você ainda vai ver muito código antigo com uma das duas correções da era pré-1.22: `i := i` dentro do laço, ou passar como argumento, `go func(n int) { ... }(i)`. Ambas continuam funcionando, mas hoje são desnecessárias (e o `go fix` moderno as remove).

---

## 🪶 4. Por que goroutines são tão baratas

Medição real: criar 100 mil goroutines que ficam paradas esperando um channel (Go 1.27, Linux, 2 núcleos):

```go
const n = 100_000
bloqueio := make(chan struct{})
for range n {
	go func() {
		<-bloqueio // fica parada até o fim
	}()
}
// ... mede tempo e memória com runtime.ReadMemStats
```

```
goroutines vivas: 100001
tempo por goroutine: 2.192µs
stack por goroutine: 2048 bytes
memória total obtida do SO: +263 MB
```

Cada goroutine parada custa exatamente **2048 bytes de stack** (mais a `struct g` do runtime, algumas centenas de bytes), e os ~2 µs por goroutine incluem criar, executar até bloquear e estacionar. Para comparar com threads do SO, medi na **mesma máquina** o ciclo "criar e esperar terminar" 10 mil vezes, em C com pthreads e em Go:

```
C (pthread_create + pthread_join):   stack padrão de uma thread: 8192 KB
                                     criar + esperar uma thread: 35.0 µs
Go (go func + channel):              criar + esperar uma goroutine: 0.65 µs
```

Cerca de **50× mais barato**. A tabela resume as diferenças:

| | Thread do SO (Linux, pthreads) | Goroutine |
| --- | --- | --- |
| Quem cria | Kernel, via syscall `clone` | Runtime, em espaço de usuário (sem syscall) |
| Stack inicial | Reserva de **8 MB** de memória virtual (`ulimit -s`), tamanho fixo | **2 KB**, cresce e encolhe |
| Criar e esperar (medido acima) | 35 µs (syscall `clone`, estruturas do kernel) | 0,65 µs |
| Troca de contexto | Entra no kernel; salva todos os registradores | Em espaço de usuário; salva poucos registradores (PC, SP, BP...) |
| Escalonada por | Escalonador do kernel | Escalonador do runtime de Go |
| Quantas por processo, na prática | Milhares | Centenas de milhares a milhões |

A diferença vem de **onde** a goroutine vive: ela é uma "thread de usuário" no sentido de [[Implementando Threads em User Space]], só que o runtime a coloca sobre várias threads do kernel ([[Implementações Híbridas]]), evitando o principal defeito das threads de usuário puras (uma syscall bloqueando todo mundo).

---

## 📈 5. A stack que cresce

### 5.1 Como funciona

Cada goroutine começa com uma stack de **2 KB**. No início de quase toda função, o compilador insere uma verificação ("ainda cabe o meu frame?"). Se não cabe, o runtime:

```
1. aloca uma stack NOVA com o dobro do tamanho        2 KB → 4 KB → 8 KB → ...
2. COPIA todos os frames para a nova stack
3. ajusta os ponteiros que apontavam para dentro da stack antiga
4. libera a antiga e continua a execução
```

Esse modelo se chama **stack contígua com cópia** (desde o Go 1.4; antes, Go usava "stacks segmentadas", que sofriam com o problema do *hot split*). Durante a coleta de lixo, uma stack que usa menos de 1/4 do espaço pode **encolher** pela metade.

> 💡 É por isso que, em Go, **ponteiros para a stack podem mudar de valor** sem você perceber, e por isso o cgo proíbe que código C guarde ponteiros para memória de Go (ver [[unsafe e cgo]]).

### 5.2 Até onde cresce

Saída real: um milhão de chamadas aninhadas funciona, mas uma recursão infinita esbarra no limite de **1 GB** por goroutine (em 64 bits):

```go
fmt.Println("1 milhão de chamadas aninhadas:", profundidade(1_000_000))
infinita(0)  // func infinita(n int) int { return infinita(n+1) + 1 }
```

```
1 milhão de chamadas aninhadas: 0
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0x3a20acf60398 stack=[0x3a20acf60000, 0x3a20ccf60000]
fatal error: stack overflow

goroutine 1 gp=0x3a208ce241e0 m=0 mp=0x572240 [running]:
main.infinita(0x2aaaa3b?)
	/tmp/gt/ex/n37/stack/main.go:14 +0x2b
main.infinita(...)
...
```

Numa thread do SO, a mesma recursão causaria `SIGSEGV` ao bater na página de guarda no fim da stack de 8 MB. Em Go, o estouro é um `fatal error` com mensagem clara (e, como todo fatal error, não pode ser recuperado). O limite pode ser alterado com `debug.SetMaxStack`.

---

## 🗓️ 6. Quem decide quem roda: visão geral do escalonador

O runtime tem um escalonador próprio, o **G-M-P**, que será estudado a fundo em [[Scheduler do Go]]. O essencial:

```
  G = goroutine          M = thread do SO ("machine")       P = processador lógico
                                                                (fila + recursos)
        fila global: [G][G][G]
                                         GOMAXPROCS = número de Ps
     ┌──────── P0 ────────┐      ┌──────── P1 ────────┐
     │ fila local [G][G]  │      │ fila local [G]     │
     │ executando: G      │      │ executando: G      │
     └─────────┬──────────┘      └─────────┬──────────┘
               M0 (thread)                 M1 (thread)
               │                           │
           núcleo 0                    núcleo 1
```

| Situação | O que o runtime faz |
| --- | --- |
| `go f()` | Cria um G e o coloca na fila local do P atual (no "próximo a rodar") |
| G bloqueia em channel, mutex, `Sleep` | G é **estacionado** (não ocupa thread); o P pega outro G |
| G faz E/S de rede | O socket é registrado no **netpoller** (epoll); G estaciona; a thread segue livre |
| G faz syscall bloqueante (ex.: ler arquivo) | O P é **entregue** a outra thread, para as demais goroutines continuarem |
| Fila local de um P vazia | Ele **rouba** metade da fila de outro P (*work stealing*) |
| G roda há mais de ~10 ms | É **preemptado** (ver 6.1) |

### 6.1 Preempção

Até o Go 1.13, uma goroutine só cedia o processador em pontos "seguros" (chamadas de função, operações de channel). Um laço apertado, sem chamadas, podia monopolizar um P. Desde o **Go 1.14**, a preempção é **assíncrona**: uma thread de monitoramento do runtime (`sysmon`) percebe goroutines rodando há mais de ~10 ms e envia o sinal **`SIGURG`** à thread delas; o tratador de sinal salva o estado e devolve o controle ao escalonador.

Saída real com **um único P** (`GOMAXPROCS(1)`) e uma goroutine presa num laço infinito vazio:

```go
runtime.GOMAXPROCS(1) // um único núcleo para código Go
go func() {
	for { // laço infinito sem nenhuma chamada de função
	}
}()
time.Sleep(10 * time.Millisecond) // deixa a goroutine gulosa começar
fmt.Println("main voltou a executar mesmo assim")
```

```
main voltou a executar mesmo assim
```

Antes do 1.14, esse programa travaria para sempre. É o mesmo princípio da interrupção de relógio que o kernel usa para preemptar processos ([[Introdução ao Escalonamento]]), só que implementado com **sinais** em espaço de usuário (ver [[Syscalls Diversas]]).

### 6.2 Ceder a vez de propósito: `runtime.Gosched`

`runtime.Gosched()` coloca a goroutine atual de volta na fila e deixa outra rodar. Saída real com um único P:

```go
runtime.GOMAXPROCS(1)
for _, nome := range []string{"A", "B"} {
	go func() {
		for i := range 3 {
			fmt.Print(nome, i, " ")
			runtime.Gosched() // cede a vez voluntariamente
		}
	}()
}
```

```
B0 A0 B1 A1 B2 A2
```

> ⚠️ `Gosched` é quase sempre desnecessário em código real; aparece em testes e em demonstrações. Se você precisa dele para o programa "funcionar", provavelmente falta sincronização.

---

## ⚙️ 7. `GOMAXPROCS`: quanto paralelismo

`GOMAXPROCS` é o **número de Ps**, ou seja, quantas goroutines podem executar código Go **ao mesmo tempo**. Não limita o número de goroutines, nem o número de threads (threads presas em syscalls não contam).

```go
runtime.NumCPU()        // CPUs lógicas visíveis ao processo
runtime.GOMAXPROCS(0)   // lê o valor atual (0 = não altera)
runtime.GOMAXPROCS(4)   // altera (raramente necessário)
// ou, de fora: GOMAXPROCS=4 ./servidor
```

### 7.1 Em containers (Go 1.25+)

Por muito tempo o padrão era simplesmente `NumCPU()`. Num container Kubernetes limitado a 2 CPUs numa máquina de 64 núcleos, Go criava 64 Ps, e o kernel **estrangulava** (*throttling*) o processo ao estourar a cota do cgroup, causando picos de latência. Por isso muita gente usava a biblioteca `automaxprocs`.

Desde o **Go 1.25**, o padrão é **ciente de containers**:

| Comportamento | Detalhe |
| --- | --- |
| Limite de CPU do cgroup | No Linux, se o limite de banda de CPU do cgroup (ver [[Cgroups]]) for menor que o número de CPUs, ele vira o padrão |
| Atualização periódica | O runtime reavalia o valor se o número de CPUs ou o limite mudar durante a execução |
| Desligado quando | Você define `GOMAXPROCS` (variável ou função), ou usa `GODEBUG=containermaxprocs=0` / `updatemaxprocs=0` |
| Voltar ao padrão | `runtime.SetDefaultGOMAXPROCS()` |

> 💡 O runtime considera o **limite** (*limit*) de CPU do cgroup, não a **requisição** (*request*) do Kubernetes.

---

## 🕳️ 8. Vazamentos de goroutines

> 💡 **Vazamento de goroutine (goroutine leak):** uma goroutine que fica **bloqueada para sempre** (num channel, num mutex, numa espera) e nunca termina. O coletor de lixo **não** libera goroutines bloqueadas: a stack, e tudo o que ela referencia, fica preso. Num servidor, cada requisição que vaza uma goroutine faz a memória crescer até o processo cair.

### 8.1 O vazamento clássico: retorno antecipado

Este é o exemplo das próprias notas de versão do Go: várias goroutines enviam resultados num channel **sem buffer**, e quem recebe **retorna no primeiro erro**. Quem ainda não enviou fica bloqueado no `ch <-` para sempre. Saída real, usando o profile **`goroutineleak`** (experimental no Go 1.26, **disponível por padrão no Go 1.27**):

```go
func buscarTodos(ids []int) ([]int, error) {
	type res struct {
		v   int
		err error
	}
	ch := make(chan res) // sem buffer
	for _, id := range ids {
		go func() {
			v, err := buscar(id)
			ch <- res{v, err} // ❌ fica preso aqui se ninguém mais receber
		}()
	}
	var out []int
	for range ids {
		r := <-ch
		if r.err != nil {
			return nil, r.err // ❌ retorna cedo e abandona os outros
		}
		out = append(out, r.v)
	}
	return out, nil
}

// em main:
_, err := buscarTodos([]int{1, 2, 3, 4, 5})   // o item 2 falha
fmt.Println("erro:", err)
time.Sleep(50 * time.Millisecond)
fmt.Println("goroutines vivas:", runtime.NumGoroutine())
pprof.Lookup("goroutineleak").WriteTo(os.Stdout, 1)
```

```
erro: falha no item 2
goroutines vivas: 4
goroutineleak profile: total 3
3 @ 0x47d82a 0x41421c 0x413e17 0x4def25 0x483561
#	0x4def24	main.buscarTodos.func1+0x44	/tmp/gt/ex/n37/leak/main.go:30
```

O item 1 foi recebido, o item 2 falhou e a função retornou: as goroutines dos itens 3, 4 e 5 ficaram presas para sempre no envio. O profile aponta **exatamente essas 3**, na **linha 30** (o `ch <- res{v, err}`). As 4 goroutines vivas são essas 3 mais a do `main`.

### 8.2 Como o detector funciona

A ideia usa o **coletor de lixo**: uma goroutine bloqueada num channel (ou mutex, ou `sync.Cond`) só pode ser acordada por alguém que **alcance** esse channel. Se o channel não é alcançável a partir de **nenhuma goroutine executável**, ninguém nunca vai acordá-la: é um vazamento comprovado. No exemplo, depois que `buscarTodos` retorna, só as próprias goroutines bloqueadas referenciam o `ch`.

| Característica | Detalhe |
| --- | --- |
| Como obter | `pprof.Lookup("goroutineleak")`, ou o endpoint `/debug/pprof/goroutineleak` do `net/http/pprof` |
| Custo | Nenhum, a menos que o profile seja coletado |
| Falsos positivos | Não: o que ele aponta não tem como acordar |
| Falsos negativos | Sim: se o channel é alcançável por uma variável global ou pela stack de uma goroutine viva, ele não acusa |

Veremos profiles em [[Profiling e Diagnóstico]].

### 8.3 As correções

```go
// ✅ 1) buffer do tamanho do número de envios: ninguém bloqueia ao enviar
ch := make(chan res, len(ids))

// ✅ 2) cancelamento: quem envia desiste quando o chamador for embora
select {
case ch <- res{v, err}:
case <-ctx.Done():
}

// ✅ 3) errgroup: espera todas e cancela o contexto no primeiro erro
g, ctx := errgroup.WithContext(ctx)
```

> ✅ **Regra de ouro:** ao escrever `go`, saiba responder **"como e quando esta goroutine termina?"**. Se a resposta for "nunca" ou "não sei", há um vazamento esperando para acontecer.

---

## 🔬 9. Por dentro: o que acontece em `go f(x)`

O compilador transforma `go f(x)` numa chamada a `runtime.newproc`, que:

```
1. empacota f e seus argumentos (já avaliados) numa closure
2. pega uma struct g reaproveitada da lista de goroutines mortas do P
   (ou aloca uma nova, com stack de 2 KB)
3. prepara o "contexto salvo" do g: PC = início de f, SP = topo da stack nova,
   e um endereço de retorno falso que leva a runtime.goexit
4. marca o g como _Grunnable e o coloca em p.runnext (o próximo a rodar)
5. se houver P ocioso, acorda uma thread para pegar trabalho
```

A `struct g` é o "bloco de controle" da goroutine, o equivalente ao registro da tabela de threads de [[Implementação de Processos]]:

```
type g struct {                       (simplificado, runtime/runtime2.go)
	stack       stack      // limites [lo, hi) da stack atual
	stackguard0 uintptr    // comparado no prólogo das funções; também usado para pedir preempção
	sched       gobuf      // registradores salvos: sp, pc, bp, ctxt...
	atomicstatus uint32    // _Grunnable, _Grunning, _Gwaiting, _Gsyscall, _Gdead...
	m           *m         // thread que está rodando este g (se estiver)
	waitreason  waitReason // "chan receive", "sync.Mutex.Lock", "select"...
	...
}
```

Os estados mapeiam diretamente para os de [[Estados de Processos]]:

| Estado do Tanenbaum | Estado da goroutine | Aparece no stack trace como |
| --- | --- | --- |
| Em execução | `_Grunning` | `[running]` |
| Pronto | `_Grunnable` | `[runnable]` |
| Bloqueado | `_Gwaiting` | `[chan receive]`, `[select]`, `[sync.Mutex.Lock]`, `[IO wait]`, `[sleep]`... |
| (em chamada ao kernel) | `_Gsyscall` | `[syscall]` |

A **troca de contexto** entre goroutines (`runtime.gogo`) restaura só os registradores do `gobuf` e salta para o PC salvo, tudo em espaço de usuário. Não há troca de tabela de páginas nem entrada no kernel, por isso custa uma fração da troca entre threads do SO.

> 💡 **Go 1.27:** se você usa *labels* de profiling (`pprof.Do`), eles passam a aparecer na linha de cabeçalho de cada goroutine nos stack traces, o que facilita saber "de qual requisição" era a goroutine que travou. Dá para desligar com `GODEBUG=tracebacklabels=0`.

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `main` retorna sem esperar | Trabalho some sem erro | WaitGroup, channel, errgroup |
| Esperar com `time.Sleep` | Funciona às vezes | Sincronização de verdade |
| Closure de laço em módulo `go 1.21` ou anterior | Todas veem o último valor | Atualize o `go.mod` para 1.22+ ou passe como argumento |
| Esperar o valor de retorno de `go f()` | Não compila | Channel ou variável sincronizada |
| Goroutine sem caminho de saída | Memória cresce; `NumGoroutine` só sobe | Buffer, `context`, errgroup; profile `goroutineleak` |
| Panic sem `recover` numa goroutine de longa duração | O servidor inteiro cai | `recover` no topo da goroutine |
| Tentar "matar" uma goroutine | Não existe API | Cancelamento cooperativo com `context` |
| Criar uma goroutine por item sem limite (milhões de itens com E/S) | Esgota arquivos abertos, conexões, memória | Limite a concorrência (pool, semáforo, `SetLimit`) |
| `GOMAXPROCS` alto demais em container antigo (< 1.25) | Throttling e latência | Atualize o Go ou defina `GOMAXPROCS` |
| Usar `Gosched` para "consertar" ordem | Comportamento frágil | Channels ou mutex |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `go f(x)` | Dispara `f` numa nova goroutine; argumentos avaliados na hora; retorno descartado |
| Fim do `main` | Mata todas as goroutines; é preciso sincronizar para esperar |
| Sem alça | Sem ID, sem join, sem kill: use WaitGroup/channels e cancelamento cooperativo |
| Laços (1.22+) | Variável nova a cada volta; o bug clássico de closure acabou |
| Custo | ~2 KB de stack, centenas de ns para criar, troca de contexto sem kernel |
| Stack | Começa com 2 KB, cresce copiando, encolhe no GC, limite de 1 GB |
| Escalonador | G-M-P, filas por P, roubo de trabalho, netpoller, entrega do P em syscalls |
| Preempção | Assíncrona com `SIGURG` desde o 1.14 |
| `GOMAXPROCS` | Número de Ps; ciente do limite de CPU do cgroup desde o 1.25 |
| Vazamentos | Goroutine bloqueada para sempre; profile `goroutineleak` (1.27) |
| `struct g` | Bloco de controle; estados equivalem aos de processos |

---

## 🔗 Conexões

**Dentro de Go**
- Concorrência vs paralelismo e o modelo CSP → [[Fundamentos de Concorrência]]
- Comunicação entre goroutines → [[Channels]], [[Select]]
- Esperar e proteger → [[sync.WaitGroup e sync.Mutex]]
- Cancelamento cooperativo → [[context.Context]]
- Pools, errgroup e limites → [[Padrões de Concorrência]]
- Captura de variáveis → [[Closures e Funções Anônimas]]
- Panics em goroutines → [[Defer, Panic e Recover]]
- A linha `go` do `go.mod` e a semântica de laço → [[Sistema de Módulos (go mod)]]
- `io.Pipe` entre goroutines → [[io.Reader e io.Writer]]
- O escalonador G-M-P em profundidade → [[Scheduler do Go]]
- Stack, heap e escape analysis → [[Memória e Escape Analysis]]
- Profiles (`goroutine`, `goroutineleak`) → [[Profiling e Diagnóstico]]
- Uma goroutine por conexão → [[Sockets e o Pacote net]], [[HTTP (net-http)]]

**Sistemas Operacionais**
- [[O Modelo Clássico de Thread]] — o que é próprio (stack, registradores, estado) e o que é compartilhado
- [[Implementando Threads em User Space]] — goroutines são threads de usuário: troca rápida, sem kernel
- [[Implementando Threads em Kernel Space]] — as Ms são threads do kernel
- [[Implementações Híbridas]] — o modelo N:M (a própria nota cita as goroutines)
- [[Threads POSIX]] — `pthread_create`/`pthread_join`, comparados a `go`/WaitGroup
- [[Estados de Processos]] — executando, pronto, bloqueado ↔ `_Grunning`, `_Grunnable`, `_Gwaiting`
- [[Implementação de Processos]] — a tabela de processos/threads ↔ `struct g`
- [[Introdução ao Escalonamento]] e [[Escalonamento de Threads]] — preempção por relógio vs preempção por sinal
- [[Syscalls Diversas]] — sinais, usados pelo runtime para preemptar (`SIGURG`)
- [[Processos]] — a stack do processo e o estouro de pilha

**Docker**
- [[Cgroups]] — o limite de CPU que define o `GOMAXPROCS` padrão desde o Go 1.25

---

⬅️ **Anterior:** [[Fundamentos de Concorrência]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Channels]] ➡️
