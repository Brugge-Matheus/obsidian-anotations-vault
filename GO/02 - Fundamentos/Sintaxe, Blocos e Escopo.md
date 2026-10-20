---
tags:
  - go
  - go/fundamentos
aliases:
  - Sintaxe do Go
  - Escopo em Go
  - Shadowing
  - Ponto e vírgula automático
ordem: 5
modulo: "02 - Fundamentos"
---
# Sintaxe, Blocos e Escopo

> **Nota 5 da trilha de Go.** Antes de falar de variáveis e tipos, precisamos entender as **peças mínimas** de um arquivo Go: quais são as palavras reservadas, como o compilador separa um comando do outro (o famoso ponto e vírgula invisível), o que é um **bloco**, onde cada nome "existe" (**escopo**) e o que acontece quando dois nomes iguais se encontram (**shadowing**). Essas regras explicam vários erros que parecem mágicos no começo.
>
> **Pré-requisitos:** [[Ambiente e Primeiro Programa]], [[Convenções da linguagem]].

---

## 🧭 De onde viemos

No [[Ambiente e Primeiro Programa]] vimos dois erros curiosos: a `{` na linha de baixo quebra a compilação, e o compilador não exige `;` no fim das linhas. Em [[Convenções da linguagem]] vimos que nomes com letra maiúscula são visíveis fora do pacote. As três coisas são consequências das regras desta nota: **análise léxica**, **inserção automática de ponto e vírgula** e **escopo**.

---

## 🔤 1. As peças léxicas de um arquivo Go

O primeiro passo do compilador (vimos em [[Como Go Funciona]]) é quebrar o texto do arquivo em **tokens**.

> 💡 **Token:** a menor unidade com significado para o compilador. Em `x := 10 + y`, os tokens são `x`, `:=`, `10`, `+` e `y`. Espaços e quebras de linha só servem para separar tokens (com uma exceção importante: a quebra de linha pode virar `;`, como veremos).

Existem cinco categorias de tokens:

| Categoria | Exemplos |
| --- | --- |
| **Identificadores** | `nome`, `Conta`, `_`, `ação`, `x1` |
| **Palavras-chave** | `func`, `if`, `for`, `return`... (25 no total) |
| **Operadores e pontuação** | `+`, `:=`, `==`, `&&`, `<-`, `(`, `{`, `,`, `;`, `...` |
| **Literais** | `42`, `3.14`, `'a'`, `"texto"`, `` `bruto` `` |
| **Comentários** | `// linha` e `/* bloco */` (descartados, mas lidos pelo `go doc`) |

### 1.1 Identificadores

Um identificador é uma sequência de **letras** (Unicode), **dígitos** e `_`, começando por letra ou `_`.

```go
nome        // ✅
_temp       // ✅ (mas não idiomático)
ação        // ✅ letras Unicode são válidas
x2          // ✅
2x          // ❌ não pode começar com dígito
minha-var   // ❌ hífen é o operador de subtração
```

> 💡 **Identificador vazio (`_`, blank identifier):** um identificador especial que significa "descarte". Você pode atribuir qualquer coisa a ele, mas nunca ler. Veremos seus usos em [[Variáveis e constantes]].

### 1.2 As 25 palavras-chave

```
DECLARAÇÕES         CONTROLE DE FLUXO      TIPOS COMPOSTOS     CONCORRÊNCIA
─────────────       ──────────────────     ───────────────     ────────────
const               if      else           struct              go
var                 for     range          interface           chan
type                switch  case           map                 select
func                default fallthrough
import              break   continue
package             goto    return
                    defer
```

Palavras-chave **não podem** ser usadas como nomes (`var func int` não compila).

### 1.3 Identificadores pré-declarados (não são palavras-chave!)

Existe um segundo grupo de nomes que já vêm definidos, mas que **não** são palavras reservadas. Eles vivem no **bloco universo** (veremos adiante):

| Grupo | Nomes |
| --- | --- |
| Tipos | `bool`, `byte`, `rune`, `string`, `error`, `any`, `comparable`, `int`, `int8`...`int64`, `uint`...`uint64`, `uintptr`, `float32`, `float64`, `complex64`, `complex128` |
| Constantes | `true`, `false`, `iota` |
| Valor zero | `nil` |
| Funções embutidas | `append`, `cap`, `clear`, `close`, `complex`, `copy`, `delete`, `imag`, `len`, `make`, `max`, `min`, `new`, `panic`, `print`, `println`, `real`, `recover` |

