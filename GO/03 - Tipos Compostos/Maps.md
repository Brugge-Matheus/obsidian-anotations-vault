---
tags:
  - go
  - go/tipos-compostos
aliases:
  - Maps em Go
  - Map
  - Swiss Tables
  - Hash table em Go
ordem: 14
modulo: "03 - Tipos Compostos"
---
# Maps

> **Nota 14 da trilha de Go.** Um **map** associa **chaves** a **valores**: "nome → idade", "id → usuário", "palavra → quantidade". É a tabela hash embutida da linguagem, com busca, inserção e remoção em tempo **O(1)** em média. Nesta nota veremos como criar e usar maps, o idioma **vírgula ok**, quais tipos podem ser chave, por que a **ordem de iteração é aleatória**, os padrões mais comuns (contagem, agrupamento, conjuntos), o pacote `maps` e, no final, como o map é implementado desde o Go 1.24 com **Swiss Tables**.
>
> **Pré-requisitos:** [[Arrays e Slices]], [[Operadores de Comparação e Lógicos]].

---

## 🧭 De onde viemos

Em [[Arrays e Slices]] o acesso era por **posição numérica** (`s[3]`). Muitas vezes, porém, queremos acessar por um **nome** ou por outro valor qualquer. Em [[Operadores de Comparação e Lógicos]] vimos que alguns tipos são **comparáveis**; essa propriedade vai decidir quem pode ser chave de map. E em [[Ponteiros]] vimos que um map, como o slice, é um valor que contém um ponteiro interno.

> 💡 **Tabela hash (hash table):** estrutura que usa uma **função hash** para transformar a chave num número, e usa esse número para saber em que posição de um array guardar o par chave-valor. Assim, encontrar uma chave não exige percorrer todos os elementos. Você estudou a estrutura em [[Hashmap]] (AED).

---

## 🗝️ 1. Criando maps

O tipo é `map[TipoDaChave]TipoDoValor`.

```go
// Literal
idades := map[string]int{
	"Ana":  30,
	"Bia":  25,     // vírgula final obrigatória (ver Sintaxe, Blocos e Escopo)
}

// make: map vazio, pronto para uso
contagem := make(map[string]int)

// make com dica de tamanho (pré-aloca espaço para ~100 entradas)
cache := make(map[int64]*Usuario, 100)

// Literal vazio
vazio := map[string]bool{}
```

### 1.1 O map nil

O zero value de um map é `nil`. E aqui existe uma **assimetria** importante (saídas reais):

```go
var m map[string]int       // nil

fmt.Println(m["x"], len(m))   // 0 0   ← LER funciona (retorna o zero value)
for range m { }               // ✅ iterar funciona (zero voltas)
delete(m, "x")                // ✅ não faz nada

m["a"] = 1                    // 💥 panic: assignment to entry in nil map
```

> ⚠️ Diferente do slice nil (onde `append` funciona), **não é possível escrever num map nil**. Sempre crie com `make` ou literal antes de inserir. Em structs, inicialize o campo de map no construtor (ver [[Structs]]).

---

## 🔧 2. Operações básicas

Saídas reais:

```go
idades := map[string]int{"Ana": 30, "Bia": 25}

idades["Caio"] = 41          // inserir
idades["Ana"]++              // atualizar (lê, soma, escreve)
fmt.Println(idades["Ana"])   // 31  ← ler
fmt.Println(idades["Zé"])    // 0   ← chave ausente: devolve o zero value, sem erro
fmt.Println(len(idades))     // 3

delete(idades, "Bia")        // remover
delete(idades, "naoexiste")  // remover chave ausente: não faz nada (sem erro)
fmt.Println(idades)          // map[Ana:31 Caio:41]

clear(idades)                // remove TODAS as entradas (Go 1.21+)
```

| Operação | Sintaxe | Custo médio |
| --- | --- | --- |
| Inserir / atualizar | `m[k] = v` | O(1) |
| Ler | `v := m[k]` | O(1) |
| Ler e saber se existe | `v, ok := m[k]` | O(1) |
| Remover | `delete(m, k)` | O(1) |
| Tamanho | `len(m)` | O(1) |
| Remover tudo | `clear(m)` | O(n) |

> 💡 `fmt.Println` imprime maps com as **chaves ordenadas** (desde o Go 1.12), para facilitar testes e depuração. Isso é só na impressão; a iteração continua aleatória.

---

## ❓ 3. O idioma "vírgula ok"

