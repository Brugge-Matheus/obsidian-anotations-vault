---
tags:
  - go
  - go/funções
---
# Closures e Funções Anônimas

> Closures são funções que capturam variáveis do escopo léxico onde foram definidas. O compilador Go aplica escape analysis: variáveis capturadas por closures escapam para a heap, pois sua vida útil ultrapassa a função que as criou.
> 

---

## 1. O Que Acontece na Memória

```
Quando você cria uma closure:

func criarContador() func() int {
    n := 0        ← normalmente ficaria na stack
    return func() int {
        n++       ← n é capturado → compilador move n para a HEAP
        return n
    }
}

Stack frame de criarContador():
┌─────────────┐
│ ptr para n  │ → aponta para a heap
└─────────────┘

Heap:
┌─────┐
│  n  │ = 0  ← alocado aqui para sobreviver ao retorno
└─────┘

A closure retornada é:
┌─────────────┬───────────────┐
│ ptr para fn │ ptr para env  │ ← "fat pointer" com ambiente capturado
└─────────────┴───────────────┘
                      ↓
              ┌───────────────┐
              │ n (na heap)   │
              └───────────────┘
```

Verificar com escape analysis:

```bash
go build -gcflags="-m" ./...
# Output: "./main.go:3:2: moved to heap: n"
```

---

## 2. Funções Anônimas

```go
// Declarar e atribuir
dobrar := func(n int) int {
	return n * 2
}
fmt.Println(dobrar(5))   // 10

// Tipo explícito
var transformar func(string) string
transformar = func(s string) string {
	return strings.ToUpper(s)
}

// IIFE — Immediately Invoked Function Expression
resultado := func(a, b int) int {
	return a + b
}(10, 20)   // chamada imediata
fmt.Println(resultado)   // 30

// Passar como argumento
sort.Slice(itens, func(i, j int) bool {
	return itens[i].Prioridade < itens[j].Prioridade
})

// Retornar como valor
func multiplicadorPor(fator int) func(int) int {
	return func(n int) int {
		return n * fator
	}
}

dobrar := multiplicadorPor(2)
triplicar := multiplicadorPor(3)
fmt.Println(dobrar(5))    // 10
fmt.Println(triplicar(5)) // 15
```

---

## 3. Closures Capturam por Referência

A closure captura a **variável** (referência), não o valor no momento da criação:

```go
x := 10

// Duas closures compartilham a MESMA variável x
incrementar := func() { x++ }
ler := func() int { return x }

incrementar()
incrementar()
fmt.Println(ler())   // 12 — x foi modificado

// A variável x original também reflete as mudanças
fmt.Println(x)   // 12 — mesma variável!
```

---

## 4. Armadilha Clássica em Loops — Antes do Go 1.22

```go
// ❌ Bug clássico (antes do Go 1.22)
// Todas as closures capturam a MESMA variável i
funcs := make([]func(), 5)
for i := 0; i < 5; i++ {
	funcs[i] = func() {
		fmt.Println(i)   // captura &i, não o valor de i
	}
}
for _, f := range funcs {
	f()   // imprime 5, 5, 5, 5, 5 — loop já terminou, i=5
}

// ✅ Correção pré-Go 1.22: criar nova variável que sombra a do loop
for i := 0; i < 5; i++ {
	i := i   // nova variável local na iteração — closure captura esta
	funcs[i] = func() { fmt.Println(i) }
}
// Imprime: 0, 1, 2, 3, 4

// ✅ Correção alternativa: passar como parâmetro
for i := 0; i < 5; i++ {
	funcs[i] = func(n int) func() {
		return func() { fmt.Println(n) }
	}(i)   // i é avaliado e copiado como parâmetro aqui
}

// ✅ Go 1.22+: variável de loop é nova a cada iteração — bug corrigido!
// O código original (❌) agora imprime 0, 1, 2, 3, 4 corretamente
for i := range 5 {
	funcs[i] = func() { fmt.Println(i) }   // cada goroutine vê seu próprio i
}
```

---

## 5. Closures em Goroutines

