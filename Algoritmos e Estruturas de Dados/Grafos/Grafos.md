---
tags:
  - algoritmos
  - estruturas-de-dados
  - grafos
  - bfs
  - dfs
---

# Grafos

Um **Grafo** representa relacionamentos entre objetos através de **vértices** (nós) conectados por **arestas** (edges). É a estrutura mais expressiva para modelar conexões arbitrárias.

**Analogia**: rede social — pessoas (vértices) conectadas por amizades (arestas).

```
REPRESENTAÇÃO VISUAL
    A ――――― B
    │ \     │
    │   \   │
    │     \ │
    D ――――― C

G = (V, E)
V = {A, B, C, D}
E = {(A,B), (A,C), (A,D), (B,C), (C,D)}
```

---

## Tipos de Grafos

### Não-direcionado vs Direcionado

```
NÃO-DIRECIONADO        DIRECIONADO (Dígrafo)
A ―――― B               A ――→ B
│      │               ↑     ↓
│      │               │     │
D ―――― C               D ←―― C
A―B significa A↔B      A→B não implica B→A
```

### Ponderado vs Não-ponderado

```
NÃO-PONDERADO          PONDERADO
A ―――― B               A ――5―― B
│      │               │3      │2
D ―――― C               D ――7―― C
```

### Grafos especiais
- **DAG** (Directed Acyclic Graph): grafo direcionado sem ciclos — usado para dependências
- **Árvore**: grafo conectado sem ciclos — caso especial de grafo
- **Grafo bipartido**: vértices divididos em dois conjuntos sem arestas internas

---

## Representações em Memória

### Lista de Adjacência — O(V + E)

Cada vértice mantém lista de vizinhos. **Melhor para grafos esparsos**.

```go
// Lista de adjacência em Go
type Grafo struct {
    adjacencia map[int][]int
}

func NovoGrafo() *Grafo {
    return &Grafo{adjacencia: make(map[int][]int)}
}

func (g *Grafo) AddAresta(v1, v2 int) {
    g.adjacencia[v1] = append(g.adjacencia[v1], v2)
    g.adjacencia[v2] = append(g.adjacencia[v2], v1) // não-direcionado
}
```

### Matriz de Adjacência — O(V²)

Matrix VxV: `adj[i][j] = 1` se há aresta. **Melhor para grafos densos ou queries de adjacência**.

```
    A B C D
A [ 0 1 1 1 ]
B [ 1 0 1 0 ]
C [ 1 1 0 1 ]
D [ 1 0 1 0 ]
```

---

## Algoritmos de Travessia

### BFS (Breadth-First Search) — O(V + E)

Explora nível por nível usando [[Queue (Fila)]]. Garante o menor caminho em grafos não-ponderados.

```go
func (g *Grafo) BFS(inicio int) []int {
    visitados := make(map[int]bool)
    fila := []int{inicio}
    resultado := []int{}

    visitados[inicio] = true

    for len(fila) > 0 {
        vertice := fila[0]
        fila = fila[1:]
        resultado = append(resultado, vertice)

        for _, vizinho := range g.adjacencia[vertice] {
            if !visitados[vizinho] {
                visitados[vizinho] = true
                fila = append(fila, vizinho)
            }
        }
    }
    return resultado
}
```

### DFS (Depth-First Search) — O(V + E)

Explora profundamente antes de retroceder. Usa [[Pilha (Stack)]] (ou recursão).

```go
func (g *Grafo) DFS(inicio int) []int {
    visitados := make(map[int]bool)
    resultado := []int{}
    g.dfsHelper(inicio, visitados, &resultado)
    return resultado
}

func (g *Grafo) dfsHelper(vertice int, visitados map[int]bool, resultado *[]int) {
    visitados[vertice] = true
    *resultado = append(*resultado, vertice)
    for _, vizinho := range g.adjacencia[vertice] {
        if !visitados[vizinho] {
            g.dfsHelper(vizinho, visitados, resultado)
        }
    }
}
```

### DFS vs BFS

| Aspecto | DFS | BFS |
|---------|-----|-----|
| Estrutura interna | Stack (recursão) | Queue |
| Memória | O(h) — altura do grafo | O(w) — largura |
| Menor caminho | Não garante | Garante (não-ponderado) |
| Uso típico | Ciclos, componentes, toposort | Menor caminho, nível |

