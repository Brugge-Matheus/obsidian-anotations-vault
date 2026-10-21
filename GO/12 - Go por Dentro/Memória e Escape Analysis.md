---
tags:
  - go
  - go/runtime
aliases:
  - Escape analysis
  - Stack vs heap
  - Alocador de memória do Go
  - mallocgc
  - Classes de tamanho
  - RSS
ordem: 49
modulo: "12 - Go por Dentro"
---
# Memória e Escape Analysis

> **Nota 49 da trilha de Go.** Abrimos o último módulo, **Go por dentro**, que conecta a linguagem diretamente ao capítulo 3 do Tanenbaum (gerenciamento de memória). A pergunta desta nota: **onde mora cada valor** de um programa Go, e quanto custa? Veremos as três regiões (stack, heap, dados globais), a **escape analysis** que o compilador faz para decidir entre stack e heap (com dez casos reais), como medir alocações, o **alocador** do runtime (classes de tamanho, caches por P, spans, arenas, o *tiny allocator*), a relação com a memória virtual do SO (por que o RSS não cai depois do GC) e as técnicas para alocar menos.
>
> **Pré-requisitos:** [[Ponteiros]], [[Variáveis e constantes]], [[Benchmarks]].

---

## 🧭 De onde viemos

Em [[Variáveis e constantes]] vimos que o compilador decide se uma variável fica na stack ou na heap, e em [[Ponteiros]] demos a regra simplificada ("se um ponteiro pode sobreviver à função, vai para a heap") e o primeiro `moved to heap`. [[Como Go Funciona]] mostrou o alocador "inspirado no TCMalloc" e as seções do binário; [[Goroutines]], a stack de 2 KB que cresce; [[Structs]], alinhamento e *padding*; [[Arrays e Slices]], a pré-alocação; [[Benchmarks]] e [[Profiling e Diagnóstico]], como medir `B/op`, `allocs/op` e quem aloca. No Tanenbaum, você estudou [[Espaços de Endereçamento]], [[Memória Virtual]], [[Paginação]], [[Tabelas de Páginas]] e [[Gerenciando a Memória Livre]]: o runtime de Go é um gerenciador de memória **em espaço de usuário**, construído sobre tudo isso.

> 💡 **Alocação:** reservar um pedaço de memória para guardar um valor. Na **stack**, é quase de graça (mover um registrador); na **heap**, envolve o alocador e, mais tarde, o coletor de lixo.

---

## 🗺️ 1. As três regiões

```
     espaço de endereçamento virtual de um processo Go
 ┌──────────────────────────────────────────────────────────┐
 │ .text / .rodata      código e constantes (ver Como Go Funciona)
 │ .data / .bss         VARIÁVEIS GLOBAIS: existem o programa inteiro
 │ ...
 │ ARENAS DA HEAP       objetos que sobrevivem às funções
 │   (blocos de 64 MB   ← alocador do Go (mallocgc)
 │    pedidos com mmap) ← liberados pelo GC
 │ ...
 │ STACKS DAS GOROUTINES  uma por goroutine, alocadas pelo runtime
 │   [G1: 2 KB] [G2: 8 KB] [G3: 2 KB] ...   (também ficam em memória do runtime)
 │ ...
 │ stack da thread principal [stack] (usada pelo runtime, não pelo seu código)
 └──────────────────────────────────────────────────────────┘
```

| | Stack (da goroutine) | Heap |
| --- | --- | --- |
| Vida útil | Até a função retornar | Enquanto alguém apontar para o valor |
| Custo de alocar | ~0 (o frame já tem o espaço) | `mallocgc`: dezenas de ns |
| Custo de liberar | 0 (o frame some no `return`) | Trabalho do GC (marcar, varrer) |
| Acesso | Muito local (cache quente) | Pode estar em qualquer lugar |
| Quem decide | O **compilador** (escape analysis) | idem |

Diferente de C (onde `malloc` é explícito) e de Java (onde quase todo objeto vai para a heap), em Go **você não escolhe**: o compilador põe na stack tudo que pode **provar** que não sobrevive à função, e manda o resto para a heap.