Como uma chave ausente devolve o zero value, `idades["Zé"] == 0` não diz se o Zé tem 0 anos ou se não existe. A leitura com **dois valores** resolve (saída real):

```go
v, ok := idades["Zé"]
fmt.Println(v, ok)          // 0 false

if idade, ok := idades["Ana"]; ok {
	fmt.Println("Ana tem", idade)
} else {
	fmt.Println("Ana não cadastrada")
}
```

> 💡 **Vírgula ok (comma ok):** padrão de Go em que uma operação devolve um segundo valor `bool` dizendo se deu certo. Aparece em maps (`v, ok := m[k]`), em *type assertions* (`s, ok := x.(string)`, ver [[Type Assertion e Type Switch]]) e no recebimento de channels (`v, ok := <-ch`, ver [[Channels]]).

Quando o zero value já é uma resposta adequada, não precisa do `ok`:

```go
contagem[palavra]++           // se não existia, começa do 0 → vira 1
if ativo[usuario] { }         // map[string]bool: ausente = false
```

---

## 🔑 4. Quem pode ser chave

> **A chave precisa ser de um tipo comparável** (aceita `==`), porque o map precisa verificar se duas chaves são iguais.

| Pode ser chave | Não pode |
| --- | --- |
| Números, strings, `bool` | Slices |
| Ponteiros (compara endereço) | Maps |
| Channels | Funções |
| Arrays de comparáveis | Structs que contêm slice, map ou função |
| Structs de comparáveis | |
| Interfaces (cuidado: panic se o valor dentro não for comparável) | |

Saídas reais:

```go
type Ponto struct{ X, Y int }
grade := map[Ponto]string{{0, 0}: "origem"}     // ✅ struct de ints como chave
fmt.Println(grade[Ponto{0, 0}])                 // origem

k := map[[]int]bool{}
// ❌ invalid map key type []int
```

Usos clássicos de chaves compostas:

```go
visitado := map[[2]int]bool{}                 // coordenada (linha, coluna) num labirinto
rota := map[struct{ De, Para string }]int{}   // distância entre duas cidades
```

### 4.1 A curiosidade do NaN como chave

Lembra que `NaN != NaN` ([[Tipos de Dados]])? Saída real:

```go
m := map[float64]int{}
m[math.NaN()] = 1
m[math.NaN()] = 2
fmt.Println(len(m), m[math.NaN()])   // 2 0
```

Cada `NaN` inserido vira uma entrada nova (nunca é "igual" à anterior) e nunca mais pode ser lido nem removido com `delete`. Por isso o Go 1.21 criou `clear(m)`, que consegue remover essas entradas. Moral: **evite floats como chave**.

---

## 🎲 5. A ordem de iteração é aleatória

```go
for chave, valor := range idades {
	fmt.Println(chave, valor)
}
```

A especificação **não garante** ordem alguma, e o runtime **embaralha de propósito** o ponto de partida a cada `range`. Três iterações do mesmo map, saída real:

```
dabc abcd abcd
```

Por quê? No início de Go, a ordem parecia estável, e programas passaram a depender dela sem perceber. Quando a implementação mudava, esses programas quebravam. A aleatorização obriga todo mundo a não depender da ordem desde o primeiro dia.

Para iterar em ordem, **ordene as chaves** (saída real):

```go
m := map[string]int{"banana": 3, "abacate": 1, "caju": 2}
for _, k := range slices.Sorted(maps.Keys(m)) {     // Go 1.23+
	fmt.Print(k, "=", m[k], " ")
}
// abacate=1 banana=3 caju=2
```

### 5.1 Modificar o map durante o `range`

É permitido, com regras definidas pela especificação:

- **Remover** uma entrada ainda não visitada: ela não será visitada
- **Alterar** o valor de uma entrada: a versão nova é vista se ainda não foi visitada
- **Adicionar** uma entrada: ela **pode ou não** aparecer na iteração

```go
for k, v := range m {
	if v == 0 {
		delete(m, k)        // ✅ remover durante a iteração é seguro e comum
	}
}
maps.DeleteFunc(m, func(k string, v int) bool { return v == 0 })   // equivalente
```

---

## 🧩 6. Maps com valores compostos

### 6.1 Valores de struct não são endereçáveis

```go
m := map[string]struct{ N int }{"a": {1}}
m["a"].N = 2
// ❌ cannot assign to struct field m["a"].N in map
```

O valor dentro de um map **não tem endereço fixo**: quando o map cresce, os valores mudam de lugar na memória. Por isso Go proíbe pegar `&m[k]` ou alterar um campo diretamente. Duas soluções (saída real da primeira):

