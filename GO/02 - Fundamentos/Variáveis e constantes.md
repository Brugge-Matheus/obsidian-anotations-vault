---
tags:
  - go
  - go/fundamentos
aliases:
  - Variáveis em Go
  - Constantes em Go
  - iota
  - Zero value
ordem: 6
modulo: "02 - Fundamentos"
---
# Variáveis e constantes

> **Nota 6 da trilha de Go.** Variáveis guardam valores que mudam; constantes guardam valores fixos, conhecidos já na compilação. Nesta nota veremos **todas as formas de declarar** variáveis em Go, o conceito de **zero value** (uma das ideias mais importantes da linguagem), as regras do `:=`, as **constantes sem tipo** (que funcionam de um jeito bem diferente de outras linguagens), o gerador `iota` e o identificador vazio `_`.
>
> **Pré-requisitos:** [[Sintaxe, Blocos e Escopo]].

---

## 🧭 De onde viemos

Em [[Sintaxe, Blocos e Escopo]] vimos as seis declarações de Go (`package`, `import`, `const`, `var`, `type`, `func`) e as regras de escopo e shadowing. Agora vamos aprofundar duas delas: `var` e `const`. Ao longo da nota, vamos usar os tipos básicos (`int`, `float64`, `string`, `bool`) de forma intuitiva; eles serão estudados a fundo na próxima nota, [[Tipos de Dados]].

---

## 📦 1. O que é uma variável

> 💡 **Variável:** um local nomeado na memória que guarda um valor de um **tipo fixo**. O nome é para o programador; o compilador transforma cada variável em um endereço (na stack, na heap ou numa região global do binário) ou até mesmo num registrador da CPU.

Em Go, toda variável tem três propriedades definidas em tempo de compilação:

```
            var idade int = 30
                 │     │     │
      nome ──────┘     │     └────── valor inicial
                       │
      tipo ────────────┘  (fixo para sempre: idade nunca vai guardar uma string)
```

---

## ✍️ 2. As formas de declarar

### 2.1 `var` com tipo e valor

```go
var nome string = "Matheus"
var idade int = 30
```

### 2.2 `var` só com tipo — recebe o zero value

```go
var contador int     // 0
var ativo bool       // false
var titulo string    // ""
```

### 2.3 `var` só com valor — o tipo é inferido

```go
var pi = 3.14159     // float64 (inferido)
var nome = "Ana"     // string
```

> 💡 **Inferência de tipo:** o compilador deduz o tipo da variável a partir do valor inicial. A variável continua sendo estaticamente tipada, você só não precisou escrever o tipo.

### 2.4 Declaração curta `:=` (só dentro de funções)

É a forma **mais usada** no dia a dia:

```go
func main() {
	nome := "Matheus"         // string
	idade := 30               // int
	altura := 1.80            // float64
	x, y := 10, 20            // várias de uma vez
	valor, err := strconv.Atoi("42")   // muito comum com funções de 2 retornos
}
```

`:=` é exatamente `var nome = valor`, só que mais curto. Duas restrições:

- **Só dentro de funções.** No nível de pacote, use `var` (o motivo: toda declaração de nível de pacote começa com uma palavra-chave, o que mantém o parsing simples)
- **Precisa criar pelo menos uma variável nova** (veremos na seção 4)

### 2.5 Bloco `var`

```go
var (
	host    = "localhost"
	porta   = 8080
	timeout time.Duration          // zero value: 0
	debug   bool                   // false
)
```

### 2.6 Qual forma usar?

| Situação | Forma idiomática |
| --- | --- |
| Dentro de função, com valor inicial | `x := valor` |
| Dentro de função, quero o zero value | `var x T` |
| Tipo do valor não é o que eu quero | `var x float64 = 10` (senão seria `int`) |
| Nível de pacote | `var x = ...` ou bloco `var (...)` |
| Várias variáveis relacionadas | bloco `var (...)` |

> ✅ Prefira `var x T` a `x := T{}` ou `x := 0` quando a intenção é "começa vazio". Fica explícito que o zero value é o valor desejado.

---

## 0️⃣ 3. Zero value — não existe variável sem valor

Esta é uma das ideias mais importantes de Go:

> **Toda variável declarada sem valor inicial recebe automaticamente o *zero value* do seu tipo. Não existe memória "lixo" em Go.**

