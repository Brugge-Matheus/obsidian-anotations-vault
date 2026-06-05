---
tags:
  - go
  - go/fundamentos
---
# Variáveis e constantes

> Em Go, toda variável tem tipo estático definido em compilação e todo valor tem um lugar bem definido na memória. O compilador aplica escape analysis para decidir onde alocar — stack ou heap — de forma transparente, sem que o programador precise gerenciar memória manualmente.
> 

---

## 1. Declaração de Variáveis

### `var` — Declaração Explícita

```go
// var nome tipo = valor
var nome string = "Alice"
var idade int = 30
var ativo bool = true

// Tipo inferido
var pi = 3.14159   // float64

// Múltiplas variáveis — bloco var
var (
	nome  string = "Alice"
	idade int    = 30
	email string            // zero value: ""
)

// Zero values — toda variável não inicializada começa com o zero do seu tipo
var n int       // 0
var s string    // ""
var b bool      // false
var p *int      // nil
var sl []int    // nil (len=0, cap=0, ptr=nil)
var m map[string]int  // nil
var fn func()   // nil
var ch chan int  // nil
var i interface{}  // nil
var err error   // nil (error é uma interface)
```

### `:=` — Declaração Curta (Short Variable Declaration)

Apenas **dentro de funções**. É a forma mais comum no dia a dia:

```go
func exemplo() {
	// Declaração com inferência de tipo
	nome := "Alice"     // string
	idade := 30         // int
	pi := 3.14          // float64
	ativo := true       // bool

	// Múltiplas variáveis em uma linha
	x, y := 10, 20
	resultado, err := calcular(x, y)

	// Reatribuição com := — PELO MENOS UMA variável deve ser nova
	nome, sobrenome := "Bob", "Silva"   // nome reatribuído, sobrenome é novo
	// nome, idade := "Carol", 25       // ❌ erro: nenhuma variável nova

	_ = nome
	_ = resultado
	_ = err
	_ = sobrenome
}
```

> ⚠️ `:=` não pode ser usado no escopo de pacote (fora de funções). Para variáveis globais, use sempre `var`.
> 

---

## 2. Stack vs Heap — Escape Analysis

O compilador de Go usa **escape analysis** para decidir onde alocar cada variável. Isso é transparente ao programador, mas entender o mecanismo ajuda a escrever código mais eficiente:

```
Stack:                          Heap:
- Cada goroutine tem sua stack  - Gerenciada pelo GC
- Começa com 2KB, cresce        - Mais lenta (sincronização)
- Alocação: mover stack ptr     - Necessária quando:
- Desalocação: automática         · Variável escapa da função
  quando função retorna           · Tamanho desconhecido em compilação
- Muito mais rápida               · Interface com tipo dinâmico
```

```go
// Fica na STACK — não escapa
func naoEscapa() int {
	x := 42
	return x   // x é copiado no retorno — o original pode ficar na stack
}

// Vai para a HEAP — escapa (retorna ponteiro)
func escapa() *int {
	x := 42
	return &x   // alguém vai usar &x depois da função — precisa sobreviver
}

// Vai para a HEAP — closure captura por referência
func criarClosure() func() int {
	n := 0   // n precisa sobreviver à função criarClosure → heap
	return func() int {
		n++
		return n
	}
}

// Interface vazia pode forçar escape (boxing)
func interfaceEscape() {
	x := 42
	var i interface{} = x   // x pode ser copiado para heap (boxing)
	fmt.Println(i)
}

// Verificar escape analysis:
go build -gcflags="-m" ./...
// "./main.go:8:2: moved to heap: x"
// "./main.go:15:2: moved to heap: n"
```

---

## 3. Zero Values — A Segurança de Go

Todo tipo em Go tem um zero value bem definido. **Nunca há lixo de memória**:

```go
// Tipos básicos
var i int          // 0
var f float64      // 0.0
var b bool         // false
var s string       // "" (string vazia, não nil!)
var r rune         // 0 (U+0000)

// Tipos compostos
var p *int              // nil (ponteiro)
var sl []int            // nil slice (len=0, cap=0, ptr=nil)
var m map[string]int    // nil map (leitura ok, escrita → panic!)
var fn func()           // nil func
var ch chan int          // nil channel

// Interfaces
var err error           // nil (interface nil)
var i interface{}       // nil

// Struct — cada campo recebe seu zero value
type Servidor struct {
	Host    string
	Porta   int
	TLS     bool
}
var s Servidor   // Servidor{Host:"", Porta:0, TLS:false}

// Zero values permitem uso imediato de muitos tipos:
var mu sync.Mutex    // pronto para uso — sem New()
var wg sync.WaitGroup  // pronto para uso
var sb strings.Builder  // pronto para uso
var buf bytes.Buffer    // pronto para uso
```

---

## 4. Tipos Numéricos — Tamanhos e Layout de Memória

```go
// Inteiros signed — complemento de dois
// int8:  1 byte  — -128 a 127
// int16: 2 bytes — -32.768 a 32.767
// int32: 4 bytes — -2.147.483.648 a 2.147.483.647
// int64: 8 bytes — -9.2×10¹⁸ a 9.2×10¹⁸
// int:   4 ou 8 bytes (depende da plataforma — 64-bit hoje em dia)

// Inteiros unsigned — representação natural
// uint8/byte:  1 byte  — 0 a 255
// uint16:      2 bytes — 0 a 65.535
// uint32:      4 bytes — 0 a 4.294.967.295
// uint64:      8 bytes — 0 a 18.4×10¹⁸
// uint:        4 ou 8 bytes
// uintptr:     tamanho de um ponteiro (8 bytes em 64-bit)

// Floats IEEE 754
// float32: 4 bytes — ~7 dígitos decimais de precisão
// float64: 8 bytes — ~15 dígitos decimais de precisão (padrão em Go)

// Aliases importantes
// byte = uint8  (dados brutos, arrays de bytes)
// rune = int32  (code points Unicode)
// any  = interface{} (Go 1.18+)

// Verificar tamanhos em runtime
import "unsafe"
fmt.Println(unsafe.Sizeof(int(0)))      // 8 (64-bit)
fmt.Println(unsafe.Sizeof(float64(0))) // 8
fmt.Println(unsafe.Sizeof(string(""))) // 16 (ptr + len)
fmt.Println(unsafe.Sizeof([]int{}))    // 24 (ptr + len + cap)
fmt.Println(unsafe.Sizeof(interface{}{})) // 16 (tipo + valor)
```

---

## 5. Constantes

Constantes são avaliadas em **tempo de compilação** — têm tipo "untyped" quando não declarado explicitamente, o que as torna mais flexíveis:

```go
// Constantes untyped — mais precisas e flexíveis
const Pi = 3.14159265358979323846   // precisão arbitrária
const E  = 2.71828182845904523536

// Untyped constants têm "kind" em vez de tipo específico:
// - untyped integer: 42, -7, 0xFF
// - untyped float: 3.14, 1e-9
// - untyped string: "hello"
// - untyped bool: true, false
// - untyped rune: 'A'

const N = 1000
var x int    = N   // ok — N é untyped, cabe em int
var y float64 = N  // ok — N é untyped, cabe em float64
var z int64   = N  // ok

// Constante tipada — menos flexível
const Pi64 float64 = 3.14159
// var n int = Pi64   // ❌ erro — Pi64 é float64, não int

// Grupo de constantes
const (
	StatusAtivo   = "ativo"
	StatusInativo = "inativo"
	StatusBloqueado = "bloqueado"
)
```

### `iota` — Gerador Sequencial

```go
type Nivel int

const (
	NivelDebug   Nivel = iota   // 0
	NivelInfo                   // 1
	NivelWarn                   // 2
	NivelError                  // 3
	NivelFatal                  // 4
)

// Com expressões
type ByteSize float64
const (
	_           = iota             // ignora 0
	KB ByteSize = 1 << (10 * iota) // 1 << 10 = 1024
	MB                             // 1 << 20 = 1.048.576
	GB                             // 1 << 30
	TB                             // 1 << 40
)

// Bitmask flags
type Permissao uint

const (
	Leitura   Permissao = 1 << iota   // 0001 = 1
	Escrita                           // 0010 = 2
	Execucao                          // 0100 = 4
)

// Verificar permissões
func tem(usuario, perm Permissao) bool {
	return usuario&perm != 0
}

p := Leitura | Escrita   // 0011 = 3
fmt.Println(tem(p, Leitura))   // true
fmt.Println(tem(p, Execucao))  // false

// String() para enums com iota
func (n Nivel) String() string {
	return [...]string{"DEBUG", "INFO", "WARN", "ERROR", "FATAL"}[n]
}
fmt.Println(NivelWarn)   // "WARN"
```

