---
tags:
  - go
  - go/tipos-compostos
aliases:
  - Arrays em Go
  - Slices em Go
  - append
  - Pacote slices
ordem: 13
modulo: "03 - Tipos Compostos"
---
# Arrays e Slices

> **Nota 13 da trilha de Go.** Arrays e slices guardam **sequências de elementos do mesmo tipo**. O array é a peça de baixo nível (tamanho fixo, faz parte do tipo, é copiado por inteiro); o slice é a ferramenta do dia a dia (tamanho dinâmico, uma "janela" sobre um array). Entender o **cabeçalho do slice** (ponteiro, tamanho e capacidade), o **compartilhamento de memória** e o funcionamento do **`append`** é provavelmente a habilidade mais importante para escrever Go correto e eficiente.
>
> **Pré-requisitos:** [[Ponteiros]], [[Strings em Profundidade]].

---

## 🧭 De onde viemos

Em [[Strings em Profundidade]] vimos que uma string é um cabeçalho de **ponteiro + tamanho** apontando para bytes imutáveis, e que fatiar (`s[2:5]`) é O(1) porque compartilha os bytes. Em [[Ponteiros]] vimos que Go sempre passa por valor, e que slices "parecem" passados por referência porque o cabeçalho contém um ponteiro. Agora vamos ver a versão **mutável** dessa ideia: o slice, e o array que mora por baixo dele.

---

## 🧱 1. Arrays: sequência de tamanho fixo

> 💡 **Array:** sequência numerada de elementos do mesmo tipo, com tamanho fixo definido em **tempo de compilação**. Os elementos ficam **contíguos** na memória.

```go
var notas [5]int                       // [0 0 0 0 0]  (zero value em todos)
primos := [5]int{2, 3, 5, 7, 11}
vogais := [...]string{"a", "e", "i"}   // [...] = o compilador conta: [3]string
esparso := [5]int{1: 10, 3: 30}        // [0 10 0 30 0]  (índices explícitos)

fmt.Println(primos[0], len(primos))    // 2 5
```

```
var a [4]int32       →   16 bytes contíguos

endereço:  base   base+4  base+8  base+12
          ┌──────┬──────┬──────┬──────┐
          │  0   │  0   │  0   │  0   │
          └──────┴──────┴──────┴──────┘
           a[0]   a[1]   a[2]   a[3]

endereço de a[i] = base + i × tamanho(int32)   → acesso O(1)
```

### 1.1 O tamanho faz parte do tipo

```go
var a [3]int
var b [5]int
a = b              // ❌ cannot use b (variable of type [5]int) as [3]int value
```

`[3]int` e `[5]int` são tipos **diferentes**. Por isso uma função que recebe `[3]int` não aceita um array de outro tamanho, o que torna arrays pouco flexíveis como parâmetros.

### 1.2 Arrays são valores: atribuir copia tudo

Saída real:

```go
a := [3]int{1, 2, 3}
b := a           // copia os 3 elementos
b[0] = 99
fmt.Println(a, b)                  // [1 2 3] [99 2 3]
fmt.Println(a == [3]int{1, 2, 3})  // true   (arrays de comparáveis são comparáveis)
```

Diferente de C, onde o nome de um array "vira" um ponteiro para o primeiro elemento, em Go o array é um **valor completo**. Passar um array de 1 MB para uma função copia 1 MB (a não ser que você passe `*[N]T`).

### 1.3 Arrays multidimensionais

```go
var m [3][3]int
m[1][2] = 6
fmt.Println(m)    // [[0 0 0] [0 0 6] [0 0 0]]

// Na memória: 9 ints contíguos, linha após linha (row-major, como C)
// [m00 m01 m02 | m10 m11 m12 | m20 m21 m22]
```

### 1.4 Quando usar arrays

Raramente diretamente. Os casos típicos:

| Uso | Exemplo |
| --- | --- |
| Tamanho fixo por natureza | `[32]byte` (hash SHA-256), `[16]byte` (UUID), `[4]byte` (IPv4), `[6]byte` (MAC) |
| Chave de map composta | `map[[2]int]bool` (arrays são comparáveis; slices não) |
| Valor que deve ser copiado | Pequenas matrizes, coordenadas |
| Por baixo de todo slice | Sempre existe um array "anônimo" guardando os dados |

