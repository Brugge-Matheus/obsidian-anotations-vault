---
tags:
  - go
  - go/concorrência
---
# Goroutines

> Goroutines são a unidade de concorrência de Go. Pense nelas como funções que executam de forma **independente e simultânea** — mas muito mais leves do que threads do sistema operacional. Um programa Go típico pode ter dezenas de milhares de goroutines rodando ao mesmo tempo sem problemas.
> 

---

## 1. O Problema que Goroutines Resolvem

Imagine que você precisa fazer 1000 requisições HTTP para uma API externa. Com código sequencial:

```go
// Abordagem sequencial — lenta
for _, url := range urls {
	resposta := fazerRequisicao(url)   // bloqueia aqui até terminar
	processar(resposta)
}
// Se cada requisição leva 100ms: 1000 × 100ms = 100 segundos!
```

Com goroutines:

```go
// Abordagem concorrente — muito mais rápida
for _, url := range urls {
	go func(u string) {
		resposta := fazerRequisicao(u)   // cada uma roda independente
		processar(resposta)
	}(url)
}
// Todas rodam "ao mesmo tempo" — tempo total ≈ 100ms (+ overhead)
```

Goroutines permitem que seu programa faça **múltiplas coisas ao mesmo tempo** sem precisar gerenciar threads manualmente.

---

## 2. O Que É uma Thread e Por Que Goroutines São Diferentes

Para entender goroutines, primeiro precisamos entender threads:

**Thread do Sistema Operacional:**

- Criada pelo OS (Linux, macOS, Windows)
- Stack fixa de **1MB a 8MB** por thread
- Criação leva **~10 microssegundos** (envolve syscall)
- Troca de contexto leva **~1-2 microssegundos** (modo kernel)
- Limite prático: **alguns milhares** de threads por processo

**Goroutine:**

- Criada pelo runtime de Go (não pelo OS)
- Stack inicial de apenas **2 kilobytes** (cresce dinamicamente)
- Criação leva **~100 nanossegundos** (100x mais rápido)
- Troca de contexto leva **~100 nanossegundos** (modo usuário)
- Limite prático: **milhões** de goroutines por processo

```go
// Demonstração: criar 100.000 goroutines é factível em Go
var wg sync.WaitGroup
for i := range 100_000 {
	wg.Add(1)
	go func(n int) {
		defer wg.Done()
		// cada goroutine começa com apenas 2KB de stack
		time.Sleep(1 * time.Second)
	}(i)
}
wg.Wait()
// Funciona sem problemas — threads do OS não conseguiriam fazer isso
```

---

## 3. O Scheduler de Go — Como Goroutines Realmente Funcionam

O runtime de Go implementa um **scheduler próprio** que gerencia goroutines. Esse scheduler usa três conceitos fundamentais chamados de **modelo G-P-M**:

```
G (Goroutine) — a tarefa a ser executada
P (Processor) — contexto de execução, tem uma fila local de goroutines
M (Machine)   — thread real do sistema operacional

Relação:
- Cada P tem UMA M associada em um dado momento
- Cada M executa UMA G por vez
- GOMAXPROCS controla quantos Ps existem (padrão = número de CPUs)

Visualização:

CPU 1          CPU 2
  │              │
  M1             M2
  │              │
  P1             P2
 ┌┴────────┐   ┌┴────────┐
 │ G em    │   │ G em    │
 │ execução│   │ execução│
 │         │   │         │
 │ fila:   │   │ fila:   │
 │ G G G G │   │ G G G   │
 └─────────┘   └─────────┘
                     ↑
              fila global: G G G G G G
```

**Como funciona na prática:**

1. Você chama `go fn()` — Go cria uma estrutura G e a coloca na fila do P atual
2. O P pega a G da fila e a passa para a M executar
3. Se a G faz uma operação bloqueante (I/O, sleep, lock), o P desatrela a M e pega outra G para executar
4. Enquanto a M espera o I/O, outras goroutines continuam rodando em outros Ps

**Work Stealing — balanceamento automático:**

```
Se P1 tem muitas goroutines e P2 está ocioso:
P2 "rouba" metade da fila de P1 para executar

P1: [G G G G G G G G] → P2 rouba → P1: [G G G G] P2: [G G G G]
```