---

## 6. Conversões de Tipo — Sempre Explícitas

```go
var i int32 = 42
var j int64 = int64(i)   // ✅ explícito
// var k int64 = i      // ❌ erro de compilação

var f float64 = 3.14
var n int = int(f)       // ✅ trunca para 3 (não arredonda)

// String conversions
r := rune('A')           // 65
s := string(r)           // "A"

n2 := 65
// s2 := string(n2)      // ⚠️ Deprecated em Go 1.15 — string(65) = "A", não "65"!
s3 := strconv.Itoa(n2)   // "65" — correto

b := []byte("hello")     // [104 101 108 108 111]
s4 := string(b)          // "hello"

// Conversões numéricas podem perder informação
var grande int64 = 300
var pequeno int8 = int8(grande)  // 44 — overflow silencioso!
```

---

## 7. Variáveis de Pacote — Escopo Global

```go
package config

import "time"

// Variáveis de pacote — alocadas na heap, vivem durante toda a execução
// Evite ao máximo — dificulta testes e rastreamento
var (
	DefaultTimeout = 30 * time.Second
	MaxRetries     = 3
)

// Constantes de pacote — preferíveis quando o valor não muda
const (
	Version = "1.2.3"
	AppName = "meu-servidor"
)

// Para valores que precisam ser configurados, prefira injeção via construtor:
type App struct {
	timeout time.Duration   // configurado no construtor
	retries int
}

func NewApp(timeout time.Duration, retries int) *App {
	return &App{timeout: timeout, retries: retries}
}
```

---

## 8. Ponteiros

```go
x := 42
p := &x              // p é *int — endereço de x
fmt.Println(*p)      // 42 — dereferenciar: ler o valor
*p = 99              // modificar o valor via ponteiro
fmt.Println(x)       // 99

// new — aloca zero value na heap e retorna ponteiro
n := new(int)        // *int → 0 na heap
*n = 100
fmt.Println(*n)      // 100

// Ponteiro nil — deref causa panic
var pNil *int
// *pNil = 1        // panic: nil pointer dereference

// Go NÃO tem aritmética de ponteiro (exceto via unsafe.Pointer)
// Isso é uma decisão de segurança — sem buffer overflows

// Comparação de ponteiros
a, b := 42, 42
pa, pb := &a, &b
fmt.Println(pa == pb)   // false — endereços diferentes
fmt.Println(*pa == *pb) // true — valores iguais
pc := pa
fmt.Println(pa == pc)   // true — mesmo endereço
```

---

## 9. Blank Identifier `_`

```go
// Descartar retornos não usados
resultado, _ := dividir(10, 2)   // ignora error — apenas quando tem certeza!
_, ok := mapa["chave"]           // ignora valor, pega apenas ok

// Importar apenas pelos side effects (init())
import _ "github.com/lib/pq"      // registra driver PostgreSQL

// Verificar implementação de interface em compilação
var _ io.Reader = (*MeuReader)(nil)  // erro se *MeuReader não implementar io.Reader

// Ignorar variáveis de loop
for _, v := range slice { fmt.Println(v) }
for i := range slice { fmt.Println(i) }  // _ implícito para valor
```

---

## 10. Resumo

| Conceito | Detalhe |
| --- | --- |
| `:=` | Apenas dentro de funções |
| `var` | Dentro e fora de funções |
| Zero value | Toda variável começa inicializada — sem lixo |
| Tipo | Sempre estático, definido em compilação |
| Conversão | Sempre explícita — nunca implícita |
| Stack vs heap | Decidido pelo compilador (escape analysis) |
| Constantes | Avaliadas em compilação — sem overhead de runtime |
| Untyped const | Mais flexíveis que typed — se encaixam em múltiplos tipos |
| `iota` | Gerador sequencial dentro de bloco `const` |
| `_` | Descarta valores sem uso, side-effect imports, verificação de interface |
| Ponteiros | Sem aritmética (exceto `unsafe`) — segurança garantida |

