---
tags:
  - go
  - go/runtime
aliases:
  - GC
  - Coletor de lixo
  - GOGC
  - GOMEMLIMIT
  - Green Tea
  - Mark and sweep
  - Weak pointers
ordem: 50
modulo: "12 - Go por Dentro"
---
# Garbage Collector

> **Nota 50 da trilha de Go.** Na nota anterior vimos **onde** a memória é alocada; agora, **como ela volta**. Go tem um **coletor de lixo** (*garbage collector*, GC) que descobre sozinho quais objetos não podem mais ser usados e libera a memória deles, **enquanto o programa continua rodando**, com pausas de microssegundos. Nesta nota veremos as alternativas (gerência manual, contagem de referências, rastreamento), o algoritmo **tricolor de marcação e varredura**, como ele roda **concorrentemente** graças à **barreira de escrita**, as fases e suas pausas, o controle de ritmo com **`GOGC`** e **`GOMEMLIMIT`** (com medições reais do trade-off memória × CPU), a leitura do `gctrace`, o novo coletor **Green Tea** (padrão desde o Go 1.26), e `AddCleanup` e **ponteiros fracos**.
>
> **Pré-requisitos:** [[Memória e Escape Analysis]], [[Ponteiros]], [[Profiling e Diagnóstico]].

---

## 🧭 De onde viemos

Em [[Ponteiros]] vimos a ideia central: o GC descobre o que está vivo **seguindo ponteiros a partir das raízes**. Em [[Memória e Escape Analysis]] vimos que só a **heap** precisa de coleta (a stack se libera sozinha), que spans são separados em "com ponteiros" e "sem ponteiros", e que o RSS não cai logo depois do GC. Em [[Profiling e Diagnóstico]] lemos uma linha do `gctrace` sem explicá-la, e em [[sync.WaitGroup e sync.Mutex]] vimos que o `sync.Pool` é esvaziado nas coletas. Do lado do SO, [[A Linguagem C]] mostrou a gerência **manual** (`malloc`/`free`) e seus bugs, e [[Paravirtualização e a Máquina Virtual Java]] apresentou a JVM, outro runtime com GC.

> 💡 **Garbage collector (coletor de lixo):** componente do runtime que identifica objetos da heap que **não são mais alcançáveis** pelo programa e devolve a memória deles ao alocador, sem que o programador precise liberar nada explicitamente.

---

## ⚖️ 1. Três formas de liberar memória

| Estratégia | Exemplo | Vantagens | Problemas |
| --- | --- | --- | --- |
| **Manual** | C (`free`), C++ (`delete`) | Controle total, custo previsível | *Use-after-free*, *double free*, vazamentos: a maior fonte de falhas de segurança em C |
| **Contagem de referências** | Python, Swift, `shared_ptr` do C++ | Libera assim que o último dono solta | Custo em toda cópia de ponteiro (atômico em código concorrente); **ciclos** nunca chegam a zero |
| **Rastreamento** (*tracing*) | Java, C#, JavaScript, **Go** | Lida com ciclos; nenhum custo ao copiar ponteiros | Precisa percorrer o grafo de objetos; pausas e uso de CPU |

```
contagem de referências e um ciclo:          rastreamento:
                                              raízes → ninguém alcança A nem B
   ┌───┐ ref=1 ┌───┐                          → os dois são lixo, ciclo ou não
   │ A │──────▶│ B │
   └───┘◀──────└───┘ ref=1
   ninguém de fora aponta, mas os contadores nunca zeram: vazamento
```

O GC de Go é, nas palavras do próprio código-fonte (`runtime/mgc.go`): **concorrente, preciso** (sabe exatamente quais palavras são ponteiros), **paralelo**, de **marcação e varredura**, com **barreira de escrita**, **não geracional** e **não compactador**.

---

## 🎨 2. Marcação tricolor

O algoritmo pinta cada objeto de uma de três cores:

| Cor | Significado |
| --- | --- |
| ⚪ **Branco** | Ainda não visto. No fim, branco = lixo |
| 🔘 **Cinza** | Visto (é alcançável), mas seus ponteiros **ainda não** foram examinados |
| ⚫ **Preto** | Visto, e todos os seus ponteiros já foram examinados |

```
1. todos brancos; as RAÍZES (globais, stacks das goroutines, registradores) ficam cinza
2. enquanto houver cinza:
      pegue um cinza → pinte de cinza tudo o que ele aponta e ainda é branco
                    → pinte-o de preto
3. sobrou branco? é inalcançável: LIXO

  raízes          A ⚫ ──▶ B ⚫ ──▶ C ⚫
     │
     └──▶ D ⚫ ──▶ E ⚫          F ⚪ ──▶ G ⚪     ← ninguém alcança F e G: varridos
```

A **varredura** (*sweep*) depois percorre os spans e trata cada bloco branco como livre. Em Go, isso é barato: cada span tem um bitmap de marcação, e "liberar" é basicamente adotar o bitmap de marcação como o novo bitmap de alocação.

---

## 🏃 3. Coletando com o programa rodando

### 3.1 O perigo

Se o programa (o *mutator*) continua rodando durante a marcação, ele pode **mudar ponteiros** debaixo do coletor:

```
estado durante a marcação:   A ⚫  (já examinado)        B 🔘 ──▶ C ⚪

o programa faz:   A.campo = C     (guarda C num objeto PRETO)
                  B.campo = nil   (apaga o único caminho cinza até C)

o coletor nunca mais olha A (é preto), e B não aponta mais para C:
C continua branco → seria LIBERADO, mas A ainda o usa → use-after-free!
```

### 3.2 A barreira de escrita

A solução é a **barreira de escrita** (*write barrier*): enquanto o GC está marcando, **toda escrita de ponteiro na heap** passa por um pequeno trecho de código que avisa o coletor. A de Go (híbrida, desde o Go 1.8, combinando as ideias de Dijkstra e Yuasa) pinta de cinza **tanto o ponteiro antigo quanto o novo**. No exemplo, ao fazer `A.campo = C`, o C vira cinza e será examinado. Além disso, objetos **alocados durante a marcação** já nascem pretos.

```go
obj.campo = ptr
// durante a marcação, o compilador transforma em algo como:
if writeBarrier.enabled {
	gcWriteBarrier(&obj.campo, ptr)    // registra o antigo e o novo num buffer do P
}
obj.campo = ptr
```

Fora da marcação, o custo é só testar uma flag. É por isso que o GC de Go **não precisa parar o mundo** para marcar.

### 3.3 As fases

```
tempo ──────────────────────────────────────────────────────────────────────▶

 ┃STW┃ ═══════════ marcação CONCORRENTE ══════════ ┃STW┃ ═════ varredura concorrente ═════
  (1)              (2) barreira ligada               (3)       (4) barreira desligada
  ↑                 25% da CPU em workers do GC      ↑         spans varridos sob demanda
  ~10–100 µs        + "assists" de quem aloca        ~10–100 µs  e por uma goroutine de fundo
```

| Fase | O que acontece | Para o programa? |
| --- | --- | --- |
| 1. Fim da varredura anterior | Liga a barreira de escrita em todos os Ps | **Sim**, brevemente |
| 2. Marcação | Workers do GC (cerca de **25%** dos Ps) percorrem raízes e heap; goroutines que alocam muito são obrigadas a **ajudar** (*assist*) | Não |
| 3. Fim da marcação | Esvazia os caches, desliga workers e barreira | **Sim**, brevemente |
| 4. Varredura | Libera os brancos, span a span, conforme a demanda | Não |

> 💡 **Assist (ajuda forçada):** se uma goroutine aloca mais rápido do que o GC consegue marcar, ela é obrigada a fazer um pouco de marcação, proporcional ao que alocou, antes de receber a memória. Isso impede que o programa "fuja" do coletor, e aparece como latência extra justamente nas goroutines que mais alocam.

---

## 🧬 4. Escolhas de projeto: sem compactação, sem gerações