---

## 4. Stack Dinâmica — Por Que Goroutines São Tão Leves

A stack de uma goroutine cresce e encolhe conforme necessário:

```
Estado inicial da goroutine:
Stack: [████] 2KB

Depois de chamar funções profundas:
Stack: [████████████████████████████] 32KB (cresceu)

Depois de retornar:
Stack: [████] 2KB (encolheu — memória liberada)

Como o crescimento funciona:
1. Em cada chamada de função, o compilador insere uma checagem:
   "ainda tenho stack suficiente?"
2. Se não: cria uma stack nova com o DOBRO do tamanho
3. Copia todos os frames para a nova stack (stack copying)
4. Atualiza todos os ponteiros que apontavam para a stack antiga
5. A stack antiga é liberada para o GC
```

Isso é por que você pode criar goroutines sem se preocupar com o tamanho da stack — ela se ajusta automaticamente. Ao contrário de threads do OS onde você precisa pré-alocar toda a stack na criação.

---

## 5. Preempção — Goroutines Não Travam o Programa

Uma preocupação válida: "se uma goroutine entra em loop infinito, ela trava tudo?"

**Antes do Go 1.14:** Sim! Goroutines só eram preemptadas em chamadas de função. Um loop `for {}` bloqueava toda a thread.

**Go 1.14+ — Preempção Assíncrona:** O runtime usa sinais do OS para interromper goroutines mesmo em loops sem chamadas de função:

```go
// Go 1.14+: isso NÃO trava o programa
go func() {
	for {
		// loop infinito puro — era problemático antes do Go 1.14
		// agora o runtime envia SIGURG a cada ~10ms para preemptar
	}
}()

// Esta goroutine ainda executa normalmente
go func() {
	fmt.Println("ainda funcionando!")
}()
```

---

## 6. Criando Goroutines — Sintaxe

A palavra-chave `go` antes de qualquer chamada de função cria uma goroutine:

```go
// Forma 1: goroutine com função nomeada
func trabalhar(id int) {
	fmt.Printf("worker %d iniciado\n", id)
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("worker %d concluído\n", id)
}

go trabalhar(1)   // dispara e continua imediatamente — não espera terminar
go trabalhar(2)
go trabalhar(3)

// Forma 2: goroutine com função anônima (mais comum)
go func() {
	fmt.Println("goroutine anônima")
}()   // ← os parênteses chamam a função imediatamente

// Forma 3: passando argumentos (importante para closures em loops)
for i := range 5 {
	go func(n int) {    // n é uma CÓPIA de i
		fmt.Println(n)   // seguro — cada goroutine tem seu próprio n
	}(i)
}
```

**O problema clássico de closure em loops (antes do Go 1.22):**

```go
// ❌ Bug clássico — ANTES do Go 1.22
for i := 0; i < 5; i++ {
	go func() {
		fmt.Println(i)   // captura &i — referência, não valor!
		// quando executar, i pode ser 5 (loop já terminou)
	}()
}
// Possível saída: 5 5 5 5 5

// ✅ Correção: passar como parâmetro
for i := 0; i < 5; i++ {
	go func(n int) {
		fmt.Println(n)   // n é uma cópia do valor de i naquele momento
	}(i)
}

// ✅ Go 1.22+: for range N cria variável nova a cada iteração
for i := range 5 {
	go func() {
		fmt.Println(i)   // cada goroutine tem seu próprio i — seguro!
	}()
}
```

---

## 7. O Problema de Sincronização — Goroutines Não Esperam

Uma goroutine dispara e o programa continua. Se `main()` terminar, **todas as goroutines são mortas** imediatamente, mesmo que não tenham terminado:

```go
func main() {
	go func() {
		time.Sleep(1 * time.Second)
		fmt.Println("goroutine terminou")   // pode nunca ser impresso!
	}()

	fmt.Println("main terminou")
	// main retorna aqui → programa encerra → goroutine é morta
}
```

**Solução: `sync.WaitGroup` para esperar goroutines:**

