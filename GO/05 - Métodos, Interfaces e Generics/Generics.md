---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Genéricos
  - Type parameters
  - Parâmetros de tipo
  - Restrições de tipo
  - Constraints
  - Métodos genéricos
ordem: 25
modulo: "05 - Métodos, Interfaces e Generics"
---
# Generics

> **Nota 25 da trilha de Go.** Generics (ou **parâmetros de tipo**) permitem escrever uma função ou um tipo **uma vez** e usá-lo com **vários tipos**, mantendo a checagem do compilador. Chegaram no **Go 1.18 (2022)**, depois de dez anos de debate, e continuam evoluindo: aliases genéricos (1.24), restrições autorreferentes (1.26) e **métodos genéricos (1.27)**. Nesta nota veremos o problema que generics resolvem, a sintaxe de funções e tipos genéricos, **restrições** (`any`, `comparable`, conjuntos de tipos com `|` e `~`), inferência, limitações, quando usar generics vs interfaces, e como o compilador os implementa.
>
> **Pré-requisitos:** [[Interfaces]], [[Type Assertion e Type Switch]], [[Arrays e Slices]].

---

## 🧭 De onde viemos

Já usamos funções genéricas sem perceber: `slices.Sort`, `slices.Contains`, `maps.Keys`, `cmp.Compare`, `min`, `max`. Em [[Interfaces]] vimos que `any` aceita tudo, mas a cada uso é preciso uma type assertion ([[Type Assertion e Type Switch]]), que só falha em tempo de execução. Generics resolvem esse dilema: **código reutilizável com segurança de tipos em compilação**.

> 💡 **Programação genérica:** escrever algoritmos e estruturas de dados em termos de **tipos a serem especificados depois**. O mesmo `Max` serve para `int`, `float64` e `string`; a mesma `Pilha` serve para inteiros, strings ou structs.

---

## 😩 1. O problema: a vida antes dos generics

Imagine uma função `Max`. Antes do Go 1.18, havia três saídas, todas ruins:

```go
// 1) Duplicar código (uma por tipo)
func MaxInt(a, b int) int { if a > b { return a }; return b }
func MaxFloat(a, b float64) float64 { if a > b { return a }; return b }
func MaxString(a, b string) string { if a > b { return a }; return b }

// 2) Usar interface{} e perder a segurança de tipos
func Max(a, b interface{}) interface{} {
	switch a := a.(type) {
	case int:
		return max(a, b.(int))     // e se b for float64? panic em produção
	// ...um case por tipo, e esquecer um vira bug
	}
	return nil
}
m := Max(3, 7).(int)               // e quem chama ainda precisa de assertion

// 3) Gerar código com ferramentas (go generate)
```

Com generics:

```go
func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

Max(3, 7)                 // 7       (T = int)
Max("abacate", "banana")  // banana  (T = string)
Max(3, "x")               // ❌ erro de COMPILAÇÃO: tipos diferentes
```

---

## ✍️ 2. Funções genéricas

```go
func Max[T cmp.Ordered](a, b T) T {
//      └┬┘ └────┬────┘
//       │       └── restrição: quais tipos T pode ser
//       └── parâmetro de tipo
	...
}
```

> 💡 **Parâmetro de tipo:** um "tipo variável", declarado entre colchetes depois do nome da função. Quem chama a função fornece (ou o compilador deduz) o **argumento de tipo** que vai substituí-lo.

> 💡 **Restrição (constraint):** uma interface que limita quais tipos podem ser usados como argumento de tipo, e que, por consequência, define **quais operações** a função pode fazer com valores desse tipo.

Saídas reais:

```go
fmt.Println(Max(3, 7), Max("abacate", "banana"), Max[float64](2, 2.5))
// 7 banana 2.5
```

### 2.1 Vários parâmetros de tipo

Saída real:

```go
func Map[T, U any](xs []T, f func(T) U) []U {
	out := make([]U, 0, len(xs))
	for _, x := range xs {
		out = append(out, f(x))
	}
	return out
}

Map([]int{1, 2, 3}, strconv.Itoa)                            // [1 2 3]  ([]string)
Map([]string{"a", "bb"}, func(s string) int { return len(s) }) // [1 2]   ([]int)
```