Como **não** são palavras-chave, eles **podem ser redeclarados** (sombreados). Isso compila (saída real):

```go
len := 3
fmt.Println(len)      // 3
true := false
fmt.Println(true)     // false   😱
```

> ⚠️ Nunca faça isso de propósito. Mas é comum alguém criar uma variável chamada `len`, `new`, `max` ou `copy` sem perceber e depois não conseguir chamar a função embutida naquele trecho. Linters (`predeclared`) avisam.

### 1.4 Comentários

```go
// Comentário de linha: o mais usado, inclusive para blocos grandes.

/*
   Comentário de bloco: usado principalmente para
   desativar trechos temporariamente ou em comentários de pacote longos.
*/

//go:noinline       ← "diretiva": comentário especial lido pelo compilador (sem espaço após //)
//go:generate ...   ← diretiva lida pelo comando go generate
```

> 💡 **Diretivas de compilador** são comentários no formato `//go:nome` (sem espaço). Elas mudam o comportamento do compilador ou da toolchain. Veremos `//go:generate`, `//go:embed` e `//go:build` em [[Build, Flags e Cross-compilation]].

---

## ⏎ 2. O ponto e vírgula invisível

A gramática de Go **usa** ponto e vírgula para separar comandos, como C. A diferença é que **você quase nunca escreve**: o analisador léxico insere automaticamente.

### 2.1 A regra exata

> Quando uma linha termina, o analisador léxico insere um `;` **se o último token da linha** for:
> - um **identificador** (`x`, `main`, `fmt`)
> - um **literal** (`10`, `"texto"`, `3.14`)
> - uma das palavras-chave `break`, `continue`, `fallthrough`, `return`
> - um dos operadores `++`, `--`, `)`, `]`, `}`

Só isso. É uma regra puramente **léxica** (olha só o último token), simples e previsível.

### 2.2 As consequências

**Consequência 1 — a `{` precisa ficar na mesma linha.**

```go
func main()      // linha termina com ")" → o lexer insere ";"
{                // vira:  func main(); {    → erro de sintaxe
```

```
./main.go:6:1: syntax error: unexpected semicolon or newline before {
```

O mesmo vale para `if`, `for`, `switch` e `else`:

```go
if x > 0 {       // ✅
}
else {           // ❌ "}" no fim da linha anterior virou "};"
}

if x > 0 {
} else {         // ✅ else na mesma linha do }
}
```

**Consequência 2 — vírgula final obrigatória em listas multilinha.**

```go
s := []int{
	1,
	2         // linha termina com literal → ";" inserido → erro
}
```

```
./main.go:6:4: syntax error: unexpected newline in composite literal; possibly missing comma or }
```

```go
s := []int{
	1,
	2,        // ✅ vírgula final: a linha termina com "," e nada é inserido
}
```

> 💡 A vírgula final é obrigatória, e isso é uma vantagem: adicionar um item novo gera um diff de **uma linha** só, sem tocar a linha anterior.

**Consequência 3 — quebre linhas depois do operador, nunca antes.**

```go
total := 10 +
	5            // ✅ linha termina com "+" (operador binário) → nada inserido

soma := 10       // linha termina com literal → ";" inserido
	+5           // vira um comando separado: "+5"
```

```
./main.go:9:2: +5 (untyped int constant 5) is not used
```

Repare que o compilador pegou o erro porque `+5` sozinho é uma expressão inútil, mas a lição é: **quebra de linha em expressão longa vai depois do operador**.

**Consequência 4 — vários comandos na mesma linha precisam de `;` explícito.**

```go
x := 1; y := 2          // válido, mas o gofmt separa em duas linhas
for i := 0; i < 3; i++  // os ";" do for são explícitos (estão no meio da linha)
if v, ok := m[k]; ok {  // ";" separa a declaração curta da condição
```

> 💡 Rob Pike explicou essa escolha: a regra é simples o bastante para que **ferramentas** (como o `gofmt`) e pessoas sempre saibam onde um comando termina, sem ambiguidades como as do JavaScript (cuja inserção automática de `;` tem regras bem mais complicadas).