---

## 🔍 2. Escape analysis

> 💡 **Escape analysis (análise de escape):** análise estática em que o compilador segue o fluxo de cada endereço (`&x`, closures, conversões) para descobrir se ele pode ser usado **depois** de a função retornar ou **por outra goroutine**. Se pode, a variável "escapa" e precisa ir para a heap.

### 2.1 Dez casos reais

Saída real de `go build -gcflags='-m'` (Go 1.27) para dez funções:

```go
type Ponto struct{ X, Y int }
var global *Ponto

func porValor() Ponto { p := Ponto{1, 2}; return p }                // 1
func porPonteiro() *Ponto { p := Ponto{1, 2}; return &p }            // 2
func guardarGlobal() { p := Ponto{3, 4}; global = &p }               // 3
func soLeitura(p *Ponto) int { return p.X + p.Y }                     // 4
func usarLocal() int { p := Ponto{5, 6}; return soLeitura(&p) }
func contador() func() int { n := 0; return func() int { n++; return n } } // 5
func imprimir() { x := 42; fmt.Println(x) }                           // 6
func sliceConstante() int { s := make([]int, 8); return len(s) }      // 7
func sliceVariavel(n int) int { s := make([]int, n); return len(s) }
func arrayGrande() int { var a [200_000]int; return len(a) }          // 8 (1,6 MB)
func enviar(ch chan *Ponto) { p := Ponto{7, 8}; ch <- &p }            // 9
func goroutine() { p := Ponto{9, 9}; go func() { _ = p.X }() }        // 10
```

```
./main.go:10:29: moved to heap: p                      ← 2  porPonteiro
./main.go:11:24: moved to heap: p                      ← 3  guardarGlobal
./main.go:12:16: p does not escape                     ← 4  soLeitura: só lê
./main.go:15:2:  moved to heap: n                      ← 5  contador (closure)
./main.go:16:9:  func literal escapes to heap
./main.go:18:40: 42 escapes to heap                    ← 6  fmt.Println
./main.go:19:38: make([]int, 8) does not escape        ← 7
./main.go:20:42: make([]int, n) does not escape
./main.go:21:30: moved to heap: a                      ← 8  arrayGrande
./main.go:22:31: moved to heap: p                      ← 9  enviar por channel
./main.go:23:41: func literal escapes to heap          ← 10 goroutine
```

| # | Caso | Resultado | Por quê |
| --- | --- | --- | --- |
| 1 | Devolver por **valor** | Stack | O valor é **copiado** para o chamador |
| 2 | Devolver **ponteiro** para local | Heap | O endereço sobrevive ao `return` |
| 3 | Guardar o endereço numa **global** | Heap | A global vive para sempre |
| 4 | Passar `&p` para quem só **lê** | Stack | O compilador analisou `soLeitura`: o ponteiro não é guardado |
| 5 | **Closure** que modifica variável capturada | Heap (`n` e a closure) | A closure é devolvida; `n` precisa viver com ela |
| 6 | Passar para `fmt.Println` (`...any`) | Heap (o `42`) | Converter para **interface** num parâmetro que escapa (o `fmt` guarda os argumentos) |
| 7 | `make` com tamanho **constante** pequeno | Stack | Tamanho conhecido |
| 7b | `make` com tamanho **variável** | "Não escapa" | Desde o Go 1.25, um buffer de 32 bytes na stack; só vai para a heap se não couber (seção 3) |
| 8 | Variável enorme (> 128 KB) | Heap | Limite de tamanho de variáveis na stack |
| 9 | Enviar **ponteiro** por channel | Heap | Outra goroutine vai usá-lo (ver [[Channels]]) |
| 10 | **Goroutine** que captura variável | Heap (a closure) | A goroutine pode viver mais que a função; repare que `p` foi **copiado** para a closure (não é modificado), então ele mesmo não precisou escapar |

`moved to heap: x` significa que a **variável** foi para a heap; `... escapes to heap` significa que um **valor** (um literal, uma closure, uma conversão) foi alocado na heap. Para ver o **motivo** de cada decisão, use `-gcflags='-m=2'`.