```go
func main() {
	var wg sync.WaitGroup

	for i := range 5 {
		wg.Add(1)         // avisar: "tem mais uma goroutine para esperar"
		go func(n int) {
			defer wg.Done()   // avisar quando terminar (via defer = sempre executa)
			fmt.Printf("worker %d\n", n)
		}(i)
	}

	wg.Wait()   // bloqueia até todas as goroutines chamarem Done()
	fmt.Println("todas terminaram")
}
```

**Regra crítica do WaitGroup:**

```go
// ✅ Add() ANTES do go — garante que o contador seja incrementado
// antes de Wait() ter a chance de ver 0
wg.Add(1)
go trabalhar()

// ❌ Add() DENTRO da goroutine — race condition!
go func() {
	wg.Add(1)    // pode executar DEPOIS de Wait() já ter retornado!
	defer wg.Done()
	trabalhar()
}()
```

---

## 8. Race Conditions — O Perigo Invisível

Quando múltiplas goroutines acessam a mesma variável e pelo menos uma escreve, temos uma **race condition**. O resultado é imprevisível porque depende da ordem de execução:

```go
// ❌ Race condition — NUNCA faça isso
contador := 0

for range 1000 {
	go func() {
		contador++   // NÃO é atômico! São 3 operações:
		             // 1. LOAD  contador → registrador
		             // 2. ADD   registrador + 1
		             // 3. STORE registrador → contador
		             // Duas goroutines podem fazer LOAD ao mesmo tempo
		             // e uma sobrescreve o incremento da outra!
	}()
}

// Resultado: pode ser qualquer valor entre 1 e 1000
// Às vezes 1000, às vezes 847, às vezes 612...
```

**Detectar races com `-race`:**

```bash
go run -race main.go
go test -race ./...

# Output do race detector:
# WARNING: DATA RACE
# Write at 0x00c0000b4010 by goroutine 7:
#   main.main.func1()
# Previous write at 0x00c0000b4010 by goroutine 6:
#   main.main.func1()
```

> ⚠️ Sempre use `-race` em CI/CD. O overhead é ~2x tempo e ~5x memória — aceitável para detectar bugs críticos.
> 

**Três formas de resolver:**

```go
// Solução 1: sync.Mutex — trava o acesso
var mu sync.Mutex
var contador int

go func() {
	mu.Lock()
	contador++
	mu.Unlock()
}()

// Solução 2: sync/atomic — operação atômica de CPU (mais rápida)
var contador atomic.Int64

go func() {
	contador.Add(1)   // instrução atômica de CPU — thread-safe por design
}()

// Solução 3: channel — comunicar em vez de compartilhar (idiomático em Go)
ch := make(chan int)
go func() {
	ch <- 1   // envia o incremento para o canal
}()
total := 0
for v := range ch {
	total += v   // apenas uma goroutine modifica total
}
```

---

## 10. Goroutine Leaks — O Problema Silencioso

Uma goroutine que **nunca termina** é um memory leak. O GC não pode coletar goroutines bloqueadas:

```go
// ❌ Goroutine leak — a goroutine fica presa para sempre
func processar() {
	ch := make(chan int)   // canal sem buffer

	go func() {
		resultado := calcularAlgo()
		ch <- resultado   // tenta enviar...
	}()

	// Mas se processar() retornar sem ler do canal:
	// a goroutine fica PRESA tentando enviar para sempre!
	return   // goroutine vaza!
}

// ✅ Solução 1: usar canal com buffer
ch := make(chan int, 1)   // a goroutine pode enviar mesmo sem ninguém recebendo

// ✅ Solução 2: usar context para cancelamento
func processar(ctx context.Context) {
	ch := make(chan int, 1)

	go func() {
		select {
		case ch <- calcularAlgo():   // tenta enviar
		case <-ctx.Done():           // ou cancela se o contexto expirar
			return
		}
	}()

	select {
	case resultado := <-ch:
		usar(resultado)
	case <-ctx.Done():
		return
	}
}
```

**Regra:** toda goroutine deve ter um caminho garantido para terminar — seja por conclusão do trabalho, fechamento de canal, ou cancelamento de context.

---

## 11. GOMAXPROCS — Controlar o Paralelismo