---

## 🧱 3. Blocos

> 💡 **Bloco:** uma sequência (possivelmente vazia) de declarações e comandos entre chaves `{ }`. Blocos podem ser aninhados e definem **onde os nomes existem**.

Além dos blocos que você escreve com `{}`, Go define **blocos implícitos**. A hierarquia completa:

```
┌───────────────────────────── BLOCO UNIVERSO ─────────────────────────────┐
│  todo o código Go: int, string, true, nil, len, append, error...          │
│                                                                           │
│  ┌──────────────────────── BLOCO DO PACOTE ───────────────────────────┐   │
│  │  todos os arquivos .go do pacote: funções, tipos, var e const      │   │
│  │  declarados FORA de funções                                        │   │
│  │                                                                    │   │
│  │  ┌──────────────── BLOCO DO ARQUIVO ───────────────────────────┐   │   │
│  │  │  só os IMPORTS (import "fmt" vale só neste arquivo)         │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                    │   │
│  │  ┌──────────────── BLOCO DA FUNÇÃO ────────────────────────────┐   │   │
│  │  │  parâmetros, retornos nomeados e variáveis locais           │   │   │
│  │  │                                                             │   │   │
│  │  │  ┌─── BLOCO IMPLÍCITO do if/for/switch ──────────────────┐  │   │   │
│  │  │  │  variáveis da cláusula de inicialização:              │  │   │   │
│  │  │  │  if v, err := f(); err != nil { ... } else { ... }    │  │   │   │
│  │  │  │                                                       │  │   │   │
│  │  │  │  ┌─── BLOCO EXPLÍCITO { } / cada case ───────────┐    │  │   │   │
│  │  │  │  │                                               │    │  │   │   │
│  │  │  │  └───────────────────────────────────────────────┘    │  │   │   │
│  │  │  └───────────────────────────────────────────────────────┘  │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────┘
```

Detalhes que costumam surpreender:

| Bloco | Detalhe |
| --- | --- |
| **Universo** | Por isso `int` e `len` não precisam de import |
| **Pacote** | Uma função declarada em `a.go` é visível em `b.go` do mesmo pacote, **sem import** |
| **Arquivo** | Imports valem **por arquivo**. Se `b.go` também usa `fmt`, precisa importar de novo |
| **`if`/`for`/`switch`** | Cada um é um bloco implícito; cada `case` e cada `else` também é um bloco |

---

## 🔭 4. Escopo

> 💡 **Escopo:** a região do código onde um nome declarado pode ser usado. Em Go o escopo é **léxico** (ou estático): é determinado pela posição no texto do programa, não pela ordem de execução.

### 4.1 As regras

| Onde o nome foi declarado | Escopo |
| --- | --- |
| Identificador pré-declarado | Todo o programa (bloco universo) |
| Fora de função (nível de pacote) | O pacote inteiro, em **todos** os arquivos, **inclusive antes da declaração** |
| Nome de pacote importado | Só o arquivo que importou |
| Parâmetro ou retorno nomeado | O corpo inteiro da função |
| Variável ou constante dentro de função | Do **ponto da declaração** até o fim do bloco mais interno que a contém |
| Tipo dentro de função | Do ponto da declaração até o fim do bloco (pode ser recursivo, como `type No struct{ prox *No }`) |
| Label (`fim:`) | O corpo da função inteira (e labels não conflitam com outros nomes) |

### 4.2 No nível do pacote, a ordem não importa

No escopo de pacote, você pode usar algo antes de declarar. Isso compila e imprime `42` (saída real):

```go
var total = dobro(limite)   // usa uma função e uma constante declaradas ABAIXO

const limite = 21

func dobro(n int) int { return n * 2 }

func main() {
	fmt.Println(total)       // 42
}
```

O compilador analisa o pacote inteiro e ordena a inicialização das variáveis pelas **dependências** entre elas (veremos em [[Criando Pacotes]]). Isso é diferente de C, onde é preciso declarar (ou pôr um protótipo) antes de usar.

### 4.3 Dentro de funções, a ordem importa

```go
func main() {
	fmt.Println(x)   // ❌ undefined: x
	x := 10
}
```

Dentro de uma função, o nome só existe **depois** da linha que o declara.

