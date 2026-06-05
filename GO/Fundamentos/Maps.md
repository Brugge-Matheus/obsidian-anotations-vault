---
tags:
  - go
  - go/fundamentos
---
# Maps

> Maps são tabelas hash com colisões tratadas por encadeamento. Go usa uma implementação específica com buckets de 8 pares cada, otimizada para cache. Entender esse modelo ajuda a evitar armadilhas de performance e concorrência.
> 

---

## 1. Implementação Interna

O runtime de Go implementa maps como uma hash table com esta estrutura:

```
hmap (header do map):
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ count    │ flags    │ B        │ noverflow │ hash0    │
│ (int)    │ (uint8)  │ (uint8)  │ (uint16) │ (uint32) │
└──────────┴──────────┴──────────┴──────────┴──────────┘
│ buckets  │ oldbuckets │ nevacuate │ extra   │
│ (*bmap)  │  (*bmap)   │ (uintptr) │ (*map..) │
└──────────┴────────────┴───────────┴─────────┘

Onde B = log2(número de buckets)
Total de buckets = 2^B

Cada bucket (bmap) armazena até 8 pares chave-valor:
┌──────────────────────────────────┐
│ tophash [8]uint8                 │  ← 8 bits altos do hash de cada chave
├──────────────────────────────────┤
│ keys [8]KeyType                  │  ← 8 chaves
├──────────────────────────────────┤
│ values [8]ValueType              │  ← 8 valores
├──────────────────────────────────┤
│ overflow *bmap                   │  ← ponteiro para bucket de overflow
└──────────────────────────────────┘
```

Isso tem implicações práticas:

- **Acesso:** O(1) amortizado — calcula hash, vai para o bucket, busca lineamente pelos 8 slots
- **Iteração:** não garante ordem — o runtime embaralha o início da iteração propositalmente para evitar dependências acidentais de ordem
- **Crescimento:** quando `count / 2^B > 6.5` (load factor), o map dobra de tamanho e rehash em background (incremental)

---

## 2. Declaração e Inicialização

```go
// Map nil — leitura retorna zero value, escrita causa PANIC
var m map[string]int
fmt.Println(m["Alice"])   // 0 — leitura ok (nil map retorna zero value)
fmt.Println(m == nil)     // true
// m["Alice"] = 30        // ❌ panic: assignment to entry in nil map

// make — forma correta para maps que serão preenchidos
idades := make(map[string]int)

// make com capacidade inicial (hint para o runtime pré-alocar buckets)
// NÃO é um limite máximo — o map pode crescer além disso
cache := make(map[string]string, 100)   // ~12 buckets pré-alocados

// Literal
capitais := map[string]string{
	"Brasil":   "Brasília",
	"França":   "Paris",
	"Japão":    "Tóquio",
}
```

> 💡 Fornecer uma capacidade inicial com `make(map[K]V, n)` evita rehashing progressivo quando você sabe quantos elementos vai inserir. Isso pode melhorar significativamente a performance em maps grandes.
> 

---

## 3. Operações CRUD

### Inserir e Atualizar

```go
m := make(map[string]int)

m["Alice"] = 30   // inserir
m["Alice"] = 31   // atualizar (mesma sintaxe)
```

### Ler

```go
v := m["Alice"]    // 31
z := m["Zé"]       // 0 — zero value para int, sem erro ou panic

// Forma com dois retornos — a forma correta quando zero value é ambíguo
v, ok := m["Alice"]
if ok {
	fmt.Println("Alice:", v)
} else {
	fmt.Println("não encontrado")
}

// Por que isso importa:
m["zero"] = 0
val1 := m["zero"]         // 0
val2 := m["inexistente"]  // 0
// Sem ok, não dá para distinguir!
v1, ok1 := m["zero"]         // 0, true
v2, ok2 := m["inexistente"]  // 0, false
```

### Deletar

```go
delete(m, "Alice")    // remove a chave
delete(m, "Quem?")    // chave inexistente — silencioso, sem erro

// Limpar todo o map (Go 1.21+)
clear(m)   // remove todos os pares, mantém a alocação de memória
```