```go
func Filtrar[T any](xs []T, ok func(T) bool) []T {
	var out []T
	for _, x := range xs {
		if ok(x) {
			out = append(out, x)
		}
	}
	return out
}

Filtrar([]int{1, 2, 3, 4, 5, 6}, func(n int) bool { return n%2 == 0 })   // [2 4 6]
```

### 2.2 Inferência de tipos

Na maioria das chamadas, o compilador **deduz** `T` pelos argumentos. Só é preciso escrever explicitamente quando não há como deduzir:

```go
Max(3, 7)                 // inferido: T = int
Max[float64](2, 2.5)      // explícito: sem isso, 2 seria int e 2.5 float64

func Zero[T any]() T { var z T; return z }
Zero[string]()            // explícito obrigatório: não há argumento para inferir
```

O **Go 1.21** melhorou muito a inferência, inclusive a partir do **tipo de destino** (testado no Go 1.24):

```go
var f func(int, int) int = Max     // ✅ infere T = int pelo tipo de f
usa(Max)                           // ✅ infere pelo tipo do parâmetro de usa

type F func(int, int) int
g := F(Max)                        // ❌ até o 1.26: cannot use generic function Max without instantiation
fs := []func(int, int) int{Max}    // ❌ idem
```

O **Go 1.27** generalizou a regra: a inferência passa a valer **em qualquer lugar** em que uma função genérica é atribuída **ou convertida** para um tipo de função compatível, então os dois últimos casos passam a compilar.

### 2.3 O zero value de `T`

Não dá para escrever `return nil` ou `return 0` num `T` qualquer. O idioma é declarar uma variável:

```go
func Primeiro[T any](xs []T) (T, bool) {
	var zero T                // zero value do tipo, seja ele qual for
	if len(xs) == 0 {
		return zero, false
	}
	return xs[0], true
}
```

---

## 🧱 3. Restrições (constraints)

Uma restrição é uma **interface**. Ela pode exigir **métodos**, listar **tipos**, ou os dois.

### 3.1 `any`: sem restrição

Aceita qualquer tipo, mas só permite operações que valem para **todos** os tipos: atribuir, passar adiante, guardar em slices/maps (como valor), declarar `var x T`. Nada de `+`, `<` ou `==`.

### 3.2 `comparable`: permite `==` e `!=`

Saída real:

```go
func Contem[T comparable](xs []T, alvo T) bool {
	for _, x := range xs {
		if x == alvo {          // só compila porque T é comparable
			return true
		}
	}
	return false
}

Contem([]string{"go", "rust"}, "go")   // true
```

`comparable` também é a restrição exigida para chaves de map genéricos (`map[K]V` com `K comparable`).

> ⚠️ Desde o Go 1.20, tipos interface (como `any`) satisfazem `comparable`. Isso significa que a comparação pode dar **panic em tempo de execução** se o valor dentro da interface não for comparável (saída real):
> ```go
> var a, b any = []int{1}, []int{1}
> Contem([]any{a}, b)
> // panic: runtime error: comparing uncomparable type []int
> ```

### 3.3 Restrições por método

Uma interface comum serve de restrição: `T` precisa ter os métodos (saída real):

```go
type Stringer interface{ String() string }

type ID int
func (i ID) String() string { return "ID-" + strconv.Itoa(int(i)) }

func JuntarTodos[T Stringer](xs []T) string {
	s := ""
	for i, x := range xs {
		if i > 0 {
			s += ","
		}
		s += x.String()
	}
	return s
}

fmt.Println(JuntarTodos([]ID{1, 2, 3}))    // ID-1,ID-2,ID-3
```

> 💡 Qual a diferença para `func JuntarTodos(xs []Stringer)`? Com a interface comum, você precisa de um `[]Stringer` (um slice de interfaces; não dá para passar um `[]ID` direto). Com generics, passa o `[]ID` como está, sem converter cada elemento.

### 3.4 Conjuntos de tipos: `|` e `~`

Para usar **operadores** (`+`, `<`, `*`...), a restrição precisa listar os tipos permitidos (saídas reais):

