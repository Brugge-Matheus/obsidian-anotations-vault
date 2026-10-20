---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Iterators
  - iter.Seq
  - Range over func
  - range sobre funções
  - yield
ordem: 26
modulo: "05 - Métodos, Interfaces e Generics"
---
# Iteradores

> **Nota 26 da trilha de Go.** Desde o **Go 1.23**, o `for range` também funciona com **funções**. Isso criou um jeito padrão de escrever **iteradores**: funções que produzem uma sequência de valores, um de cada vez, para serem consumidos com `range`. Iteradores permitem percorrer estruturas próprias (árvores, resultados de banco, linhas de arquivo) com a mesma sintaxe de slices e maps, encadear transformações sem criar slices intermediários e representar sequências **infinitas**. Nesta nota veremos o problema que eles resolvem, os tipos `iter.Seq` e `iter.Seq2`, o protocolo do `yield`, como escrever e compor iteradores, os iteradores da biblioteca padrão, iteradores "pull" e como o compilador os transforma.
>
> **Pré-requisitos:** [[Generics]], [[Closures e Funções Anônimas]], [[Controle de Fluxo]].

---

## 🧭 De onde viemos

Em [[Controle de Fluxo]], o `range` funcionava com slices, arrays, strings, maps, channels e inteiros, e mencionamos que desde o 1.23 ele também aceita funções. Em [[Closures e Funções Anônimas]] fizemos um gerador de Fibonacci com closure (`f()` a cada valor). Em [[Generics]] vimos tipos parametrizados. Iteradores são a combinação dos três.

> 💡 **Iterador:** um objeto ou função que dá acesso aos elementos de uma sequência, um por vez, sem expor como a sequência é armazenada (ou mesmo se ela está armazenada: os valores podem ser calculados sob demanda).

---

## 😩 1. O problema: cada um iterava de um jeito

Antes do Go 1.23, não havia um jeito padrão de percorrer uma coleção personalizada. Cada biblioteca inventava o seu:

```go
// 1) Devolver um slice com tudo: simples, mas aloca tudo de uma vez (e se forem 10 milhões?)
func (a *Arvore) Valores() []int

// 2) Callback: eficiente, mas não dá para usar "for", e parar no meio é estranho
func (a *Arvore) Percorrer(f func(v int) bool)

// 3) Método Next(): estilo bufio.Scanner, sql.Rows
for rows.Next() { rows.Scan(&v) }
if err := rows.Err(); err != nil { ... }

// 4) Channel + goroutine: bonito no range, mas lento e vaza goroutine se parar no meio
func (a *Arvore) Canal() <-chan int
```

O Go 1.23 padronizou: **um iterador é uma função**, e o `range` sabe consumir essa função.

---

## 🧩 2. Os tipos `iter.Seq` e `iter.Seq2`

O pacote `iter` define dois tipos genéricos:

```go
package iter

type Seq[V any]     func(yield func(V) bool)          // sequência de valores
type Seq2[K, V any] func(yield func(K, V) bool)       // sequência de pares (índice/valor, chave/valor)
```

Lendo devagar:

- Um `Seq[V]` é uma **função** que recebe outra função, chamada por convenção de **`yield`**
- O iterador chama `yield(v)` **para cada valor** que quer entregar
- `yield` devolve um `bool`: **`true`** = "pode continuar", **`false`** = "pare" (o laço deu `break`, `return`...)
- Quando `yield` devolve `false`, o iterador **deve parar** imediatamente

> 💡 **`yield`** ("produzir", "ceder"): nome tradicional, vindo de Python, C# e outras linguagens, para a operação de entregar um valor ao consumidor e esperar ele pedir o próximo. Em Go, `yield` não é palavra-chave: é só o nome convencional do parâmetro.

---

## ✍️ 3. Escrevendo o primeiro iterador

Saída real:

```go
func Contar(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) {        // entrega i; se o consumidor parou, sai
				return
			}
		}
	}
}

for i := range Contar(3) {
	fmt.Print(i, " ")         // 0 1 2
}
```

