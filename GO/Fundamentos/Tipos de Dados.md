---
tags:
  - go
  - go/fundamentos
---
# Tipos de Dados

> Go é estaticamente tipado — cada variável tem um tipo fixo em tempo de compilação. O sistema de tipos é simples e ortogonal: há tipos básicos, tipos compostos, e tipos definidos pelo usuário. O compilador usa essa informação para verificações de segurança e otimizações de código.
> 

---

## 1. Tipos Básicos (Primitivos)

### Booleano

```go
var b bool = true
var f bool = false

// Apenas true e false — sem truthy/falsy como em JS/Python/Ruby
// if 0 { }     ❌ erro de compilação em Go
// if "" { }    ❌ erro de compilação em Go
if b { }        // ✅ apenas bool
```

### Inteiros

```go
// Signed — guardam negativos e positivos
int8    // 1 byte:  -128        a 127
int16   // 2 bytes: -32.768     a 32.767
int32   // 4 bytes: -2.147.483.648    a 2.147.483.647
int64   // 8 bytes: -9.2×10¹⁸  a 9.2×10¹⁸

// Unsigned — apenas positivos, dobram o range máximo
uint8   // 1 byte:  0 a 255
uint16  // 2 bytes: 0 a 65.535
uint32  // 4 bytes: 0 a 4.294.967.295
uint64  // 8 bytes: 0 a 18.4×10¹⁸

// Dependentes da plataforma (tamanho = pointer size do OS)
int     // 32 ou 64 bits
uint    // 32 ou 64 bits
uintptr // grande o suficiente para um ponteiro (para aritmética com unsafe)

// Aliases com semântica específica
byte = uint8   // dados brutos, I/O
rune = int32   // Unicode code point
```

**Representação na memória:**

```
int8:   [bbbbbbbb]                    — 8 bits, complemento de dois
int32:  [bbbbbbbb bbbbbbbb bbbbbbbb bbbbbbbb]   — 32 bits
int64:  [64 bits, little-endian no x86-64]

// Complemento de dois: -1 é representado como todos os bits em 1
// Isso permite overflow/underflow previsível (wrap-around silencioso)
var x int8 = 127
x++
// x = -128  (0111_1111 + 1 = 1000_0000 = -128 em complemento de dois)
```

### Floats

Go usa **IEEE 754** para representação de ponto flutuante:

```go
float32   // 4 bytes: ~7  dígitos decimais de precisão, range ±3.4×10³⁸
float64   // 8 bytes: ~15 dígitos decimais de precisão, range ±1.8×10³⁰⁸

// float64 é o padrão — use float32 apenas quando necessário por memória/API
var f float64 = 3.14
var g = 3.14    // inferido como float64

// Valores especiais IEEE 754
import "math"
posInf := math.Inf(1)    // +Inf
negInf := math.Inf(-1)   // -Inf
nan    := math.NaN()     // NaN

// NaN != NaN (por definição IEEE 754)
fmt.Println(nan == nan)          // false!
fmt.Println(math.IsNaN(nan))     // true — forma correta de verificar

// Precisão — por que 0.1 + 0.2 != 0.3
fmt.Println(0.1 + 0.2)          // 0.30000000000000004
fmt.Println(0.1 + 0.2 == 0.3)   // false!

// Para comparar floats, use epsilon
const epsilon = 1e-9
func iguaisFloat(a, b float64) bool {
	return math.Abs(a-b) < epsilon
}
```

### Números Complexos

```go
complex64   // partes float32
complex128  // partes float64

c1 := complex(3, 4)     // 3+4i
c2 := 2 + 3i             // sintaxe literal

fmt.Println(real(c1))    // 3
fmt.Println(imag(c1))    // 4

// Operações
soma := c1 + c2
produto := c1 * c2
modulo := math.Sqrt(real(c1)*real(c1) + imag(c1)*imag(c1))  // = 5
```

---

## 2. String

```go
// String: sequência imutável de bytes UTF-8
// Internamente: struct { ptr *byte; len int }

s := "hello, 世界"

len(s)           // 13 — bytes (não caracteres!)
s[0]             // 104 — byte, não caractere
string(s[0])     // "h"

// Iteração por bytes (pode quebrar multibyte)
for i := 0; i < len(s); i++ {
	fmt.Printf("%d: %x\n", i, s[i])
}

// Iteração por runes (correta para Unicode)
for i, r := range s {
	fmt.Printf("offset %d: %c\n", i, r)
}

// Subtring é O(1) — novo header, mesmo array de bytes
sub := s[0:5]   // "hello"
```

---

## 3. Tipos Compostos

### Array `[N]T`