---

## 🪟 2. Slices: uma janela sobre um array

> 💡 **Slice:** um descritor de um **trecho contíguo de um array**. Não guarda os dados: guarda um **ponteiro** para o primeiro elemento visível, o **tamanho** (quantos elementos estão visíveis) e a **capacidade** (quantos elementos existem do início da janela até o fim do array).

### 2.1 O cabeçalho de 24 bytes

```
s := make([]int, 3, 5)

   cabeçalho (24 bytes)              array subjacente (na heap ou na stack)
 ┌──────────────────────┐           ┌─────┬─────┬─────┬─────┬─────┐
 │ ptr  ────────────────┼─────────▶ │  0  │  0  │  0  │  0  │  0  │
 ├──────────────────────┤           └─────┴─────┴─────┴─────┴─────┘
 │ len  = 3             │             s[0]  s[1]  s[2]  ░░░░░░░░░░░
 ├──────────────────────┤             └── visíveis ──┘  └ reservados ┘
 │ cap  = 5             │                               (só o append usa)
 └──────────────────────┘
```

- `len(s)`: quantos elementos você pode acessar com `s[i]`
- `cap(s)`: quantos cabem antes de precisar de um array novo
- Acessar `s[i]` com `i >= len(s)` dá panic, **mesmo que exista capacidade**:

```go
z := []int{1, 2, 3}
z[5] = 1    // panic: runtime error: index out of range [5] with length 3
```

> 💡 **Checagem de limites (bounds checking):** antes de cada acesso por índice, Go verifica se o índice está dentro do slice. É isso que impede os *buffer overflows* de C. O compilador remove a checagem quando consegue provar que o índice é seguro (por exemplo, em `for i := range s`).

### 2.2 Criando slices

```go
// 1) Literal: cria o array e o slice de uma vez
nums := []int{1, 2, 3}             // len 3, cap 3

// 2) make: tamanho (e capacidade opcional) explícitos, elementos zerados
a := make([]int, 3)                // [0 0 0], len 3, cap 3
b := make([]int, 0, 100)           // [], len 0, cap 100 (pronto para 100 appends)

// 3) Fatiando um array ou outro slice
arr := [5]int{10, 20, 30, 40, 50}
s := arr[1:4]                      // [20 30 40]

// 4) Zero value: slice nil
var vazio []int                    // nil, len 0, cap 0
```

### 2.3 Slice nil vs slice vazio

Saída real:

```go
var ns []int
fmt.Println(ns == nil, len(ns), cap(ns))   // true 0 0
ns = append(ns, 1)                         // ✅ append em nil funciona
fmt.Println(ns)                            // [1]

e := []int{}
fmt.Println(e == nil)                      // false
```

| | `var s []int` (nil) | `s := []int{}` (vazio) |
| --- | --- | --- |
| `s == nil` | `true` | `false` |
| `len`, `cap` | 0, 0 | 0, 0 |
| `range`, `append`, `len` | funcionam | funcionam |
| JSON (`encoding/json`) | `null` | `[]` |

> ✅ Na maior parte do código, trate os dois igualmente e teste "vazio" com `len(s) == 0`, nunca com `s == nil`. Prefira `var s []int` para começar vazio. A diferença só importa em serialização (JSON) e em APIs que distinguem "ausente" de "vazio".

---

## ✂️ 3. Fatiamento: `s[início:fim]`

`s[i:j]` cria um **novo cabeçalho** que vê os elementos de `i` até `j-1`. **Não copia** nada. Saída real:

```go
s := []int{10, 20, 30, 40, 50}

s1 := s[1:4]
fmt.Println(s1, len(s1), cap(s1))   // [20 30 40] 3 4
```

```
array:      [10] [20] [30] [40] [50]
índice:       0    1    2    3    4

s     = ptr→0, len 5, cap 5      ████ ████ ████ ████ ████
s[1:4]= ptr→1, len 3, cap 4           ████ ████ ████ ░░░░
                                      └─ visível ─┘ └cap┘
```