### Tamanho

```go
fmt.Println(len(m))   // número de pares chave-valor atualmente no map
// len() é O(1) — o count é mantido no header do hmap
```

---

## 4. Iteração

```go
m := map[string]int{"Alice": 30, "Bob": 25, "Carol": 28}

// Chave e valor
for nome, idade := range m {
	fmt.Printf("%s: %d\n", nome, idade)
}

// Apenas chaves
for nome := range m {
	fmt.Println(nome)
}

// Apenas valores (raramente útil sem a chave)
for _, idade := range m {
	fmt.Println(idade)
}
```

> ⚠️ A ordem de iteração é **aleatória E propositalmente não-determinística**. O runtime embaralha o bucket inicial a cada iteração como medida de segurança — evita que código acidentalmente dependa de uma ordem específica.
> 

### Iterar em Ordem

```go
import (
	"maps"    // Go 1.21+
	"slices"
)

m := map[string]int{"Alice": 30, "Bob": 25, "Carol": 28}

// Go 1.21+: extrair chaves com maps.Keys
chaves := slices.Collect(maps.Keys(m))   // ou: slices.Sorted(maps.Keys(m))
slices.Sort(chaves)

for _, nome := range chaves {
	fmt.Printf("%s: %d\n", nome, m[nome])
}

// Antes do Go 1.21 (ainda funciona):
chaves2 := make([]string, 0, len(m))
for k := range m {
	chaves2 = append(chaves2, k)
}
sort.Strings(chaves2)
```

---

## 5. Tipos de Chave

Qualquer tipo **comparável** pode ser chave. Internamente, o runtime usa a função de hash específica do tipo:

```go
// ✅ Tipos válidos
map[string]int
map[int]string
map[float64]bool     // ⚠️ floats como chave: NaN != NaN, pode causar perda de dados
map[bool]string
map[[3]int]string    // array é comparável (slice não é!)

// Struct como chave — todos os campos devem ser comparáveis
type Coord struct{ X, Y int }
grade := map[Coord]string{
	{0, 0}: "origem",
	{1, 0}: "direita",
}

// Ponteiro como chave — compara endereços, não conteúdo
type Config struct{ Nome string }
c1 := &Config{"prod"}
c2 := &Config{"prod"}
m := map[*Config]int{c1: 1}
fmt.Println(m[c1])   // 1
fmt.Println(m[c2])   // 0 — c1 != c2 (endereços diferentes)

// ❌ Tipos inválidos como chave (erro de compilação)
// map[[]int]string    — slice não é comparável
// map[map[string]int]string  — map não é comparável
// map[func()]string   — função não é comparável
```

---

## 6. Maps com Valores Complexos

### Map de Slices — Padrão de Agrupamento

```go
// Agrupar dados — zero value de []string é nil, append funciona em nil slice
index := make(map[string][]string)

palavras := []string{"go", "é", "simples", "go", "é", "rápido"}
for i, p := range palavras {
	index[p] = append(index[p], fmt.Sprintf("pos%d", i))
}
fmt.Println(index["go"])   // [pos0 pos3]
fmt.Println(index["é"])    // [pos1 pos4]
```

### Map de Ponteiros para Structs

```go
type Usuario struct {
	Nome  string
	Email string
	Idade int
}

// Map de ponteiros — permite modificar os valores diretamente
usuarios := map[int]*Usuario{
	1: {Nome: "Alice", Email: "alice@ex.com", Idade: 30},
}

// Modificar campo diretamente
usuarios[1].Idade = 31   // ✅ funciona com ponteiro

// Com map de valores (não ponteiros), precisa substituir o struct inteiro
usuariosValor := map[int]Usuario{
	1: {Nome: "Alice", Idade: 30},
}
u := usuariosValor[1]
u.Idade = 31
usuariosValor[1] = u   // necessário: structs em maps não são endereçáveis
```

---

## 7. Pacote `maps` (Go 1.21+)

