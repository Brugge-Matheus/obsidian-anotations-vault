---
tags:
  - go
  - go/fundamentos
---
# Arrays e Slices

> Arrays e Slices parecem similares mas têm semânticas completamente diferentes. Arrays são valores com tamanho fixo; Slices são views dinâmicas sobre arrays. Entender o layout de memória de cada um é essencial para escrever código Go eficiente.
> 

---

## 1. Arrays — Valores na Stack

Um array tem **tamanho fixo** definido em tempo de compilação. O tamanho faz parte do tipo — `[3]int` e `[5]int` são tipos completamente diferentes e incompatíveis.

### Layout de Memória

Arrays em Go são armazenados como bytes contíguos — diretamente na stack (para tamanhos pequenos) ou na heap (quando escapam ou são grandes):

```go
var a [4]int32   // 16 bytes contíguos na memória
┌────┬────┬────┬────┐
│ 0  │ 0  │ 0  │ 0  │  ← 4 × 4 bytes = 16 bytes
└────┴────┴────┴────┘
 a[0] a[1] a[2] a[3]
```

```go
// Forma explícita — zero values
var notas [5]int                         // [0, 0, 0, 0, 0]

// Com valores iniciais
primos := [5]int{2, 3, 5, 7, 11}

// Tamanho inferido pelo compilador com [...]
vogais := [...]string{"a", "e", "i", "o", "u"}   // tipo: [5]string

// Inicializar posições específicas (resto fica com zero value)
sparse := [5]int{1: 10, 3: 30}   // [0, 10, 0, 30, 0]
```

### Arrays são Valores — Cópia Completa

Esta é a diferença crítica em relação a slices. Atribuir um array **copia todos os bytes**:

```go
a := [3]int{1, 2, 3}
b := a   // cópia de 3 × 8 bytes = 24 bytes

b[0] = 99
fmt.Println(a)   // [1, 2, 3] — não foi alterado
fmt.Println(b)   // [99, 2, 3]

// Para passar por referência, use ponteiro
func dobrar(arr *[3]int) {
	for i := range arr {
		arr[i] *= 2
	}
}
dobrar(&a)
fmt.Println(a)   // [2, 4, 6]
```

> 💡 Passar um array grande por valor em uma função copia todos os bytes — pode ser caro. Use `*[N]T` para evitar a cópia ou prefira slices.
> 

### Arrays Multidimensionais

```go
// Matriz 3×3 — armazenada como 9 inteiros contíguos na memória
// Row-major order (linha por linha) como em C
matriz := [3][3]int{
	{1, 2, 3},
	{4, 5, 6},
	{7, 8, 9},
}
// Layout na memória: [1][2][3][4][5][6][7][8][9]

fmt.Println(matriz[1][2])   // 6

// Acesso é O(1): endereço = base + (linha × colunas + coluna) × sizeof(T)
```

---

## 2. Slices — A Estrutura Principal

Um slice é um **view sobre um array** — não armazena dados diretamente. Internamente é uma struct de três campos:

### Anatomia de um Slice

```go
// Internamente (simplificado do runtime):
type SliceHeader struct {
    Data unsafe.Pointer  // ponteiro para o primeiro elemento visível
    Len  int             // número de elementos acessíveis
    Cap  int             // número de elementos até o fim do array subjacente
}

// No x86-64: 3 × 8 bytes = 24 bytes por slice header
```

```
s := make([]int, 3, 5)

Stack:
┌──────────┬─────┬─────┐
│  ptr     │ len │ cap │
│ → heap   │  3  │  5  │
└──────────┴─────┴─────┘

Heap (array subjacente):
┌────┬────┬────┬────┬────┐
│ 0  │ 0  │ 0  │ -- │ -- │  ← 5 elementos alocados, 3 visíveis
└────┴────┴────┴────┴────┘
  [0]  [1]  [2]  [3]  [4]
```

### Declaração