```go
// 1) Ler, alterar a cópia, escrever de volta
u := us["a"]
u.N = 2
us["a"] = u
fmt.Println(us)              // map[a:{2}]

// 2) Guardar ponteiros
usuarios := map[int]*Usuario{1: {Nome: "Ana"}}
usuarios[1].Idade = 31       // ✅ altera o objeto apontado (o ponteiro não muda de lugar)
```

### 6.2 Map de slices: agrupamento

Saída real:

```go
grupos := map[int][]string{}
for _, nome := range []string{"ana", "bia", "caio", "rui"} {
	grupos[len(nome)] = append(grupos[len(nome)], nome)
}
fmt.Println(grupos)          // map[3:[ana bia rui] 4:[caio]]
```

Funciona porque o zero value de `[]string` (nil) aceita `append`.

### 6.3 Map de maps

```go
notas := map[string]map[string]float64{}

// ⚠️ o map interno precisa ser criado antes de usar
if notas["ana"] == nil {
	notas["ana"] = map[string]float64{}
}
notas["ana"]["go"] = 9.5
```

Muitas vezes uma chave composta é mais simples: `map[[2]string]float64{{"ana", "go"}: 9.5}`.

---

## 🧪 7. Padrões comuns

### 7.1 Contagem

Saída real:

```go
contagem := map[string]int{}
for _, w := range strings.Fields("o gato e o rato e o cão") {
	contagem[w]++
}
fmt.Println(contagem["o"], contagem["e"])   // 3 2
```

### 7.2 Conjunto (set)

Go não tem um tipo "set". O idioma é um map com valor `struct{}`, que ocupa **zero bytes** (saída real: `unsafe.Sizeof(struct{}{})` = 0):

```go
set := map[string]struct{}{}
set["go"] = struct{}{}               // adicionar
_, tem := set["go"]                  // pertence? → true
delete(set, "go")                    // remover

vistos := map[int]bool{}             // alternativa mais legível, 1 byte por valor
if !vistos[x] { vistos[x] = true }
```

### 7.3 Cache / memoização

```go
var memo = map[int]int{}

func fib(n int) int {
	if n < 2 {
		return n
	}
	if v, ok := memo[n]; ok {
		return v
	}
	memo[n] = fib(n-1) + fib(n-2)
	return memo[n]
}
```

### 7.4 Despachar por nome

```go
comandos := map[string]func(args []string) error{
	"criar":   criar,
	"remover": remover,
}
if f, ok := comandos[nome]; ok {
	return f(args)
}
return fmt.Errorf("comando desconhecido: %s", nome)
```

---

## 📦 8. O pacote `maps` (Go 1.21+)

```go
import "maps"

c := maps.Clone(m)                  // cópia RASA (os valores são copiados; ponteiros continuam apontando para o mesmo)
maps.Copy(destino, origem)          // copia todos os pares (sobrescreve)
maps.Equal(m1, m2)                  // mesmo conteúdo?
maps.DeleteFunc(m, func(k string, v int) bool { return v < 0 })

// Iteradores (Go 1.23+)
for k, v := range maps.All(m) { }
chaves := slices.Collect(maps.Keys(m))     // em ordem aleatória
ordenadas := slices.Sorted(maps.Keys(m))   // em ordem
valores := slices.Collect(maps.Values(m))
novo := maps.Collect(seq)                  // cria um map a partir de um iterador
```

---

## 📨 9. Maps são passados "por referência"? (de novo: não exatamente)

Saída real:

```go
um := map[string]int{"a": 1}
f := func(mm map[string]int) { mm["b"] = 2 }
f(um)
fmt.Println(um)    // map[a:1 b:2]   ← a função alterou o map do chamador
```

Uma variável de map é, por baixo, **um único ponteiro** para a estrutura interna (`unsafe.Sizeof` de um map é 8 bytes, como vimos em [[Tipos de Dados]]). Passar o map copia o ponteiro, e as duas cópias apontam para a mesma tabela. Diferente do slice, aqui até o crescimento é visível para todos, porque o cabeçalho com tamanho e capacidade está **dentro** da estrutura apontada.

---

## 🔒 10. Maps e concorrência

> ⚠️ **Maps não são seguros para uso concorrente.** Se uma goroutine escreve enquanto outra lê ou escreve, o comportamento é indefinido. O runtime detecta muitos desses casos e aborta o programa com `fatal error: concurrent map writes` (que nem `recover` consegue capturar).

