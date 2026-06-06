---
tags:
  - algoritmos
  - estruturas-de-dados
  - queue
  - fifo
---

# Queue (Fila)

Uma **Queue** é uma estrutura de dados **FIFO** (First In, First Out): o primeiro elemento inserido é o primeiro a sair. Inserção no final (rear), remoção no início (front).

**Analogia**: fila do banco — quem chega primeiro é atendido primeiro.

```
ENTRADA (enqueue) →
┌───┬───┬───┬───┬───┐
│ 5 │ 3 │ 8 │ 1 │ 7 │
└───┴───┴───┴───┴───┘
FRONT                REAR
← SAÍDA (dequeue)
```

---

## Operações Fundamentais

| Operação | Complexidade | Descrição |
|----------|-------------|-----------|
| `enqueue(x)` | O(1) | Adiciona no final |
| `dequeue()` | O(1) | Remove e retorna o início |
| `front()` | O(1) | Lê o início sem remover |
| `isEmpty()` | O(1) | Verifica se vazia |

---

## Implementação em Go

```go
// Queue usando slice — simples, mas dequeue é O(n) sem anel circular
// Para produção, use container/list ou array circular
type Queue[T any] struct {
    items []T
}

func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item) // O(1) amortizado
}

func (q *Queue[T]) Dequeue() (T, bool) {
    if q.IsEmpty() {
        var zero T
        return zero, false
    }
    item := q.items[0]
    q.items = q.items[1:] // O(n) — shift — use array circular para O(1)
    return item, true
}

func (q *Queue[T]) Front() (T, bool) {
    if q.IsEmpty() {
        var zero T
        return zero, false
    }
    return q.items[0], true
}

func (q *Queue[T]) IsEmpty() bool {
    return len(q.items) == 0
}

// Queue circular O(1) para enqueue e dequeue
type CircularQueue[T any] struct {
    items    []T
    front    int
    rear     int
    size     int
    capacity int
}

func NewCircularQueue[T any](cap int) *CircularQueue[T] {
    return &CircularQueue[T]{
        items:    make([]T, cap),
        capacity: cap,
    }
}

func (q *CircularQueue[T]) Enqueue(item T) bool {
    if q.size == q.capacity {
        return false // cheia
    }
    q.items[q.rear] = item
    q.rear = (q.rear + 1) % q.capacity // wraparound!
    q.size++
    return true
}

func (q *CircularQueue[T]) Dequeue() (T, bool) {
    if q.size == 0 {
        var zero T
        return zero, false
    }
    item := q.items[q.front]
    q.front = (q.front + 1) % q.capacity // wraparound!
    q.size--
    return item, true
}
```

---

## Implementações Possíveis

### 1. Array Circular
```
┌───┬───┬───┬───┬───┐
│ 8 │ 1 │   │   │ 7 │
└───┴───┴───┴───┴───┘
  2   3           1   ← índices reais
      ↑           ↑
    front        rear

próximo índice = (índice + 1) % capacidade
Reutiliza espaços sem mover dados → O(1) para tudo
```

### 2. Linked List
```
FRONT → [5|→] → [3|→] → [8|→] → [1|NULL] ← REAR
        ↑                              ↑
     dequeue                       enqueue
```

### 3. Duas Stacks
```
Stack de entrada:  [1][8][3] ← push aqui
Stack de saída:    []        ← pop daqui

Dequeue: se saída vazia, move tudo da entrada para saída
Amortizado O(1) — cada elemento é movido no máximo uma vez
```

---

## BFS com Queue

Queue é a estrutura central do BFS em [[Grafos]] e [[Árvores Binárias]]:

```go
func bfs(grafo map[int][]int, inicio int) []int {
    visitados := make(map[int]bool)
    queue := &Queue[int]{}
    resultado := []int{}

    queue.Enqueue(inicio)
    visitados[inicio] = true

    for !queue.IsEmpty() {
        no, _ := queue.Dequeue()
        resultado = append(resultado, no)

        for _, vizinho := range grafo[no] {
            if !visitados[vizinho] {
                visitados[vizinho] = true
                queue.Enqueue(vizinho) // processa nível por nível
            }
        }
    }
    return resultado
}
```

---

## Variações Especializadas

### Priority Queue
Elementos têm prioridades — não segue estritamente FIFO. Implementada com heap binário.

```go
import "container/heap"

// Go tem heap.Interface para priority queues
// O elemento de maior/menor prioridade sai primeiro → O(log n)
```

### Deque (Double-ended Queue)
Inserção e remoção em ambas as extremidades.

```
addFirst() ←→ [5][3][8][1] ←→ addLast()
removeFirst()              removeLast()
```

---

## Conexão com Sistemas Operacionais

- **Escalonamento de processos (CPU Scheduler)**: o SO mantém **run queues** — uma fila de processos prontos para executar em cada CPU. O CFS do Linux usa uma fila com prioridade (red-black tree), mas conceitualmente é uma queue
- **Buffer de I/O de disco**: requisições de leitura/escrita entram em uma queue — o algoritmo de escalonamento de disco (CFQ, BFQ, deadline) decide a ordem de execução para minimizar seek time
- **Filas de interrupções**: interrupções de hardware são enfileiradas para processamento — o kernel processa em ordem, garantindo fairness
- **Spooler de impressão**: trabalhos de impressão entram em uma fila FIFO — o daemon processa um por vez
- **Pipe do Unix**: `cmd1 | cmd2` é implementado como uma queue circular no kernel — `cmd1` escreve na fila, `cmd2` lê. Bloqueia quando cheia ou vazia
- **Socket receive buffer**: pacotes TCP chegam e entram em uma queue no kernel até o processo fazer `read()` — tamanho configurável via `SO_RCVBUF`

## Conexão com Go

- **Channels em Go são queues**: `ch := make(chan int, 5)` cria uma queue circular de capacidade 5 no runtime. `ch <- x` é enqueue; `<-ch` é dequeue. Bloqueiam quando cheios/vazios
- **Scheduler do Go (goroutines)**: cada P (processor) tem uma **run queue local** de goroutines prontas para executar. Quando vazia, tenta roubar da global queue ou de outros Ps (work stealing)
- **`container/ring`**: implementação de buffer circular na stdlib — ideal para implementar queues de tamanho fixo eficientes
- **`container/heap`**: implementa priority queue — use para implementar filas de prioridade em Go. Ex: Dijkstra, Prim, event-driven simulações
- **Select com timeout**: `select { case v := <-ch: ... case <-time.After(1s): ... }` permite dequeue com timeout — idioma comum para queues com deadline
