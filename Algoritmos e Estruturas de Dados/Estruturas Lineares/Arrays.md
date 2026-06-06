---
tags:
  - algoritmos
  - estruturas-de-dados
  - arrays
  - memoria
---

# Arrays

Um **array verdadeiro** é um bloco contíguo de memória onde elementos do mesmo tipo são armazenados em posições sequenciais. O acesso a qualquer elemento é O(1) porque usa aritmética pura de ponteiros.

Veja [[Big O Notation]] para entender as complexidades.

---

## Como funciona na memória

```
MEMÓRIA DO COMPUTADOR (endereços em bytes)
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 100 │ 101 │ 102 │ 103 │ 104 │ 105 │ 106 │ 107 │
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ 10  │ 20  │ 30  │ 40  │ 50  │  ?  │  ?  │  ?  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  [0]   [1]   [2]   [3]   [4]
  ↑ TODOS CONTÍGUOS na memória
```

### Fórmula de acesso O(1)

```
endereço[i] = endereço_base + (i × tamanho_do_tipo)

array[3] = 100 + (3 × 1) = 103   ← acesso instantâneo!
```

---

## Arrays Verdadeiros vs JavaScript

Arrays em linguagens de baixo nível têm **memória contígua**. JavaScript "arrays" são objetos disfarçados (hash tables).

### Array verdadeiro em Go

```go
// Array de tamanho fixo — alocado na stack
var diasSemana [7]string = [7]string{
    "Segunda", "Terça", "Quarta", "Quinta",
    "Sexta", "Sábado", "Domingo",
}

// Acesso O(1) — aritmética direta de ponteiro
fmt.Println(diasSemana[2]) // "Quarta"

// Slice — view dinâmica sobre um array
nums := []int{10, 20, 30, 40, 50}
sub := nums[1:3] // [20, 30] — sem cópia de memória!
```

### Slice em Go: o array dinâmico

```go
// Estrutura interna de um slice
// type slice struct {
//     array unsafe.Pointer  // ponteiro para o array subjacente
//     len   int             // comprimento atual
//     cap   int             // capacidade total
// }

s := make([]int, 0, 4) // len=0, cap=4

// append: O(1) amortizado
s = append(s, 1) // len=1, cap=4
s = append(s, 2) // len=2, cap=4
s = append(s, 3) // len=3, cap=4
s = append(s, 4) // len=4, cap=4
s = append(s, 5) // len=5, cap=8 → REALOCAÇÃO! copia tudo para novo array maior
```

---

## Características Fundamentais

| Propriedade | Array Fixo | Slice (Go) |
|------------|-----------|------------|
| Tamanho | Fixo em compilação | Dinâmico |
| Alocação | Stack (geralmente) | Heap |
| Acesso por índice | O(1) | O(1) |
| Inserção no meio | O(n) | O(n) |
| Cache-friendly | Sim | Sim |

---

## Por que arrays são cache-friendly

O processador lê memória em **linhas de cache** (geralmente 64 bytes). Como elementos de um array são contíguos, ao acessar `array[0]`, o hardware pré-carrega `array[1]`, `array[2]`... na cache automaticamente (**spatial locality**).

```
Linha de cache (64 bytes):
[array[0]][array[1]][array[2]]...[array[15]]  ← lidos de uma vez!

Linked List (elementos espalhados):
[nó em 0x1000] → cache miss → [nó em 0x8500] → cache miss → ...
```

---

## Análise de Complexidade

| Operação | Complexidade | Motivo |
|----------|-------------|--------|
| Acesso por índice | O(1) | Aritmética de ponteiro |
| Busca | O(n) | Percorrer sequencialmente |
| Inserção no final | O(1) amortizado | Sem mover elementos |
| Inserção no meio | O(n) | Shift dos elementos seguintes |
| Remoção do meio | O(n) | Shift dos elementos seguintes |

---

## Quando usar vs Linked List

**Use array/slice quando:**
- Acesso frequente por índice
- Iteração sequencial (cache locality)
- Tamanho razoavelmente previsível

**Use [[Linked Lists]] quando:**
- Inserções/remoções frequentes no início ou meio
- Tamanho muito variável sem limite superior
- Não precisa de acesso aleatório

---

## Conexão com Sistemas Operacionais

- **Memória virtual**: a tabela de páginas pode ser vista como um array de entradas — cada entrada mapeia um endereço virtual para físico. Acesso O(1) com TLB (Translation Lookaside Buffer como cache)
- **Buffer de I/O**: o kernel usa arrays circulares (ring buffers) para bufferizar leituras/escritas de disco e rede — evita cópias desnecessárias
- **Stack do processo**: a call stack de cada processo é literalmente um array com ponteiro de topo (SP — stack pointer). `push`/`pop` são O(1)
- **Cache de CPU**: o hardware implementa cache em linhas contíguas — a contiguidade dos arrays é a razão pela qual são mais cache-friendly que [[Linked Lists]]
- **`mmap()`**: mapeia arquivos ou memória em arrays de bytes no espaço do processo — acesso por offset é O(1), igual a um array

## Conexão com Go

- **`slice`** é a abstração de array do Go: header com ponteiro, len e cap. Operação `s[i]` compila para aritmética de ponteiro pura
- **`append()`** amortiza realocações: quando cap é atingido, dobra a capacidade (≤1024 elementos) ou cresce 25% (>1024). O custo O(n) da cópia é amortizado em O(1) por operação
- **`copy(dst, src)`** usa `memmove` internamente — cópia otimizada pelo OS/hardware
- **Arrays em Go são valores**: `var a [5]int; b := a` copia todos os 5 elementos. Slices são referências — `b := a[:]` compartilha o backing array
- **`unsafe.Slice`** permite criar slice a partir de ponteiro arbitrário — útil para interop com C e para entender como slices funcionam internamente
