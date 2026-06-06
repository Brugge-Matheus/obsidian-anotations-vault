---
tags:
  - algoritmos
  - estruturas-de-dados
  - linked-list
  - ponteiros
---

# Linked Lists

Uma **Linked List** é uma estrutura linear onde cada elemento (nó) armazena um dado e um ponteiro para o próximo nó. Diferente de [[Arrays]], os nós podem estar em qualquer posição na memória — o que torna inserção O(1) mas acesso O(n).

---

## Estrutura na memória

```
LINKED LIST
HEAD → [42|•] → [17|•] → [89|•] → [33|NULL]

DISTRIBUIÇÃO FÍSICA (endereços reais)
0x1000: [42 | 0x3500]  ← nó 1: dado + ponteiro pro próximo
0x3500: [17 | 0x7200]  ← nó 2
0x7200: [89 | 0x1800]  ← nó 3
0x1800: [33 | NULL  ]  ← nó 4: fim da lista

↑ Nós ESPALHADOS pela memória — sem contiguidade!
```

Contraste com [[Arrays]]: lá os elementos são contíguos, aqui estão em endereços arbitrários ligados por ponteiros.

---

## Tipos

### Singly Linked (Simples)
```
[A|→] → [B|→] → [C|→] → [D|NULL]
Navegação apenas para frente.
```

### Doubly Linked (Dupla)
```
NULL ← [←|A|→] ⇄ [←|B|→] ⇄ [←|C|→] → NULL
Navegação bidirecional. Permite remoção O(1) dado o nó.
```

### Circular
```
[A|→] → [B|→] → [C|→] → [D|→]
  ↑                           │
  └───────────────────────────┘
Último aponta para o primeiro. Usado em round-robin.
```

---

## Operações

### Inserção no início — O(1)
```
ANTES: HEAD → [3|→] → [8|→] → [1|NULL]

1. novoNó = [5|→]
2. novoNó.next = HEAD
3. HEAD = novoNó

DEPOIS: HEAD → [5|→] → [3|→] → [8|→] → [1|NULL]
```

### Busca — O(n)
```go
// Percorre nó a nó — sem atalho possível
func (ll *LinkedList) Find(data int) *Node {
    current := ll.head
    for current != nil {
        if current.data == data {
            return current
        }
        current = current.next
    }
    return nil
}
```

### Remoção — O(n) para encontrar + O(1) para remover
```go
// Precisa encontrar o nó anterior: O(n)
// A remoção em si é O(1): apenas ajusta ponteiros
func (ll *LinkedList) Remove(data int) {
    if ll.head == nil {
        return
    }
    if ll.head.data == data {
        ll.head = ll.head.next // O(1)
        return
    }
    current := ll.head
    for current.next != nil {       // O(n) para encontrar
        if current.next.data == data {
            current.next = current.next.next // O(1) para remover
            return
        }
        current = current.next
    }
}
```

---

## Implementação em Go

```go
type Node struct {
    data int
    next *Node
}

type LinkedList struct {
    head *Node
    size int
}

func (ll *LinkedList) AddFront(data int) {
    newNode := &Node{data: data, next: ll.head}
    ll.head = newNode
    ll.size++
}

func (ll *LinkedList) AddBack(data int) {
    newNode := &Node{data: data}
    if ll.head == nil {
        ll.head = newNode
        ll.size++
        return
    }
    current := ll.head
    for current.next != nil { // O(n) — precisa achar o final
        current = current.next
    }
    current.next = newNode
    ll.size++
}

func (ll *LinkedList) ToSlice() []int {
    result := make([]int, 0, ll.size)
    current := ll.head
    for current != nil {
        result = append(result, current.data)
        current = current.next
    }
    return result
}

// Uso
ll := &LinkedList{}
ll.AddFront(3)
ll.AddFront(1)
ll.AddFront(5)
fmt.Println(ll.ToSlice()) // [5, 1, 3]
```

---

## Análise de Performance

| Operação | Complexidade | Motivo |
|----------|-------------|--------|
| Acesso por índice | O(n) | Percorrer n nós |
| Busca por valor | O(n) | Varredura sequencial |
| Inserção no início | O(1) | Apenas atualiza ponteiros |
| Inserção no final | O(n) | Precisa encontrar o último |
| Remoção do início | O(1) | Atualiza HEAD |
| Remoção no meio | O(n) | Encontrar + ajustar ponteiros |

**Overhead de memória:**
- Array: apenas dados
- Linked List: dados + ponteiro por nó (~8 bytes extra em 64-bit)

---

## Comparação com Arrays

```
Acesso ao 1000º elemento:
Array:       endereço = base + (999 × sizeof) → O(1)
Linked List: 999 saltos de ponteiro          → O(n)

1000 inserções no início:
Array:       realocar + copiar dados → O(n) cada vez
Linked List: atualizar ponteiros     → O(1) cada vez
```

---

## Variações Especializadas

**Skip List**: múltiplos níveis de ponteiros para busca O(log n)
```
Nível 2: [1] ─────────→ [5] ─────────→ [9]
Nível 1: [1]→[2]→[3]→[5]→[7]→[9]
```
Usado no Redis como estrutura de sorted sets.

---

## Conexão com Sistemas Operacionais

- **Lista de processos**: o kernel Linux usa `list_head` (lista duplamente encadeada intrusiva) para listas de processos — cada `task_struct` tem ponteiros `prev`/`next` embutidos, evitando alocação extra
- **Buffer cache de disco**: páginas sujas (dirty pages) são mantidas em listas encadeadas para flush periódico — inserção O(1) ao marcar como suja
- **Free list de memória**: alocadores como o slab allocator mantêm linked lists de blocos livres de cada tamanho
- **Lista de módulos do kernel**: `lsmod` mostra módulos em uma linked list — inserção/remoção de módulos é O(1)
- **Tabela de processos (process groups, sessions)**: hierarquia pai-filho em linked lists — `SIGCHLD` propaga através dessas listas

## Conexão com Go

- O runtime do Go usa linked lists internamente para **goroutines aguardando em channels** — quando um canal está cheio/vazio, goroutines entram em uma lista de espera encadeada
- **`container/list`**: implementação de doubly linked list na stdlib Go — use `list.List` quando precisar de inserções/remoções O(1) com posição conhecida
- **`sync.Pool`** usa linked lists de objetos livres por P (processador Go) para evitar alocações frequentes
- Em Go, listas encadeadas são raramente a escolha certa — `slice` geralmente vence em performance real por causa de cache locality. Use linked list apenas quando a inserção O(1) em posição conhecida for crítica