```go
import "maps"

m := map[string]int{"a": 1, "b": 2, "c": 3}

// Clonar — cópia rasa (shallow)
copia := maps.Clone(m)
copia["a"] = 99
fmt.Println(m["a"])   // 1 — não afetado

// Copiar pares de uma fonte para destino (sobrescreve)
dst := map[string]int{"x": 10}
maps.Copy(dst, m)
fmt.Println(dst)   // map[x:10 a:1 b:2 c:3]

// Deletar com predicado
maps.DeleteFunc(m, func(k string, v int) bool {
	return v > 1   // deleta "b" e "c"
})

// Verificar igualdade
maps.Equal(m1, m2)
maps.EqualFunc(m1, m2, func(v1, v2 int) bool { return v1 == v2 })

// Iterar de forma determinística (Go 1.23+)
// iter.Seq e iter.Seq2 para iteradores
for k, v := range maps.All(m) {
	fmt.Println(k, v)
}
```

---

## 8. Maps e Concorrência

Maps **não são thread-safe** por design (para não penalizar o caso de uso single-threaded com overhead de lock):

```go
// ❌ Race condition — causa panic em produção (Go detecta isso com -race)
m := make(map[string]int)
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
// panic: concurrent map writes

// ✅ Opção 1: sync.RWMutex — controle manual (mais eficiente para casos simples)
type MapSeguro[K comparable, V any] struct {
	mu sync.RWMutex
	m  map[K]V
}

func (ms *MapSeguro[K, V]) Set(k K, v V) {
	ms.mu.Lock()
	defer ms.mu.Unlock()
	ms.m[k] = v
}

func (ms *MapSeguro[K, V]) Get(k K) (V, bool) {
	ms.mu.RLock()
	defer ms.mu.RUnlock()
	v, ok := ms.m[k]
	return v, ok
}

// ✅ Opção 2: sync.Map — otimizado para leitura frequente, escrita rara
// e chaves estáveis (uma vez escritas, não mudam)
var sm sync.Map
sm.Store("chave", 42)
v, ok := sm.Load("chave")
sm.Delete("chave")
sm.LoadOrStore("chave", 99)   // atômico: carrega ou armazena

// Iterar sobre sync.Map
sm.Range(func(k, v any) bool {
	fmt.Println(k, v)
	return true   // retorne false para parar
})
```

---

## 9. Padrões Comuns

### Contar Ocorrências

```go
palavras := []string{"go", "é", "rápido", "go", "é", "simples", "go"}

contagem := make(map[string]int)
for _, p := range palavras {
	contagem[p]++   // zero value de int é 0 — funciona diretamente
}
// map[é:2 go:3 rápido:1 simples:1]
```

### Set (conjunto) com `map[T]struct{}`

`struct{}` ocupa 0 bytes — é o tipo de zero tamanho do Go. Usar como valor do map cria um "set" eficiente:

```go
// struct{} ocupa 0 bytes na memória
// Go otimiza: todos os *struct{} podem apontar para o mesmo endereço (zerobase)
visitado := make(map[string]struct{})

visitado["alice"] = struct{}{}
visitado["bob"] = struct{}{}

// Verificar pertencimento
_, existe := visitado["alice"]   // true
_, existe = visitado["carol"]    // false

// Deletar do set
delete(visitado, "bob")

// map[T]bool também funciona mas gasta 1 byte por entrada
// Para sets de alta cardinalidade, a diferença de memória é significativa
```

### Memoization / Cache

```go
type MemoFunc[T comparable, R any] struct {
	mu    sync.Mutex
	cache map[T]R
	fn    func(T) R
}

func (m *MemoFunc[T, R]) Call(arg T) R {
	m.mu.Lock()
	defer m.mu.Unlock()
	if v, ok := m.cache[arg]; ok {
		return v
	}
	v := m.fn(arg)
	m.cache[arg] = v
	return v
}

fib := &MemoFunc[int, int]{
	cache: make(map[int]int),
	fn:    nil, // inicializado abaixo para permitir recursão
}
// (a recursão com generics requer uma closure)
```

---

## 10. Resumo