```go
// ❌ Antes do Go 1.22 — race condition + valor errado
for i := 0; i < 5; i++ {
	go func() {
		fmt.Println(i)   // i pode ser qualquer valor quando executa
	}()
}

// ✅ Passar como parâmetro — sempre correto
for i := 0; i < 5; i++ {
	go func(n int) {
		fmt.Println(n)   // n é uma cópia do valor de i no momento da chamada
	}(i)
}

// ✅ Go 1.22+ — for range N é seguro diretamente
for i := range 5 {
	go func() {
		fmt.Println(i)   // cada goroutine tem seu próprio i
	}()
}
```

---

## 6. Padrões Práticos

### Gerador (Stateful Iterator)

```go
func gerarFibonacci() func() int {
	a, b := 0, 1
	return func() int {
		v := a
		a, b = b, a+b
		return v
	}
}

fib := gerarFibonacci()
for range 10 {
	fmt.Print(fib(), " ")   // 0 1 1 2 3 5 8 13 21 34
}
```

### Memoization com Closure

```go
func memoizar(fn func(int) int) func(int) int {
	cache := make(map[int]int)   // capturado pela closure
	return func(n int) int {
		if v, ok := cache[n]; ok {
			return v
		}
		v := fn(n)
		cache[n] = v
		return v
	}
}

var fib func(int) int
fib = memoizar(func(n int) int {
	if n <= 1 {
		return n
	}
	return fib(n-1) + fib(n-2)
})

fmt.Println(fib(50))   // rápido — com cache
```

### Middleware / Decorator

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Closure que "envolve" um handler com logging
func comLogging(fn HandlerFunc) HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		inicio := time.Now()
		fn(w, r)   // chama o handler original
		fmt.Printf("%s %s — %v\n", r.Method, r.URL.Path, time.Since(inicio))
	}
}

// Closure que adiciona retry com estado
func comRetry(maxTentativas int, fn func() error) func() error {
	return func() error {
		for i := range maxTentativas {
			if err := fn(); err == nil {
				return nil
			} else if i < maxTentativas-1 {
				time.Sleep(time.Duration(1<<i) * 100 * time.Millisecond)
			}
		}
		return fmt.Errorf("falhou após %d tentativas", maxTentativas)
	}
}
```

### Opções Funcionais — O Padrão Mais Elegante

```go
type Servidor struct {
	host    string
	porta   int
	timeout time.Duration
	debug   bool
}

type Opcao func(*Servidor)   // tipo "opção" é uma closure que configura

func ComHost(host string) Opcao {
	return func(s *Servidor) { s.host = host }
}

func ComPorta(porta int) Opcao {
	return func(s *Servidor) { s.porta = porta }
}

func ComTimeout(d time.Duration) Opcao {
	return func(s *Servidor) { s.timeout = d }
}

func ComDebug() Opcao {
	return func(s *Servidor) { s.debug = true }
}

func NovoServidor(opcoes ...Opcao) *Servidor {
	s := &Servidor{
		host:    "localhost",
		porta:   8080,
		timeout: 30 * time.Second,
	}
	for _, opt := range opcoes {
		opt(s)   // cada closure configura s
	}
	return s
}

// API limpa e extensível
s := NovoServidor(
	ComHost("0.0.0.0"),
	ComPorta(443),
	ComTimeout(60*time.Second),
	ComDebug(),
)
```

---

## 7. Closures e `defer`

```go
// defer com closure — captura variáveis por referência
func operacao() (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic: %v", r)   // modifica o retorno nomeado!
		}
	}()
	// ... código que pode panic
	return
}

// Medição de tempo com closure
func medirTempo(nome string) func() {
	inicio := time.Now()
	return func() {
		fmt.Printf("%s: %v\n", nome, time.Since(inicio))
	}
}

func processamento() {
	defer medirTempo("processamento")()   // () chama e retorna a closure
	// ... trabalho
}
```

---

## 8. Closures como Callbacks e Higher-Order Functions

```go
// Map
func mapear[T, R any](slice []T, fn func(T) R) []R {
	resultado := make([]R, len(slice))
	for i, v := range slice {
		resultado[i] = fn(v)
	}
	return resultado
}

