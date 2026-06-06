---
tags:
  - algoritmos
  - estruturas-de-dados
  - btree
  - banco-de-dados
  - sistemas-de-arquivos
---

# B-Tree

Uma **B-Tree** é uma árvore auto-balanceada onde cada nó pode ter **múltiplas chaves e filhos**. Otimizada para sistemas que leem/escrevem grandes blocos de dados — como bancos de dados e sistemas de arquivos.

**Analogia**: sistema de arquivo com pastas que contêm múltiplos documentos e subpastas, organizadas em ordem.

```
B-TREE DE ORDEM 3 (máximo 5 chaves por nó)
                [10, 20, 30]
               /    |    |    \
        [5, 7]   [12, 15]  [25]  [35, 40, 45]
```

Veja [[Árvores Binárias]] para a versão com apenas 2 filhos por nó.

---

## Por que B-Trees existem?

[[Árvores Binárias]] têm altura O(log₂ n) — para 1 milhão de chaves, ~20 níveis. Cada nível pode exigir um **acesso a disco** (I/O). Disco é ~10.000x mais lento que RAM.

**Solução**: aumentar o número de filhos por nó (ordem m). Altura cai para O(log_m n).

```
BST (m=2), n=1.000.000: altura ≈ 20 → 20 I/Os
B-Tree (m=1000):         altura ≈ 2  → 2-3 I/Os

Redução dramática de I/O!
```

---

## Propriedades (B-Tree de ordem m)

1. **Raiz**: 1 a m-1 chaves
2. **Nós internos**: ⌈m/2⌉-1 a m-1 chaves
3. **Folhas**: todas no mesmo nível (balanceamento perfeito)
4. **Chaves ordenadas**: dentro de cada nó
5. **Propriedade de busca**: subárvore esquerda < chave < subárvore direita

### Estrutura do nó

```
NÓ B-TREE (ordem m)
┌───┬────┬────┬───┬────┬────┬───┬────┐
│ n │ k₁ │ P₁ │...│ kₙ │ Pₙ │   │leaf│
└───┴────┴────┴───┴────┴────┴───┴────┘
  ↑    ↑    ↑               ↑     ↑
count chave ponteiro      ponteiro boolean
```

---

## Operações

### Busca — O(log_m n)

```
BUSCAR 25 em [10, 20, 30]:
25 > 20 e 25 < 30 → entrar no filho entre 20 e 30
Encontrar 25 na folha → ENCONTRADO

Em disco: cada comparação = 1 acesso a bloco de disco
```

### Inserção — sempre em folha, split se necessário

```
INSERIR 13 em folha cheia [10, 15, 20, 25]:
1. Split: [10, 15] | promover 20 | [25]
2. Inserir 13 em [10, 15] → [10, 13, 15]

O split pode propagar até a raiz, aumentando a altura
```

### Remoção — merge ou redistribuição

```
CASOS:
1. Folha com > mínimo chaves → simplesmente remover
2. Folha com mínimo → "borrow" do irmão ou merge
3. Nó interno → substituir por predecessor/sucessor
```

---

## Implementação em Go (busca)

```go
const ordem = 5 // máximo 4 chaves, 5 filhos por nó

type BTreeNode struct {
    keys     []int
    children []*BTreeNode
    isLeaf   bool
}

func (node *BTreeNode) search(key int) bool {
    i := 0
    // encontra posição onde key deveria estar
    for i < len(node.keys) && key > node.keys[i] {
        i++
    }
    // chave encontrada neste nó
    if i < len(node.keys) && key == node.keys[i] {
        return true
    }
    // se é folha, não existe
    if node.isLeaf {
        return false
    }
    // busca no filho apropriado
    return node.children[i].search(key)
}
```

---

## B+Tree — Variação Mais Comum em Banco de Dados

Na **B+Tree**, dados ficam **apenas nas folhas**, que são ligadas por ponteiros:

```
B+TREE:
              [10, 20]
             /    |    \
         [5,7]   [12,15] [25,30]
        /  |  \   |   |   |   |
      [1,5][7,8][12][15][25][30,35]
         ↕     ↕    ↕   ↕   ↕    ↕
       data  data data data data data
        ←── linked list de folhas ──→
```

**Vantagens sobre B-Tree:**
- **Range queries eficientes**: percorrer folhas ligadas sem subir a árvore
- **Nós internos menores**: só chaves, sem dados → mais chaves por bloco → altura menor
- Todos os dados no mesmo nível → scans sequenciais eficientes

---

## Onde B-Trees são Usadas

### Banco de Dados
- **MySQL InnoDB**: índices clustered B+Tree — dados ordenados por primary key
- **PostgreSQL**: índices B-Tree para queries com `=`, `<`, `>`, `BETWEEN`
- **SQLite**: B+Tree tanto para dados quanto para índices

### Sistemas de Arquivos
- **NTFS**: Master File Table usa B+Tree para metadados
- **HFS+ / APFS**: catálogo de arquivos em B-Tree
- **ext4**: diretórios grandes usam htree (variante de B-Tree)
- **ZFS**: metadados em B-Trees para consistência transacional

### Key-Value Stores
- **LevelDB / RocksDB**: LSM-Trees (inspiradas em B-Trees mas otimizadas para writes)
- **BoltDB** (Go nativo): B+Tree pura, usada pelo etcd e muitas ferramentas Go

---

## Análise de Performance

| Operação | Complexidade | Nota |
|----------|-------------|------|
| Busca | O(log_m n) | m = ordem da árvore |
| Inserção | O(log_m n) | Splits em cascata no pior caso |
| Remoção | O(log_m n) | Merges podem propagar |
| Range query | O(log_m n + k) | k = resultados (B+Tree ideal) |

---

## Conexão com Sistemas Operacionais

- **Sistemas de arquivos**: ext4, NTFS, HFS+, APFS usam B-Trees para diretórios e metadados. Cada nó da B-Tree é alinhado ao tamanho de um bloco de disco (4KB tipicamente) — minimiza I/Os
- **Journaling**: ao fazer transações em sistemas de arquivos (ext4 journal, ZFS), as operações na B-Tree são agrupadas em batches para reduzir I/Os
- **Buffer pool do banco de dados**: MySQL InnoDB mantém páginas de B-Tree em memória (buffer pool). Um "page read" = busca na B+Tree + possível leitura de disco
- **Tamanho do nó = tamanho da página**: nós de B-Tree são projetados para ter exatamente o tamanho de uma página de disco (4KB ou 16KB). Isso minimiza leituras parciais e fragmentação

## Conexão com Go

- **BoltDB**: banco de dados embarcado em Go puro, usa B+Tree. Base do **etcd** (Kubernetes) e **bbolt**. Código Go: `db.View(func(tx *bolt.Tx) error { ... })`
- **`golang.org/x/exp/slices`**: busca binária em slices ordenadas — conceitualmente similar à busca em nós de B-Tree
- **Implementar índice**: para implementar um índice range-query eficiente em Go, use BoltDB ou implemente B+Tree sobre `[]byte` com páginas
- **LSM-Trees em Go**: LevelDB port `syndtr/goleveldb` e RocksDB binding `linxGnu/grocksdb` — alternativas a B-Trees para write-heavy workloads