```go
import "runtime"

// Ver quantos CPUs lógicos estão disponíveis
fmt.Println(runtime.NumCPU())         // ex: 8

// Ver quantos Ps (processadores) estão ativos
fmt.Println(runtime.GOMAXPROCS(0))   // 0 = apenas ler, não mudar

// Mudar (raramente necessário — o padrão é bom)
runtime.GOMAXPROCS(4)   // usar apenas 4 CPUs
runtime.GOMAXPROCS(1)   // útil para debugging de race conditions

// Via variável de ambiente (antes de iniciar o programa)
// GOMAXPROCS=4 ./meuservidor
```

**Importante:** `GOMAXPROCS` controla o **paralelismo** (execução simultânea real em múltiplos CPUs), não a **concorrência** (múltiplas goroutines alternando em um CPU). Você pode ter 10.000 goroutines com `GOMAXPROCS=1`.

---

## 12. Padrão Completo — Worker Pool

O padrão mais comum de goroutines em produção: um número fixo de workers processando jobs de uma fila:

```go
func workerPool(ctx context.Context, numWorkers int, jobs []string) []string {
	jobCh := make(chan string, len(jobs))
	resultCh := make(chan string, len(jobs))
	var wg sync.WaitGroup

	// Iniciar workers
	for range numWorkers {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case job, ok := <-jobCh:
					if !ok {
						return   // canal fechado — worker encerra
					}
					// processar o job
					resultCh <- processar(job)

				case <-ctx.Done():
					return   // contexto cancelado — encerrar
				}
			}
		}()
	}

	// Enviar todos os jobs
	for _, job := range jobs {
		jobCh <- job
	}
	close(jobCh)   // sinaliza que não há mais jobs

	// Aguardar workers e fechar resultados
	go func() {
		wg.Wait()
		close(resultCh)
	}()

	// Coletar resultados
	var resultados []string
	for r := range resultCh {
		resultados = append(resultados, r)
	}
	return resultados
}
```

---

## 13. Regras de Ouro para Goroutines

```
1. SEMPRE garanta que goroutines possam terminar
   → Use WaitGroup, channels com buffer, ou context

2. SEMPRE passe variáveis de loop como parâmetros
   → go func(n int) { }(i)  — antes do Go 1.22

3. NUNCA acesse dados compartilhados sem sincronização
   → Use mutex, atomic, ou channel

4. SEMPRE use -race em desenvolvimento e CI/CD
   → go test -race ./...

5. PREFIRA channels para comunicação de dados
   → "não comunique compartilhando memória, compartilhe memória comunicando"

6. USE context para cancelamento e timeout
   → func fn(ctx context.Context, ...) error

7. CUIDADO com goroutine leaks
   → Toda goroutine deve ter um caminho para terminar

8. PREFIRA WaitGroup ou errgroup para esperar goroutines
   → golang.org/x/sync/errgroup para propagação de erros
```

---

## Conexão com Sistemas Operacionais

### Goroutines vs Threads de SO — O Modelo de Execução

Em [[O Modelo Clássico de Thread]], uma thread é a unidade de execução do SO: ela tem seu próprio stack, registradores salvos e estado de escalonamento, mas compartilha o espaço de endereçamento com outras threads do mesmo processo. O SO gerencia essas threads diretamente no kernel e decide quando cada uma roda.

Goroutines são fundamentalmente diferentes em *onde* vivem. Go implementa o que [[Implementando Threads em User Space]] chama de "threads de espaço do usuário" (green threads): o runtime de Go é uma biblioteca que emula um escalonador dentro do próprio processo, sem envolver o kernel a cada troca de contexto.

```
Modelo de Threads do SO (Kernel Space):
  Processo
    ├── Thread 1  ←──── escalonada pelo kernel
    ├── Thread 2  ←──── kernel troca contexto (~1-2 µs, modo privilegiado)
    └── Thread 3  ←──── kernel aloca stack fixa (1–8 MB cada)

Modelo de Goroutines do Go (User Space + Kernel híbrido):
  Processo
    ├── OS Thread (M1)  ←── criada pelo kernel
    │     └── P1 (scheduler Go)
    │           ├── G1  ←── escalonada pelo runtime (~100 ns, modo usuário)
    │           ├── G2
    │           └── G3
    └── OS Thread (M2)  ←── criada pelo kernel
          └── P2
                ├── G4
                └── G5
```