```go
// Tamanho fixo, parte do tipo, valor semântico (cópia ao atribuir)
var a [5]int            // [0, 0, 0, 0, 0]
b := [3]string{"a", "b", "c"}
c := [...]int{1, 2, 3}  // tamanho inferido pelo compilador

// sizeof: N × sizeof(T)
// [5]int64 = 5 × 8 = 40 bytes, contíguos na memória
```

### Slice `[]T`

```go
// Header de 24 bytes (ptr + len + cap), aponta para array subjacente
s := []int{1, 2, 3, 4, 5}
s2 := make([]int, 3, 5)   // len=3, cap=5
s3 := s[1:4]               // view: [2, 3, 4]
```

### Map `map[K]V`

```go
// Hash table — O(1) amortizado para operações básicas
m := make(map[string]int)
m["a"] = 1
v, ok := m["a"]
```

### Struct

```go
// Campos contíguos na memória, com padding para alinhamento
type Ponto struct {
	X, Y float64
}
// sizeof(Ponto) = 16 bytes (dois float64 de 8 bytes cada)
```

### Ponteiro `*T`

```go
// Endereço de memória de um valor T
x := 42
p := &x    // *int
*p = 99    // deref: modifica x
```

### Função `func`

```go
// Funções são valores de primeira classe
var fn func(int) int = func(n int) int { return n * 2 }
```

### Interface

```go
// Par dinâmico (tipo, valor) — permite polimorfismo
var r io.Reader = os.Stdin
```

### Channel `chan T`

```go
// Pipe tipado para comunicação entre goroutines
ch := make(chan int, 10)
ch <- 42
v := <-ch
```

---

## 4. Tipos Definidos pelo Usuário

### `type` — Criar Novo Tipo

```go
// type NovoNome TipoBase
type Celsius float64
type Fahrenheit float64
type UserID int64

// NovoTipo é DISTINTO do tipo base — sem conversão implícita
var temp Celsius = 100
var f float64 = temp   // ❌ erro de compilação

// Conversão explícita
var f2 float64 = float64(temp)   // ✅

// Vantagem: pode adicionar métodos
func (c Celsius) ToFahrenheit() Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}
```

### `type` com `~` em Type Constraints (Go 1.18+)

```go
// ~ significa "qualquer tipo cujo tipo subjacente é X"
type Numerico interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
	~uint | ~uint8 | ~float32 | ~float64
}

// Isso permite que tipos definidos pelo usuário também satisfaçam:
type Celsius float64   // satisfaz Numerico (subjacente é float64)
type UserID int64      // satisfaz Numerico (subjacente é int64)
```

### Type Alias (Go 1.9+)

```go
// type Alias = TipoOriginal — mesmo tipo, apenas nome diferente
type MyString = string   // MyString e string são exatamente o mesmo tipo

// byte e rune são aliases da stdlib:
// type byte = uint8
// type rune = int32
// type any  = interface{}  (Go 1.18+)
```

---

## 5. Tipo `any` (Go 1.18+)

```go
// any é alias de interface{} — pode guardar qualquer valor
var v any

v = 42
v = "texto"
v = []int{1, 2, 3}
v = struct{ Nome string }{"Alice"}

// Para recuperar o valor, use type assertion
if s, ok := v.(string); ok {
	fmt.Println(s)
}
```

---

## 6. Tamanhos e Alinhamento

```go
import "unsafe"

// Tamanhos em bytes no AMD64
fmt.Println(unsafe.Sizeof(bool(false)))    // 1
fmt.Println(unsafe.Sizeof(int8(0)))        // 1
fmt.Println(unsafe.Sizeof(int16(0)))       // 2
fmt.Println(unsafe.Sizeof(int32(0)))       // 4
fmt.Println(unsafe.Sizeof(int64(0)))       // 8
fmt.Println(unsafe.Sizeof(int(0)))         // 8 (64-bit)
fmt.Println(unsafe.Sizeof(float32(0)))     // 4
fmt.Println(unsafe.Sizeof(float64(0)))     // 8
fmt.Println(unsafe.Sizeof(string("")))     // 16 (ptr + len)
fmt.Println(unsafe.Sizeof([]int{}))        // 24 (ptr + len + cap)
fmt.Println(unsafe.Sizeof(map[int]int{}))  // 8 (ponteiro para hmap)
fmt.Println(unsafe.Sizeof(func(){}))       // 8 (ponteiro para código)
fmt.Println(unsafe.Sizeof(interface{}{}))  // 16 (tipo + valor)
fmt.Println(unsafe.Sizeof(chan int(nil)))   // 8 (ponteiro para hchan)
```

---

## 7. Resumo — Quando Usar Cada Tipo Numérico

