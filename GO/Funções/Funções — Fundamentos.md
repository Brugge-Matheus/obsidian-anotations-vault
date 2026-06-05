---
tags:
  - go
  - go/funções
---
# Funções — Fundamentos

> Funções em Go são blocos fundamentais de código. Além das funções normais, Go tem closures, funções de primeira classe, múltiplos retornos e um sistema de defer/panic/recover. O compilador aplica escape analysis para decidir onde alocar variáveis de funções.
> 

---

## 1. Declaração Básica

```go
// func nomeDaFuncao(parametros) tipoDeRetorno { }
func saudar(nome string) string {
	return "Olá, " + nome + "!"
}

// Sem parâmetros e sem retorno
func inicializar() {
	fmt.Println("inicializando...")
}

// Void (sem retorno)
func imprimirLinha(msg string) {
	fmt.Println(msg)
}
```

---

## 2. Como Funções São Compiladas

Internamente, uma chamada de função em Go:

1. **Argumentos** são colocados em registradores (ABI register-based desde Go 1.17) ou na stack (para argumentos que não cabem em registradores)
2. O endereço de retorno é empilhado na stack
3. A função cria seu **stack frame** — espaço para variáveis locais
4. O resultado é retornado via registradores

```
// Go 1.17+ usa register-based calling convention (AMD64: AX, BX, CX, DI, SI, R8, R9, R10, R11)
// Antes de 1.17: stack-based (todos os argumentos na stack)

func somar(a, b int) int { return a + b }

// Assembly gerado (AMD64):
// MOVQ AX, a      // a ← primeiro argumento (registrador AX)
// ADDQ BX, AX     // AX ← a + b (BX é o segundo argumento)
// RET             // retorna com resultado em AX
```

---

## 3. Parâmetros

### Parâmetros do Mesmo Tipo

```go
// Forma longa
func somar(a int, b int) int { return a + b }

// Forma curta — tipo compartilhado
func somar(a, b int) int { return a + b }

// Mistura
func calcular(x, y float64, label string) string {
	return fmt.Sprintf("%s: %.2f", label, x+y)
}
```

### Passagem por Valor vs Ponteiro

Em Go, **tudo é passado por valor** — mas o "valor" de um ponteiro é o endereço:

```go
// Passa uma CÓPIA do int — não modifica o original
func dobrar(n int) int {
	return n * 2
}

// Passa o ENDEREÇO — modifica via deref
func dobrarInPlace(n *int) {
	*n *= 2   // deref: acessa o int na memória, não a cópia do ponteiro
}

// Slices, maps e channels são passados por valor MAS seus headers
// contêm ponteiros — então modificações nos dados internos são visíveis
func adicionarItem(s *[]int, item int) {
	*s = append(*s, item)   // append pode realocar — precisa de ponteiro para o header
}

func modificarSlice(s []int) {
	s[0] = 99   // modifica o array subjacente — visível fora
}

x := []int{1, 2, 3}
modificarSlice(x)
fmt.Println(x[0])   // 99 — visível! (modifica o array, não o header)
```

---

## 4. Retorno

### Retorno Simples

```go
func quadrado(n float64) float64 {
	return n * n
}
```

### Múltiplos Retornos — Implementação

Go retorna múltiplos valores via registradores (até 9 valores no AMD64 ABI) ou via stack para mais valores. Não há tupla — são valores independentes:

```go
func dividir(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("divisão por zero")
	}
	return a / b, nil
}

// Consumir os dois retornos
resultado, err := dividir(10, 3)
if err != nil {
	log.Fatal(err)
}
fmt.Printf("%.4f\n", resultado)   // 3.3333
```

### Retornos Nomeados

Nomeiam os valores de retorno — úteis para documentação, `defer` que modifica o erro, e em funções curtas:

```go
func minMax(nums []int) (min, max int) {
	if len(nums) == 0 {
		return 0, 0   // naked return não recomendado em casos não-óbvios
	}
	min, max = nums[0], nums[0]
	for _, n := range nums[1:] {
		if n < min { min = n }
		if n > max { max = n }
	}
	return   // naked return — retorna min e max como estão
}
```