O que acontece no `for i := range Contar(3)`:

```
for i := range seq { CORPO }

é transformado pelo compilador em algo como:

seq(func(i int) bool {     ← o CORPO do laço vira a função yield
	CORPO
	return true            ← chegar ao fim do corpo = "continue"
})                         ← break = return false

Contar(3) chama yield(0) → executa o corpo com i=0 → true
              yield(1) → executa o corpo com i=1 → true
              yield(2) → executa o corpo com i=2 → true
          o for de dentro termina → o range termina
```

### 3.1 Sequências infinitas e `break`

Como os valores são produzidos **sob demanda**, a sequência pode ser infinita. Quem consome decide quando parar (saída real):

```go
func Fibonacci() iter.Seq[int] {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for {                     // laço infinito...
			if !yield(a) {        // ...até o consumidor parar
				return
			}
			a, b = b, a+b
		}
	}
}

for f := range Fibonacci() {
	if f > 50 {
		break                     // faz o yield devolver false
	}
	fmt.Print(f, " ")             // 0 1 1 2 3 5 8 13 21 34
}
```

### 3.2 A regra de ouro: respeite o `false`

Se o iterador **ignorar** o `false` e chamar `yield` de novo, o runtime detecta e dá panic (saída real):

```go
func Ruim() iter.Seq[int] {
	return func(yield func(int) bool) {
		yield(1)                  // ❌ ignora o retorno
		yield(2)
	}
}

for v := range Ruim() {
	fmt.Println("recebi", v)
	break
}
// recebi 1
// panic: runtime error: range function continued iteration after function for loop body returned false
```

> ✅ Todo `yield(...)` deve estar num `if !yield(...) { return }`.

---

## 🌳 4. Iteradores para os seus tipos

O caso de uso mais importante: dar `range` a estruturas próprias. Uma árvore binária percorrida em ordem, com parada antecipada funcionando (saída real):

```go
type No struct {
	Valor    int
	Esq, Dir *No
}

func (n *No) EmOrdem() iter.Seq[int] {
	return func(yield func(int) bool) {
		n.percorrer(yield)
	}
}

// devolve false se o consumidor pediu para parar
func (n *No) percorrer(yield func(int) bool) bool {
	if n == nil {
		return true
	}
	return n.Esq.percorrer(yield) && yield(n.Valor) && n.Dir.percorrer(yield)
}

//        4
//      /   \
//     2     6
//    / \     \
//   1   3     7
for v := range arv.EmOrdem() {
	if v > 5 {
		break                     // a recursão inteira para, graças aos && em curto-circuito
	}
	fmt.Print(v, " ")             // 1 2 3 4
}
```

Repare como o curto-circuito do `&&` ([[Operadores de Comparação e Lógicos]]) propaga o "pare" por toda a recursão.

### 4.1 `Seq2`: pares

Saída real:

```go
func Enumerar[V any](s []V) iter.Seq2[int, V] {
	return func(yield func(int, V) bool) {
		for i, v := range s {
			if !yield(i, v) {
				return
			}
		}
	}
}

for i, s := range Enumerar([]string{"a", "b"}) {
	fmt.Print(i, s, " ")          // 0a 1b
}
```

Convenção de nomes na biblioteca padrão: um método `All()` devolve `Seq2` com todos os pares; `Keys()` e `Values()` devolvem `Seq`; `Backward()` percorre ao contrário.

---

## 🔗 5. Compondo iteradores

Como iteradores são funções, dá para escrever **adaptadores** genéricos que recebem um iterador e devolvem outro. Nada é calculado até alguém consumir (avaliação **preguiçosa**). Saída real:

```go
func Filtrar[V any](seq iter.Seq[V], ok func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if ok(v) && !yield(v) {
				return
			}
		}
	}
}

func Limitar[V any](seq iter.Seq[V], n int) iter.Seq[V] {
	return func(yield func(V) bool) {
		if n <= 0 {
			return
		}
		i := 0
		for v := range seq {
			if !yield(v) {
				return
			}
			i++
			if i == n {
				return
			}
		}
	}
}

pares := Filtrar(Fibonacci(), func(n int) bool { return n%2 == 0 })
fmt.Println(slices.Collect(Limitar(pares, 5)))   // [0 2 8 34 144]
```