| Expressão | Significado | len | cap |
| --- | --- | --- | --- |
| `s[i:j]` | de `i` até `j-1` | `j-i` | `cap(s)-i` |
| `s[:j]` | do início até `j-1` | `j` | `cap(s)` |
| `s[i:]` | de `i` até o fim | `len(s)-i` | `cap(s)-i` |
| `s[:]` | tudo | `len(s)` | `cap(s)` |
| `s[i:j:k]` | de `i` até `j-1`, **limitando a capacidade** até `k` | `j-i` | `k-i` |

### 3.1 Fatias compartilham a memória

Esta é a consequência mais importante (saída real):

```go
s := []int{10, 20, 30, 40, 50}
s1 := s[1:4]
s1[0] = 99
fmt.Println(s)    // [10 99 30 40 50]   ← alterar s1 alterou s!
```

É o mesmo array por baixo. Isso é ótimo para performance (fatiar é grátis) e perigoso se você esquece.

### 3.2 A expressão de três índices

`s[i:j:k]` limita a capacidade do resultado (saída real):

```go
s2 := s[1:3:4]
fmt.Println(s2, len(s2), cap(s2))   // [99 30] 2 3
```

Limitar a capacidade serve para proteger o array original de `append`s feitos sobre a fatia (veremos por quê na seção 4.3).

---

## ➕ 4. `append`: como o slice cresce

```go
s = append(s, 4)               // um elemento
s = append(s, 5, 6, 7)         // vários
s = append(s, outro...)        // todos os elementos de outro slice
b = append(b, "texto"...)      // caso especial: string em []byte
```

### 4.1 O algoritmo

```
append(s, v):
  se len(s) < cap(s):
      escreve v em s[len], devolve cabeçalho com len+1     → mesmo array
  senão:
      aloca um array NOVO e maior
      copia os elementos antigos
      escreve v
      devolve cabeçalho apontando para o array novo         → array diferente!
```

Por isso a regra de ouro:

> ⚠️ **Sempre atribua o resultado: `s = append(s, v)`.** O `append` pode devolver um cabeçalho apontando para outro array; a variável antiga continuaria vendo o array velho.

### 4.2 Quanto ele cresce

Saída real da capacidade a cada realocação, adicionando 2000 elementos um por um:

```
1 2 4 8 16 32 64 128 256 512 848 1280 1792 2560
└────────── dobra ──────────┘ └─ cresce menos ─┘
```

A regra atual (desde o Go 1.18):

- Até 256 elementos: **dobra** a capacidade
- Acima disso: cresce aproximadamente `cap += (cap + 3×256) / 4`, uma transição suave de ×2 para ×1,25
- Depois, o tamanho é **arredondado para a classe de tamanho** do alocador (por isso aparecem números como 848 em vez de 832)

> 💡 **Custo amortizado:** cada realocação copia tudo, mas como a capacidade cresce em proporção ao tamanho, o custo médio por `append` é **O(1) amortizado**. É a mesma análise de vetores dinâmicos que você viu em [[Arrays]] (AED).

> ✅ Se você sabe (ou estima) quantos elementos vai ter, **pré-aloque**: `make([]T, 0, n)`. Evita todas as realocações e cópias.
> ```go
> resultado := make([]string, 0, len(usuarios))
> for _, u := range usuarios {
> 	resultado = append(resultado, u.Nome)
> }
> ```

### 4.3 A armadilha: `append` numa fatia sobrescreve o original

Saída real:

```go
base := []int{1, 2, 3, 4}
p := base[:2]           // [1 2], len 2, cap 4  ← tem capacidade sobrando!
p = append(p, 100)      // cabe na capacidade → escreve em base[2]
fmt.Println(base, p)    // [1 2 100 4] [1 2 100]   ← o 3 sumiu!
```

O `append` encontrou espaço na capacidade e escreveu **por cima** de um elemento que pertencia a `base`. Correções:

```go
p := base[:2:2]                    // ✅ cap = 2: o append é obrigado a alocar
p := slices.Clone(base[:2])        // ✅ cópia independente
```