### 2.2 Inlining muda o resultado

Uma função construtora que devolve ponteiro (caso 2) **escapa por definição**. Mas se ela for **inlinada** (ver [[Funções — Fundamentos]]), o corpo é copiado para dentro do chamador, e a análise passa a ver o uso real. Saída real:

```go
func NovoPonto(x, y int) *Ponto { return &Ponto{x, y} } // pequena: pode ser inlinada

//go:noinline
func NovoPontoSemInline(x, y int) *Ponto { return &Ponto{x, y} }
```

```
./main.go:10:42: &Ponto{...} escapes to heap         ← dentro da função isolada
./main.go:17:17: inlining call to NovoPonto
./main.go:17:17: &Ponto{...} does not escape          ← no chamador, depois do inlining

allocs com inlining: 0 | sem inlining: 1
```

O idiomático `NovoX(...) *X` não é necessariamente caro: se a função for pequena e o ponteiro não sair do chamador, a alocação **desaparece**.

---

## 📏 3. Medindo alocações

Saída real com `testing.AllocsPerRun` (funções com `//go:noinline` para isolar o efeito):

```
allocs por chamada:
  porValor():           0
  porPonteiro():        1
  sliceVariavel(4):     0 (32 bytes: cabe no buffer da stack)
  sliceVariavel(5):     1 (40 bytes: não cabe)
```

Ferramentas, da mais pontual à mais ampla:

| Ferramenta | Mostra |
| --- | --- |
| `go build -gcflags='-m'` | **Decisões** do compilador, por linha |
| `testing.AllocsPerRun`, `b.ReportAllocs()` | **Quantas** alocações uma operação faz (ver [[Benchmarks]]) |
| Profile `allocs` | **Onde**, no programa todo, a memória é alocada (ver [[Profiling e Diagnóstico]]) |
| `runtime.MemStats`, `runtime/metrics` | Totais do processo |

---

## 🏗️ 4. O alocador do runtime

### 4.1 Classes de tamanho

O alocador não atende pedidos "do tamanho exato": arredonda para uma de **67 classes de tamanho** (8, 16, 24, 32, 48, 64, 80... até 32 768 bytes). Saída real, medindo quantos bytes cada `make([]byte, n)` realmente reservou:

```
bytes realmente reservados por make([]byte, n):
  n=1      → 1        ← tiny allocator (seção 4.3)
  n=9      → 16
  n=17     → 24
  n=33     → 48       ← 15 bytes desperdiçados
  n=100    → 112
  n=1000   → 1024
  n=5000   → 5378     ← classe de 5376 (+ ruído da medição)
  n=33000  → 40960    ← "objeto grande": arredondado para páginas de 8 KB (5 páginas)
```

É a estratégia de **listas segregadas por tamanho** (*segregated fits*): cada classe tem seus próprios blocos de tamanho fixo. Comparando com [[Gerenciando a Memória Livre]]:

| Gerenciador clássico (Tanenbaum) | Alocador do Go |
| --- | --- |
| Lista de buracos com *first fit*, *best fit* | Uma lista por classe: achar um bloco livre é imediato |
| **Fragmentação externa** (buracos pequenos entre blocos) | Praticamente eliminada: dentro de uma página, todos os blocos têm o mesmo tamanho |
| Juntar buracos adjacentes ao liberar | Desnecessário |
| | Em troca: **fragmentação interna** (o arredondamento; até ~12% nas classes médias) |

### 4.2 A hierarquia: do P ao kernel

```
 goroutine chama mallocgc(48 bytes)
      │
      ▼
 ┌───────────────┐  mcache: um por P (ver Scheduler do Go), SEM trava
 │ mcache do P   │  tem um "span" aberto para cada classe de tamanho
 └──────┬────────┘  → o caso comum: pegar o próximo bloco livre do span
        │ span da classe acabou
        ▼
 ┌───────────────┐  mcentral: um por classe de tamanho, COM trava
 │ mcentral[48B] │  lista de spans com blocos livres
 └──────┬────────┘
        │ nenhum span disponível
        ▼
 ┌───────────────┐  mheap: gerencia PÁGINAS de 8 KB
 │    mheap      │  monta um span novo (1 ou mais páginas)
 └──────┬────────┘
        │ sem páginas livres
        ▼
 ┌───────────────┐  pede uma ARENA de 64 MB ao kernel (mmap, memória virtual)
 │    kernel     │
 └───────────────┘
```