| Situação | Tipo Recomendado |
| --- | --- |
| Contadores, índices, tamanhos | `int` |
| Tamanhos de arquivo, offsets | `int64` |
| Dados binários, bytes | `byte` (`uint8`) |
| Caracteres Unicode | `rune` (`int32`) |
| IDs de banco de dados | `int64` |
| Timestamps Unix | `int64` |
| Flags de bitmask | `uint` ou `uint64` |
| Cálculos financeiros | `int64` (centavos) ou `decimal` (lib externa) |
| Coordenadas, física, matemática | `float64` |
| Buffers de GPU, audio 32-bit | `float32` |
| Paridade com API C/OS | tamanho específico (`int32`, etc.) |

---

## Conexão com Sistemas Operacionais

### Tamanho de `int` e `uint` = word size do processador — [[Processadores]]

O fato de `int` ter 4 bytes em plataformas 32-bit e 8 bytes em 64-bit não é arbitrário: esse é exatamente o conceito de **word size** discutido em [[Processadores]]. A "palavra" da CPU é a unidade natural de processamento — um registrador de propósito geral em um CPU x86-64 tem 64 bits, então operações com `int` de 64 bits são executadas em um único ciclo de clock sem extensão de sinal. Usar `int32` forçado em uma máquina 64-bit pode até ser mais lento, pois o processador precisa mascarar os bits superiores.

### `uintptr` = tamanho de um ponteiro = tamanho de um registrador — [[Processadores]]

Em [[Processadores]] vimos que os registradores (PC, SP) têm o mesmo tamanho que a word da arquitetura. Um ponteiro em C ou Go precisa caber em um registrador para ser passado eficientemente entre funções. Por isso `uintptr` tem 8 bytes em 64-bit: o endereço virtual precisa caber no registrador de 64 bits que a CPU usa para operações de memória (MOV, LOAD, STORE).

### Complemento de dois — [[Bits e Bytes]]

A representação de inteiros com sinal em Go usa **complemento de dois**, como descrito em [[Bits e Bytes]]. O exemplo no arquivo (`int8(127) + 1 == -128`) é exatamente o wrap-around do complemento de dois: `0111_1111 + 1 = 1000_0000`, e em complemento de dois `1000_0000` representa -128. Isso não é um bug — é o comportamento definido da aritmética binária.

### Overflow silencioso — [[Processos]]

Go permite overflow/underflow de inteiros sem pânico (wrap-around silencioso). Em [[Processos]] vimos a distinção entre **overflow aritmético** (wrap-around de inteiros, como aqui) e **stack overflow** (estouro da pilha). São conceitos diferentes: overflow aritmético é definido e previsível em complemento de dois; stack overflow é uma condição de erro no gerenciamento de memória.

### `float64` IEEE 754 e a FPU — [[Processadores]]

Go usa IEEE 754 para floats (mencionado na seção 1). Em [[Processadores]], a **FPU (Floating-Point Unit)** é a unidade de hardware responsável por executar operações de ponto flutuante. Processadores modernos têm FPU integrada (via instruções SSE/AVX em x86-64), e `float64` é o formato nativo da FPU de 64 bits. Por isso `float64` é o padrão em Go — operações com `float32` em CPUs modernas frequentemente exigem conversão adicional.

### Alinhamento de memória — [[Processadores]]

Os requisitos de alinhamento dos tipos (um `int64` precisa estar em endereço múltiplo de 8) existem porque o **barramento de memória** da CPU transfere dados em blocos alinhados à word size. Como descrito em [[Processadores]], um acesso desalinhado pode exigir dois ciclos de barramento em vez de um, ou até causar uma exceção de hardware em arquiteturas mais restritas (como alguns ARMs). O compilador Go insere padding automaticamente nas structs para garantir que os campos fiquem alinhados.

### `sizeof(interface{})` = 16 bytes — dois ponteiros — [[Espaços de Endereçamento]]

Uma interface em Go ocupa 16 bytes: um ponteiro para o descritor de tipo (itab) e um ponteiro para o valor concreto. Ambos são endereços virtuais no espaço de endereçamento do processo, como descrito em [[Espaços de Endereçamento]]. Em uma máquina 64-bit cada ponteiro tem 8 bytes, totalizando 16 bytes por interface — independentemente do tipo concreto armazenado.

### `byte = uint8` em contextos de I/O — [[Arquivos]]

`byte` é alias de `uint8`. Em operações de I/O (leitura de arquivos, pipes, sockets), o kernel sempre opera com sequências de bytes crus — a menor unidade endereçável. Em [[Arquivos]] e [[System Calls]], as syscalls como `read()` e `write()` recebem um ponteiro para buffer de bytes e um tamanho. Por isso as interfaces de I/O em Go usam `[]byte` como tipo fundamental de transferência de dados.