> ⚠️ "Naked return" (return sem argumentos) só é recomendado em funções muito curtas (≤ 5 linhas). Em funções longas, dificulta entender o que está sendo retornado.
> 

---

## 5. Funções Variádicas

O parâmetro variádico é internamente um slice. Deve ser o último parâmetro:

```go
func somar(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

fmt.Println(somar())              // 0
fmt.Println(somar(1, 2, 3))      // 6

// Passar slice com ...
numeros := []int{1, 2, 3, 4, 5}
fmt.Println(somar(numeros...))   // 15

// Internamente: nums é []int — o mesmo slice, sem cópia
// Mas ⚠️: modificar nums dentro da função afeta o slice original se passado com ...
```

---

## 6. Funções como Valores de Primeira Classe

Funções são valores como qualquer outro — têm tipo, podem ser atribuídas e passadas:

```go
// Tipo de função
type Transformacao func(int) int

// Atribuir
dobrar := func(n int) int { return n * 2 }
var t Transformacao = dobrar

// Higher-order functions
func aplicar(nums []int, fn func(int) int) []int {
	resultado := make([]int, len(nums))
	for i, n := range nums {
		resultado[i] = fn(n)
	}
	return resultado
}

dobrados := aplicar([]int{1, 2, 3}, func(n int) int { return n * 2 })
```

---

## 7. Closures

Uma closure é uma função que **captura variáveis do ambiente lexical** onde foi definida. O compilador aloca essas variáveis na heap (não na stack), pois sua vida útil ultrapassa a função que as criou:

```go
func criarContador() func() int {
	n := 0   // alocado na HEAP (não na stack) — escape analysis detecta o escape
	return func() int {
		n++      // captura n por referência
		return n
	}
}

c1 := criarContador()
fmt.Println(c1())   // 1
fmt.Println(c1())   // 2

c2 := criarContador()   // novo n independente na heap
fmt.Println(c2())   // 1
```

### Armadilha Clássica em Loops (antes do Go 1.22)

```go
// ❌ Bug clássico (antes do Go 1.22)
funcs := make([]func(), 3)
for i := 0; i < 3; i++ {
	funcs[i] = func() {
		fmt.Println(i)   // captura a VARIÁVEL i (referência), não o valor
	}
}
funcs[0]()   // 3 (não 0!)
funcs[1]()   // 3
funcs[2]()   // 3

// ✅ Correção pré-Go 1.22: criar nova variável a cada iteração
for i := 0; i < 3; i++ {
	i := i   // nova variável i que "sombra" a do loop
	funcs[i] = func() { fmt.Println(i) }
}

// ✅ Go 1.22+: cada iteração cria automaticamente uma nova variável
// O comportamento mudou — o código do ❌ agora imprime 0, 1, 2
```

---

## 8. `defer` — Execução Adiada

`defer` adia a execução para quando a função retornar. É implementado como uma lista ligada de "deferred calls" no goroutine frame:

```go
func lerArquivo(caminho string) (string, error) {
	f, err := os.Open(caminho)
	if err != nil {
		return "", err
	}
	defer f.Close()   // executa quando lerArquivo retornar, em qualquer caso

	dados, err := io.ReadAll(f)
	if err != nil {
		return "", err   // f.Close() ainda é chamado!
	}
	return string(dados), nil
}
```

### Ordem LIFO e Avaliação de Argumentos

```go
func exemplo() {
	defer fmt.Println("3º")
	defer fmt.Println("2º")
	defer fmt.Println("1º")
}
// Imprime: 1º, 2º, 3º (ordem inversa)

// Argumentos avaliados NO MOMENTO do defer (não na execução)
x := 10
defer fmt.Println("x =", x)   // captura x=10 agora
x = 99
// No retorno imprime: x = 10 (não 99)

// Para capturar o valor final, use uma closure
defer func() { fmt.Println("x =", x) }()   // captura x por referência
x = 99
// Imprime: x = 99
```

### Defer com Retornos Nomeados — Modificar o Erro

```go
func executar() (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic capturado: %v", r)
		}
	}()
	// pode panic
	return
}
```