> 💡 **Zero value (valor zero):** o valor padrão de um tipo, que é o que você obtém quando todos os bits da memória daquele valor são zero. Para números é `0`, para `bool` é `false`, para strings é `""`, e para ponteiros, slices, maps, channels, funções e interfaces é `nil`.

Saída real:

```go
var a int
var b float64
var c string
var d bool
var e []int
var f map[string]int
var g *int
var h error
fmt.Printf("%v %v %q %v %v %v %v %v\n", a, b, c, d, e, f, g, h)
// 0 0 "" false [] map[] <nil> <nil>
```

| Tipo | Zero value |
| --- | --- |
| `int`, `int8`...`uint64`, `float32`, `float64`, `complex128`, `byte`, `rune` | `0` |
| `bool` | `false` |
| `string` | `""` (string vazia, **não** `nil`) |
| ponteiro `*T` | `nil` |
| slice `[]T` | `nil` (mas `len` = 0 e dá para iterar e fazer `append`) |
| map `map[K]V` | `nil` (leitura funciona, **escrita causa panic**) |
| channel `chan T` | `nil` |
| função `func(...)` | `nil` |
| interface (`error`, `any`, ...) | `nil` |
| array `[N]T` | N elementos com o zero value de T |
| struct | cada campo com seu zero value |

### 3.1 Por que isso importa: "make the zero value useful"

Compare com C:

```c
int contador;           // em C, variável local: valor INDEFINIDO (o que estava na memória)
contador++;             // comportamento imprevisível
```

Em Go, o mesmo código é sempre seguro: `contador` começa em `0`. E a biblioteca padrão é desenhada para que o zero value de vários tipos **já esteja pronto para uso**, sem construtor:

```go
var mu sync.Mutex         // mutex destravado, pronto
var wg sync.WaitGroup     // contador zerado, pronto
var sb strings.Builder    // builder vazio, pronto
var buf bytes.Buffer      // buffer vazio, pronto
var s []int               // slice nil, mas append funciona
s = append(s, 1)          // ✅ [1]
```

Esse é o provérbio *"Make the zero value useful"* que vimos em [[Convenções da linguagem]]. Quando você criar seus próprios tipos (em [[Structs]]), tente seguir o mesmo princípio.

### 3.2 Como o zero value é garantido

Não é mágica nem custo grande:

- **Variáveis globais** vão para a seção `.bss` do binário, que o kernel entrega **já zerada** quando carrega o programa (páginas anônimas novas sempre vêm zeradas, por segurança; ver [[Memória Virtual]])
- **Variáveis na stack** são zeradas pelo código que o compilador gera na entrada da função (quando necessário)
- **Variáveis na heap** são zeradas pelo alocador do runtime (que também reaproveita memória já zerada vinda do SO)

---

## 🔁 4. Atribuição, reatribuição e as regras do `:=`

### 4.1 `=` atribui, `:=` declara

```go
x := 10     // declara x e atribui 10
x = 20      // atribui 20 ao x que já existe
x := 30     // ❌ no new variables on left side of :=
```

### 4.2 Com várias variáveis, basta UMA ser nova

```go
a, err := passo1()
b, err := passo2()   // ✅ b é nova; err é REUTILIZADO (só recebe atribuição)
```

Isso existe justamente para o padrão de erros em sequência. **Mas atenção**: a reutilização só acontece se `err` estiver **no mesmo bloco**. Em um bloco interno, `:=` cria um `err` novo que sombreia o externo (a armadilha que vimos em [[Sintaxe, Blocos e Escopo]]).

### 4.3 Atribuição múltipla é avaliada "em paralelo"

O lado direito é **todo avaliado antes** de qualquer atribuição. Isso permite trocar valores sem variável temporária (saída real):

```go
x, y := 1, 2
x, y = y, x
fmt.Println(x, y)   // 2 1

a, b := 0, 1
for range 8 {
	fmt.Print(a, " ")
	a, b = b, a+b   // Fibonacci: os dois lados usam os valores ANTIGOS
}
// 0 1 1 2 3 5 8 13
```

### 4.4 Variável local não usada é erro

```go
func main() {
	a, b := 1, 2
	fmt.Println(a)
}
// ./main.go:4:5: declared and not used: b
```

A regra vale para variáveis **locais**. Variáveis de pacote, parâmetros de função e constantes não usadas **não** geram erro.