```go
// Slice nil — len=0, cap=0, ptr=nil
var s []int
fmt.Println(s == nil)   // true
fmt.Println(len(s))     // 0 — seguro
fmt.Println(cap(s))     // 0

// Literal — o compilador cria um array subjacente anônimo
nums := []int{1, 2, 3, 4, 5}

// make(tipo, len, cap) — controle explícito
a := make([]int, 3)       // len=3, cap=3, todos zerados
b := make([]int, 3, 5)    // len=3, cap=5, indices 3 e 4 reservados mas invisíveis

// make com len=0 — slice pronto para append (padrão eficiente)
resultado := make([]int, 0, cap(entrada))   // pré-aloca sem elementos visíveis
```

### Operação de Slicing

Criar um sub-slice é **O(1)** — apenas cria um novo header:

```go
s := []int{10, 20, 30, 40, 50}
// Layout:  ptr=&s[0], len=5, cap=5

s1 := s[1:4]    // ptr=&s[1], len=3, cap=4  → [20, 30, 40]
s2 := s[:3]     // ptr=&s[0], len=3, cap=5  → [10, 20, 30]
s3 := s[2:]     // ptr=&s[2], len=3, cap=3  → [30, 40, 50]

// Sintaxe de três índices: s[low:high:max]
// cap do resultado = max - low
s4 := s[1:3:4]  // ptr=&s[1], len=2, cap=3  → [20, 30]
// Impede que s4 acesse além do índice 3 via append
```

### Sub-slices Compartilham o Array Subjacente

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:4]   // [2, 3, 4]

sub[0] = 99
fmt.Println(original)   // [1, 99, 3, 4, 5] — AFETOU o original!
fmt.Println(sub)        // [99, 3, 4]

// Para evitar: use a sintaxe de três índices para limitar o cap,
// forçando o append a alocar novo array antes de modificar
isolado := original[1:4:4]   // cap=3 — append vai alocar novo array
```

---

## 3. `append` — O Mecanismo de Crescimento

`append` é a operação mais importante de slices. Entender seu comportamento é crucial:

```go
s = append(s, elemento)                    // um elemento
s = append(s, elem1, elem2, elem3)         // múltiplos
s = append(s, outroSlice...)               // outro slice (spread)
s = append(s[:i], s[i+1:]...)              // deletar índice i
s = append(s[:i], append([]T{x}, s[i:]...)...)  // inserir em i
```

### Como o Crescimento Funciona Internamente

Quando `len == cap`, `append` precisa alocar um novo array. A estratégia de crescimento mudou ao longo das versões:

```
Go < 1.18:  se cap < 1024: nova cap = cap * 2
             se cap ≥ 1024: nova cap = cap * 1.25

Go 1.18+:   fórmula mais suave para evitar crescimento brusco
             média entre dobrar e menos que dobrar, baseada em thresholds
             (evita overallocation excessiva para slices grandes)
