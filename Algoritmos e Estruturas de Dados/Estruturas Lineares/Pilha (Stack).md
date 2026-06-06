---
tags:
  - algoritmos
  - estruturas-de-dados
  - stack
  - lifo
---

# Pilha (Stack)

Uma **Stack** é uma estrutura de dados **LIFO** (Last In, First Out): o último elemento inserido é o primeiro a sair. Todas operações acontecem no **topo**.

**Analogia**: pilha de pratos — só dá para adicionar ou remover do topo.

```
    ↑ PUSH (inserir)
    ↓ POP (remover)
┌─────────────┐
│      7      │ ← TOP
├─────────────┤
│      1      │
├─────────────┤
│      8      │
├─────────────┤
│      5      │ ← BOTTOM
└─────────────┘
```

---

## Operações Fundamentais

| Operação | Complexidade | Descrição |
|----------|-------------|-----------|
| `push(x)` | O(1) | Adiciona no topo |
| `pop()` | O(1) | Remove e retorna o topo |
| `peek()` | O(1) | Lê o topo sem remover |
| `isEmpty()` | O(1) | Verifica se vazia |

---

## Implementação em Go

```go
// Stack genérica usando slice como backing array
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item) // O(1) amortizado
}

func (s *Stack[T]) Pop() (T, bool) {
    if s.IsEmpty() {
        var zero T
        return zero, false
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1] // remove último elemento
    return top, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if s.IsEmpty() {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

// Exemplo: verificador de parênteses balanceados
func paresBaanceados(expr string) bool {
    stack := &Stack[rune]{}
    pares := map[rune]rune{')': '(', ']': '[', '}': '{'}

    for _, ch := range expr {
        switch ch {
        case '(', '[', '{':
            stack.Push(ch)
        case ')', ']', '}':
            top, ok := stack.Pop()
            if !ok || top != pares[ch] {
                return false
            }
        }
    }
    return stack.IsEmpty()
}

fmt.Println(paresBaanceados("({[]})")) // true
fmt.Println(paresBaanceados("({[})"))  // false
```

---

## Implementações Possíveis

### 1. Array (backing de slice)
```
[5][3][8][1]
              ↑ top = índice len-1
Vantagem: cache-friendly, sem overhead de ponteiros
```

### 2. Linked List
```
TOP → [1|→] → [8|→] → [3|→] → [5|NULL]
push: adiciona no início → O(1)
pop:  remove do início   → O(1)
Vantagem: sem limite de tamanho, sem realocação
```

---

## Aplicações Clássicas

### 1. Avaliação de expressões
```
Expressão: "3 + 4 * 2"
Stack de operadores gerencia precedência
```

### 2. Navegação em browser
```
Histórico de páginas:
push("pagina-a") → push("pagina-b") → pop() volta para "pagina-a"
```

### 3. Undo/Redo
```
Ações: [digitar "A", digitar "B", digitar "C"]
Undo: pop() → desfaz "C"
Undo: pop() → desfaz "B"
```

### 4. DFS iterativo em [[Grafos]]
```go
func dfs(grafo map[int][]int, inicio int) []int {
    visitados := make(map[int]bool)
    stack := &Stack[int]{}
    resultado := []int{}

    stack.Push(inicio)
    for !stack.IsEmpty() {
        no, _ := stack.Pop()
        if visitados[no] {
            continue
        }
        visitados[no] = true
        resultado = append(resultado, no)
        for _, vizinho := range grafo[no] {
            if !visitados[vizinho] {
                stack.Push(vizinho)
            }
        }
    }
    return resultado
}
```

---

## Comparação: Stack vs Queue

| Aspecto | Stack (LIFO) | [[Queue (Fila)]] (FIFO) |
|---------|-------------|---------|
| Princípio | Último entra, primeiro sai | Primeiro entra, primeiro sai |
| Inserção | Topo | Final (rear) |
| Remoção | Topo | Início (front) |
| Uso típico | Recursão, undo/redo, DFS | Scheduling, BFS, I/O |

---

## Conexão com Sistemas Operacionais

- **Call stack do processo**: cada chamada de função empurra um **stack frame** (variáveis locais, endereço de retorno, registradores salvos) na stack do processo. `push`/`pop` de frames são O(1) — movem apenas o stack pointer (SP)
- **Interrupções e context switch**: ao ocorrer uma interrupção, o hardware faz `push` do estado da CPU na kernel stack. Ao retornar, `pop` restaura o estado — isso é gerenciado pelo SO
- **Kernel stack**: cada processo no Linux tem uma kernel stack separada (geralmente 8KB ou 16KB) usada durante syscalls e interrupções. Overflow da kernel stack = kernel panic
- **Stack de chamadas de sistema**: quando seu programa faz `write(fd, buf, n)`, o kernel empilha frames de funções internas na kernel stack até chegar ao driver de dispositivo
- **Linguagens de máquina**: instruções `PUSH`/`POP` do x86 operam na stack de hardware — são a base de toda chamada de função

## Conexão com Go

- **Goroutine stack**: cada goroutine começa com uma stack pequena (~2-4KB) que **cresce dinamicamente** (segmented stacks no início, contiguous stacks hoje). Diferente de threads OS que têm stack fixa de 1-8MB
- **Stack growth**: Go detecta quando a goroutine está prestes a ultrapassar o tamanho atual e realoca uma stack maior, copiando o conteúdo — transparente para o programador
- **Escape analysis**: o compilador decide se variáveis vão para stack (O(1), sem GC) ou heap (GC). `go build -gcflags="-m"` mostra as decisões
- **Defer**: implementado como uma linked list de funções adiadas associada ao frame atual — executada em ordem LIFO quando a função retorna
- **`runtime.Stack()`**: captura o stack trace de todas goroutines — útil para debugging de deadlocks