// Filter
func filtrar[T any](slice []T, pred func(T) bool) []T {
	var resultado []T
	for _, v := range slice {
		if pred(v) {
			resultado = append(resultado, v)
		}
	}
	return resultado
}

// Reduce
func reduzir[T, R any](slice []T, inicial R, fn func(R, T) R) R {
	acc := inicial
	for _, v := range slice {
		acc = fn(acc, v)
	}
	return acc
}

// Composição
nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

soma := reduzir(
	mapear(
		filtrar(nums, func(n int) bool { return n%2 == 0 }),
		func(n int) int { return n * n },
	),
	0,
	func(acc, n int) int { return acc + n },
)
fmt.Println(soma)   // 4+16+36+64+100 = 220
```

---

## 9. Resumo

| Conceito | Detalhe |
| --- | --- |
| Closures capturam por | Referência (a variável, não o valor) |
| Variáveis capturadas ficam na | Heap (escape analysis) |
| Loop bug (< Go 1.22) | Todas closures viam o valor final de i |
| Go 1.22+ | Variável de loop é nova por iteração — bug corrigido |
| Goroutines + closures | Passe i como parâmetro para segurança |
| Padrão de opções funcionais | Opcao = func(*T) — muito usado em libs Go |
| IIFE | `func() { ... }()` — executada imediatamente |
| Performance | Closures que capturam têm overhead de heap |

---

## 10. Conexão com Sistemas Operacionais

### Closures na Memória — Função + Ambiente Capturado

Uma closure em Go é representada internamente como um **fat pointer**: um ponteiro para o código da função + um ponteiro para uma struct alocada na heap com as variáveis capturadas:

```
func criarContador() func() int {
    n := 0
    return func() int { n++; return n }
}

c := criarContador()

Memória:
               Stack (criarContador já retornou)
               [frame destruído]

               Heap:
               ┌─────────────────────┐
               │  closure struct     │
               │  ┌───────────────┐  │
               │  │  funcptr      │──┼──→ código da função anônima
               │  │  n    = 1     │  │    (TEXT segment, somente leitura)
               │  └───────────────┘  │
               └─────────────────────┘
                         ↑
               c (variável na stack do caller):
               ┌─────────────────────┐
               │  ptr para closure   │─────────────────────────────┘
               └─────────────────────┘

Chamada c():
  1. Carrega o funcptr da struct de closure
  2. Passa o ptr da struct como argumento implícito (contexto)
  3. Executa: n++ usando o n da struct na heap
  4. Retorna n
```

A variável `n` **escapou para a heap** porque sua vida útil ultrapassa o frame da função `criarContador`. O compilador detecta isso via **escape analysis**. Ver [[Gerenciamento de Memória]].

---

### Escape Analysis — Compilador Decide Stack vs Heap

O compilador Go analisa estaticamente o **tempo de vida** de cada variável para decidir onde alocá-la:

```bash
go build -gcflags="-m" ./...
# "moved to heap: n"     ← variável capturada pela closure escapou
# "func literal escapes" ← a própria closure escapou para a heap
```

```go
// Caso 1 — closure NÃO captura variáveis externas: sem heap
func dobrar(n int) int {
    f := func(x int) int { return x * 2 }  // não captura nada
    return f(n)
    // f fica na stack — o compilador pode até inlinar
}

// Caso 2 — closure captura variável: var vai para heap
func contador() func() int {
    n := 0           // "moved to heap: n"
    return func() int { n++; return n }
    // n alocado na heap — sobrevive ao retorno de contador()
}

// Caso 3 — closure usada localmente: pode ficar na stack
func localSomente() int {
    x := 10
    f := func() int { return x * 2 }  // x pode ficar na stack
    return f()   // f não escapa — compilador pode otimizar
}
```

O escape analysis é a ponte entre o modelo de [[Gerenciamento de Memória]] (stack rápida vs heap com GC) e o comportamento das closures em Go.

---

### Function Values — Ponteiros Indiretos para Código

Em Go, uma variável do tipo `func(...)` é um **ponteiro indireto** — diferente de function pointers em C, carrega o contexto da closure junto:

```
C — function pointer simples:
  int (*fn)(int) = &dobrar;
  fn(5);   // assembly: CALL [fn]  ← jump direto para dobrar