### 4.4 Passando slices para funções

Saída real:

```go
func zerarPrimeiro(s []int) { s[0] = 0 }

y := []int{1, 2, 3}
zerarPrimeiro(y)
fmt.Println(y)        // [0 2 3]   ← a função alterou o elemento (mesmo array)
```

```go
func adicionar(s []int, v int) {
	s = append(s, v)  // altera só a CÓPIA do cabeçalho
	s[0] = -1
}

x := make([]int, 3, 10)
adicionar(x, 7)
fmt.Println(x, len(x), x[:4])   // [-1 0 0] 3 [-1 0 0 7]
```

O que aconteceu:

```
main: x = {ptr, len 3, cap 10}         func: s = {ptr, len 3, cap 10}  (cópia)
                                        append → s = {ptr, len 4, cap 10}
                     ┌──────────────────────────────────┐
                     ▼                                  │
array: [ -1 | 0 | 0 | 7 | ░ | ░ ... ]   ← os dois compartilham o array
main enxerga só len 3: [-1 0 0]; o 7 está lá, mas "fora da janela" de x
```

> ✅ **Regra:** uma função que pode mudar o **tamanho** do slice deve **devolver** o slice novo, como o próprio `append` faz: `func adicionar(s []int, v int) []int { return append(s, v) }`.

---

## 📋 5. `copy`: cópia de verdade

`copy(destino, origem)` copia `min(len(destino), len(origem))` elementos e devolve quantos copiou (saída real):

```go
c := make([]int, 2)
n := copy(c, []int{7, 8, 9})
fmt.Println(n, c)       // 2 [7 8]
```

```go
clone := make([]int, len(original))
copy(clone, original)                 // forma clássica

clone := slices.Clone(original)       // forma moderna (Go 1.21+)

copy(s[1:], s)                        // ✅ copy funciona com regiões sobrepostas (como memmove)
copy(b, "texto")                      // caso especial: string → []byte
```

---

## 🗑️ 6. Removendo, inserindo e limpando

```go
s := []int{1, 2, 3, 4, 5}

// Remover índices [1, 3) mantendo a ordem — O(n)
s = slices.Delete(s, 1, 3)            // [1 4 5]

// Inserir na posição 1
s = slices.Insert(s, 1, 8, 9)         // [1 8 9 4 5]

// Remover sem manter a ordem — O(1): troca pelo último
i := 2
s[i] = s[len(s)-1]
s = s[:len(s)-1]

// Esvaziar mantendo a capacidade (reaproveitar o array)
s = s[:0]

// Zerar todos os elementos (len continua igual)
clear(s)                              // Go 1.21+
```

> 💡 Desde o Go 1.22, `slices.Delete` **zera** os elementos que ficaram "sobrando" no fim do array. Isso evita que ponteiros esquecidos ali mantenham objetos vivos (vazamento de memória discreto com slices de ponteiros).

---

## 🧰 7. O pacote `slices` (Go 1.21+)

Funções genéricas (funcionam com qualquer tipo de elemento; veremos como em [[Generics]]). Saídas reais:

```go
o := []int{3, 1, 4, 1, 5, 9, 2, 6}

slices.Sort(o)                        // [1 1 2 3 4 5 6 9]  (in-place)
slices.BinarySearch(o, 5)             // 5 true  (exige ordenado)
slices.Contains(o, 7)                 // false
slices.Index(o, 4)                    // 4
slices.Max(o), slices.Min(o)          // 9 1
slices.Compact(o)                     // [1 2 3 4 5 6 9]  (remove repetidos ADJACENTES)
slices.Reverse(o)
slices.Equal(a, b)
slices.Clone(o)
slices.Concat(a, b, c)                // Go 1.22+
slices.Repeat([]int{0}, 5)            // Go 1.23+

// Ordenar structs por um campo (estável = mantém a ordem dos empates)
type U struct {
	Nome  string
	Idade int
}
us := []U{{"Bia", 30}, {"Ana", 25}, {"Caio", 30}}
slices.SortStableFunc(us, func(a, b U) int { return cmp.Compare(a.Idade, b.Idade) })
// [{Ana 25} {Bia 30} {Caio 30}]

slices.IndexFunc(us, func(u U) bool { return u.Nome == "Caio" })   // 2
slices.ContainsFunc(us, func(u U) bool { return u.Idade > 40 })    // false
```

