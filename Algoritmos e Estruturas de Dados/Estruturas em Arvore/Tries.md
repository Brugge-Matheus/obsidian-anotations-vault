---
tags:
  - algoritmos
  - estruturas-de-dados
  - trie
  - prefix-tree
  - strings
---

# Tries

Uma **Trie** (pronuncia-se "try") é uma árvore especializada para strings onde cada nó representa um caractere. Caminhos da raiz às folhas formam palavras. Também chamada de **Prefix Tree** ou **Digital Tree**.

**Analogia**: dicionário onde palavras com o mesmo prefixo seguem o mesmo caminho.

```
TRIE COM: ["CAT", "CAR", "CARD", "CARE", "DOG"]

         ROOT
        /    \
       C      D
       |      |
       A      O
       |      |
       T*     G*
      / \
    (ε)  R*
        / \
       D*  E*

* = isEndOfWord = true
```

---

## Estrutura do Nó

```
NÓ TRIE
┌─────────────────────────────────────┐
│           isEndOfWord (bool)        │
├─────────────────────────────────────┤
│   children: map[rune]*TrieNode      │
│   (ou array[26] para a-z apenas)    │
└─────────────────────────────────────┘
```

Vantagem do `map` vs array: economiza memória para alfabetos esparsos (Unicode, emojis, strings com dígitos).

---

## Operações

### Inserção — O(m), m = comprimento da palavra

```
INSERIR "CAR":
ROOT → criar 'C' → criar 'A' → criar 'R' → isEndOfWord = true
```

### Busca — O(m)

```
BUSCAR "CAT":
ROOT → C (existe?) → A (existe?) → T (existe? isEndOfWord?) → true
```

### Verificar Prefixo — O(p), p = comprimento do prefixo

```
TEM PALAVRA COM PREFIXO "CA"?
ROOT → C → A → existe! (não precisa checar isEndOfWord)
```

---

## Implementação em Go

```go
type TrieNode struct {
    children    map[rune]*TrieNode
    isEndOfWord bool
}

func newTrieNode() *TrieNode {
    return &TrieNode{children: make(map[rune]*TrieNode)}
}

type Trie struct {
    root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{root: newTrieNode()}
}

func (t *Trie) Insert(word string) {
    current := t.root
    for _, ch := range word {
        if current.children[ch] == nil {
            current.children[ch] = newTrieNode()
        }
        current = current.children[ch]
    }
    current.isEndOfWord = true
}

func (t *Trie) Search(word string) bool {
    current := t.root
    for _, ch := range word {
        if current.children[ch] == nil {
            return false
        }
        current = current.children[ch]
    }
    return current.isEndOfWord
}

func (t *Trie) StartsWith(prefix string) bool {
    current := t.root
    for _, ch := range prefix {
        if current.children[ch] == nil {
            return false
        }
        current = current.children[ch]
    }
    return true
}

// Autocomplete — encontra todas palavras com dado prefixo
func (t *Trie) Autocomplete(prefix string) []string {
    current := t.root
    for _, ch := range prefix {
        if current.children[ch] == nil {
            return nil
        }
        current = current.children[ch]
    }
    var result []string
    t.collectWords(current, prefix, &result)
    return result
}

func (t *Trie) collectWords(node *TrieNode, prefix string, result *[]string) {
    if node.isEndOfWord {
        *result = append(*result, prefix)
    }
    for ch, child := range node.children {
        t.collectWords(child, prefix+string(ch), result)
    }
}

// Uso
trie := NewTrie()
for _, w := range []string{"cat", "car", "card", "care", "careful", "dog"} {
    trie.Insert(w)
}

fmt.Println(trie.Search("cat"))           // true
fmt.Println(trie.Search("caring"))        // false
fmt.Println(trie.StartsWith("car"))       // true
fmt.Println(trie.Autocomplete("car"))     // [car card care careful]
```

---

## Análise de Performance

| Operação | Complexidade | Observação |
|----------|-------------|------------|
| Insert | O(m) | m = comprimento da palavra |
| Search | O(m) | Independente do total de palavras |
| StartsWith | O(p) | p = comprimento do prefixo |
| Autocomplete | O(p + n) | n = número de resultados |

### Comparação com HashMap para strings

| Estrutura | Busca exata | Prefixo | Autocomplete |
|-----------|------------|---------|--------------|
| **Trie** | O(m) | O(p) | O(p + k) |
| **HashMap** | O(m) | O(n×m) | O(n×m) |
| **BST** | O(m log n) | O(m log n) | O(m log n + k) |

Trie ganha em operações de prefixo. HashMap ganha em busca exata com menor overhead de memória.

---

## Variações

### Compressed Trie (Patricia Tree)
Comprime caminhos lineares (um único filho) em um único nó:
```
NORMAL: A → B → C → D*   ==>   COMPRESSED: A → "BCD"*
Economiza nós em prefixos longos sem bifurcação
```

### Suffix Trie
Armazena todos os sufixos de uma string — usado em bioinformática para busca de padrões em DNA.

### Ternary Search Trie (TST)
Cada nó tem 3 filhos: menor, igual, maior. Combina eficiência de Trie com economia de memória de BST.

---

## Casos de Uso

- **Autocomplete**: Google Search, IDEs (IntelliSense), terminais
- **Spell checker**: verificar se palavra existe no dicionário
- **IP routing**: tabelas de roteamento usam **longest prefix match** — encontrar a rota mais específica para um IP
- **DNS**: hierarquia de domínios é uma trie (`.com` → `google` → `www`)
- **T9 / predictive text**: teclados antigos de celular

---

## Conexão com Sistemas Operacionais

- **Tabela de roteamento IP**: o kernel Linux usa uma **trie compacta** (LC-Trie / LPC-Trie) para armazenar rotas IP. Ao receber um pacote, faz **longest prefix match** para encontrar qual rota usar — operação O(k) onde k = bits do endereço IP (32 para IPv4, 128 para IPv6)
- **`/proc/net/fib_trie`**: você pode ver a trie de roteamento do kernel Linux em tempo real neste arquivo
- **Namespace de arquivos**: o VFS do Linux resolve caminhos como `/usr/local/bin/go` percorrendo uma estrutura similar a uma trie — cada componente do caminho é um nó
- **Shell autocomplete**: bash/zsh usam tries internas para completar comandos e nomes de arquivos
- **Filtros de pacotes (iptables/nftables)**: regras com prefixos de endereços IP são armazenadas em tries para matching eficiente

## Conexão com Go

- **URL routing em Go**: frameworks como `httprouter` e `chi` usam radix trees (compressed tries) para rotear URLs como `/users/:id/posts` → handler. O(m) onde m = comprimento da URL
- **`net/http` do stdlib**: o `ServeMux` padrão do Go usa matching por prefixo — semanticamente uma trie, embora implementado com mapa simples
- **`strings.HasPrefix`, `strings.HasSuffix`**: operações O(m) que a trie torna O(1) por nível ao processar múltiplas strings em batch
- **`golang.org/x/text`**: operações Unicode usam tries compactas para mapear caracteres a propriedades (categoria, capitalização, etc.)
- **Implementar router**: para um router HTTP de alta performance em Go, implemente uma radix tree ou use `httprouter` — muito mais eficiente que slice de regexp