Go — function value (com possível closure):
  f := dobrar        // sem captura: f = {ptr_para_código, nil}
  f := criarCont()   // com closure: f = {ptr_para_código, ptr_para_env}
  f()   // assembly: 
        //   MOVQ  (f), AX        ← carrega funcptr
        //   MOVQ  f, DX          ← carrega ptr do ambiente (contexto)
        //   CALL  AX             ← chamada INDIRETA via registrador
```

A instrução `CALL AX` (chamada indireta via registrador) é mais cara que `CALL funcao` (chamada direta) — o preditor de branch não consegue prever o destino com tanta antecedência. Por isso closures têm um overhead extra em hot paths. Ver [[Processadores]].

---

### Goroutines + Closures — Variáveis Vivem na Heap

O padrão `go func() { use(x) }()` é ubíquo em Go, mas tem uma implicação importante de memória:

```go
func servirRequests(servidor *Servidor) {
    for {
        conn, err := servidor.Accept()
        if err != nil { break }

        go func() {
            // conn é capturado pela closure
            // → conn escapa para a heap
            // → sobrevive ao loop iteration
            servirConexao(conn)
        }()
    }
}
```

```
O que acontece na memória:

Loop iteration 1:
  conn1 alocado → escape analysis: escapa → alocado na heap
  closure criada → struct na heap: { funcptr, conn: conn1 }
  go scheduler: nova goroutine com a closure
  Loop continua (conn1 ainda vivo — goroutine segura referência)

Loop iteration 2:
  conn2 alocado → heap, nova struct de closure
  ...

GC coleta conn1 apenas quando a goroutine de conn1 terminar
```

A variável `conn` tem **tempo de vida vinculado à goroutine** — o GC não pode coletar enquanto a goroutine mantiver a referência. Isso é gerenciamento de tempo de vida de memória, tema central de [[Processos]] e [[Gerenciamento de Memória]].

---

### Higher-Order Functions e Callbacks — Analogia com o OS

Closures e funções de primeira classe em Go espelham padrões usados em todo o kernel Linux e APIs de OS:

```
OS e bibliotecas C usam function pointers para callbacks:

  // signal handler — registrar função para sinal SIGINT:
  signal(SIGINT, meu_handler);   // function pointer

  // File operations no kernel Linux (struct file_operations):
  struct file_operations {
      ssize_t (*read)(struct file*, char __user*, size_t, loff_t*);
      ssize_t (*write)(struct file*, const char __user*, size_t, loff_t*);
      int     (*open)(struct inode*, struct file*);
      // ... tabela de ponteiros de função
  };
  // Cada driver implementa sua própria tabela — polimorfismo via ponteiros

  // Callbacks de I/O assíncrono (aio, io_uring):
  io_uring_sqe->user_data = (uint64_t)callback_fn;
```

```go
// Go equivalente — mais seguro e com contexto capturado:

// Signal handler com closure (captura estado):
signal.Notify(sigChan, syscall.SIGINT)
go func() {
    <-sigChan
    servidor.Desligar()   // captura servidor — imposível em C sem global
}()

// File operation "driver" em Go (io.Reader/Writer):
type MeuLeitor struct { buf []byte; pos int }
func (r *MeuLeitor) Read(p []byte) (int, error) { ... }
// Tabela de métodos → equivalente ao struct file_operations do kernel

// Callback de I/O com closure:
http.HandleFunc("/api/dados", func(w http.ResponseWriter, r *http.Request) {
    responder(w, config)   // captura config — sem global!
})
```

O mecanismo de callbacks com function pointers no OS é análogo a closures em Go, mas Go adiciona: type safety, contexto capturado (sem variáveis globais), e GC (sem necessidade de free manual). Ver [[Processadores]] e [[System Calls]].