| Característica | Muitos GCs (JVM, .NET) | Go | Por que Go escolheu assim |
| --- | --- | --- | --- |
| **Compactação** (mover objetos para juntar espaço livre) | Sim | **Não** | Ponteiros podem ir para código C (cgo) e para dentro de objetos (ponteiros internos); o alocador por classes já evita fragmentação externa |
| **Gerações** (coletar os objetos jovens com mais frequência) | Sim | **Não** | A escape analysis já mantém muitos objetos de vida curta na **stack**, onde não custam nada ao GC; gerações exigiriam barreiras de escrita **sempre ligadas** |
| Objetivo principal | Vazão | **Latência baixa** (pausas de µs) | Servidores |

A consequência prática: em Go, a melhor forma de reduzir o custo do GC é **alocar menos** na heap (ver [[Memória e Escape Analysis]]).

---

## 🎚️ 5. Quando coletar: `GOGC` e `GOMEMLIMIT`

### 5.1 `GOGC`: a meta proporcional

O próximo ciclo começa quando a heap cresce uma certa **proporção** além do que estava vivo depois do último ciclo:

```
meta da heap = vivo + (vivo + stacks + globais) × GOGC / 100

GOGC=100 (padrão), 50 MB vivos  →  próximo GC perto de 100 MB
GOGC=200                        →  perto de 150 MB  (menos ciclos, mais memória)
GOGC=50                         →  perto de 75 MB   (mais ciclos, menos memória)
GOGC=off                        →  nunca (a menos que haja GOMEMLIMIT)
```

Medição real: um programa com **~50 MB vivos** (árvores com milhões de ponteiros) que gera muito lixo, rodado com vários `GOGC`:

```
GOGC=25   GCs: 25   | CPU do GC: 61.6% | tempo: 364ms | heap máx (HeapSys):  75 MB
GOGC=50   GCs: 13   | CPU do GC: 46.0% | tempo: 238ms | heap máx (HeapSys):  95 MB
GOGC=100  GCs: 6    | CPU do GC: 34.4% | tempo: 177ms | heap máx (HeapSys): 127 MB
GOGC=200  GCs: 3    | CPU do GC: 27.0% | tempo: 162ms | heap máx (HeapSys): 171 MB
GOGC=400  GCs: 2    | CPU do GC: 23.5% | tempo: 164ms | heap máx (HeapSys): 163 MB
GOGC=off  GCs: 0    | CPU do GC:   —   | tempo: 161ms | heap máx (HeapSys): 199 MB
```

É o trade-off fundamental, com números: **memória ↔ CPU**. Com `GOGC=25` o heap fica pequeno (75 MB), mas o GC consome 62% da CPU e o programa leva o dobro do tempo; com `GOGC=off`, nenhum ciclo, mas o heap só cresce. (A "CPU do GC" aqui é de um programa feito para estressar o coletor; serviços típicos ficam na casa de poucos por cento.)

### 5.2 `GOMEMLIMIT` (Go 1.19+): um teto de memória

`GOGC` não sabe quanta memória a máquina (ou o container) tem. `GOMEMLIMIT` define um **limite suave** para a memória total do runtime: ao se aproximar dele, o GC passa a coletar com mais frequência, independentemente do `GOGC`. Saída real (mesmo programa):

```
GOGC=off GOMEMLIMIT=150MiB  GCs: 1    | CPU do GC: 13.3% | tempo: 175ms | heap máx: 151 MB
GOGC=100 GOMEMLIMIT=80MiB   GCs: 13   | CPU do GC: 56.2% | tempo: 402ms | heap máx:  75 MB
```

| Combinação | Comportamento |
| --- | --- |
| `GOGC=off` + `GOMEMLIMIT=X` | "Use a memória que quiser até X, só então colete": máxima eficiência quando a memória está sobrando (1 ciclo só, acima) |
| `GOGC=100` + `GOMEMLIMIT` um pouco abaixo do limite do container | **A recomendação para containers**: comportamento normal, mas o GC aperta antes do *OOM killer* agir (ver [[Cgroups]]) |
| Limite **perto** do tamanho dos dados vivos | O GC roda sem parar (13 ciclos, 56% de CPU, o dobro do tempo, acima) |