Repare: partimos de uma sequência **infinita** (`Fibonacci`), filtramos e limitamos, e só os valores necessários foram calculados. Nenhum slice intermediário foi criado.

> 💡 **Avaliação preguiçosa (lazy evaluation):** os valores só são produzidos quando alguém os pede. O oposto é a avaliação **ansiosa**, como em `Filtrar(slice)` devolvendo um slice novo completo.

> 💡 A biblioteca padrão não inclui `Filter`/`Map` para iteradores (de propósito, para não estimular cadeias longas e difíceis de ler). O pacote experimental `golang.org/x/exp/xiter` tem várias. Um `for` com `if` costuma ser igualmente claro.

---

## 📚 6. Iteradores na biblioteca padrão

Saídas reais dos exemplos:

```go
// slices (Go 1.23)
slices.All(s)          // Seq2[int, E]: índice e valor
slices.Values(s)       // Seq[E]
slices.Backward(s)     // Seq2 de trás para frente
for i, v := range slices.Backward([]int{1, 2, 3}) { ... }   // 2:3 1:2 0:1
slices.Collect(seq)    // iterador → slice
slices.Sorted(seq)     // iterador → slice ordenado
slices.Chunk(s, n)     // pedaços de n elementos

// maps (Go 1.23)
maps.Keys(m), maps.Values(m), maps.All(m)
slices.Sorted(maps.Keys(m))              // [a b c]: o jeito moderno de iterar um map em ordem
maps.Collect(seq2)                        // iterador de pares → map

// strings e bytes (Go 1.24)
strings.Lines(s)                          // linhas (com o \n)
strings.SplitSeq("x,y,z", ",")            // x y z, sem criar slice
strings.FieldsSeq(s)

// reflect (Go 1.26)
for campo, valor := range reflect.ValueOf(x).Fields() { ... }
```

---

## 🔄 7. Iteradores "push" e "pull"

Os iteradores que vimos são **push**: o iterador **empurra** valores para o corpo do laço, e quem controla o ritmo é o iterador. Às vezes precisamos do contrário, **puxar** valores sob demanda, por exemplo para percorrer **duas** sequências ao mesmo tempo (intercalar, comparar, fazer "zip"). Para isso existe `iter.Pull` (saída real):

```go
next, stop := iter.Pull(Contar(3))
defer stop()                      // ⚠️ sempre chame stop: libera os recursos do iterador

for {
	v, ok := next()               // pede o próximo valor
	if !ok {
		break                     // acabou
	}
	fmt.Print("pull:", v, " ")    // pull:0 pull:1 pull:2
}
```

```go
// Exemplo: as duas sequências são iguais?
func Iguais[V comparable](a, b iter.Seq[V]) bool {
	nextA, stopA := iter.Pull(a)
	defer stopA()
	nextB, stopB := iter.Pull(b)
	defer stopB()
	for {
		va, okA := nextA()
		vb, okB := nextB()
		if okA != okB || va != vb {
			return false
		}
		if !okA {
			return true
		}
	}
}
```

> 💡 `iter.Pull` é implementado com **corrotinas** no runtime: uma troca de contexto leve entre o iterador e quem chama `next`, sem criar uma goroutine comum nem passar pelo escalonador. É bem mais barato do que usar channels para o mesmo fim.

---

## 🔬 8. Por dentro: o compilador reescreve o laço

O `for ... range` sobre uma função é **reescrito** pelo compilador: o corpo do laço vira uma **closure** (o `yield`), e `break`, `continue`, `return` e `goto` dentro do corpo são traduzidos para retornos dessa closure com um estado indicando o que aconteceu:

```go
for v := range seq {
	if v == 3 { continue }
	if v == 5 { return v }      // return da FUNÇÃO de fora, de dentro do corpo
	use(v)
}

// conceitualmente vira:
var resultado int
var saiuComReturn bool
seq(func(v int) bool {
	if v == 3 { return true }               // continue → "pode mandar o próximo"
	if v == 5 {
		resultado, saiuComReturn = v, true
		return false                        // return → "pare", e lembra de retornar depois
	}
	use(v)
	return true
})
if saiuComReturn { return resultado }
```

Consequências:

- `defer` dentro do corpo do laço pertence à **função de fora**, como num laço comum (o compilador cuida disso)
- Se o iterador e o corpo forem simples, o compilador **inlina** tudo e o resultado é tão rápido quanto um `for` escrito à mão
- O runtime verifica o protocolo (por isso o panic quando o iterador ignora o `false`)

> 💡 Comparação com outras linguagens: Python e C# usam **geradores** (funções com `yield` como palavra-chave, que pausam e retomam). Rust e Java usam objetos com método `next()` (iteradores "pull"). Go escolheu o modelo "push" com funções comuns e closures, que não exige nenhuma mudança no runtime para o caso comum, e oferece o "pull" via `iter.Pull` quando necessário.

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Ignorar o retorno de `yield` | `panic: range function continued iteration...` | `if !yield(v) { return }` |
| Esquecer `stop()` do `iter.Pull` | Recursos do iterador não liberados | `defer stop()` logo após o `Pull` |
| Iterador que não para em sequência infinita | Laço infinito se o consumidor não der `break` | Documente; ofereça `Limitar` |
| Consumir o mesmo iterador duas vezes esperando o mesmo resultado | Depende de como foi escrito (alguns são de uso único) | Documente se é reutilizável |
| Cadeias longas de adaptadores | Código difícil de ler | Um `for` com `if` costuma ser mais claro |
| Erros durante a iteração | `Seq` não tem lugar para erro | Use `Seq2[V, error]` ou um método `Err()` depois do laço |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Iterador | Função `func(yield func(V) bool)`; consumida com `range` (Go 1.23+) |
| `iter.Seq[V]` / `iter.Seq2[K,V]` | Tipos padrão para sequências de valores / pares |
| `yield` | Entrega um valor; `false` = pare imediatamente |
| Infinitos | Possíveis: valores calculados sob demanda |
| Seus tipos | Métodos `All()`, `Keys()`, `Values()`, `Backward()` devolvendo iteradores |
| Composição | Adaptadores genéricos, avaliação preguiçosa |
| Biblioteca padrão | `slices`, `maps` (1.23), `strings`/`bytes` (1.24), `reflect` (1.26) |
| Pull | `iter.Pull` → `next()` e `stop()`; usa corrotinas |
| Por dentro | O corpo do laço vira a closure `yield`; geralmente inlinado |

---

## 🔗 Conexões

**Dentro de Go**
- `range` e suas formas → [[Controle de Fluxo]]
- Closures por trás de cada iterador → [[Closures e Funções Anônimas]]
- Tipos genéricos `Seq[V]` → [[Generics]]
- `slices.Collect`, `slices.Sorted` → [[Arrays e Slices]]
- `maps.Keys` e iteração ordenada → [[Maps]]
- `strings.Lines`, `strings.SplitSeq` → [[Strings em Profundidade]]
- A alternativa antiga com channels (e seus custos) → [[Channels]]
- Iteradores de campos em `reflect` → [[Reflection]]

**Algoritmos e Estruturas de Dados**
- [[Árvores Binárias]] — percursos em ordem, pré-ordem e pós-ordem como iteradores
- [[Grafos]] — busca em largura/profundidade expostas como `iter.Seq[No]`
- [[Linked Lists]] — percorrer uma lista com `range`

**Sistemas Operacionais**
- [[Implementando Threads em User Space]] — `iter.Pull` usa corrotinas, uma forma de troca de contexto cooperativa em espaço de usuário

---

⬅️ **Anterior:** [[Generics]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[O Padrão error em Go]] ➡️