### Performance de Defer

Defer teve overhead significativo em versões antigas. Em Go 1.14+, o compilador implementou **"open-coded defer"**: para funções com poucos defers conhecidos em tempo de compilação, o compilador os inlina diretamente no código, eliminando o overhead da lista ligada.

---

## 9. `panic` e `recover`

```go
// panic — interrompe a execução imediatamente, propaga pela call stack
func validarPorta(porta int) {
	if porta < 1 || porta > 65535 {
		panic(fmt.Sprintf("porta inválida: %d", porta))
	}
}

// recover — captura o panic em um defer (único lugar onde funciona)
func executarSeguro(fn func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			switch v := r.(type) {
			case error:
				err = fmt.Errorf("panic: %w", v)
			default:
				err = fmt.Errorf("panic: %v", v)
			}
		}
	}()
	fn()
	return
}
```

> ⚠️ Use panic/recover apenas em situações excepcionais — bugs de programação, estados impossíveis. Para erros esperados, retorne `error`.
> 

---

## 10. `init()` — Inicialização de Pacote

```go
package config

var DB *sql.DB

func init() {
	var err error
	DB, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatalf("falha ao conectar: %v", err)
	}
}

// Múltiplos init() em um pacote — todos executam, na ordem de aparição
// Múltiplos arquivos — os inits executam na ordem de compilação dos arquivos
// (Use sparingly — dificulta o teste e a inicialização controlada)
```

---

## 11. Boas Práticas

```go
// ✅ Nomes de funções: verbos, descritivos
func calcularMedia(nums []float64) float64 { ... }
func validarEmail(email string) error { ... }

// ✅ Funções pequenas com uma responsabilidade
// Máximo que cabe visualmente na tela (≈ 30-40 linhas)

// ✅ Retorno antecipado (guard clauses)
func processar(dados []byte) error {
	if len(dados) == 0 {
		return errors.New("dados vazios")
	}
	// lógica sem aninhamento
	return nil
}

// ✅ Quando há muitos parâmetros, use struct de opções
type OpcoesConexao struct {
	Host     string
	Porta    int
	Timeout  time.Duration
	TLS      bool
}
func conectar(opts OpcoesConexao) (*Conn, error) { ... }

// ✅ Funções que retornam funções — para configuração lazy
func comTimeout(d time.Duration) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.TimeoutHandler(next, d, "timeout")
	}
}
```

---

## 12. Conexão com Sistemas Operacionais

### Calling Convention — Como Argumentos Chegam à Função

Toda linguagem precisa de uma **ABI (Application Binary Interface)** que define como argumentos e retornos transitam entre funções. Em Go, essa convenção mudou radicalmente na versão 1.17:

```
Antes do Go 1.17 — stack-based (igual ao C cdecl):
                    ┌─────────────┐  ← SP (Stack Pointer)
                    │   arg b     │
                    │   arg a     │
                    │ return addr │  ← CALL empilha aqui
                    │ saved RBP   │
                    │ local vars  │
                    └─────────────┘

Go 1.17+ em AMD64 — register-based ABI:
  args de entrada: AX, BX, CX, DI, SI, R8, R9, R10, R11
  retornos:        AX, BX, CX, DI, SI, R8, R9, R10, R11
  float:           X0–X14 (registradores SSE)
  overflow:        stack (argumentos que não cabem em registradores)
```

Veja no assembly real — `go tool compile -S main.go`:

```asm
// func somar(a, b int) int { return a + b }
// a chega em AX, b em BX
TEXT main.somar(SB)
    ADDQ    BX, AX    // AX = a + b
    RET               // retorna com resultado em AX
```

O Linux x86-64 System V ABI (usado pelo kernel) usa: `RDI, RSI, RDX, RCX, R8, R9` para argumentos e `RAX` para retorno — Go deliberadamente escolheu registradores diferentes para facilitar a distinção nos traces. Ver [[Processadores]].

---

### Stack Frame — O Que Acontece em Cada Chamada de Função

Cada chamada de função cria um **stack frame** — uma região de memória na stack do goroutine:

```
Memória da stack cresce para baixo (endereços decrescentes):

Antes de CALL:
  [caller frame]
  ┌─────────────────┐ ← RSP (Stack Pointer) do caller
  
Instrução CALL somar (= PUSH RIP; JMP somar):
  ┌─────────────────┐
  │  return address │ ← endereço da próxima instrução do caller
  └─────────────────┘ ← novo RSP

Dentro de somar — prólogo da função:
  ┌─────────────────┐
  │  return address │
  │  saved RBP      │ ← PUSH RBP; MOV RBP, RSP
  │  variável local │
  │  variável local │
  └─────────────────┘ ← RSP ajustado (SUB RSP, N)

Instrução RET (= POP RIP; JMP RIP):
  restaura RSP, volta para o return address
```

No modelo de [[Processos]], a call stack é a região da memória de um processo/thread dedicada a armazenar frames de função. O OS aloca a stack com tamanho fixo (tipicamente 1–8 MB por thread POSIX).

---

### Stack de Goroutine vs Stack de Thread POSIX

Esta é uma das diferenças fundamentais entre Go e linguagens que usam threads do OS diretamente:

```
Thread POSIX (criada com pthread_create):
  Stack alocada pelo OS — tamanho FIXO em tempo de criação
  Padrão Linux: 8 MB por thread
  Configurável: ulimit -s ou pthread_attr_setstacksize()
  
  Problema: criar 10.000 threads = 80 GB de stack reservado
  (mesmo que cada thread use apenas 1 KB)

Goroutine (gerenciada pelo runtime Go):
  Stack começa em 2 KB (Go 1.4+, antes era 8 KB)
  Cresce dinamicamente quando necessário (segmented stacks → Go 1.3)
  Go 1.3+ usa "stack copying": quando a stack esgota,
  o runtime aloca uma nova stack 2x maior e copia tudo
  
  Isso permite ter 1.000.000 de goroutines com pouca memória

  Stack mínima:    2 KB   (por goroutine)
  Stack máx padrão: 1 GB  (configurável via runtime/debug.SetMaxStack)
```

A implementação de goroutines como **threads em user space** é exatamente o que [[Implementando Threads em User Space]] descreve — o runtime Go é o scheduler que multiplexa goroutines em threads do OS (modelo M:N).

---

### Instrução CALL e RET — Nível de Hardware

A chamada de função, por baixo de tudo, é implementada por duas instruções da CPU:

```
CALL destino:
  1. Empilha o endereço da próxima instrução (RIP/PC) na stack
     → RSP = RSP - 8
     → [RSP] = RIP atual
  2. Salta para "destino"
     → RIP = destino

RET:
  1. Desempilha o endereço de retorno
     → RIP = [RSP]
     → RSP = RSP + 8
  2. Executa a partir do endereço desempilhado

O registrador RIP (Instruction Pointer / Program Counter) sempre
aponta para a PRÓXIMA instrução a ser executada.
```

O **Program Counter (PC)** e o **Stack Pointer (SP)** são os dois registradores mais críticos para entender chamadas de função em qualquer arquitetura. Ver [[Processadores]].

---

### Inlining — Eliminando o Overhead de CALL/RET

O compilador Go automaticamente **inlina** funções pequenas, eliminando o overhead de CALL/RET e melhorando a previsão de branch:

```go
// Esta função SERÁ inlinada (leaf, pequena, sem loops):
func quadrado(n int) int { return n * n }

// Chamada:
x := quadrado(5)

// Após inlining — o compilador substitui pela lógica direta:
x := 5 * 5   // CALL quadrado não existe no binário final
```

Para ver o que o compilador inlina:

```bash
go build -gcflags="-m" ./...
# ./main.go:3:6: can inline quadrado
# ./main.go:7:13: inlining call to quadrado
```

O inlining tem benefícios além de eliminar CALL/RET:
1. Elimina o setup/teardown do stack frame
2. Permite ao compilador otimizar o código inlinado com contexto do caller
3. Melhora **branch prediction** — a CPU não precisa prever saltos indiretos

Ver [[Processadores]] para entender o pipeline e por que branch misprediction é custoso (15–20 ciclos de penalidade em CPUs modernas).