---

## 🕳️ 5. O identificador vazio `_`

O `_` aceita qualquer valor e o descarta. É a forma de dizer explicitamente "eu sei que existe esse valor e não quero usá-lo".

```go
// 1. Ignorar um dos retornos de uma função
n, _ := strconv.Atoi("42")         // ⚠️ ignorar erro só quando há certeza

// 2. Ignorar índice ou valor no range
for _, v := range lista { }
for i := range lista { }           // aqui nem precisa: omita o valor

// 3. Import só pelo efeito colateral (roda o init() do pacote)
import _ "github.com/lib/pq"

// 4. Garantir em compilação que um tipo implementa uma interface
var _ io.Reader = (*MeuLeitor)(nil)

// 5. Pular valores no iota (seção 7)
const ( _ = iota; KB = 1 << (10 * iota) )
```

Veremos os usos 3 e 4 em [[Criando Pacotes]] e [[Interfaces]].

---

## 🔒 6. Constantes

> 💡 **Constante:** um valor fixo, calculado e conhecido em **tempo de compilação**. Não ocupa uma posição de memória mutável: o compilador substitui o nome pelo valor diretamente no código gerado.

```go
const Pi = 3.14159265358979323846
const AppName = "minha-api"
const MaxConexoes = 100

const (
	StatusAtivo   = "ativo"
	StatusInativo = "inativo"
)
```

Regras:

- Só podem ser de tipos **básicos**: números, `string`, `bool` (e tipos definidos sobre eles). Não existe constante de slice, map ou struct
- O valor precisa ser calculável em compilação: `const x = len("abc")` ✅, `const y = math.Sqrt(2)` ❌ (é uma chamada de função em tempo de execução)
- Não dá para pegar o endereço (`&Pi` não compila)

### 6.1 Constantes sem tipo (untyped) — a parte que surpreende

Esta é uma ideia original de Go. Uma constante declarada sem tipo explícito é uma constante **untyped**: ela representa um **valor matemático exato**, que só ganha um tipo concreto quando é usada.

```go
const k = 10          // untyped: é "o número 10", ainda sem tipo

var i8 int8 = k       // ✅ aqui vira int8
var f32 float32 = k   // ✅ aqui vira float32
var d time.Duration = k * time.Second   // ✅ aqui vira time.Duration
fmt.Println(i8, f32)  // 10 10
```

Compare com uma **variável** com o mesmo valor:

```go
v := 10               // variável: tipo int, definido agora
var f float32 = v     // ❌ cannot use v (variable of type int) as float32 value
```

E com uma **constante tipada**:

```go
const c float64 = 1.5
var n int = c         // ❌ cannot use c (constant 1.5 of type float64) as int value
```

> 💡 **Por que isso existe?** Para conciliar duas coisas: Go **não** tem conversão implícita entre tipos (segurança), mas seria insuportável escrever `float64(2) * x` toda hora. Constantes sem tipo resolvem isso: `2 * x` funciona para `x` de qualquer tipo numérico, porque `2` se adapta.

### 6.2 Precisão arbitrária

Constantes sem tipo são calculadas pelo compilador com **precisão arbitrária** (pelo menos 256 bits). Elas podem guardar valores que não cabem em nenhum tipo, desde que o resultado final caiba (saída real):

```go
const Enorme = 1 << 100          // 2^100: não cabe em int64!
fmt.Println(Enorme >> 98)        // 4   ✅ o resultado cabe

var i8 int8 = 300
// ❌ cannot use 300 (untyped int constant) as int8 value in variable declaration (overflows)
```

O overflow de constantes é pego **em compilação**. (O overflow de variáveis, em tempo de execução, é silencioso; veremos em [[Operações Aritméticas]].)

### 6.3 O tipo padrão de cada constante

Quando uma constante sem tipo precisa virar um tipo concreto e o contexto não diz qual (por exemplo, em `x := k`), ela usa seu **tipo padrão** (saída real):

```go
fmt.Printf("%T %T %T %T\n", 10, 10.0, 'a', 2i)
// int float64 int32 complex128
```

| Tipo de constante | Exemplo | Tipo padrão |
| --- | --- | --- |
| Inteira | `10` | `int` |
| Ponto flutuante | `10.0`, `1e3` | `float64` |
| Rune | `'a'` | `rune` (= `int32`) |
| Complexa | `2i` | `complex128` |
| String | `"oi"` | `string` |
| Booleana | `true` | `bool` |

