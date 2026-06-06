---
tags:
  - algoritmos
  - estruturas-de-dados
  - hashmap
  - hash-table
---

# Hashmap

Um **HashMap** (Hash Table) mapeia **chaves** para **valores** usando uma **função hash** para calcular diretamente o índice de armazenamento — resultado: acesso O(1) no caso médio.

**Analogia**: um dicionário físico — você vai direto à letra "P" sem ler do início.

```
CHAVE → hash_function() → ÍNDICE → VALOR

"nome"  → hash() → 3  → "João"
"idade" → hash() → 7  → 25
"cidade"→ hash() → 1  → "São Paulo"
```

---

## Como funciona internamente

### 1. Função Hash

Transforma qualquer chave em um índice do array interno:

```go
func hashSimples(key string, tamanho int) int {
    hash := 0
    for _, ch := range key {
        hash += int(ch)
    }
    return hash % tamanho // índice dentro dos limites do array
}

hashSimples("nome", 10)  // → 3
hashSimples("amor", 10)  // → 3  ← COLISÃO! mesmas letras, mesma soma
```

### 2. Tratamento de Colisões

Quando duas chaves geram o mesmo índice:

**Chaining (Encadeamento)** — cada bucket contém uma lista:
```
índice 3: ["nome"→João] → ["nick"→José] → NULL
```

**Open Addressing** — procura próximo slot disponível:
```
índice 3 ocupado → tenta índice 4 → tenta 5...
```

### 3. Load Factor e Rehashing

```
Load Factor = elementos / tamanho_do_array

Quando LF > 0.75:
- Cria array com dobro do tamanho
- Recalcula hash de TODOS elementos (O(n))
- Amortizado: O(1) por inserção

Antes: [A][B][C][D][E][F][•][ ] → LF = 6/8 = 0.75 → rehash!
Depois:[A][ ][B][ ][C][ ][D][ ][ ][E][ ][ ][F][ ][ ][ ]
```

---

## Implementação em Go

```go
// Go tem map nativo — implementado como hashmap com chaining
m := make(map[string]int)

m["alice"] = 90
m["bob"] = 75
m["charlie"] = 88

// Lookup O(1) amortizado
score, ok := m["alice"]
if ok {
    fmt.Println(score) // 90
}

// Delete O(1) amortizado
delete(m, "bob")

// Iterar — ordem NÃO é garantida (propositalmente randomizado)
for chave, valor := range m {
    fmt.Printf("%s: %d\n", chave, valor)
}

// Contar frequências — padrão clássico
palavras := []string{"go", "é", "rápido", "go", "é", "bom"}
freq := make(map[string]int)
for _, p := range palavras {
    freq[p]++ // se chave não existe, zero value é 0
}
// {"go": 2, "é": 2, "rápido": 1, "bom": 1}
```

### Hashmap concorrente

```go
import "sync"

// sync.Map — otimizado para leituras concorrentes
var sm sync.Map

sm.Store("chave", "valor")
val, ok := sm.Load("chave")
sm.Delete("chave")

// Para writes frequentes, prefira mutex + map normal
var mu sync.RWMutex
m := make(map[string]int)

// Escrita
mu.Lock()
m["x"] = 1
mu.Unlock()

// Leitura concorrente
mu.RLock()
v := m["x"]
mu.RUnlock()
```

---

## Análise de Performance

| Operação | Caso Médio | Pior Caso | Motivo do Pior Caso |
|----------|-----------|-----------|---------------------|
| Insert | O(1) | O(n) | Todas chaves colidem no mesmo bucket |
| Lookup | O(1) | O(n) | Precisa percorrer toda a chain |
| Delete | O(1) | O(n) | Idem |
| Space | O(n) | O(n) | — |

**Fatores que afetam performance:**
- Qualidade da função hash (distribuição uniforme)
- Load factor (< 0.75 garante O(1) amortizado)
- Tratamento de colisões escolhido

---

## Hashmap vs Outras Estruturas

| Estrutura | Busca | Inserção | Ordem | Uso Típico |
|-----------|-------|----------|-------|------------|
| **HashMap** | O(1) | O(1) | Não | Acesso rápido chave-valor |
| [[Arrays]] | O(n) | O(1) final | Sim | Acesso por índice numérico |
| [[Linked Lists]] | O(n) | O(1) início | Sim | Inserções frequentes |
| [[Árvores Binárias]] (BST) | O(log n) | O(log n) | Sim | Dados ordenados |

---

## Padrões Comuns em Go

```go
// Set (conjunto) — map com valor vazio
set := make(map[string]struct{})
set["a"] = struct{}{}
set["b"] = struct{}{}
_, existe := set["a"] // true

// Memoização com map
memo := make(map[int]int)
func fib(n int) int {
    if n <= 1 { return n }
    if v, ok := memo[n]; ok { return v }
    resultado := fib(n-1) + fib(n-2)
    memo[n] = resultado
    return resultado
}

// Grouping — agrupar por chave
type Pessoa struct { Nome, Cidade string }
pessoas := []Pessoa{{"Alice", "SP"}, {"Bob", "RJ"}, {"Carol", "SP"}}
porCidade := make(map[string][]Pessoa)
for _, p := range pessoas {
    porCidade[p.Cidade] = append(porCidade[p.Cidade], p)
}
```

---

## Conexão com Sistemas Operacionais

- **Page cache (buffer cache)**: o kernel mantém um hashmap `(device, block_number) → page` para saber rapidamente se uma página de disco já está em memória RAM. Acesso O(1) evita leituras desnecessárias ao disco
- **Inode cache**: `icache` do Linux mapeia `inode_number → inode_struct` em memória — evita reler metadados de arquivo do disco. Baseado em hashtable
- **VFS (Virtual Filesystem)**: a dentry cache (dcache) mapeia `(parent_inode, filename) → dentry` — resolve caminhos de arquivo sem acessar disco. É o maior consumidor de memória do kernel em servidores de arquivos
- **Tabela de processos**: o kernel usa hashtable para mapear `PID → task_struct` — `kill(pid, sig)` precisa encontrar o processo rapidamente
- **TCP connection table**: hash de `(src_ip, src_port, dst_ip, dst_port) → socket` — permite ao kernel rotear pacotes recebidos ao socket correto em O(1)
- **`/proc/sys/net/ipv4/tcp_max_syn_backlog`**: controla o tamanho da hashtable de conexões TCP em SYN_RECV — SYN flood ataca justamente esta estrutura

## Conexão com Go

- **`map` em Go** é implementado com chaining — cada bucket contém até 8 pares chave-valor inline (sem ponteiros extras). Overflow buckets são criados quando necessário
- **Hash seed aleatório**: desde Go 1.0, o seed da função hash de `map` é randomizado por processo — previne ataques de **hash flooding** (enviar chaves que colidem propositalmente para criar O(n) behavior)
- **`map` não é thread-safe**: acessos concorrentes causam `fatal error: concurrent map read and map write`. Use `sync.RWMutex` ou `sync.Map`
- **Iteração aleatória**: Go randomiza a ordem de iteração de `map` — evita que código dependa de uma ordem que não é garantida
- **Comparação**: chaves de `map` devem ser **comparáveis** (`==` definido). Slices e maps não podem ser chaves — use structs ou strings como chaves compostas