| Operação | Sintaxe | Complexidade | Observação |
| --- | --- | --- | --- |
| Declarar (nil) | `var m map[K]V` | — | Só leitura |
| Inicializar | `make(map[K]V)` ou `make(map[K]V, n)` | O(1) | Pronto para uso |
| Inserir/Atualizar | `m[k] = v` | O(1) amortizado | — |
| Ler | `v := m[k]` | O(1) amortizado | Zero value se ausente |
| Verificar | `v, ok := m[k]` | O(1) amortizado | ok=false se ausente |
| Deletar | `delete(m, k)` | O(1) amortizado | Silencioso se ausente |
| Limpar | `clear(m)` | O(n) | Go 1.21+ |
| Tamanho | `len(m)` | O(1) | Mantido no header |
| Iterar | `for k, v := range m` | O(n) | Ordem aleatória |

---

## Conexão com Sistemas Operacionais

### Hash tables dentro do próprio SO

O SO também usa hash tables internamente para gerenciar seus recursos. A **tabela de processos** pode ser indexada por PID usando uma estrutura hash; a tabela de **inodes** usa hash para mapear números de inode a estruturas em memória; o **page cache** (cache de páginas de arquivo) é indexado por `(device, inode, offset)` em uma hash table. O `hmap` do Go e as estruturas internas do kernel são o mesmo padrão — hash com encadeamento por bucket — aplicado em contextos diferentes.

### Map não é thread-safe — [[Threads POSIX]]

A seção 8 mostra que maps causam panic em acessos concorrentes. Em [[Threads POSIX]] estudamos por que isso acontece: duas threads executando simultaneamente podem interleaved suas leituras e escritas na hash table, corrompendo ponteiros internos (e.g., dois `append` simultâneos ao array de buckets). O sistema operacional não oferece proteção contra esse tipo de corrida dentro de um processo — a responsabilidade é do programador, usando `sync.Mutex` ou `sync.RWMutex`.

### Race condition em map — [[Processadores]]

Uma corrida de dados em map é um problema de nível de hardware. Em [[Processadores]] vimos que operações LOAD e STORE no nível de CPU não são automaticamente atômicas para estruturas compostas. Um `m["a"] = 1` pode envolver múltiplas instruções de máquina (calcular hash, encontrar bucket, escrever valor, atualizar count). Se outra goroutine interromper no meio, o estado interno do `hmap` fica inconsistente. O detector de corrida de Go (`-race`) usa instrumentação de LOAD/STORE para detectar exatamente esse tipo de acesso não sincronizado.

### Ordem de iteração randomizada — [[Modelando a Multiprogramação]]

O runtime de Go embaralha deliberadamente o bucket inicial de iteração. Isso conecta ao princípio discutido em [[Modelando a Multiprogramação]]: o SO torna o escalonamento não-determinístico propositalmente para evitar que programas acidentalmente dependam de uma ordem específica de execução. O mesmo raciocínio se aplica aqui — se a iteração de map fosse sempre determinística, programas poderiam acidentalmente depender dessa ordem e quebrarem sutilmente em versões futuras do runtime ou em sistemas com hardware diferente.

### `hash0` e segurança — seed aleatório por processo

O campo `hash0` no `hmap` é uma semente aleatória gerada quando o map é criado. Sem ela, um atacante que conhece o algoritmo de hash poderia enviar chaves que colidem deliberadamente, degradando o map de O(1) para O(n) — ataque de **hash flooding**. Isso é o mesmo problema que motivou a randomização de endereços (**ASLR**) discutida em [[Hardware de Proteção]] e [[Espaços de Endereçamento]]: sem aleatoriedade, estruturas internas previsíveis viram vetores de ataque.

### Crescimento e rehashing — [[Processos]]

Quando o load factor supera 6.5, o map dobra de tamanho e inicia o rehashing incremental. Internamente, isso envolve `mmap()` para alocar novos buckets na heap. Em [[Processos]] vimos que a heap cresce via chamadas ao kernel — cada rehash de um map grande é essencialmente um `malloc()` de maior escala. O Go faz o rehash de forma incremental (movendo alguns buckets por operação) para não bloquear a goroutine por muito tempo, estratégia similar ao que alocadores modernos fazem para evitar picos de latência.