> 💡 **Span:** sequência contígua de páginas de 8 KB dividida em blocos de uma única classe de tamanho. Um span da classe de 48 bytes, com 1 página, tem 170 blocos (e 32 bytes de sobra no fim).

O caminho rápido (pegar um bloco do `mcache`) não usa nenhuma trava, porque cada P tem o seu: é o mesmo truque de dados "por CPU" que o kernel Linux usa. O desenho vem do **TCMalloc** (*Thread-Caching Malloc*) do Google.

### 4.3 Casos especiais

| Pedido | Tratamento |
| --- | --- |
| < 16 bytes **sem ponteiros** | **Tiny allocator**: vários objetos minúsculos dividem um bloco de 16 bytes (por isso `make([]byte, 1)` custou **1 byte** em média na saída acima) |
| ≤ 32 KB | Classes de tamanho (caminho de 4.2) |
| > 32 KB | "Objeto grande": páginas inteiras direto do `mheap` |
| Tamanho 0 | Todos apontam para o mesmo endereço (`runtime.zerobase`): `struct{}` não custa nada |

Os spans também são separados em **com ponteiros** (*scan*) e **sem ponteiros** (*noscan*): um `[]byte` vai para um span *noscan*, que o GC nem precisa olhar por dentro. É o mesmo motivo das seções `.data` vs `.noptrdata` que vimos em [[Como Go Funciona]].

> 💡 **Go 1.27:** o compilador passou a chamar rotinas de alocação **especializadas por tamanho** para objetos pequenos (< 80 bytes), até 30% mais baratas cada, com ganho de ~1% em programas que alocam muito.

---

## 🖥️ 5. A memória do processo vista pelo SO

### 5.1 Virtual não é física

O runtime reserva arenas com `mmap`, mas isso é só **memória virtual**: uma página só ganha um quadro de memória física quando é **tocada** pela primeira vez (falta de página, ver [[Paginação]] e [[Tabelas de Páginas]]). Por isso há vários números de "memória usada":

| Número | O que mede | De onde vem |
| --- | --- | --- |
| `HeapAlloc` / `HeapInuse` | Objetos vivos / spans em uso | `runtime.MemStats` |
| `Sys` | Tudo que o runtime obteve do SO (virtual) | `runtime.MemStats` |
| `HeapReleased` | Memória devolvida ao SO (mas ainda reservada) | `runtime.MemStats` |
| **RSS** (*Resident Set Size*) | Páginas **físicas** de fato ocupadas | `/proc/<pid>/status` (`VmRSS`), `top`, o limite do container |

### 5.2 Por que o RSS não cai depois do GC

Saída real: alocar e tocar 100 MB, soltar tudo, coletar:

```
RSS inicial: 2252 kB
após alocar 100 MB: RSS 105696 kB | HeapInuse 100 MB | Sys 111 MB
após GC:            RSS 105804 kB | HeapInuse 0 MB   | HeapReleased 6 MB    ← ainda 105 MB!
após FreeOSMemory:  RSS 3052 kB   | HeapReleased 107 MB
```

Depois do GC, os objetos estão mortos (`HeapInuse 0`), mas as páginas continuam **com o processo**: o runtime as mantém para reutilizar, porque devolver e pedir de novo custa caro (syscalls e faltas de página). Quem devolve é o **scavenger**, uma goroutine de fundo que, aos poucos, chama `madvise(MADV_DONTNEED)` nas páginas livres há mais tempo, liberando os quadros físicos sem desfazer a reserva virtual. `debug.FreeOSMemory()` força um GC e devolve tudo de uma vez (raramente necessário em produção).

> ⚠️ Em containers, quem mata o processo por falta de memória (o *OOM killer* do cgroup) olha o **RSS**, não o `HeapInuse`. O controle certo é o `GOMEMLIMIT`, que faz o GC e o scavenger trabalharem para manter a memória total abaixo de um teto (ver [[Garbage Collector]] e [[Cgroups]]).

