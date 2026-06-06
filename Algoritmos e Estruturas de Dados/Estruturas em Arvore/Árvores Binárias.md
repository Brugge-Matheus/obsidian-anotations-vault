---
tags:
  - algoritmos
  - estruturas-de-dados
  - arvore-binaria
  - bst
---

# Árvores Binárias

Uma **Árvore Binária** é uma estrutura hierárquica onde cada nó tem **no máximo dois filhos**: esquerdo e direito. Representa relações pai-filho de forma recursiva.

**Analogia**: árvore genealógica onde cada pessoa tem no máximo dois filhos.

```
REPRESENTAÇÃO VISUAL
         (A)      ← Root (raiz)
        /   \
      (B)   (C)   ← Level 1
     /  \   / \
   (D) (E)(F)(G)  ← Level 2 (folhas)
```

---

## Terminologia Essencial

| Termo | Significado |
|-------|-------------|
| Root | Nó sem pai (topo da árvore) |
| Leaf | Nó sem filhos |
| Height | Maior distância da raiz até folha |
| Depth | Distância de um nó até a raiz |
| Subtree | Árvore formada por um nó e seus descendentes |

---

## Binary Search Tree (BST)

A variação mais importante: filho esquerdo < pai < filho direito.

```
    (8)
   /   \
  (3)  (10)
 /  \    \
(1) (6)  (14)
   /  \   /
  (4)(7)(13)
```

### Implementação em Go

```go
type Node struct {
    value       int
    left, right *Node
}

type BST struct {
    root *Node
}

func (t *BST) Insert(value int) {
    t.root = insertNode(t.root, value)
}

func insertNode(node *Node, value int) *Node {
    if node == nil {
        return &Node{value: value}
    }
    if value < node.value {
        node.left = insertNode(node.left, value)
    } else if value > node.value {
        node.right = insertNode(node.right, value)
    }
    return node
}

func (t *BST) Search(value int) bool {
    return searchNode(t.root, value)
}

func searchNode(node *Node, value int) bool {
    if node == nil {
        return false
    }
    if value == node.value {
        return true
    }
    if value < node.value {
        return searchNode(node.left, value)
    }
    return searchNode(node.right, value)
}

// In-order traversal → retorna valores em ordem crescente
func (t *BST) InOrder() []int {
    result := []int{}
    inOrderTraversal(t.root, &result)
    return result
}

func inOrderTraversal(node *Node, result *[]int) {
    if node == nil {
        return
    }
    inOrderTraversal(node.left, result)
    *result = append(*result, node.value)
    inOrderTraversal(node.right, result)
}

// Uso
bst := &BST{}
for _, v := range []int{8, 3, 10, 1, 6, 14, 4, 7, 13} {
    bst.Insert(v)
}
fmt.Println(bst.InOrder())  // [1 3 4 6 7 8 10 13 14]
fmt.Println(bst.Search(6))  // true
```

---

## Algoritmos de Travessia

### In-order (Esquerda → Raiz → Direita)
```
Resultado: 1, 2, 3, 4, 5, 6, 7 — valores ORDENADOS em BST
Uso: obter elementos em ordem crescente
```

### Pre-order (Raiz → Esquerda → Direita)
```
Resultado: 4, 2, 1, 3, 6, 5, 7
Uso: serializar/copiar a árvore
```

### Post-order (Esquerda → Direita → Raiz)
```
Resultado: 1, 3, 2, 5, 7, 6, 4
Uso: deletar árvore (filhos antes do pai), cálculo de expressões
```

### Level-order / BFS
```
Resultado: 4, 2, 6, 1, 3, 5, 7 — nível por nível
Uso: encontrar menor caminho, visualização
```

---

## Tipos de Árvores Binárias

| Tipo | Propriedade | Uso |
|------|-------------|-----|
| **BST** | esq < pai < dir | Busca eficiente |
| **Complete** | todos níveis preenchidos exceto último | Heaps |
| **Full** | todo nó tem 0 ou 2 filhos | Expressões |
| **Perfect** | todos níveis completamente cheios | Análise teórica |
| **Balanced** | diferença de altura ≤ 1 | AVL, Red-Black |

---

## Análise de Performance

| Operação | BST Médio | BST Pior | Árvore Balanceada |
|----------|-----------|----------|-------------------|
| Search | O(log n) | O(n) | O(log n) |
| Insert | O(log n) | O(n) | O(log n) |
| Delete | O(log n) | O(n) | O(log n) |
| Traversal | O(n) | O(n) | O(n) |

**Pior caso da BST**: inserir dados já ordenados → árvore degenerada vira lista:
```
1 → 2 → 3 → 4 → 5   (altura = n-1, busca O(n))
```

---

## Árvores Auto-balanceadas

Para garantir O(log n) mesmo com dados ordenados:

**AVL Tree**: diferença de altura entre subárvores ≤ 1. Mais rígida, mais rotações.

**Red-Black Tree**: nós coloridos (vermelho/preto) com regras de balanceamento. Mais flexível, menos rotações. Usada em `std::map` (C++), `TreeMap` (Java), `map` do kernel Linux.

---

## Conexão com Sistemas Operacionais

- **CFS do Linux (Completely Fair Scheduler)**: usa **red-black tree** para ordenar processos pelo `vruntime` (tempo virtual de CPU). O processo com menor vruntime (mais à esquerda) é o próximo a executar → busca/inserção O(log n)
- **Área de memória virtual (`vm_area_struct`)**: o mapa de memória de cada processo é um red-black tree de VMAs (Virtual Memory Areas). `mmap()`, `munmap()` e page faults navegam nessa árvore
- **Inodes e entradas de diretório**: sistemas de arquivos como ext4 usam árvores para organizar entradas em diretórios grandes (htree = hash tree)
- **Árvores de expressão**: compiladores e shells usam expression trees (árvores binárias) para representar expressões aritméticas e booleanas antes de gerar código
- **Timer wheel do kernel**: timers de alta precisão são organizados em árvores binárias para ordenação eficiente

## Conexão com Go

- **`sort.Search()`**: implementa busca binária em slice já ordenada — internamente similar à busca em BST
- **`golang.org/x/exp/maps`**: utilitários para maps, incluindo sorted keys — internamente usa sort O(n log n)
- **`container/heap`**: implementa heap (árvore binária completa) — base para priority queues. Interface `heap.Interface` exige `Len`, `Less`, `Swap`, `Push`, `Pop`
- **JSON/XML parsing**: parsers em Go constroem ASTs (Abstract Syntax Trees) — árvores binárias ou n-árias que representam a estrutura do documento
- **Goroutine tree**: goroutines têm relação pai-filho implícita (goroutine que criou outra), embora o Go não exponha isso diretamente