Para não entrar numa **espiral de morte** (o GC rodando 100% do tempo tentando respeitar um limite impossível), o runtime limita o GC a cerca de **50% da CPU** quando é o limite de memória que o está empurrando; a partir daí, deixa a memória passar do teto.

```go
debug.SetGCPercent(200)            // = GOGC=200, em tempo de execução
debug.SetMemoryLimit(900 << 20)    // = GOMEMLIMIT=900MiB
```

> 💡 Além disso, um ciclo é **forçado** a cada 2 minutos se nenhum acontecer (para o scavenger devolver memória e os cleanups rodarem).

---

## 📜 6. Lendo o `gctrace`

`GODEBUG=gctrace=1` imprime uma linha por ciclo. Uma linha real do programa da seção 5:

```
gc 4 @0.055s 14%: 0.026+24+0.055 ms clock, 0.053+1.8/10/19+0.11 ms cpu, 36->46->45 MB, 43 MB goal, 0 MB stacks, 0 MB globals, 2 P
```

| Trecho | Significado |
| --- | --- |
| `gc 4` | Quarto ciclo |
| `@0.055s` | Segundos desde o início do programa |
| `14%` | Fração da CPU gasta pelo GC desde o início |
| `0.026+24+0.055 ms clock` | Tempo de parede de cada fase: **pausa** inicial + marcação **concorrente** + **pausa** final. As pausas foram de 26 µs e 55 µs |
| `0.053+1.8/10/19+0.11 ms cpu` | CPU de cada fase; a do meio é dividida em **assists** / workers dedicados / workers ociosos |
| `36->46->45 MB` | Heap no **início** da marcação → no **fim** → **vivo** marcado |
| `43 MB goal` | A meta calculada para este ciclo |
| `2 P` | Ps (`GOMAXPROCS`) usados |

O heap passou da meta (46 > 43 MB) porque o programa continuou alocando durante a marcação, e quase tudo estava vivo (45 MB). Repare nas pausas: dezenas de **microssegundos**, com 45 MB de heap. As pausas do GC de Go não crescem com o tamanho da heap; o p99 medido nesse programa ficou entre 57 e 262 µs.

---

## 🍵 7. Green Tea: o coletor atual (Go 1.26+)

O algoritmo clássico examina os objetos na ordem em que os descobre, saltando por toda a heap: para objetos **pequenos**, cada um pode estar numa linha de cache e numa página diferentes, e a CPU passa a maior parte do tempo esperando a memória (o efeito que medimos em [[Benchmarks]]).

O **Green Tea** (experimental no 1.25, **padrão no Go 1.26**) muda a unidade de trabalho: em vez de enfileirar **objetos**, enfileira **spans** (páginas de 8 KB). Quando encontra um ponteiro para um objeto pequeno, marca-o e coloca o **span** numa fila (FIFO); quando o span sai da fila, muitos objetos dele já foram marcados nesse meio-tempo, e todos são examinados **juntos**, com acesso à memória sequencial e metadados compartilhados. Em CPUs amd64 recentes (Intel Ice Lake, AMD Zen 4 em diante), a varredura usa até instruções vetoriais (SIMD).

Medição real, o mesmo programa com e sem Green Tea (`GOEXPERIMENT=nogreenteagc`), três execuções cada:

```
Green Tea:  GCs: 6 | CPU do GC: 32.4% | tempo: 187ms | heap máx: 127 MB
antigo:     GCs: 5 | CPU do GC: 44.8% | tempo: 225ms | heap máx: 147 MB
Green Tea:  GCs: 6 | CPU do GC: 33.5% | tempo: 167ms | heap máx: 127 MB
antigo:     GCs: 5 | CPU do GC: 45.1% | tempo: 206ms | heap máx: 151 MB
Green Tea:  GCs: 6 | CPU do GC: 33.2% | tempo: 193ms | heap máx: 127 MB
antigo:     GCs: 5 | CPU do GC: 45.9% | tempo: 210ms | heap máx: 151 MB
```