Funções que produzem **iteradores** (Go 1.23+), úteis com `range` (ver [[Iteradores]]):

```go
for i, v := range slices.All(s) { }        // índice e valor
for v := range slices.Values(s) { }        // só valores
for i, v := range slices.Backward(s) { }   // de trás para frente
ordenadas := slices.Sorted(maps.Keys(m))   // coleta um iterador num slice ordenado
for lote := range slices.Chunk(s, 100) { } // pedaços de 100 elementos
```

> 💡 Antes do Go 1.21, ordenava-se com o pacote `sort` (`sort.Ints`, `sort.Slice`). Ele ainda funciona, mas `slices` é mais rápido (genérico, sem interface) e mais seguro em tipos.

---

## 🧮 8. Slices de slices (matrizes dinâmicas)

```go
linhas, colunas := 3, 4
m := make([][]int, linhas)
for i := range m {
	m[i] = make([]int, colunas)
}
m[2][3] = 42
```

```
m (slice de cabeçalhos)        cada linha: um array separado na heap
┌─────────┐
│ m[0] ───┼──▶ [0 0 0 0]
│ m[1] ───┼──▶ [0 0 0 0]         ← linhas podem ficar longe umas das outras
│ m[2] ───┼──▶ [0 0 0 42]
└─────────┘
```

Para matrizes grandes e densas (processamento de imagem, cálculo numérico), uma única fatia contígua é mais rápida, porque aproveita melhor o cache:

```go
plana := make([]int, linhas*colunas)
plana[2*colunas+3] = 42            // equivale a m[2][3]
```

> 💡 **Localidade espacial:** quando a CPU lê um endereço, ela traz para o cache uma **linha de cache** inteira (tipicamente 64 bytes). Percorrer elementos contíguos aproveita cada linha trazida; saltar pela memória desperdiça. Você estudou a hierarquia de memória em [[Memória]].

---

## 🔬 9. Por dentro

### 9.1 A estrutura no runtime

No código do runtime (`runtime/slice.go`), um slice é literalmente:

```go
type slice struct {
	array unsafe.Pointer   // ponteiro para o primeiro elemento
	len   int
	cap   int
}
```

E `append` além da capacidade chama `runtime.growslice`, que calcula a nova capacidade, pede memória ao alocador (`mallocgc`), copia com `memmove` e devolve o novo cabeçalho.

### 9.2 Stack ou heap?

O array subjacente **não** vai sempre para a heap:

```go
func f() int {
	buf := make([]int, 8)     // tamanho constante, pequeno, não escapa
	return buf[0]             // → array na STACK (sem alocação, sem GC)
}

func g(n int) []int {
	return make([]int, n)     // devolvido ao chamador → ESCAPA para a heap
}
```

O escape analysis decide (`go build -gcflags=-m` mostra). Nas versões 1.25 e 1.26, o compilador passou a colocar na stack o array de slices em **mais situações** (inclusive alguns `make` com tamanho variável, quando o tamanho é pequeno). Veremos em [[Memória e Escape Analysis]].

### 9.3 Checagem de limites e sua eliminação

```go
for i := 0; i < len(s); i++ {
	total += s[i]          // o compilador PROVA que 0 <= i < len(s) → sem checagem
}

_ = s[3]                   // "dica" ao compilador: uma checagem só aqui...
a, b, c, d := s[0], s[1], s[2], s[3]   // ...e nenhuma nestas
```

`go build -gcflags=-d=ssa/check_bce/debug=1` mostra onde as checagens ficaram.

### 9.4 Retenção de memória

Como uma fatia mantém **o array inteiro** vivo, guardar um pedaço pequeno de um slice enorme retém toda a memória:

```go
func primeirosBytes(arquivo []byte) []byte {
	return arquivo[:16]                  // ⚠️ mantém os 100 MB vivos
}
func primeirosBytesOk(arquivo []byte) []byte {
	return slices.Clone(arquivo[:16])    // ✅ só 16 bytes ficam vivos
}
```

---

## ⚖️ 10. Array vs slice

| | Array `[N]T` | Slice `[]T` |
| --- | --- | --- |
| Tamanho | Fixo, parte do tipo | Dinâmico |
| Atribuição / passagem | Copia todos os elementos | Copia o cabeçalho (24 bytes) |
| Zero value | N elementos zerados | `nil` |
| Comparável com `==` | Sim (se `T` for) | Não (use `slices.Equal`) |
| Pode ser chave de map | Sim | Não |
| Cresce | Não | Com `append` |
| Uso | Tamanhos fixos por natureza | **Quase sempre** |

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Não reatribuir o `append` | Elementos "somem" | `s = append(s, v)` |
| `append` em fatia com capacidade sobrando | Sobrescreve o original | `s[i:j:j]` ou `slices.Clone` |
| Alterar elementos de uma fatia | Altera o slice de origem | Copie se precisar de independência |
| Função que faz `append` sem devolver | Chamador não vê o crescimento | Devolva o slice novo |
| Testar vazio com `== nil` | `[]int{}` não é nil | `len(s) == 0` |
| Índice ≥ `len` (mesmo < `cap`) | panic | Respeite `len`; use `append` para crescer |
| Pedaço pequeno de slice gigante guardado | Memória não liberada | `slices.Clone` |
| Não pré-alocar com tamanho conhecido | Realocações e cópias | `make([]T, 0, n)` |
| `slices.Compact` sem ordenar | Duplicatas não adjacentes ficam | Ordene antes |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Array | Tamanho fixo no tipo, contíguo, é um **valor** (copiado inteiro) |
| Slice | Cabeçalho {ptr, len, cap} de 24 bytes sobre um array |
| `len` / `cap` | Visíveis / disponíveis até o fim do array |
| nil vs vazio | Comportam-se igual; teste com `len(s) == 0` |
| Fatiar | O(1), **compartilha** o array |
| `s[i:j:k]` | Limita a capacidade |
| `append` | Escreve na capacidade ou realoca (×2 até 256, depois ~×1,25) |
| Regra do append | Sempre `s = append(s, ...)` |
| `copy` / `slices.Clone` | Cópia independente |
| Pacote `slices` | Sort, BinarySearch, Contains, Index, Delete, Insert, iteradores |
| Performance | Pré-alocar; contiguidade favorece o cache |

---

## 🔗 Conexões

**Dentro de Go**
- O cabeçalho imutável equivalente → [[Strings em Profundidade]]
- Por que slices não são comparáveis → [[Operadores de Comparação e Lógicos]]
- Passagem por valor e ponteiros internos → [[Ponteiros]]
- Funções genéricas como `slices.Sort` → [[Generics]]
- `range` e iteradores sobre slices → [[Controle de Fluxo]], [[Iteradores]]
- Funções variádicas recebem um slice → [[Funções Variádicas]]
- Stack vs heap do array subjacente → [[Memória e Escape Analysis]]
- Medir o efeito da pré-alocação → [[Benchmarks]]

**Sistemas Operacionais**
- [[Memória]] — linhas de cache e localidade espacial explicam por que dados contíguos são rápidos
- [[Gerenciando a Memória Livre]] — o alocador do Go arredonda pedidos para classes de tamanho, como os alocadores que o Tanenbaum descreve
- [[Espaços de Endereçamento]] — a checagem de limites protege o espaço do processo de leituras e escritas fora do buffer

**Algoritmos e Estruturas de Dados**
- [[Arrays]] — vetor dinâmico, crescimento geométrico e custo amortizado O(1)
- [[Algoritmos de Ordenação]] e [[Algoritmos de Busca]] — `slices.Sort` (pdqsort) e `slices.BinarySearch`
- [[Pilha (Stack)]] e [[Queue (Fila)]] — implementadas naturalmente com slices e `append`/fatiamento

---

⬅️ **Anterior:** [[Strings em Profundidade]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Maps]] ➡️