As soluções (estudadas no módulo de concorrência):

```go
// 1) Map + mutex: a solução padrão
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}
func (c *Cache) Get(k string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	v, ok := c.m[k]
	return v, ok
}

// 2) sync.Map: otimizado para casos específicos (muitas leituras, chaves estáveis)
var sm sync.Map
sm.Store("a", 1)
v, ok := sm.Load("a")
```

Veremos os detalhes em [[sync.WaitGroup e sync.Mutex]]. E o detector de corridas (`go test -race`) aponta esses acessos (ver [[Goroutines]]).

---

## 🔬 11. Por dentro: Swiss Tables (Go 1.24+)

Até o Go 1.23, o map usava uma tabela hash clássica com **buckets** de 8 entradas e **encadeamento** de buckets de overflow. No **Go 1.24** a implementação foi reescrita usando o design **Swiss Table** (criado no Google para C++, na biblioteca Abseil), com algumas adaptações exigidas pela especificação de Go.

### 11.1 Hash: dividido em duas partes

Cada chave passa por uma função hash de 64 bits (com uma **semente aleatória por map**, para que um atacante não consiga prever colisões). O hash é dividido:

```
hash (64 bits)
┌───────────────────────────────────────────────────────┬─────────┐
│                     H1 (57 bits)                      │ H2 (7b) │
└───────────────────────────────────────────────────────┴─────────┘
  escolhe ONDE começar a procurar                         "impressão digital"
                                                          guardada junto ao slot
```

### 11.2 Grupos de 8 slots + palavra de controle

A tabela é dividida em **grupos** de 8 slots. Cada grupo tem uma **palavra de controle** de 8 bytes, com **1 byte por slot**:

```
                 palavra de controle (8 bytes)
               ┌────┬────┬────┬────┬────┬────┬────┬────┐
 grupo i:      │ 3A │ vaz│ 15 │ 3A │ del│ 7F │ vaz│ 02 │   ← cada byte: H2 do slot,
               └────┴────┴────┴────┴────┴────┴────┴────┘     ou "vazio", ou "removido"
               ┌────┬────┬────┬────┬────┬────┬────┬────┐
 slots:        │k,v │    │k,v │k,v │    │k,v │    │k,v │   ← chaves e valores
               └────┴────┴────┴────┴────┴────┴────┴────┘
```

Para buscar uma chave:

1. Usa `H1` para escolher o grupo inicial
2. Compara o `H2` da chave com **os 8 bytes de controle de uma só vez** (com instruções **SIMD** quando disponíveis, ou truques aritméticos de bits)
3. Só nos slots cujo byte bateu, compara a chave inteira (o `H2` de 7 bits colide por acaso só ~1 vez em 128)
4. Se o grupo tem um slot vazio e a chave não estava lá, ela não existe; senão, segue para o próximo grupo (*sondagem*)

> 💡 **SIMD (Single Instruction, Multiple Data):** instruções da CPU que aplicam a mesma operação a vários dados de uma vez. Aqui, uma única instrução compara 8 (ou 16) bytes de controle ao mesmo tempo. É a mesma família de instruções que vimos no `math/bits` e nos floats em [[Operações Aritméticas]].

> 💡 **Endereçamento aberto:** em vez de pendurar listas encadeadas em cada posição (encadeamento), as entradas ficam no próprio array, e em caso de conflito procura-se a próxima posição livre. Swiss Tables são uma forma de endereçamento aberto. Isso dá melhor **localidade de cache** que o encadeamento.

Como 8 slots são verificados de uma vez, a tabela pode ficar bem mais cheia antes de crescer (fator de carga máximo de **7/8**), o que reduz o uso de memória.

### 11.3 Diretório de tabelas: crescimento sem pausas longas

Um map grande não é **uma** Swiss Table, e sim um **diretório** de várias tabelas, cada uma com no máximo **1024 entradas**. Os bits mais altos do hash escolhem a tabela (*hashing extensível*):

```
                 diretório (indexado pelos bits altos do hash)
               ┌──────┬──────┬──────┬──────┐
               │  00  │  01  │  10  │  11  │
               └──┬───┴──┬───┴──┬───┴──┬───┘
                  │      │      └──┬───┘        ← duas entradas podem apontar
                  ▼      ▼         ▼               para a mesma tabela
             ┌───────┐┌───────┐┌───────┐
             │tabela ││tabela ││tabela │   cada uma: até 1024 entradas
             │   A   ││   B   ││   C   │   em grupos de 8 slots
             └───────┘└───────┘└───────┘
```