### 4.4 Variáveis do `if`, `for` e `switch` têm vida curta

```go
if n, err := strconv.Atoi("42"); err != nil {
	fmt.Println("erro:", err)
} else {
	fmt.Println("número:", n)   // n e err também existem no else
}
fmt.Println(n)                  // ❌ undefined: n — o bloco implícito do if acabou
```

Esse é um recurso **muito usado**: ele limita o nome ao trecho em que ele faz sentido, o que é bom para a legibilidade. Veremos mais em [[Controle de Fluxo]].

---

## 👥 5. Shadowing (sombreamento)

> 💡 **Shadowing (sombreamento):** quando um bloco interno declara um nome igual ao de um bloco externo. Dentro do bloco interno, o nome novo "esconde" o antigo; o antigo continua existindo e volta a ser visível quando o bloco interno termina.

```go
x := 1
{
	x := 2                         // NOVA variável x, que sombreia a externa
	fmt.Println("dentro:", x)      // dentro: 2
}
fmt.Println("fora:", x)            // fora: 1   (a externa nunca mudou)
```

Na memória, são **duas variáveis diferentes**:

```
bloco da função                 bloco interno
┌────────────────┐              ┌────────────────┐
│ x (externo) = 1│              │ x (interno) = 2│  ← o nome "x" aqui
└────────────────┘              └────────────────┘    aponta para esta
        ▲
        └── depois do }, "x" volta a apontar para cá
```

### 5.1 A armadilha clássica: `:=` sombreando `err`

Este é **um dos bugs mais comuns em Go** (saída real):

```go
func buscar() (int, error) { return 0, errors.New("falhou") }

func main() {
	var err error

	if true {
		v, err := buscar()   // ⚠️ := cria um NOVO err dentro do if
		_ = v
		_ = err
	}

	fmt.Println("err externo:", err)   // err externo: <nil>   ← o erro sumiu!
}
```

O `:=` declara **todas** as variáveis da esquerda que ainda não existem **naquele bloco**. Como o `if` abre um bloco novo, tanto `v` quanto `err` são declarados de novo, e o `err` de fora nunca recebe o erro.

Como evitar:

```go
var err error
var v int
if true {
	v, err = buscar()   // ✅ "=" atribui às variáveis que já existem
}
```

> ✅ O analisador `shadow` (`go vet -vettool=$(which shadow)`, ou o linter `govet` com `shadow` habilitado no golangci-lint) avisa sobre esse padrão.

### 5.2 Sombreamento de pacote

```go
import "path"

func processar(path string) {    // o parâmetro "path" sombreia o PACOTE path
	path.Join("a", "b")          // ❌ path.Join undefined (type string has no field or method Join)
}
```

Use nomes como `p`, `caminho` ou `filePath` quando um pacote importado tem o nome que você queria usar.

### 5.3 Shadowing nem sempre é ruim

Há usos idiomáticos e intencionais:

```go
// 1. Reduzir o tipo de uma variável num trecho (type switch, veremos em Type Assertion)
switch v := v.(type) {
case int:    // aqui v é int
case string: // aqui v é string
}

// 2. "Reatribuir" um parâmetro depois de normalizar
func saudar(nome string) {
	nome = strings.TrimSpace(nome)   // aqui é "=", não sombreia — só reatribui
}
```

---

## 🔢 6. Literais — uma visão geral

> 💡 **Literal:** um valor escrito diretamente no código-fonte, como `42` ou `"oi"`. O compilador converte o texto no valor correspondente.

Testado com saída real:

```go
fmt.Println('é', 0b1010, 0o17, 0x1F, 1_000_000, 1e3, 3i)
// 233 10 15 31 1000000 1000 (0+3i)
```

| Tipo de literal | Exemplos | Observação |
| --- | --- | --- |
| Inteiro decimal | `42`, `1_000_000` | `_` separa dígitos para legibilidade |
| Binário | `0b1010` | = 10 |
| Octal | `0o17` (ou `017`, forma antiga) | = 15 |
| Hexadecimal | `0x1F`, `0xFF` | = 31, 255 |
| Ponto flutuante | `3.14`, `.5`, `1e3`, `6.02e23`, `0x1p-2` | `1e3` = 1000.0 |
| Imaginário | `3i`, `2.5i` | parte de um número complexo |
| Rune | `'a'`, `'é'`, `'\n'`, `'\u00e9'` | um caractere Unicode; vale o *code point* (`'é'` = 233) |
| String interpretada | `"olá\n"` | aceita sequências de escape |
| String bruta (raw) | `` `C:\pasta\n` `` | sem escapes, pode ter várias linhas |