> 💡 Desde o **Go 1.25**, no Linux, o runtime **nomeia** suas regiões de memória anônima (`[anon: Go: heap]`, `[anon: Go: stack]`...) em `/proc/<pid>/maps`, facilitando ferramentas de diagnóstico. Isso exige um kernel compilado com `CONFIG_ANON_VMA_NAME`; o kernel da máquina destes experimentos não tem a opção, então os nomes não apareceram nela.

---

## 🧰 6. Alocando menos

Só depois de um profile mostrar que alocação é o problema (ver [[Profiling e Diagnóstico]]):

| Técnica | Exemplo | Nota |
| --- | --- | --- |
| Devolver valores pequenos, não ponteiros | `func f() Ponto` | Seção 2 |
| Pré-alocar a capacidade | `make([]T, 0, n)`, `strings.Builder.Grow`, `make(map[K]V, n)` | [[Arrays e Slices]], [[Maps]] |
| Reutilizar buffers | `buf = buf[:0]`; `sync.Pool` para temporários | [[sync.WaitGroup e sync.Mutex]] |
| Evitar `any` em caminhos quentes | `strconv.AppendInt` em vez de `fmt.Sprintf` | [[Interfaces]] |
| Passar `[]byte` de destino | APIs `AppendXxx(dst, ...)` da stdlib | [[Strings em Profundidade]] |
| Evitar closures que capturam por referência em laços quentes | Parâmetros explícitos | [[Closures e Funções Anônimas]] |
| Ordenar campos de structs | Menos *padding* = objetos menores, classe de tamanho menor | [[Structs]] |
| Manter funções construtoras pequenas | Inlining pode eliminar a alocação | Seção 2.2 |

> ⚠️ Não transforme o código num campo minado de otimizações sem medir. Na maioria dos programas, a alocação não é o gargalo, e código claro vale mais.

---

## 🔬 7. Por dentro

### 7.1 Como a análise funciona

O compilador monta um **grafo de fluxo de endereços**: cada variável é um nó, e cada atribuição de endereço (`q = &p`, `return &p`, `ch <- &p`, captura numa closure, conversão para interface) é uma aresta. Alguns nós especiais representam "a heap" (globais, parâmetros de funções desconhecidas, outras goroutines). Se existe um caminho de `&p` até "a heap", ou se um ponteiro para `p` pode viver mais que o frame de `p`, então `p` escapa. A análise é feita **por pacote** e usa resumos de cada função ("o parâmetro 1 escapa para o resultado"), o que permitiu ver que `soLeitura` não guardava o ponteiro.

Ela é **conservadora**: na dúvida (uma chamada via interface cujo tipo concreto é desconhecido, por exemplo), assume que escapa. Por isso a desvirtualização (inclusive por PGO, ver [[Profiling e Diagnóstico]]) pode reduzir alocações.

### 7.2 O caminho de `mallocgc`

```
mallocgc(tamanho, tipo, zerar)
  ├─ tamanho 0 → &zerobase
  ├─ < 16 B e sem ponteiros → tiny allocator
  ├─ ≤ 32 KB → classe de tamanho → mcache do P → (mcentral → mheap → mmap)
  ├─ > 32 KB → páginas direto do mheap
  ├─ se o GC está marcando: a goroutine pode ser obrigada a AJUDAR (assist),
  │  proporcional ao quanto aloca (ver Garbage Collector)
  └─ amostragem para o profile de memória (1 a cada ~512 KB)
```

### 7.3 Paralelo com o SO