Quando uma tabela enche, **só ela** é dividida em duas. O pior caso de uma inserção é copiar ~1024 entradas, e não o map inteiro. Isso é importante para servidores sensíveis a latência: nenhuma inserção sozinha causa uma pausa enorme.

Maps pequenos (até 8 entradas) são um caso especial: um único grupo, sem diretório.

### 11.4 Por que não copiar o design do C++ direto

Duas exigências da especificação de Go complicaram a adaptação:

1. **Modificar durante o `range`** é permitido em Go (em C++/Abseil não é). O iterador de Go guarda uma referência à tabela onde começou, usa-a só para decidir a **ordem**, e antes de devolver cada entrada confere na tabela atual se ela ainda existe e qual o valor mais recente
2. **Crescimento sem pausas longas**, resolvido com o diretório de tabelas

### 11.5 O resultado

Segundo o time do Go, operações de map ficaram **até 60% mais rápidas** em micro-benchmarks, com ganho médio de ~1,5% de CPU em aplicações completas.

> 💡 Você não precisa saber nada disso para usar maps. Mas explica três coisas que você observa de fora: por que **a ordem é imprevisível**, por que **valores não são endereçáveis** (mudam de lugar ao crescer) e por que maps **não são seguros para concorrência** (uma escrita pode estar reorganizando grupos enquanto outra goroutine lê).

---

## ⚠️ 12. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Escrever em map nil | `panic: assignment to entry in nil map` | `make` ou literal antes de usar |
| Confundir ausente com zero | `m[k] == 0` é ambíguo | `v, ok := m[k]` |
| Depender da ordem do `range` | Resultados mudam a cada execução | Ordene as chaves |
| Alterar campo de struct dentro do map | Erro de compilação | Reescreva o valor ou use ponteiros |
| Map interno não inicializado (map de maps) | panic ao escrever | Crie o interno antes, ou use chave composta |
| Floats como chave | NaN e problemas de precisão | Use outro tipo de chave |
| Acesso concorrente | `fatal error: concurrent map writes` | Mutex ou `sync.Map` |
| `maps.Clone` achando que é cópia profunda | Ponteiros compartilhados | Copie os valores manualmente se necessário |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Tipo | `map[K]V`; K precisa ser comparável |
| Criação | `make(map[K]V)`, `make(map[K]V, n)` ou literal |
| Map nil | Lê e itera; **escrever dá panic** |
| Chave ausente | Devolve o zero value |
| Vírgula ok | `v, ok := m[k]` distingue ausente de zero |
| `delete` / `clear` | Remove uma / todas as entradas |
| Iteração | Ordem aleatória de propósito; ordene as chaves se precisar |
| Valores | Não endereçáveis (`m[k].Campo = x` não compila) |
| Set | `map[T]struct{}` |
| Passagem | Variável é um ponteiro: alterações são vistas pelo chamador |
| Concorrência | Não é seguro; use mutex ou `sync.Map` |
| Implementação | Swiss Tables (1.24+): grupos de 8, controle de 1 byte por slot, SIMD, diretório de tabelas |

---

## 🔗 Conexões

**Dentro de Go**
- Tipos comparáveis → [[Operadores de Comparação e Lógicos]]
- Structs como chave e como valor → [[Structs]]
- Iteradores `maps.Keys`, `maps.All` → [[Iteradores]]
- Funções genéricas do pacote `maps` → [[Generics]]
- Acesso concorrente seguro → [[sync.WaitGroup e sync.Mutex]]
- Decodificar JSON dinâmico em `map[string]any` → [[JSON (marshal e unmarshal)]]

**Sistemas Operacionais**
- [[Memória]] — por que o endereçamento aberto (dados juntos no array) é mais amigável ao cache que listas encadeadas
- [[Race Condition]] e [[Regiões Críticas]] — o map é uma região crítica clássica quando compartilhado entre goroutines
- [[Processadores]] — instruções SIMD usadas na comparação dos bytes de controle

**Algoritmos e Estruturas de Dados**
- [[Hashmap]] — funções hash, colisões, encadeamento vs endereçamento aberto e fator de carga
- [[Big O Notation]] — O(1) médio vs O(n) no pior caso
- [[Grafos]] — listas de adjacência são naturalmente `map[No][]No`

**Nginx**
- [[Cache no Servidor (Cache Reverso)]] — um cache em memória é, na essência, um map protegido por mutex com política de expiração

---

⬅️ **Anterior:** [[Arrays e Slices]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Structs]] ➡️