```

```go
s := make([]int, 0)
for i := 0; i < 10; i++ {
	s = append(s, i)
	fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// len=1  cap=1
// len=2  cap=2
// len=3  cap=4   ← realocou, dobrou
// len=4  cap=4
// len=5  cap=8   ← realocou, dobrou
// len=8  cap=8
// len=9  cap=16  ← realocou, dobrou
// len=10 cap=16

// Pré-alocar quando o tamanho é conhecido evita realocações
s2 := make([]int, 0, 10)   // cap=10 desde o início
for i := 0; i < 10; i++ {
	s2 = append(s2, i)     // sem realocações
}
```

> ⚠️ **Sempre reatribua o resultado do append**: `s = append(s, v)`. Se o append precisar realocar, a variável `s` ainda apontaria para o array antigo.
> 

---

## 4. `copy` — Cópia Independente

`copy` copia elementos entre slices com arrays subjacentes separados:

```go
original := []int{1, 2, 3, 4, 5}

// copy(dst, src) — retorna min(len(dst), len(src))
copia := make([]int, len(original))
n := copy(copia, original)   // n = 5

copia[0] = 99
fmt.Println(original)   // [1, 2, 3, 4, 5] — não afetado

// Copiar para slice menor
pequeno := make([]int, 3)
copy(pequeno, original)   // copia apenas 3 elementos
fmt.Println(pequeno)      // [1, 2, 3]

// Forma concisa: append para copiar
copia2 := append([]int(nil), original...)   // idiomático mas aloca
copia3 := original[:len(original):len(original)]   // não copia! ainda compartilha

// copy é implementado com memmove — otimizado pelo compilador
// e seguro para overlapping slices (diferente de memcpy)
```

---

## 5. Deletar Elementos

Go não tem função `delete` para slices. Padrões idiomáticos:

```go
s := []int{1, 2, 3, 4, 5}
i := 2   // deletar índice 2 (valor 3)

// Mantém ordem — O(n) (usa copy internamente)
s = append(s[:i], s[i+1:]...)
fmt.Println(s)   // [1, 2, 4, 5]

// Não mantém ordem — O(1) (swap com último + shrink)
s[i] = s[len(s)-1]
s = s[:len(s)-1]
fmt.Println(s)   // [1, 2, 5, 4] (ou similar)

// Limpar preservando a alocação (Go 1.21+)
import "slices"
s = s[:0]          // len=0, mantém cap — elementos ainda na memória
clear(s[:cap(s)])  // zera os bytes — evita memory leak com slices de ponteiros
```

---

## 6. Pacote `slices` (Go 1.21+)

Go 1.21 introduziu o pacote `slices` com funções genéricas para operações comuns:

```go
import "slices"

s := []int{3, 1, 4, 1, 5, 9, 2, 6}

// Ordenação
slices.Sort(s)                          // [1, 1, 2, 3, 4, 5, 6, 9]
slices.SortFunc(s, func(a, b int) int { return b - a })  // decrescente

// Busca (em slice ordenado)
i, encontrado := slices.BinarySearch(s, 5)   // 5, true

// Verificações
slices.Contains(s, 5)           // true
slices.Equal(s, outra)          // true se todos os elementos iguais
slices.EqualFunc(s, t, func(a, b int) bool { return a == b })

// Índice
slices.Index(s, 5)              // 4 (primeira ocorrência)
slices.Max(s)                   // 9
slices.Min(s)                   // 1

// Operações
slices.Compact(s)               // remove duplicatas adjacentes (slice deve estar ordenado)
slices.Reverse(s)               // inverte in-place
slices.Clone(s)                 // cópia independente
s = slices.Delete(s, 2, 4)     // deleta elementos de índice 2 a 3 (mantém ordem)

// Inserir
s = slices.Insert(s, 2, 99, 100)   // insere 99, 100 no índice 2
```

---

## 7. Slices 2D

```go
// Declaração e inicialização
matriz := [][]int{
	{1, 2, 3},
	{4, 5, 6},
	{7, 8, 9},
}

// Criar programaticamente — cada linha é um slice independente
linhas, colunas := 3, 4
m := make([][]int, linhas)
for i := range m {
	m[i] = make([]int, colunas)
}

// Performance: para matrizes densas, use slice 1D com indexação manual
// Isso garante localidade de cache (cache-friendly)
linhas, colunas = 1000, 1000
flat := make([]int, linhas*colunas)
// Acesso: flat[linha*colunas + coluna]
flat[2*colunas+3] = 42   // equivale a matriz[2][3]
```

---

## 8. Arrays vs Slices — Quando Usar

|  | Array `[N]T` | Slice `[]T` |
| --- | --- | --- |
| Tamanho | Fixo em compilação | Dinâmico |
| Tipo | `[N]T` — N é parte do tipo | `[]T` |
| Atribuição | Cópia completa dos bytes | Cópia do header (ptr, len, cap) |
| nil | Não existe array nil | Slice pode ser nil |
| Comparação | `==` funciona se T comparável | `==` não funciona (use `slices.Equal`) |
| Passa para função | Por valor (cópia) ou `*[N]T` | Sempre como header (referência efetiva) |
| Uso | Tamanho genuinamente fixo | Quase sempre |
| Exemplos reais | `[32]byte` (SHA256), `[2]float64` (ponto 2D) | Listas, buffers, resultados de query |

> 💡 **Regra prática:** use slices por padrão. Arrays apenas quando: (1) o tamanho é fixo e conhecido em compilação, (2) você precisa de semântica de valor, ou (3) está implementando algo que requer um array para satisfazer uma interface (como `crypto/sha256`).
>

---

## Conexão com Sistemas Operacionais

### Arrays contíguos e localidade de cache — [[Memória]]

Arrays em Go são armazenados como bytes contíguos na memória, o que tem impacto direto na performance de cache. Em [[Memória]], estudamos a **hierarquia de memória** e o conceito de **cache lines** (tipicamente 64 bytes). Quando você itera sequencialmente por um array, cada acesso a um elemento provavelmente já carregou os elementos vizinhos na cache line anterior — isso é **localidade espacial**. Listas encadeadas (linked lists) quebram essa propriedade porque os nós ficam espalhados na heap.

### Matriz flat (1D acessada como 2D) = cache-friendly — [[Memória]]

O padrão `flat[linha*colunas + coluna]` mostrado na seção 7 é diretamente explicado pelo modelo de cache de [[Memória]]: uma cache line carrega 64 bytes de uma vez. Uma matriz representada como `[][]int` (slice de slices) aloca cada linha em posições arbitrárias da heap — iteração linha por linha gera cache misses. A representação flat garante que elementos consecutivos de uma linha (e até de linhas adjacentes) cabem na mesma cache line.

### `append` e realoção na heap — [[Processos]]

Quando `len == cap` e `append` precisa crescer, ele chama o alocador de memória do runtime Go, que por sua vez pode chamar `mmap()` para obter novas páginas do SO. Em [[Processos]] vimos que a heap cresce via `brk()` ou `mmap()` — é exatamente esse mecanismo que ocorre sob o capô de um `append` que ultrapassa a capacidade. A estratégia de dobrar a capacidade amortiza o custo de syscall, minimizando as chamadas ao kernel.

### `copy` usa `memmove` — [[System Calls]] e [[Processos]]

`copy` é implementado internamente via `memmove`, que é uma operação de cópia de bloco de memória otimizada pelo compilador/CPU (instruções `MOVSQ` em x86-64). Em [[System Calls]] estudamos que operações de cópia de memória dentro de um mesmo processo não necessitam de syscall — `memmove` opera em user space. Já em [[Processos]], o conceito de **copy-on-write (COW)** aparece: quando o kernel faz fork, ele não copia a memória imediatamente, apenas marca as páginas como compartilhadas e copia sob demanda. O `copy` em Go é a versão explícita e imediata dessa cópia dentro do processo.

### Slice nil = ponteiro nulo no espaço virtual — [[Espaços de Endereçamento]]

Um slice nil tem `ptr = nil`, que em termos de [[Espaços de Endereçamento]] significa que o ponteiro contém o endereço virtual `0x0`. Toda tentativa de deferenciar esse endereço gera um **page fault** que o SO converte em sinal SIGSEGV (o "nil pointer dereference" que Go transforma em panic). A MMU protege a página zero exatamente para detectar acessos acidentais a ponteiros nulos.

### Arrays grandes escapam para a heap — [[Processos]]

Quando o compilador detecta que um array é grande demais para ficar na stack sem risco de stack overflow, ele move a alocação para a heap (escape analysis). Em [[Processos]] vimos que um stack overflow ocorre quando a pilha cresce além do seu limite (seja o limite fixo de uma thread OS, seja o limite dinâmico da goroutine). Um `[40_000_000]byte` literalmente não cabe na stack de 2 KB de uma goroutine — o compilador o promove para a heap automaticamente.

### `make()` aloca na heap — [[Processos]]

`make([]T, n)` sempre aloca o array subjacente na heap. Isso é equivalente ao `malloc()` de C. Em [[Processos]] vimos que a heap é o segmento de memória de tamanho variável de um processo, crescido via `brk()`/`mmap()`, e gerenciado pelo alocador. Em Go, o **Garbage Collector** substitui o `free()` manual — mas o mecanismo de alocação subjacente é o mesmo.

### Estratégia de crescimento do slice vs retenção de memória — [[Processos]]

Em [[Processos]] aprendemos que `free()` em C não necessariamente devolve memória ao SO imediatamente — o alocador (glibc) costuma manter a memória em pool para reutilização futura. Go tem comportamento similar: mesmo após o GC coletar slices, o runtime tende a reter as páginas de memória em seu próprio pool antes de devolvê-las ao SO. A estratégia de crescimento exponencial do slice (dobrar capacidade) segue a mesma lógica de amortização de custo de syscall que os alocadores de memória em C usam.