| SO (Tanenbaum, cap. 3) | Runtime de Go |
| --- | --- |
| Espaço de endereçamento do processo ([[Espaços de Endereçamento]]) | Arenas da heap + stacks das goroutines dentro dele |
| `brk`/`mmap` para aumentar a memória do processo | `mmap` de arenas de 64 MB |
| Gerência de memória livre com bitmaps e listas ([[Gerenciando a Memória Livre]]) | Bitmaps de alocação por span + listas por classe de tamanho |
| Paginação e faltas de página ([[Paginação]], [[Tabelas de Páginas]]) | Páginas da arena viram físicas no primeiro toque |
| Devolver memória ao SO | Scavenger com `madvise` |
| Pilha do processo de tamanho fixo ([[Processos]]) | Stacks de goroutine que crescem copiando (ver [[Goroutines]]) |
| Alocador de objetos do kernel (slab) | `mcache`/`mcentral` por classe; `sync.Pool` |

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| "Ponteiro é sempre mais rápido" | Mais alocações, mais GC | Valores para structs pequenas; meça |
| Achar que `new`/`&T{}` sempre vai para a heap | Otimizações desnecessárias | O compilador decide; veja `-gcflags=-m` |
| `fmt.Sprintf` em laços quentes | Alocações por conversão para `any` | `strconv`, `Append*` |
| Slice pequeno de tamanho variável assumido na heap | — | Até 32 bytes fica na stack (1.25+) |
| Medir memória pelo `top` logo após o GC | "Vazamento" que não existe | Olhe `HeapInuse`; espere o scavenger |
| Container morto por OOM com heap pequeno | RSS alto (memória retida, stacks, fragmentação) | `GOMEMLIMIT` abaixo do limite do cgroup |
| Guardar um subslice de um slice enorme | O array inteiro continua vivo | `slices.Clone` do pedaço necessário |
| Otimizar alocação sem profile | Código pior, ganho nulo | Profile primeiro |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Regiões | Stack (por goroutine), heap (GC), globais |
| Escape analysis | O compilador prova o que pode ficar na stack; na dúvida, heap |
| Escapa quando | Ponteiro devolvido, guardado em global/heap, enviado por channel, capturado por closure/goroutine, convertido para interface que escapa, grande demais |
| Inlining | Pode eliminar alocações de construtores |
| `-gcflags=-m` | Mostra as decisões; `-m=2` mostra o motivo |
| Alocador | 67 classes; `mcache` (por P, sem trava) → `mcentral` → `mheap` (páginas de 8 KB) → arenas de 64 MB |
| Tiny / grandes | < 16 B sem ponteiros compartilham blocos; > 32 KB em páginas inteiras |
| Fragmentação | Interna (arredondamento) em troca de quase nenhuma externa |
| RSS | Não cai logo após o GC; scavenger devolve aos poucos; `GOMEMLIMIT` para containers |

---

## 🔗 Conexões

**Dentro de Go**
- A regra simplificada e o primeiro `moved to heap` → [[Ponteiros]], [[Variáveis e constantes]]
- Quem libera a heap → [[Garbage Collector]]
- O P de cada `mcache` → [[Scheduler do Go]]
- Stacks que crescem → [[Goroutines]]
- Inlining → [[Funções — Fundamentos]]
- Interfaces e conversões que alocam → [[Interfaces]]
- Pré-alocação e subslices → [[Arrays e Slices]], [[Maps]]
- Alinhamento e padding → [[Structs]]
- `sync.Pool` → [[sync.WaitGroup e sync.Mutex]]
- Medir → [[Benchmarks]], [[Profiling e Diagnóstico]]
- Seções do binário e o runtime → [[Como Go Funciona]]
- Ponteiros que o GC não vê → [[unsafe e cgo]]

**Sistemas Operacionais**
- [[Espaços de Endereçamento]] e [[Espaços de Endereçamento e Registradores-Base e Limite]] — o espaço onde tudo isso vive
- [[Memória Virtual]], [[Paginação]] e [[Tabelas de Páginas]] — reserva virtual vs páginas físicas
- [[Gerenciando a Memória Livre]] — bitmaps, listas, fragmentação
- [[Troca de Processos (Swapping)]] — o que acontece quando a memória física acaba
- [[Memória]] — hierarquia e por que localidade importa
- [[Processos]] — a pilha do processo

**Docker**
- [[Cgroups]] — o limite de memória que mede RSS

---

⬅️ **Anterior:** [[Build, Flags e Cross-compilation]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Garbage Collector]] ➡️