Cerca de **26% menos CPU de GC** neste programa (repleto de objetos pequenos com ponteiros, o caso que o Green Tea mira), dentro da faixa de 10–40% que a equipe de Go relata para programas reais. A mudança foi **invisível** para o código: nenhuma linha muda, só a versão do Go.

---

## 🪦 8. Cleanups e ponteiros fracos

### 8.1 `runtime.AddCleanup` (Go 1.24+)

Às vezes um objeto Go é dono de um recurso **externo** (memória de C, um descritor). `AddCleanup` registra uma função para rodar **depois** que o objeto for coletado. Ele substitui o antigo `runtime.SetFinalizer`, que tinha armadilhas (ressuscitava o objeto, não funcionava com ciclos, atrasava a liberação).

### 8.2 Ponteiros fracos: o pacote `weak` (Go 1.24+)

Um **ponteiro fraco** aponta para um objeto **sem mantê-lo vivo**: se só restarem ponteiros fracos, o GC coleta o objeto e o ponteiro fraco passa a devolver `nil`. Útil para caches e tabelas de canonicalização que não devem impedir a coleta. Saída real:

```go
img := &Imagem{Nome: "logo.png", Dados: make([]byte, 1<<20)}

fraco := weak.Make(img) // NÃO mantém a imagem viva
runtime.AddCleanup(img, func(nome string) {
	fmt.Println("cleanup: memória de", nome, "foi liberada")
}, img.Nome) // o argumento NÃO pode ser img (senão ela nunca morreria)

runtime.GC()
fmt.Println("com referência forte, Value():", fraco.Value() != nil)

runtime.KeepAlive(img) // img é usada até aqui
img = nil
runtime.GC()
time.Sleep(10 * time.Millisecond) // cleanups rodam numa goroutine do runtime
fmt.Println("sem referência forte, Value():", fraco.Value())
```

```
com referência forte, Value(): true
cleanup: memória de logo.png foi liberada
sem referência forte, Value(): <nil>
```

| Regra | Por quê |
| --- | --- |
| A função de cleanup **não** pode referenciar o objeto | Ela seria uma raiz e o manteria vivo para sempre |
| Cleanups rodam **quando** o GC decidir, numa goroutine do runtime | Não sirva-se deles para liberar recursos escassos (feche arquivos com `Close`, não com cleanup) |
| `runtime.KeepAlive(x)` | Garante que `x` é considerado vivo até aquele ponto (o compilador poderia considerar `img` morta antes da última linha que a usa) |

---

## 🔬 9. Por dentro

| Peça | Implementação |
| --- | --- |
| Bits de marcação | Um bitmap por span; varrer = trocar o bitmap de alocação pelo de marcação |
| Fila de trabalho cinza | Buffers por P (`gcWork`), sem trava no caso comum; no Green Tea, filas de spans |
| Raízes | Globais (`.data`/`.bss` com mapas de ponteiros gerados pelo compilador), stacks das goroutines (cada goroutine é parada **individualmente** só para ter a stack escaneada), registradores |
| Precisão | O compilador grava, para cada tipo e cada ponto de cada função, **quais palavras são ponteiros**: por isso `.data` e `.noptrdata` são separados, e por isso aritmética de ponteiros é proibida (ver [[unsafe e cgo]]) |
| Objetos grandes | Divididos em pedaços de 128 KB (*oblets*) para paralelizar a marcação |
| Barreira de escrita | Inserida pelo compilador em escritas de ponteiros na heap; enfileira num buffer por P |
| Ritmo | O *pacer* estima quando começar para terminar perto da meta, usando a taxa de alocação observada |

