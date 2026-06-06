---
tags:
  - moc
  - algoritmos
  - estruturas-de-dados
---

# MOC — Algoritmos e Estruturas de Dados

Mapa de conhecimento sobre algoritmos e estruturas de dados. Estudo fundamentado em implementações práticas com conexões aos internals de Sistemas Operacionais e à linguagem Go.

---

## Fundamentos

| Nota | Conceito Central |
|------|-----------------|
| [[Complexidade de Algoritmos]] | O que é e por que analisar |
| [[Big O Notation]] | O(1) · O(n) · O(log n) · O(n²) · O(2^n) |

---

## Estruturas de Dados

### Lineares

| Estrutura | Acesso | Inserção Início | Caso de Uso |
|-----------|--------|-----------------|-------------|
| [[Arrays]] | O(1) | O(n) | Acesso por índice, cache-friendly |
| [[Linked Lists]] | O(n) | O(1) | Inserções frequentes no início |
| [[Pilha (Stack)]] | O(1) topo | O(1) | Recursão, undo/redo, call stack |
| [[Queue (Fila)]] | O(1) front | O(1) | Scheduling, BFS, I/O buffers |

### Mapeamento

| Estrutura | Busca Média | Caso de Uso |
|-----------|-------------|-------------|
| [[Hashmap]] | O(1) | Cache, índices, contadores |

### Hierárquicas / Árvores

| Estrutura | Busca | Caso de Uso |
|-----------|-------|-------------|
| [[Árvores Binárias]] | O(log n) | BST, heaps, parsers |
| [[B-Tree]] | O(log_m n) | Bancos de dados, sistemas de arquivos |
| [[Tries]] | O(m) | Autocomplete, roteamento, DNS |

### Grafos

| Estrutura | Traversal | Caso de Uso |
|-----------|-----------|-------------|
| [[Grafos]] | O(V + E) | Redes, rotas, dependências |

---

## Algoritmos

### Busca

| Algoritmo | Complexidade | Pré-requisito |
|-----------|-------------|---------------|
| [[Algoritmos de Busca#Busca Linear\|Busca Linear]] | O(n) | Nenhum |
| [[Algoritmos de Busca#Busca Binária\|Busca Binária]] | O(log n) | Array ordenado |
| [[Algoritmos de Busca#DFS\|DFS]] | O(V + E) | Árvore ou grafo |
| [[Algoritmos de Busca#BFS\|BFS]] | O(V + E) | Árvore ou grafo |

### Ordenação

| Algoritmo | Pior Caso | Médio | Estável |
|-----------|-----------|-------|---------|
| [[Algoritmos de Ordenação#Bubble Sort\|Bubble Sort]] | O(n²) | O(n²) | Sim |
| [[Algoritmos de Ordenação#Selection Sort\|Selection Sort]] | O(n²) | O(n²) | Não |
| [[Algoritmos de Ordenação#Insertion Sort\|Insertion Sort]] | O(n²) | O(n²) | Sim |
| [[Algoritmos de Ordenação#Merge Sort\|Merge Sort]] | O(n log n) | O(n log n) | Sim |
| [[Algoritmos de Ordenação#Quick Sort\|Quick Sort]] | O(n²) | O(n log n) | Não |
| [[Algoritmos de Ordenação#Counting Sort\|Counting Sort]] | O(n+k) | O(n+k) | Sim |

---

## Conexões com outras áreas

- **[[MOC - Sistemas Operacionais]]**: Scheduling usa [[Queue (Fila)]]; call stack do processo usa [[Pilha (Stack)]]; sistemas de arquivos usam [[B-Tree]]; virtual memory mapping usa [[Hashmap]]
- **[[MOC - GO]]**: `slice` é um [[Arrays]] dinâmico; `map` é um [[Hashmap]]; goroutines usam [[Queue (Fila)]] internamente; interfaces permitem polimorfismo em estruturas
- **[[MOC - Docker]]**: Union filesystem (OverlayFS) usa estruturas de árvore; roteamento de rede usa [[Tries]] para longest prefix match
- **[[MOC - Redes]]**: Tabelas de roteamento são [[Tries]]; BFS/DFS em [[Grafos]] modelam redes; buffers de pacotes são [[Queue (Fila)]]
