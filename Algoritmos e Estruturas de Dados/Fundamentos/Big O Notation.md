---
tags:
  - algoritmos
  - big-o
  - complexidade
  - fundamentos
---

# Big O Notation

**Big O Notation** é a linguagem matemática para descrever como o tempo ou espaço de um algoritmo cresce em função do tamanho da entrada `n`. Sempre considera o **pior caso** e ignora constantes.

Veja [[Complexidade de Algoritmos]] para o contexto geral de análise.

---

## Hierarquia de Complexidades

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2^n) < O(n!)
  ↑         ↑       ↑         ↑           ↑        ↑        ↑
Melhor                                               Pior
```

---

## O(1) — Tempo Constante

Não importa o tamanho da entrada: sempre o mesmo número de operações.

```go
// O(1) — acesso direto por índice
func primeiroPar(nums []int) int {
    return nums[1] // sempre 1 operação, independente do tamanho
}

// O(1) — verificar paridade
func isPar(n int) bool {
    return n%2 == 0
}
```

**Exemplos reais**: acesso a array por índice, inserção no topo de uma [[Pilha (Stack)]], lookup em [[Hashmap]].

---

## O(log n) — Tempo Logarítmico

A cada passo, o problema é **dividido pela metade**. Para 1 milhão de elementos, são apenas ~20 operações.

```go
// O(log n) — busca binária
func buscaBinaria(arr []int, alvo int) int {
    inicio, fim := 0, len(arr)-1
    for inicio <= fim {
        meio := (inicio + fim) / 2
        if arr[meio] == alvo {
            return meio
        } else if arr[meio] < alvo {
            inicio = meio + 1
        } else {
            fim = meio - 1
        }
    }
    return -1
}
```

**Exemplos reais**: [[Algoritmos de Busca#Busca Binária|busca binária]], busca em [[B-Tree]], inserção em árvores balanceadas.

---

## O(n) — Tempo Linear

Cresce proporcionalmente ao tamanho da entrada.

```go
// O(n) — somar todos elementos
func somaArray(nums []int) int {
    total := 0
    for _, n := range nums { // percorre cada elemento uma vez
        total += n
    }
    return total
}
```

**Exemplos reais**: busca linear, percorrer [[Linked Lists]], imprimir todos elementos.

---

## O(n log n) — Tempo Linear-Logarítmico

Divide o problema (log n) e processa cada parte (n). É o **ótimo teórico** para algoritmos de comparação.

```go
// O(n log n) — merge sort
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }
    meio := len(arr) / 2
    esq := mergeSort(arr[:meio])
    dir := mergeSort(arr[meio:])
    return merge(esq, dir)
}

func merge(esq, dir []int) []int {
    resultado := make([]int, 0, len(esq)+len(dir))
    i, j := 0, 0
    for i < len(esq) && j < len(dir) {
        if esq[i] <= dir[j] {
            resultado = append(resultado, esq[i])
            i++
        } else {
            resultado = append(resultado, dir[j])
            j++
        }
    }
    resultado = append(resultado, esq[i:]...)
    resultado = append(resultado, dir[j:]...)
    return resultado
}
```

**Exemplos reais**: [[Algoritmos de Ordenação#Merge Sort|Merge Sort]], [[Algoritmos de Ordenação#Quick Sort|Quick Sort]] (médio), `sort.Slice()` em Go.

---

## O(n²) — Tempo Quadrático

Dois loops aninhados. Dobrar a entrada = 4x mais operações.

```go
// O(n²) — bubble sort
func bubbleSort(arr []int) []int {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
    return arr
}
```

**Exemplos reais**: [[Algoritmos de Ordenação#Bubble Sort|Bubble Sort]], [[Algoritmos de Ordenação#Selection Sort|Selection Sort]], busca de pares.

---

## O(2^n) — Tempo Exponencial

Cada elemento dobra o trabalho. Impraticável acima de ~30 elementos.

```go
// O(2^n) — gerar todos subconjuntos
func subconjuntos(nums []int) [][]int {
    resultado := [][]int{{}}
    for _, num := range nums {
        var novos [][]int
        for _, sub := range resultado {
            novo := make([]int, len(sub)+1)
            copy(novo, sub)
            novo[len(sub)] = num
            novos = append(novos, novo)
        }
        resultado = append(resultado, novos...)
    }
    return resultado
}
```

**Exemplos reais**: força bruta para Caixeiro Viajante, geração de combinações.

---

## O(n!) — Tempo Fatorial

Gera todas as permutações. 10 elementos = 3.628.800 operações. 15 elementos = 1,3 trilhão.

**Exemplos reais**: permutações completas, alguns algoritmos de matching.

---

## Resumo Visual

| Complexidade | n=10 | n=100 | n=1000 | Exemplo |
|-------------|------|-------|--------|---------|
| O(1) | 1 | 1 | 1 | Array index |
| O(log n) | 3 | 7 | 10 | Busca binária |
| O(n) | 10 | 100 | 1.000 | Busca linear |
| O(n log n) | 33 | 664 | 9.966 | Merge Sort |
| O(n²) | 100 | 10.000 | 1.000.000 | Bubble Sort |
| O(2^n) | 1.024 | 10^30 | ∞ | Subsets |

---

## Conexão com Sistemas Operacionais

- **Scheduler do Linux (CFS)**: usa **red-black tree** (árvore auto-balanceada) para gerenciar processos prontos para execução → inserção e busca em O(log n)
- **Alocador de memória**: o `malloc` usa **free lists** com busca O(n) no pior caso; allocators modernos (buddy system, slab) garantem O(log n)
- **Sistema de arquivos ext4**: htree para diretórios com muitos arquivos — O(log n) ao invés de O(n) da busca linear em diretórios simples
- **Page cache**: lookup de páginas em memória usa [[Hashmap]] → O(1) para verificar se página está em cache

## Conexão com Go

- **`sort.Slice()`**: implementa pdqsort (introsort), garantindo O(n log n) no pior caso — evita o O(n²) do quicksort puro
- **`map`**: amortizado O(1) para get/set — implementa [[Hashmap]] com buckets e rehashing
- **`append()`**: amortizado O(1) — quando o backing array está cheio, realoca com crescimento exponencial, distribuindo o custo O(n) da cópia
- **Binary search**: `sort.Search()` usa busca binária O(log n) — recebe uma função e encontra o menor índice onde ela é verdadeira
- **`sync.Map`**: otimizado para leituras concorrentes; internamente separa reads (O(1) via atomic) de writes (O(1) com lock)