| Conceito de SO | No GC de Go |
| --- | --- |
| [[Gerenciando a Memória Livre]] com bitmaps | Bitmaps de alocação e de marcação por span |
| Interromper processos em pontos seguros ([[Introdução ao Escalonamento]]) | Pausas STW: cada P chega a um ponto seguro |
| Contabilizar uso de CPU por atividade | Os 25% de workers, os assists e o limite de 50% |
| Limites de memória do cgroup ([[Cgroups]]) | `GOMEMLIMIT` |
| Algoritmos de substituição de página que escolhem o que liberar ([[Memória Virtual]]) | Aqui não há escolha: libera-se exatamente o inalcançável |

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Tentar "ajudar" com `runtime.GC()` | Pausas e CPU desperdiçadas | Deixe o pacer decidir |
| `GOGC` baixo para "economizar memória" | CPU alta, latência ruim | `GOMEMLIMIT` com `GOGC` padrão |
| Sem `GOMEMLIMIT` em container | Morte por OOM com heap "pequeno" | `GOMEMLIMIT` ≈ 80–90% do limite |
| `GOMEMLIMIT` perto dos dados vivos | GC sem parar, latência alta | Limite com folga; reduza os dados vivos |
| Muitos ponteiros em estruturas gigantes | Marcação cara | Estruturas sem ponteiros (índices em vez de ponteiros; `[]T` em vez de `[]*T`) |
| Liberar recursos em cleanups/finalizers | Arquivos e conexões esgotam antes do GC rodar | `Close` explícito com `defer` |
| Cleanup que referencia o próprio objeto | O objeto nunca é coletado | Passe só o que o cleanup precisa |
| Medir memória sem considerar o GC | Conclusões erradas | `gctrace`, `runtime/metrics`, profile `inuse_space` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Tipo de GC | Rastreamento; marcação e varredura; concorrente, preciso, paralelo; sem compactação, sem gerações |
| Tricolor | Branco (não visto), cinza (a examinar), preto (examinado); branco no fim = lixo |
| Barreira de escrita | Ligada só na marcação; pinta o ponteiro antigo e o novo; permite marcar sem parar o mundo |
| Fases | STW curto → marcação concorrente (25% + assists) → STW curto → varredura concorrente |
| Pausas | Dezenas a centenas de µs, independentes do tamanho da heap |
| `GOGC` | Meta = vivo × (1 + GOGC/100); troca memória por CPU |
| `GOMEMLIMIT` | Teto suave; recomendado em containers; GC limitado a ~50% da CPU |
| `gctrace` | Pausas, CPU, heap início→fim→vivo, meta |
| Green Tea (1.26) | Marca por spans; melhor localidade; ~26% menos CPU de GC no experimento |
| `AddCleanup` / `weak` | Limpeza pós-coleta; ponteiros que não mantêm vivo |
| Reduzir custo | Alocar menos; menos ponteiros |

---

## 🔗 Conexões

**Dentro de Go**
- O que vai para a heap e o alocador → [[Memória e Escape Analysis]]
- Raízes e ponteiros → [[Ponteiros]]
- `gctrace`, `runtime/metrics`, profile de heap → [[Profiling e Diagnóstico]]
- O escalonador que roda os workers do GC e para os Ps → [[Scheduler do Go]]
- `sync.Pool` e o cache vítima → [[sync.WaitGroup e sync.Mutex]]
- Stacks escaneadas e que crescem → [[Goroutines]]
- Ponteiros escondidos do GC, memória de C → [[unsafe e cgo]]
- Medir o impacto de alocações → [[Benchmarks]]
- Seções `.data` vs `.noptrdata` → [[Como Go Funciona]]

**Sistemas Operacionais**
- [[Gerenciando a Memória Livre]] — bitmaps e listas, agora com coleta automática
- [[A Linguagem C]] — gerência manual e seus riscos
- [[Paravirtualização e a Máquina Virtual Java]] — outro runtime com GC
- [[Memória Virtual]] e [[Paginação]] — páginas físicas, RSS
- [[Introdução ao Escalonamento]] — pontos seguros e pausas
- [[Processadores]] e [[Memória]] — localidade, a motivação do Green Tea

**Docker**
- [[Cgroups]] — limite de memória e `GOMEMLIMIT`

---

⬅️ **Anterior:** [[Memória e Escape Analysis]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Scheduler do Go]] ➡️