Uma consequência que pega muita gente (saída real):

```go
var v = 7 / 2      // as DUAS são constantes inteiras → divisão inteira → 3
var w = 7 / 2.0    // uma é float → o resultado é float → 3.5
fmt.Println(v, w)  // 3 3.5
```

---

## 🔢 7. `iota` — gerador de sequências

> 💡 **`iota`:** identificador pré-declarado que, dentro de um bloco `const`, vale o **índice da linha** (0 na primeira especificação, 1 na segunda, e assim por diante). Combinado com a regra de que uma linha vazia **repete a expressão anterior**, ele gera sequências automaticamente.

### 7.1 Enumerações

Go não tem `enum`. O padrão idiomático é um **tipo definido** + constantes com `iota` (saída real):

```go
type Nivel int

const (
	Debug Nivel = iota   // 0  (iota = 0)
	Info                 // 1  (repete "Nivel = iota" com iota = 1)
	Warn                 // 2
	Error                // 3
)

func (n Nivel) String() string {
	switch n {
	case Debug:
		return "DEBUG"
	case Info:
		return "INFO"
	case Warn:
		return "WARN"
	case Error:
		return "ERROR"
	}
	return fmt.Sprintf("Nivel(%d)", int(n))
}

fmt.Println(Warn, Nivel(7))   // WARN Nivel(7)
```

O método `String()` faz o `fmt` imprimir o nome em vez do número. Veremos métodos em [[Métodos e Receptores]] e a interface `fmt.Stringer` em [[Interfaces]].

> ⚠️ Os "enums" de Go **não são fechados**: `Nivel(7)` é um valor válido do tipo. Sempre trate o caso desconhecido (o `return` final acima). A ferramenta `stringer` (`go generate`) gera o método `String()` automaticamente.

### 7.2 Começando em 1 ou pulando valores

```go
const (
	_ = iota          // descarta o 0
	Segunda           // 1
	Terca             // 2
)

type Status int
const (
	Desconhecido Status = iota  // 0 → zero value significa "não definido" (boa prática!)
	Ativo                       // 1
	Inativo                     // 2
)
```

> ✅ Reserve o `0` para "desconhecido/não definido". Assim, uma variável `var s Status` não inicializada não é confundida com um estado válido.

### 7.3 Expressões com `iota` (saída real)

```go
type ByteSize uint64

const (
	_           = iota                // 0, descartado
	KB ByteSize = 1 << (10 * iota)    // 1 << 10 = 1024
	MB                                // 1 << 20
	GB                                // 1 << 30
)
fmt.Println(KB, MB, GB)   // 1024 1048576 1073741824

const (
	A = iota * 10   // 0
	B               // 10
	_               // 20 (pulado)
	D               // 30
)

const (
	P, Q = iota, iota + 100   // iota é o mesmo na linha inteira: 0, 100
	R, S                      // 1, 101
)
```

### 7.4 Flags com bits

```go
type Permissao uint8

const (
	Leitura  Permissao = 1 << iota   // 0001
	Escrita                          // 0010
	Execucao                         // 0100
)

p := Leitura | Escrita              // 0011: combina flags
temEscrita := p&Escrita != 0        // true: testa uma flag
```

Isso é exatamente o modelo de permissões `rwx` que você viu em [[Proteção]]. Os operadores de bits serão explicados em [[Operações Aritméticas]].

---

## 🌐 8. Variáveis de nível de pacote

```go
package config

var (
	TimeoutPadrao = 30 * time.Second   // exportada: outros pacotes podem LER E ALTERAR
	maxTentativas = 3                  // não exportada
)
```

Elas vivem durante **toda a execução** do programa e ficam em regiões fixas do binário (`.data`/`.noptrdata` se têm valor inicial, `.bss`/`.noptrbss` se começam zeradas), como vimos em [[Como Go Funciona]].

> ⚠️ **Evite estado global mutável.** Variáveis de pacote exportadas podem ser alteradas por qualquer parte do programa, o que dificulta testes e cria condições de corrida quando há goroutines. Prefira constantes (quando o valor é fixo) ou passar a configuração explicitamente (em structs, veremos em [[Structs]]).

---

## 🔬 9. Por dentro: onde as variáveis moram

