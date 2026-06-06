---
tags:
  - algoritmos
  - ordenacao
  - sorting
  - divide-and-conquer
---

# Algoritmos de Ordenação

Algoritmos de ordenação organizam elementos em uma ordem específica. A escolha depende do tamanho dos dados, se já estão parcialmente ordenados e se precisam de estabilidade.

Veja [[Big O Notation]] para entender as complexidades comparadas.

---

## 1. Bubble Sort — O(n²)

Compara pares adjacentes e troca se fora de ordem. Repete até não haver mais trocas.

```go
func bubbleSort(arr []int) []int {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        trocou := false
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
                trocou = true
            }
        }
        if !trocou { // otimização: parar se já ordenado
            break
        }
    }
    return arr
}
```

**Complexidade**: O(n²) pior/médio, O(n) melhor (já ordenado).
**Uso real**: praticamente nenhum — valor didático apenas.

---

## 2. Selection Sort — O(n²)

Encontra o menor elemento e o coloca na posição correta, repetindo para o restante.

```go
func selectionSort(arr []int) []int {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if arr[j] < arr[minIdx] {
                minIdx = j
            }
        }
        arr[i], arr[minIdx] = arr[minIdx], arr[i]
    }
    return arr
}
```

**Complexidade**: O(n²) em todos os casos.
**Vantagem**: mínimo de swaps (O(n)) — útil quando swap é custoso (ex: escrita em flash).

---

## 3. Insertion Sort — O(n²) / O(n) best case

Pega cada elemento e o insere na posição correta na parte já ordenada.

```go
func insertionSort(arr []int) []int {
    for i := 1; i < len(arr); i++ {
        chave := arr[i]
        j := i - 1
        for j >= 0 && arr[j] > chave {
            arr[j+1] = arr[j]
            j--
        }
        arr[j+1] = chave
    }
    return arr
}
```

**Complexidade**: O(n²) pior, O(n) melhor (quase ordenado).
**Uso real**: excelente para arrays pequenos (< 20 elementos). Usado como base do Timsort e pdqsort para partições pequenas.

---

## 4. Merge Sort — O(n log n)

Divide o array ao meio recursivamente, ordena cada metade, depois mescla. **Divide e conquiste**.

```go
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

**Complexidade**: O(n log n) sempre (pior, médio e melhor).
**Espaço**: O(n) auxiliar (precisa de array extra).
**Estável**: sim — elementos iguais mantêm ordem relativa original.
**Uso real**: `sort.Stable()` em Go; ordenação de linked lists; merge em banco de dados.

---

## 5. Quick Sort — O(n log n) médio

Escolhe um pivô, particiona em menores/maiores, ordena recursivamente. **Divide e conquiste**.

```go
func quickSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }

    pivo := arr[len(arr)-1]
    var esq, dir []int

    for _, v := range arr[:len(arr)-1] {
        if v <= pivo {
            esq = append(esq, v)
        } else {
            dir = append(dir, v)
        }
    }

    result := quickSort(esq)
    result = append(result, pivo)
    result = append(result, quickSort(dir)...)
    return result
}

// Versão in-place (mais eficiente em memória)
func quickSortInPlace(arr []int, low, high int) {
    if low < high {
        p := particionar(arr, low, high)
        quickSortInPlace(arr, low, p-1)
        quickSortInPlace(arr, p+1, high)
    }
}

func particionar(arr []int, low, high int) int {
    pivo := arr[high]
    i := low - 1
    for j := low; j < high; j++ {
        if arr[j] <= pivo {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[high] = arr[high], arr[i+1]
    return i + 1
}
```

**Complexidade**: O(n log n) médio, O(n²) pior (pivô ruim com dados ordenados).
**Espaço**: O(log n) na stack de recursão.
**Não estável**: elementos iguais podem trocar de posição.
**Uso real**: `sort.Slice()` do Go usa **pdqsort** — variante que evita O(n²) com detecção de padrões.

---

## 6. Counting Sort — O(n + k)

Conta a frequência de cada elemento, reconstrói o array ordenado. **Não comparativo**.

```go
func countingSort(arr []int, max int) []int {
    contador := make([]int, max+1)
    for _, v := range arr {
        contador[v]++
    }

    resultado := make([]int, 0, len(arr))
    for i, count := range contador {
        for j := 0; j < count; j++ {
            resultado = append(resultado, i)
        }
    }
    return resultado
}

arr := []int{4, 2, 2, 8, 3, 3, 1}
fmt.Println(countingSort(arr, 8)) // [1 2 2 3 3 4 8]
```

**Complexidade**: O(n + k), onde k = valor máximo.
**Limitação**: funciona apenas para inteiros em intervalo conhecido.
**Uso real**: ordenar IDs, idades, scores — qualquer inteiro com range pequeno.

---

## Comparação Geral

| Algoritmo | Pior Caso | Médio | Melhor | Espaço | Estável |
|-----------|-----------|-------|--------|--------|---------|
| Bubble | O(n²) | O(n²) | O(n) | O(1) | Sim |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | Não |
| Insertion | O(n²) | O(n²) | O(n) | O(1) | Sim |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Sim |
| Quick | O(n²) | O(n log n) | O(n log n) | O(log n) | Não |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(k) | Sim |

---

## Qual usar?

- **Dados pequenos (< 20)**: Insertion Sort — simples e cache-friendly
- **Precisa de estabilidade**: Merge Sort
- **Caso geral**: Quick Sort (pdqsort) — melhor performance prática
- **Inteiros em range pequeno**: Counting Sort
- **Linked Lists**: Merge Sort — Quicksort é ruim sem acesso aleatório

---

## Conexão com Sistemas Operacionais

- **Scheduler do kernel**: `sort` é usado internamente pelo kernel para ordenar listas de processos, timers e dispositivos durante boot
- **Merge sort em filesystems**: sistemas de arquivos como ext4 usam merge sort para ordenar entradas de diretório antes de escrever em disco — minimiza I/O buscando contiguidade
- **`qsort()` da libc**: a função C padrão `qsort()` implementa um quicksort híbrido — base de muitos programas do sistema
- **`sort` command do Unix**: ordenação de arquivos grandes usa merge sort externo — divide o arquivo em blocos que cabem na RAM, ordena cada bloco, depois faz merge dos arquivos temporários. Complexidade I/O: O(n log n)
- **Caches e TLBs**: internamente, hardware mantém entradas de cache em estruturas que precisam de ordenação eficiente para flush/replacement policies

## Conexão com Go

- **`sort.Slice()`**: usa **pdqsort** (pattern-defeating quicksort) — combina quicksort, insertion sort e heapsort. O(n log n) garantido no pior caso (diferente do quicksort puro)
- **`sort.Stable()`**: usa merge sort para garantir estabilidade — use quando a ordem de elementos iguais importa (ex: ordenar por data e depois por nome)
- **`slices.Sort()` (Go 1.21+)**: pacote `slices` da stdlib com API mais ergonômica e genérica — `slices.SortFunc(s, func(a, b T) int { ... })`
- **Benchmarking sorts**: use `testing.B` para medir no seu caso específico — quicksort tem melhor cache locality que merge sort (in-place vs O(n) extra)
- **Ordenar structs**: padrão Go para ordenar por múltiplos campos: implementar `sort.Interface` (`Len`, `Less`, `Swap`) ou usar `sort.Slice` com closure
