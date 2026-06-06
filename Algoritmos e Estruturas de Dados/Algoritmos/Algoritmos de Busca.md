---
tags:
  - algoritmos
  - busca
  - dfs
  - bfs
  - busca-binaria
---

# Algoritmos de Busca

Algoritmos de busca localizam elementos em estruturas de dados. A escolha depende do tipo de estrutura e se os dados estão ordenados.

Veja [[Big O Notation]] para entender as complexidades, e [[Grafos]] para DFS/BFS em grafos completos.

---

## 1. Busca Linear — O(n)

Percorre sequencialmente até encontrar o elemento. Funciona em qualquer sequência (ordenada ou não).

```go
func buscaLinear(arr []int, alvo int) int {
    for i, v := range arr {
        if v == alvo {
            return i
        }
    }
    return -1
}

fmt.Println(buscaLinear([]int{5, 2, 8, 1, 9}, 8)) // 2
fmt.Println(buscaLinear([]int{5, 2, 8, 1, 9}, 7)) // -1
```

**Quando usar**: dados não ordenados, estruturas pequenas, busca única.
**Evitar quando**: dados ordenados (use busca binária) ou buscas frequentes (use [[Hashmap]]).

---

## 2. Busca Binária — O(log n)

Divide o array ordenado repetidamente ao meio. A cada passo, **descarta metade** dos elementos.

```go
func buscaBinaria(arr []int, alvo int) int {
    inicio, fim := 0, len(arr)-1

    for inicio <= fim {
        meio := inicio + (fim-inicio)/2 // evita overflow vs (inicio+fim)/2

        if arr[meio] == alvo {
            return meio
        } else if arr[meio] < alvo {
            inicio = meio + 1 // alvo na metade direita
        } else {
            fim = meio - 1 // alvo na metade esquerda
        }
    }
    return -1
}

arr := []int{1, 3, 5, 7, 9, 11, 13}
fmt.Println(buscaBinaria(arr, 7))  // 3
fmt.Println(buscaBinaria(arr, 4))  // -1

// Versão com sort.Search (stdlib Go)
import "sort"
idx := sort.SearchInts(arr, 7) // retorna posição de inserção
if idx < len(arr) && arr[idx] == 7 {
    fmt.Println("encontrado em", idx)
}
```

**Pré-requisito**: array **ordenado**.
**Performance**:
- 1.000 elementos → no máximo 10 comparações
- 1.000.000 elementos → no máximo 20 comparações
- 1.000.000.000 elementos → no máximo 30 comparações

---

## 3. DFS (Depth-First Search) — O(V + E)

Explora o máximo possível em profundidade antes de retroceder. Usa [[Pilha (Stack)]] (implícita na recursão).

```go
// DFS em árvore binária — ver [[Árvores Binárias]]
func dfsArvore(raiz *Node, alvo int) bool {
    if raiz == nil {
        return false
    }
    if raiz.value == alvo {
        return true
    }
    return dfsArvore(raiz.left, alvo) || dfsArvore(raiz.right, alvo)
}

// DFS em grafo — ver [[Grafos]]
func dfsGrafo(grafo map[int][]int, inicio, alvo int) bool {
    visitados := make(map[int]bool)
    return dfsHelper(grafo, inicio, alvo, visitados)
}

func dfsHelper(grafo map[int][]int, atual, alvo int, visitados map[int]bool) bool {
    if atual == alvo {
        return true
    }
    visitados[atual] = true
    for _, vizinho := range grafo[atual] {
        if !visitados[vizinho] {
            if dfsHelper(grafo, vizinho, alvo, visitados) {
                return true
            }
        }
    }
    return false
}
```

**Usos**: detectar ciclos, toposort, encontrar componentes conectados, resolver labirintos, backtracking.

---

## 4. BFS (Breadth-First Search) — O(V + E)

Explora nível por nível usando [[Queue (Fila)]]. **Garante menor caminho** em grafos não-ponderados.

```go
// BFS — menor número de saltos entre dois nós
func bfsDistancia(grafo map[int][]int, origem, destino int) int {
    if origem == destino {
        return 0
    }

    visitados := map[int]bool{origem: true}
    fila := []int{origem}
    distancia := 0

    for len(fila) > 0 {
        distancia++
        proxNivel := []int{}

        for _, vertice := range fila {
            for _, vizinho := range grafo[vertice] {
                if vizinho == destino {
                    return distancia
                }
                if !visitados[vizinho] {
                    visitados[vizinho] = true
                    proxNivel = append(proxNivel, vizinho)
                }
            }
        }
        fila = proxNivel
    }
    return -1 // não encontrado
}
```

**Usos**: menor caminho (não-ponderado), crawlers web, redes sociais (amigos a N graus), broadcasting em redes.

---

## 5. Dijkstra — O((V + E) log V)

BFS generalizado para grafos **ponderados** com pesos positivos. Usa priority queue.

```go
type Edge struct { destino, peso int }

func dijkstra(grafo map[int][]Edge, origem int) map[int]int {
    dist := make(map[int]int)
    // omitido por brevidade — veja [[Grafos]] para implementação completa
    return dist
}
```

---

## Comparação Geral

| Algoritmo | Estrutura | Complexidade | Menor Caminho? |
|-----------|-----------|-------------|----------------|
| Busca Linear | Array/lista | O(n) | N/A |
| Busca Binária | Array ordenado | O(log n) | N/A |
| DFS | Grafo/Árvore | O(V + E) | Não garante |
| BFS | Grafo/Árvore | O(V + E) | Sim (não-ponderado) |
| Dijkstra | Grafo ponderado | O((V+E) log V) | Sim (ponderado) |

---

## Conexão com Sistemas Operacionais

- **Busca binária em tabelas do kernel**: o kernel Linux usa busca binária para localizar handlers de syscall, entradas em tabelas de interrupção e outros arrays ordenados
- **`find` e localização de arquivos**: o sistema de arquivos usa travessia DFS para percorrer diretórios. `find /` faz um DFS do sistema de arquivos inteiro
- **Roteamento de pacotes**: roteadores usam variações de BFS/Dijkstra para construir tabelas de roteamento. OSPF (Open Shortest Path First) é baseado em Dijkstra
- **Detecção de deadlock**: DFS em grafos de alocação de recursos detecta ciclos (deadlocks). O kernel monitora isso para recursos como mutexes e semáforos
- **`locate` / `updatedb`**: banco de dados de arquivos usa busca binária em índice ordenado — O(log n) para encontrar arquivo vs O(n) do `find`

## Conexão com Go

- **`sort.Search()`**: busca binária genérica. Recebe `n` (tamanho) e função `f(i) bool`. Encontra menor `i` onde `f(i)` é true — O(log n)
- **`sort.SearchInts()`, `sort.SearchStrings()`**: wrappers de busca binária para tipos comuns
- **`bytes.Index()`, `strings.Index()`**: busca de substring — usa algoritmo otimizado (similar a Boyer-Moore) O(n+m)
- **`sync/atomic` + arrays**: busca binária thread-safe em arrays imutáveis grandes sem lock — padrão comum para lookup tables em Go
- **Testes com `testing.B`**: use benchmark para medir busca linear vs binária nos seus dados reais — O(n) pode ser mais rápido que O(log n) para n pequeno por conta da cache locality