Você **não escolhe** onde uma variável é alocada em Go. O compilador decide:

```
┌──────────────────────────────── memória do processo ────────────────────────────────┐
│                                                                                     │
│  .data / .bss         variáveis de pacote — existem do início ao fim do programa    │
│                                                                                     │
│  HEAP                 variáveis que "escapam" da função (ex.: o endereço delas é    │
│  (gerenciada pelo GC) retornado ou guardado em outro lugar)                         │
│                                                                                     │
│  STACK de cada        variáveis locais que não escapam — liberadas de graça         │
│  goroutine            quando a função retorna                                       │
│                                                                                     │
│  registradores        muitas variáveis locais nem chegam à memória: o compilador    │
│                       mantém o valor num registrador da CPU                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

E as **constantes** não moram em lugar nenhum como variável: são "coladas" no código como valores imediatos das instruções (ou, no caso de strings, ficam em `.rodata`).

```bash
go build -gcflags='-m' .
# ./main.go:12:2: moved to heap: x     ← o compilador explica cada decisão
```

Essa decisão se chama **escape analysis**. Ela depende de ponteiros, que veremos em [[Ponteiros]], e é estudada em profundidade em [[Memória e Escape Analysis]].

---

## ⚠️ 10. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| `:=` em bloco interno sombreia variável externa | `if ok { v, err := f() }` | Use `=` se a variável já existe |
| Divisão entre constantes inteiras | `var media = 7 / 2` → 3 | `7 / 2.0` ou `float64(total) / float64(n)` |
| Escrever em map nil | `var m map[string]int; m["a"] = 1` → panic | `m := make(map[string]int)` (ver [[Maps]]) |
| `:=` fora de função | `x := 1` no nível de pacote | `var x = 1` |
| Ignorar erro com `_` | `n, _ := strconv.Atoi(s)` | Trate o erro (ver [[O Padrão error em Go]]) |
| Enum sem valor "desconhecido" | `Ativo = iota` vale 0 | Comece com `Desconhecido = iota` |
| Estado global mutável | `var Config = ...` exportada | Constantes ou injeção explícita |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `var x T = v` | Forma completa; válida em qualquer lugar |
| `var x T` | Recebe o zero value |
| `x := v` | Declaração curta, só em funções, tipo inferido |
| `:=` com várias variáveis | Pelo menos uma nova; as outras (do mesmo bloco) são reatribuídas |
| Zero value | `0`, `false`, `""`, `nil`; nunca há lixo |
| Atribuição múltipla | Lado direito avaliado antes: `a, b = b, a` |
| Variável local não usada | Erro de compilação |
| `_` | Descarta valores; imports por efeito colateral; checagem de interface |
| Constante | Valor de compilação; só tipos básicos |
| Constante sem tipo | Valor exato, se adapta ao contexto, precisão arbitrária |
| Tipo padrão | `int`, `float64`, `rune`, `complex128`, `string`, `bool` |
| `iota` | Índice da linha no bloco `const`; linhas vazias repetem a expressão |
| Enum idiomático | `type X int` + `const ( ... = iota )` + método `String()` |

---

## 🔗 Conexões

**Dentro de Go**
- Os tipos básicos em profundidade → [[Tipos de Dados]]
- Escopo e shadowing com `:=` → [[Sintaxe, Blocos e Escopo]]
- Operadores de bits usados nas flags → [[Operações Aritméticas]]
- Ponteiros e o operador `&` → [[Ponteiros]]
- Stack, heap e escape analysis → [[Memória e Escape Analysis]]
- Métodos como `String()` → [[Métodos e Receptores]]
- Zero value útil em tipos próprios → [[Structs]]

**Sistemas Operacionais**
- [[Memória Virtual]] e [[Paginação]] — páginas anônimas novas chegam zeradas do kernel; é por isso que `.bss` e a memória nova da heap já começam com zero value
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — os segmentos `.data`, `.bss` e `.rodata` onde moram as variáveis de pacote e as strings constantes
- [[Proteção]] — as permissões `rwx` são o exemplo clássico de flags com bits, o mesmo padrão de `1 << iota`
- [[A Linguagem C]] — em C, variáveis locais não inicializadas contêm lixo; Go elimina essa classe de bug

---

⬅️ **Anterior:** [[Sintaxe, Blocos e Escopo]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Tipos de Dados]] ➡️