Veremos cada um em detalhe nas notas de tipos: [[Tipos de Dados]] e [[Strings em Profundidade]].

---

## 🧾 7. Declarações: a forma geral

Tudo em Go que dá nome a algo usa uma de **seis** declarações:

```go
package main              // 1. cláusula de pacote (sempre a primeira)

import "fmt"              // 2. import

const Pi = 3.14           // 3. constante
var contador int          // 4. variável
type Celsius float64      // 5. tipo
func main() { ... }       // 6. função (ou método)
```

E as quatro primeiras aceitam a **forma agrupada**, com parênteses:

```go
import (
	"fmt"
	"os"
)

const (
	A = 1
	B = 2
)

var (
	nome  string
	idade int
)

type (
	ID    int64
	Email string
)
```

> 💡 Repare no padrão **nome antes do tipo** (`var idade int`), o contrário de C (`int idade`). A justificativa oficial: lida da esquerda para a direita, a declaração soa natural ("variável idade do tipo int"), e tipos complexos ficam legíveis. Compare `var f func(int) []string` com o equivalente de C, que exige ler "em espiral".

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Solução |
| --- | --- | --- |
| `{` ou `else` na linha de baixo | `syntax error: unexpected semicolon or newline` | Mantenha na mesma linha (o `gofmt` não consegue consertar código que não compila) |
| Falta da vírgula final em lista multilinha | `unexpected newline in composite literal` | Sempre termine cada item com `,` |
| Quebra de linha antes do operador | Expressão vira dois comandos | Quebre **depois** do operador |
| `:=` dentro de bloco sombreando variável externa | Erro "some", valor não é atualizado | Use `=` quando a variável já existe |
| Variável com nome de pacote ou de função embutida | `undefined` em chamadas a `path.Join`, `len()` etc. | Escolha outro nome |
| Esperar que um import em `a.go` valha em `b.go` | `undefined: fmt` | Imports são por arquivo |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Tokens | Identificadores, palavras-chave, operadores, literais, comentários |
| Palavras-chave | 25, não podem ser nomes |
| Pré-declarados | `int`, `len`, `true`, `nil`... vivem no bloco universo e **podem** ser sombreados |
| Ponto e vírgula | Inserido no fim da linha após identificador, literal, `return`/`break`/..., `)`, `]`, `}`, `++`, `--` |
| Consequências | `{` na mesma linha, vírgula final obrigatória, quebrar após o operador |
| Blocos | universo → pacote → arquivo (imports) → função → implícitos (`if`/`for`/`switch`) → `{}` |
| Escopo no pacote | Pacote inteiro, ordem não importa |
| Escopo local | Da declaração até o fim do bloco |
| Shadowing | Nome interno esconde o externo; cuidado com `:=` e `err` |
| Declarações | `package`, `import`, `const`, `var`, `type`, `func` |

---

## 🔗 Conexões

**Dentro de Go**
- Declarando variáveis e constantes com essas regras → [[Variáveis e constantes]]
- Variáveis na cláusula de inicialização do `if`/`for`/`switch` → [[Controle de Fluxo]]
- Ordem de inicialização das variáveis de pacote e `init()` → [[Criando Pacotes]]
- Type switch (shadowing intencional) → [[Type Assertion e Type Switch]]
- Como o compilador transforma tokens em AST → [[Como Go Funciona]]

**Sistemas Operacionais**
- [[A Linguagem C]] — C exige `;` explícito e declaração antes do uso (ou protótipos); Go automatiza o primeiro e elimina o segundo no nível de pacote
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — em C, um nome é compartilhado entre arquivos via `extern` em headers; em Go, todos os arquivos do pacote compartilham o mesmo bloco de pacote
- [[Interpretador de Comandos Shell]] — o shell também usa `;` e quebras de linha como separadores de comandos

---

⬅️ **Anterior:** [[Convenções da linguagem]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Variáveis e constantes]] ➡️