---

## Conexão com Sistemas Operacionais

### Stack vs Heap — o mesmo modelo de [[Processos]]

O modelo de memória que o Go usa é exatamente o descrito em [[Processos]]: cada goroutine possui uma **pilha (stack)** para contexto de execução e variáveis locais, e existe uma **heap** global gerenciada para dados de vida longa. A diferença prática entre `naoEscapa()` e `escapa()` neste arquivo é a mesma diferença entre uma variável que vive apenas enquanto o frame de função está ativo e uma que exige `malloc` — em Go o compilador faz essa decisão automaticamente via escape analysis, mas o resultado em memória é idêntico ao modelo C/Unix estudado em [[Processos]].

### Goroutine stack vs thread stack — [[Processos]]

Conforme mostrado na seção 2, a goroutine começa com **2 KB de stack** e cresce sob demanda (Go usa stacks segmentadas ou copiadas). Uma thread de SO típica começa com 1 a 8 MB de stack fixo, o que é um dos motivos de threads serem caras. Em [[Processos]] vimos que o **stack overflow** ocorre quando a pilha cresce além do limite — em Go esse limite é dinâmico: o runtime detecta que a stack está prestes a transbordar e aloca uma nova stack maior, copiando o conteúdo. Isso elimina stack overflow acidental sem desperdiçar memória como uma thread tradicional faz.

### Escape analysis e alocação na heap — [[Processos]]

Quando o compilador detecta que uma variável "escapa" da função (e.g., `return &x`), ele emite uma chamada equivalente a `malloc` no runtime do Go. O ponteiro retornado aponta para a heap do processo — o mesmo segmento de memória que cresce via `brk()` ou `mmap()` em [[Processos]]. O GC é responsável pelo equivalente do `free()`.

### Variáveis globais de pacote = segmento `.data` / `.bss` — [[Arquivos de Cabeçalho e o Modelo de Execução]]

As variáveis declaradas no escopo de pacote (seção 7) não ficam na stack nem passam pelo GC da mesma forma: elas residem nos segmentos estáticos do binário ELF. Se inicializadas com valor, vão para o segmento `.data`; se zero value, vão para `.bss`. Isso é exatamente o que [[Arquivos de Cabeçalho e o Modelo de Execução]] descreve ao cobrir os segmentos de um binário compilado. Constantes de pacote (`const`) geralmente ficam incorporadas diretamente no segmento de código ou em `.rodata`.

### Zero values = sem lixo de memória — [[Processos]]

Em [[Processos]] vimos que o SO **zera as páginas de memória** antes de entregá-las a um processo, por segurança (um processo não deve ler dados de outro). O comportamento de zero values em Go (seção 3) é o reflexo dessa garantia no nível da linguagem: toda variável começa no zero do seu tipo porque a memória subjacente já foi zerada. Isso elimina a classe de bugs de "lixo de memória não inicializada" comum em C.

### Aritmética de ponteiro proibida — [[Espaços de Endereçamento]]

Go não permite aritmética de ponteiro direta (exceto via `unsafe.Pointer`). Isso conecta diretamente ao modelo de [[Espaços de Endereçamento]]: a MMU protege regiões de memória e acessar um endereço arbitrário causaria uma falha de segmentação (page fault → SIGSEGV). Ao proibir aritmética de ponteiro, Go garante que o programador só acessa endereços válidos dentro dos limites do seu espaço de endereçamento virtual — sem buffer overflows como os que afetam código C.

### `uintptr` = endereço virtual — [[Espaços de Endereçamento]]

O tipo `uintptr` (mencionado na seção 4 como "tamanho de um ponteiro") é literalmente um inteiro que representa um **endereço virtual** no espaço de endereçamento do processo. Como descrito em [[Espaços de Endereçamento]], cada processo enxerga um espaço de endereçamento virtual isolado; `uintptr` é a representação Go desse endereço. O tipo tem 8 bytes em arquiteturas de 64 bits porque o barramento de endereços é de 64 bits — embora na prática apenas 48 bits sejam usados em x86-64.