---

## Algoritmos Clássicos

### Dijkstra — menor caminho em grafos ponderados

```go
import "container/heap"

func Dijkstra(grafo map[int][]Edge, origem int) map[int]int {
    dist := make(map[int]int)
    // inicializa com infinito
    pq := &PriorityQueue{}
    heap.Push(pq, &Item{vertex: origem, dist: 0})
    dist[origem] = 0

    for pq.Len() > 0 {
        u := heap.Pop(pq).(*Item)
        for _, aresta := range grafo[u.vertex] {
            newDist := dist[u.vertex] + aresta.peso
            if d, ok := dist[aresta.destino]; !ok || newDist < d {
                dist[aresta.destino] = newDist
                heap.Push(pq, &Item{vertex: aresta.destino, dist: newDist})
            }
        }
    }
    return dist
}
// Complexidade: O((V + E) log V) com priority queue
```

### Ordenação Topológica — para DAGs

```go
// Algoritmo de Kahn — usa in-degree
func TopoSort(grafo map[int][]int, vertices int) []int {
    inDegree := make(map[int]int)
    for _, vizinhos := range grafo {
        for _, v := range vizinhos {
            inDegree[v]++
        }
    }

    queue := []int{}
    for v := 0; v < vertices; v++ {
        if inDegree[v] == 0 {
            queue = append(queue, v)
        }
    }

    resultado := []int{}
    for len(queue) > 0 {
        v := queue[0]
        queue = queue[1:]
        resultado = append(resultado, v)
        for _, u := range grafo[v] {
            inDegree[u]--
            if inDegree[u] == 0 {
                queue = append(queue, u)
            }
        }
    }
    return resultado
}
// Uso: build systems (Make, Gradle), gerenciadores de dependências
```

---

## Casos de Uso

| Domínio | Aplicação | Algoritmo |
|---------|-----------|-----------|
| **Redes** | Menor caminho entre servidores | Dijkstra, BFS |
| **SO** | Detecção de deadlock | DFS (ciclo em grafo de recursos) |
| **Build systems** | Ordem de compilação | Toposort (DFS) |
| **GPS** | Menor rota entre cidades | Dijkstra, A* |
| **Redes sociais** | "Pessoas que você pode conhecer" | BFS (k níveis) |
| **Web crawling** | Indexar páginas | BFS |

---

## Conexão com Sistemas Operacionais

- **Detecção de deadlock**: o kernel mantém um **grafo de recursos** — processos são vértices, arestas representam "espera por recurso". Um ciclo nesse grafo = deadlock. DFS detecta ciclos em O(V+E)
- **Scheduling com dependências**: `systemd` resolve a ordem de inicialização de serviços usando toposort em um DAG de dependências — `Requires=`, `After=` criam arestas
- **Roteamento de pacotes de rede**: roteadores executam algoritmos de grafo (OSPF usa Dijkstra, BGP usa variantes) para calcular tabelas de roteamento
- **OverlayFS / union mounts** (Docker): camadas de filesystem formam um grafo DAG — leitura percorre as camadas de cima para baixo (DFS implícito)
- **Call graph do kernel**: análise de segurança do kernel Linux usa grafos de chamadas para detectar caminhos que atingem código privilegiado

## Conexão com Go

- **Módulos Go (`go.mod`)**: dependências formam um DAG — `go mod tidy` faz toposort para resolver versões compatíveis (Minimal Version Selection)
- **`golang.org/x/tools/go/callgraph`**: análise estática de call graphs em programas Go — útil para encontrar caminhos de código não utilizados
- **Goroutine dependency graph**: ferramentas como `go tool trace` visualizam o grafo de dependências entre goroutines e channels — útil para detectar deadlocks
- **Kubernetes**: o estado do cluster é um grafo de objetos (Pod → Service → Ingress). O controller-manager usa toposort implícito para orquestrar reconciliações
- **`gonum/graph`**: biblioteca Go para algoritmos de grafo — BFS, DFS, Dijkstra, MST (Kruskal, Prim), toposort, todos prontos para uso