O modelo G-P-M de Go é uma instância do modelo N:M descrito em [[Implementações Híbridas]]: N goroutines mapeadas para M threads de SO. Isso contrasta com o modelo 1:1 (uma thread de SO por thread de usuário, como pthreads) e com o modelo N:1 (todas as threads de usuário em uma única thread de SO, como em [[Implementando Threads em User Space]] puro).

### Stack da Goroutine vs Stack de Thread do SO — [[Processos]]

Em [[Processos]], a stack é uma região de memória contígua pré-alocada que cresce para baixo. Uma thread do SO recebe sua stack no momento da criação: tipicamente 1 MB no macOS, 8 MB no Linux (ajustável via `ulimit -s`). Esse tamanho é **fixo** — se uma função recursiva profunda ultrapassar o limite, ocorre **stack overflow**, um erro fatal discutido em [[Processos]].

Go elimina o stack overflow de uma forma elegante:

```
Thread de SO (stack fixa):
  ┌────────────────────────────────┐  ← limite: 8 MB (Linux)
  │ frame: main()                  │
  │ frame: handleRequest()         │
  │ frame: parsarJSON()            │
  │ frame: recursão profunda...    │
  │ frame: recursão profunda...    │
  │ ??? stack overflow aqui        │  ← SIGSEGV / crash
  └────────────────────────────────┘

Goroutine (stack dinâmica):
  ┌──────┐  2 KB inicial
  │frame1│
  └──────┘
  ← precisa de mais espaço →
  ┌────────────┐  4 KB (dobrou)
  │frame1      │
  │frame2      │
  │frame3      │
  └────────────┘
  ← precisa de mais espaço →
  ┌──────────────────────┐  8 KB (dobrou de novo)
  │ todos os frames      │  ← stack copying: copia tudo para nova região
  └──────────────────────┘
```

O compilador Go insere uma checagem ("stack guard") no prólogo de cada função. Se não há espaço suficiente, o runtime chama `morestack()`, aloca uma nova stack com o dobro do tamanho e copia todos os frames para ela — atualizando todos os ponteiros que apontavam para a stack antiga. Isso é transparente para o programador.

O custo inicial baixo (2 KB) é o que permite criar centenas de milhares de goroutines sem esgotar memória — algo impossível com threads de SO de stack fixa.

### O Scheduler G-P-M — Multiprogramação em User Space

[[Modelando a Multiprogramação]] explica que um sistema operacional usa multiprogramação para manter a CPU ocupada: quando um processo bloqueia em I/O, o SO escala outro. O scheduler de Go reimplementa exatamente esse conceito, mas dentro do processo, gerenciando goroutines em vez de processos.

[[O Modelo de Processos]] fala de **pseudoparalelismo**: um único CPU executando múltiplos processos rapidamente, criando a ilusão de paralelismo. O scheduler de Go faz o mesmo com goroutines em cada thread de SO:

```
Pseudoparalelismo em nível de SO (O Modelo de Processos):
  CPU → P1 → P2 → P3 → P1 → P2 → ...  (troca em ~ms, gerenciada pelo kernel)

Pseudoparalelismo em nível de Go (dentro do processo):
  M1 → G1 → G4 → G7 → G1 → G9 → ...  (troca em ~µs, gerenciada pelo runtime)
  M2 → G2 → G5 → G3 → G8 → G2 → ...
```

O P (Processor) é o análogo do "contexto de CPU virtual" — ele mantém a fila local de goroutines e o estado necessário para executar Go code. GOMAXPROCS determina quantos Ps existem, e portanto quantas goroutines podem executar verdadeiramente em paralelo.

### GOMAXPROCS e Paralelismo Real — [[Processadores]]

[[Processadores]] descreve multicore, hyperthreading e a distinção entre concorrência (múltiplas tarefas progredindo) e paralelismo (múltiplas tarefas executando simultaneamente em cores físicos distintos).

`GOMAXPROCS` mapeia diretamente sobre esse conceito:

- `GOMAXPROCS=1`: todas as goroutines concorrem em uma thread de SO. Pseudoparalelismo puro, mesmo em um processador multicore.
- `GOMAXPROCS=N` (padrão = `runtime.NumCPU()`): Go cria N threads de SO e N Ps. Se a máquina tem 8 cores físicos, até 8 goroutines podem executar verdadeiramente em paralelo.
- Em ambientes com hyperthreading (cada core físico exposto como 2 "logical CPUs"), `runtime.NumCPU()` retorna o número de CPUs lógicas, não físicas. O Go não faz distinção — trata cada CPU lógica como um slot de P.

### Work Stealing — Balanceamento de Carga

O work stealing implementado pelo scheduler do Go é análogo ao balanceamento de carga discutido em [[Modelando a Multiprogramação]]: o objetivo é manter todos os "processadores" (Ps) ocupados. Se P2 esvazia sua fila local de goroutines enquanto P1 tem dezenas esperando, P2 rouba metade da fila de P1.

```
Antes do work stealing:
  P1: [G G G G G G G G]   P2: []  ← ocioso

Depois do work stealing:
  P1: [G G G G]           P2: [G G G G]  ← equilibrado
```

Isso elimina o custo de comunicação com o kernel para redistribuição de carga — tudo acontece em memória compartilhada entre as threads do processo.

### Preempção Assíncrona com SIGURG — [[Processos]] (Sinais de Alarme)

[[Processos]] descreve sinais UNIX como mecanismo de notificação assíncrona: um processo pode registrar handlers para sinais e o kernel entrega o sinal interrompendo a execução normal. Sinais de alarme (`SIGALRM`) permitem que um processo solicite ao SO ser notificado após N segundos.

O Go 1.14 introduziu preempção assíncrona usando exatamente esse mecanismo. O runtime envia `SIGURG` (sinal de "urgent data" em sockets, escolhido por ser raramente usado por aplicações) a cada ~10 ms para as threads de SO que têm goroutines rodando. O signal handler do runtime verifica se a goroutine atual rodou por tempo suficiente e, se sim, a preempta.

```
Sem preempção assíncrona (Go < 1.14):
  G1: [for { /* CPU puro */ }]  ← nunca yield → M1 travada → P1 inútil
  G2: esperando na fila de P1  ← nunca executa enquanto G1 rodar

Com preempção assíncrona (Go >= 1.14):
  G1: [for { }] → SIGURG após ~10ms → salva registradores → suspende G1
  G2: agendada no P1 → executa
  G1: retorna à fila → executa depois
```

Isso é fundamentalmente diferente de como threads de SO funcionam: no caso de threads, é o kernel que envia o sinal de timer (via interrupt de hardware) e faz a troca. No Go, o runtime imita esse comportamento usando o mecanismo de sinais UNIX do SO.

### User-Space Scheduling — [[Implementando Threads em User Space]]

[[Implementando Threads em User Space]] descreve as vantagens e desvantagens de implementar threads completamente em user space: troca de contexto muito rápida (sem syscall), mas a desvantagem de que uma thread bloqueada em syscall trava toda a thread de SO subjacente, bloqueando todas as outras threads de usuário.

O Go resolve esse problema de forma elegante: quando uma goroutine faz uma syscall bloqueante (como `read()` em um arquivo), o runtime **desatrela** o P da M (thread de SO) antes de entrar na syscall. Isso libera o P para ser atrelado a outra M (ou uma nova M é criada), e as outras goroutines continuam executando sem interrupção.

```
G1 vai fazer syscall bloqueante (ex: read()):
  ANTES:  M1 ↔ P1 ↔ G1 (executando)

  1. Runtime detecta: "G1 vai bloquear em syscall"
  2. Desatrela P1 de M1
  3. M1 entra na syscall com G1 (bloqueada no kernel)
  4. P1 é atrelado a M2 (ou nova thread criada)
  5. P1 continua executando G2, G3, G4... normalmente

  DURANTE: M1 ↔ G1 (bloqueada no kernel)
           M2 ↔ P1 ↔ G2, G3... (executando normalmente)

  QUANDO syscall retorna:
  6. M1 tenta reatrelar a P1 (ou outro P disponível)
  7. G1 volta para uma fila de P e é reescalonada
```

Essa é a razão pela qual I/O de rede em Go não bloqueia outras goroutines — o runtime usa `epoll`/`kqueue` (I/O não-bloqueante no nível do SO) e o goroutine scheduler para multiplexar eficientemente.