```go
type Numero interface {
	~int | ~int64 | ~float64
}

func Somar[T Numero](xs []T) T {
	var total T
	for _, x := range xs {
		total += x              // só compila porque TODOS os tipos do conjunto suportam +
	}
	return total
}

type Celsius float64

Somar([]int{1, 2, 3})           // 6
Somar([]float64{1.5, 2.5})      // 4
Somar([]Celsius{10, 20.5})      // 30.5   ← aceito por causa do ~
Somar([]string{"a"})
// ❌ string does not satisfy Numero (string missing in ~int | ~float64)
```

| Sintaxe | Significado |
| --- | --- |
| `int \| float64` | Exatamente `int` ou exatamente `float64` |
| `~float64` | Qualquer tipo cujo **tipo subjacente** é `float64` (inclui `Celsius`, `time.Duration` é `~int64`...) |
| `interface{ ~int; String() string }` | Subjacente `int` **e** com método `String()` |

> 💡 O `~` (til) existe porque tipos definidos ([[Tipos de Dados]]) são tipos distintos. Sem ele, `Celsius` não seria aceito por uma restrição que lista `float64`.

> ⚠️ Interfaces com conjuntos de tipos **só podem ser usadas como restrição**, não como tipo de variável (mensagem real): `cannot use type Numero outside a type constraint: interface contains type constraints`.

### 3.5 Restrições prontas

| Restrição | Pacote | Tipos |
| --- | --- | --- |
| `any` | pré-declarada | todos |
| `comparable` | pré-declarada | os que aceitam `==` |
| `cmp.Ordered` | `cmp` (Go 1.21) | inteiros, floats e strings (os que aceitam `<`) |
| `constraints.Integer`, `Float`, `Signed`... | `golang.org/x/exp/constraints` | grupos numéricos (fora da biblioteca padrão) |

---

## 🏗️ 4. Tipos genéricos

Structs, slices, maps e funções também podem ter parâmetros de tipo. O clássico: uma estrutura de dados (saídas reais):

```go
type Pilha[T any] struct {
	itens []T
}

func (p *Pilha[T]) Push(v T) { p.itens = append(p.itens, v) }

func (p *Pilha[T]) Pop() (T, error) {
	var zero T
	if len(p.itens) == 0 {
		return zero, errors.New("pilha vazia")
	}
	v := p.itens[len(p.itens)-1]
	p.itens = p.itens[:len(p.itens)-1]
	return v, nil
}

func (p *Pilha[T]) Len() int { return len(p.itens) }

var p Pilha[string]           // o argumento de tipo é obrigatório no tipo
p.Push("a")
p.Push("b")
v, _ := p.Pop()
fmt.Println(v, p.Len())       // b 1
```

```go
type Par[K comparable, V any] struct {
	Chave K
	Valor V
}

par := Par[string, int]{"idade", 30}
fmt.Printf("%+v %T\n", par, par)     // {Chave:idade Valor:30} main.Par[string,int]
```

Outros exemplos úteis:

```go
type Conjunto[T comparable] map[T]struct{}
type Arvore[T cmp.Ordered] struct { Raiz *No[T] }
type No[T cmp.Ordered] struct { Valor T; Esq, Dir *No[T] }
type Resultado[T any] struct { Valor T; Err error }
```

> 💡 Repare: nos métodos de um tipo genérico, o receptor repete os parâmetros (`(p *Pilha[T])`), mas **sem** a restrição.

---

## 🆕 5. Novidades recentes

### 5.1 Aliases genéricos (Go 1.24)

```go
type Lista[T any] = []T          // alias com parâmetro de tipo
var l Lista[int] = []int{1, 2}   // l é um []int de verdade (saída real: %T → []int)
```

Útil para migrar tipos genéricos entre pacotes sem quebrar quem usa.

### 5.2 Restrições autorreferentes (Go 1.26)

Um tipo genérico pode citar a si mesmo na sua lista de parâmetros. Isso permite expressar "um tipo que sabe se somar com outro do mesmo tipo":

```go
type Adder[A Adder[A]] interface {
	Add(A) A
}

func Somar2[A Adder[A]](x, y A) A {
	return x.Add(y)
}
```

### 5.3 Métodos genéricos (Go 1.27)

Até o Go 1.26, **métodos não podiam ter parâmetros de tipo próprios**: só podiam usar os do tipo (`Pilha[T]`). Funções genéricas relacionadas a um tipo precisavam ficar no nível do pacote. No **Go 1.27**, um método pode declarar seus próprios parâmetros de tipo:

```go
// Exemplo real da biblioteca padrão (math/rand/v2), adicionado no Go 1.27:
func (r *Rand) N[Int intType](n Int) Int

// O mesmo raciocínio para os seus tipos:
func (p *Pilha[T]) Mapear[U any](f func(T) U) *Pilha[U] {
	var nova Pilha[U]
	for _, v := range p.itens {
		nova.Push(f(v))
	}
	return &nova
}
```

Restrições que continuam valendo:

- **Métodos de interfaces** não podem ter parâmetros de tipo
- Um método genérico **não pode** ser usado para satisfazer um método de interface

O motivo: uma interface é satisfeita em **tempo de execução** (via itab, ver [[Interfaces]]), e o runtime não teria como gerar, sob demanda, uma versão do método para um tipo que só aparece na hora.

---

## 🚧 6. O que generics NÃO fazem

| Limitação | Exemplo |
| --- | --- |
| Sem especialização | Não dá para escrever uma versão especial de `F[T]` para `T = string` (use um type switch em `any(x)` se precisar) |
| Sem parâmetros de tipo em métodos de interface | `interface{ M[T any]() }` é inválido |
| Sem variância | `[]Gato` não é `[]Animal`, nem com generics |
| Sem operadores sobre `T` sem restrição | `a + b` exige um conjunto de tipos que suporte `+` |
| Sem acesso a campos via restrição | Restrições não podem exigir "ter o campo `Nome`"; exija um método `Nome()` |
| Sem metaprogramação | Generics de Go são deliberadamente simples; nada de templates de C++ calculando em compilação |

---

## ⚖️ 7. Generics ou interfaces?

As duas ferramentas resolvem problemas diferentes:

| Pergunta | Interface | Generics |
| --- | --- | --- |
| O que varia? | O **comportamento** (métodos diferentes) | O **tipo** (mesmo algoritmo) |
| Valores de tipos diferentes juntos? | ✅ `[]Forma{Retangulo{}, Circulo{}}` | ❌ uma `Pilha[T]` tem um único `T` |
| Precisa de operadores (`+`, `<`)? | ❌ | ✅ com conjuntos de tipos |
| Quando o tipo é decidido? | Em execução (despacho dinâmico) | Em compilação |
| Exemplo típico | `io.Reader`, `http.Handler`, `error` | `slices.Sort`, `maps.Keys`, `sync.Map` tipado, estruturas de dados |

Recomendações do time do Go:

> ✅ **Comece com código concreto.** Escreva a função para o tipo que você tem. Se precisar da **mesma** função para outros tipos, aí sim generalize.
>
> ✅ Use generics para **estruturas de dados** e **funções sobre slices, maps e channels** que funcionam igual para qualquer tipo de elemento.
>
> ✅ Se o que varia é o **comportamento**, use **interfaces**.
>
> ⚠️ Não use generics só para "evitar uma interface": se a função só chama métodos, uma interface comum geralmente é mais simples.

---

## 🔬 8. Por dentro: como o compilador implementa

Há duas estratégias clássicas para compilar generics:

| Estratégia | Como | Prós | Contras | Quem usa |
| --- | --- | --- | --- | --- |
| **Monomorfização** | Gera uma cópia do código para cada tipo usado | Máximo desempenho | Binário maior, compilação lenta | C++, Rust |
| **Boxing** | Uma única versão que trabalha com ponteiros/objetos | Binário pequeno | Indireções, alocações | Java (com *type erasure*) |

Go usa um **meio-termo** chamado **GC shape stenciling com dicionários**:

> 💡 **GC shape (forma):** dois tipos têm a mesma "forma" se têm o mesmo tipo subjacente, ou se são **ponteiros** (todos os ponteiros têm a mesma forma).

```
Max[int], Max[ID], Max[time.Duration]...   → formas diferentes? int e ID têm subjacente int:
                                              MESMA forma → compartilham uma cópia
Max[float64]                                → outra forma → outra cópia
Max[*Usuario], Max[*Pedido], Max[*bytes.Buffer] → TODOS ponteiros → uma única cópia

Cada cópia recebe um parâmetro oculto: o DICIONÁRIO,
com informações do tipo concreto (descritores de tipo, itabs, funções de comparação...)
```

Consequências práticas:

- Para tipos **não-ponteiro** (números, strings, structs), o desempenho é praticamente o de código escrito à mão
- Para **ponteiros e interfaces**, chamadas de método via restrição passam pelo dicionário (parecido com uma chamada de interface), e podem não ser inlinadas
- O binário cresce menos que com monomorfização completa
- Funções genéricas pequenas podem ser **inlinadas** no local da chamada, e aí o compilador conhece o tipo exato

> 💡 Na prática: `slices.Sort` (genérico) é **mais rápido** que o antigo `sort.Slice` (que usava interface e reflection), porque compara elementos diretamente, sem chamada indireta.

### 8.1 Simplificação da especificação (Go 1.25)

A especificação original usava o conceito de **"core type"** para decidir quais operações eram válidas sobre um parâmetro de tipo. No Go 1.25 esse conceito foi removido e substituído por regras mais diretas, o que tornou a especificação mais simples e abriu espaço para recursos como os métodos genéricos do 1.27. Para quem usa a linguagem, quase nada mudou.

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Generalizar cedo demais | Código abstrato e difícil de ler | Comece concreto; generalize com 2+ usos reais |
| Esquecer o `~` | Tipos definidos (`Celsius`) recusados | `~float64` |
| Usar restrição de conjunto como variável | `cannot use type ... outside a type constraint` | Restrições com tipos são só para `[T ...]` |
| `comparable` com `any` dentro | panic ao comparar valores não comparáveis | Tipos concretos ou cuidado com interfaces |
| `return nil` num `T` | Não compila | `var zero T; return zero` |
| Generics onde cabia interface | Complexidade sem ganho | Se só chama métodos, use interface |
| Esperar especialização por tipo | Não existe | Type switch em `any(x)` ou funções separadas |
| Esperar método genérico satisfazendo interface | Não é permitido (1.27) | Função ou método concreto |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Função genérica | `func F[T Restricao](x T) T` |
| Inferência | O compilador deduz `T` na maioria das chamadas (melhorada em 1.21 e 1.27) |
| `any` | Qualquer tipo, sem operações especiais |
| `comparable` | Permite `==`; exigido para chaves de map |
| `cmp.Ordered` | Permite `<`, `>` |
| Conjuntos de tipos | `~int \| ~float64`: permite operadores; `~` inclui tipos definidos |
| Tipo genérico | `type Pilha[T any] struct{...}`; métodos com receptor `(p *Pilha[T])` |
| Zero value | `var zero T` |
| 1.24 / 1.26 / 1.27 | Aliases genéricos / restrições autorreferentes / **métodos genéricos** |
| Generics vs interface | Varia o tipo → generics; varia o comportamento → interface |
| Implementação | GC shape stenciling + dicionários (meio-termo entre C++ e Java) |

---

## 🔗 Conexões

**Dentro de Go**
- Restrições são interfaces → [[Interfaces]]
- A alternativa antiga (`any` + assertions) → [[Type Assertion e Type Switch]]
- Funções genéricas da biblioteca padrão → [[Arrays e Slices]] (`slices`), [[Maps]] (`maps`), [[Operadores de Comparação e Lógicos]] (`cmp`)
- Iteradores genéricos (`iter.Seq[V]`) → [[Iteradores]]
- Tipos definidos e o `~` → [[Tipos de Dados]]
- Métodos e receptores → [[Métodos e Receptores]]
- `errors.AsType` (genérico, 1.26) → [[Wrapping e Unwrapping de Erros]]

**Algoritmos e Estruturas de Dados**
- [[Pilha (Stack)]], [[Queue (Fila)]], [[Linked Lists]], [[Árvores Binárias]], [[B-Tree]], [[Tries]] — estruturas de dados são o caso de uso número um de generics
- [[Algoritmos de Ordenação]] e [[Algoritmos de Busca]] — implementados uma vez para qualquer tipo ordenável
- [[Grafos]] — `Grafo[V comparable]` com lista de adjacência `map[V][]V`

**Sistemas Operacionais**
- [[A Linguagem C]] — em C, "genérico" é feito com `void*` e macros, sem segurança de tipos (como o `interface{}` de antes do Go 1.18)

---

⬅️ **Anterior:** [[Type Assertion e Type Switch]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Iteradores]] ➡️
