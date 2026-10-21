---
tags:
  - go
  - go/tooling
aliases:
  - Benchmark
  - go test -bench
  - b.Loop
  - benchstat
  - Medir desempenho em Go
ordem: 46
modulo: "11 - Tooling e Testes"
---
# Benchmarks

> **Nota 46 da trilha de Go.** Ao longo da trilha repetimos "meça antes de otimizar". Esta nota mostra **como medir direito** com o pacote `testing`: a função `BenchmarkXxx`, o `b.Loop()` (Go 1.24+) e o problema de **eliminação de código morto** que ele resolve, medindo alocações, sub-benchmarks, benchmarks **paralelos**, comparando versões com rigor estatístico usando **`benchstat`**, e os efeitos de hardware (cache da CPU) que distorcem medições ingênuas. Os números aqui são todos reais, e alguns contrariam a intuição.
>
> **Pré-requisitos:** [[Testes com go test]], [[Funções — Fundamentos]].

---

## 🧭 De onde viemos

Em [[Testes com go test]] vimos o pacote `testing` e o binário de teste. Várias notas mediram coisas com `testing.Benchmark` dentro de um `main` (o custo de channels em [[Channels]], de mutex e atômicos em [[sync.WaitGroup e sync.Mutex]], de iteradores em [[Padrões de Concorrência]]). Outras deixaram perguntas para cá: o efeito da pré-alocação em [[Arrays e Slices]], structs grandes vs ponteiros em [[Structs]] e [[Ponteiros]], `Mutex` vs `RWMutex`. Em [[Processadores]] e [[Memória]] você viu caches L1/L2/L3 e hierarquia de memória: eles vão aparecer nos números.

> 💡 **Benchmark (microbenchmark):** medição repetida e controlada do tempo (e dos recursos) gastos por uma operação pequena, para comparar implementações ou detectar regressões de desempenho.

---

## ⏱️ 1. A estrutura

```go
func BenchmarkJuntar(b *testing.B) {
	partes := gerarPartes()          // setup: roda UMA vez (com b.Loop)
	b.ReportAllocs()                 // inclui B/op e allocs/op na saída
	for b.Loop() {                   // Go 1.24+: o framework decide quantas voltas
		Juntar(partes)
	}
}
```

```bash
go test -bench=.                     # benchmarks NÃO rodam sem -bench
go test -run='^$' -bench=Juntar .    # -run='^$' pula os testes comuns
go test -bench=. -benchmem           # alocações em todos
go test -bench=. -count=6            # repetir (para estatística)
go test -bench=. -benchtime=3s       # tempo alvo por benchmark (padrão 1s)
go test -bench=. -benchtime=100x     # número fixo de iterações
go test -bench=. -cpu=1,2,4          # repetir com vários GOMAXPROCS
```

Lendo uma linha de resultado (real):

```
BenchmarkJuntar-2   	   79002	     16829 ns/op	   37000 B/op	      99 allocs/op
     │          │          │              │                │                │
     │          │          │              │                │                └ alocações na heap por operação
     │          │          │              │                └ bytes alocados por operação
     │          │          │              └ tempo médio por operação
     │          │          └ quantas iterações foram medidas
     │          └ GOMAXPROCS
     └ nome
```

### 1.1 Como o número de iterações é escolhido

O framework roda o benchmark com N crescente (1, depois uma estimativa, depois outra...) até que o tempo total chegue perto do `-benchtime`, e reporta `tempo / N`. Com o estilo antigo (`for i := 0; i < b.N; i++`), a **função inteira** é chamada a cada rodada de calibração; com `b.Loop()`, a função é chamada **uma vez** e o próprio `Loop` controla as rodadas.

---

## 🗑️ 2. O problema do código morto, e o `b.Loop`

O compilador elimina cálculos cujo resultado ninguém usa (ver [[Como Go Funciona]]). Num benchmark, isso pode apagar exatamente o que você queria medir. Saída real:

```go
func quadrado(x int) int { return x * x }

func BenchmarkQuadradoBN(b *testing.B) {
	for i := 0; i < b.N; i++ {
		quadrado(i)          // resultado descartado
	}
}

func BenchmarkQuadradoLoop(b *testing.B) {
	i := 0
	for b.Loop() {
		quadrado(i)          // b.Loop mantém argumentos e resultado "vivos"
		i++
	}
}
```

```
BenchmarkQuadradoBN-2     	1000000000	         0.3589 ns/op
BenchmarkQuadradoLoop-2   	1000000000	         0.9997 ns/op
```

**0,36 ns** é menos que um ciclo de clock de 2,1 GHz (0,48 ns): é o custo do laço **vazio**. A multiplicação foi inlinada e depois apagada, porque o resultado não era usado. Com `b.Loop()`, o compilador é instruído a manter vivos os argumentos e resultados das chamadas no corpo do laço, então a operação é de fato medida.

Antes do Go 1.24, a defesa era atribuir o resultado a uma variável global (`sink = quadrado(i)`). Com `b.Loop` isso não é mais necessário, e desde o **Go 1.26** o `b.Loop` também deixou de impedir inlining no corpo, então não há mais motivo para usar `b.N` em código novo.

### 2.1 O setup roda uma vez só

Saída real, contando quantas vezes a função de benchmark (e portanto o setup) foi chamada:

```go
func BenchmarkSetupBN(b *testing.B) {
	setupsBN++
	d := preparar()                  // monta 1000 ints
	for i := 0; i < b.N; i++ {
		_ = slices.Contains(d, 999)
	}
	b.ReportMetric(float64(setupsBN), "setups")      // métrica personalizada
}
// BenchmarkSetupLoop: igual, mas com for b.Loop()
```

```
BenchmarkSetupBN-2     	 1670678	       727.6 ns/op	         5.000 setups
BenchmarkSetupLoop-2   	 3382560	       367.2 ns/op	         1.000 setups
```

Com `b.N`, o setup rodou **5 vezes** (uma por rodada de calibração); com `b.Loop`, **uma**. Com setups caros (abrir um banco, gerar 1 GB de dados), isso faz diferença no tempo total. (A diferença de ns/op entre as duas versões veio do código gerado para cada laço; o ponto aqui é a coluna `setups`.)

| | `for i := 0; i < b.N; i++` | `for b.Loop()` |
| --- | --- | --- |
| Setup antes do laço | Roda a cada rodada de calibração | Roda uma vez por `-count` |
| `b.ResetTimer()` depois do setup | Necessário | Desnecessário (o timer só conta dentro do `Loop`) |
| Código morto | Pode ser eliminado | Argumentos e resultados mantidos vivos |
| Uso | Código antigo | **Padrão desde o Go 1.24** |

### 2.2 Controlando o cronômetro

```go
for b.Loop() {
	b.StopTimer()
	entrada := gerarEntradaAleatoria()   // não entra na medição
	b.StartTimer()
	processar(entrada)
}
```

> ⚠️ `StopTimer`/`StartTimer` dentro do laço têm custo próprio (leem o relógio e as estatísticas de memória). Para operações de nanossegundos, eles distorcem o resultado; prefira preparar as entradas antes do laço.

---

## 💾 3. Alocações

`-benchmem` (ou `b.ReportAllocs()`) adiciona **B/op** e **allocs/op**: quantos bytes e quantas alocações **na heap** cada operação fez. Isso importa porque cada alocação custa tempo agora (o alocador) e depois (o coletor de lixo, ver [[Garbage Collector]]). O exemplo clássico, concatenar 100 strings:

```go
func Juntar(partes []string) string {          // s += p: uma string nova a cada volta
	s := ""
	for _, p := range partes {
		s += p
	}
	return s
}

func JuntarBuilder(partes []string) string {   // calcula o tamanho e aloca UMA vez
	n := 0
	for _, p := range partes {
		n += len(p)
	}
	var b strings.Builder
	b.Grow(n)
	for _, p := range partes {
		b.WriteString(p)
	}
	return b.String()
}
```

```
Juntar          16829 ns/op	   37000 B/op	      99 allocs/op
JuntarBuilder     613 ns/op	     704 B/op	       1 allocs/op
```

O motivo está em [[Strings em Profundidade]]: strings são imutáveis, então cada `+=` copia tudo o que já existia. São 99 alocações e ~37 KB copiados para produzir um texto de 700 bytes, um custo **quadrático**. A seção 5 compara as duas com rigor.

> 💡 `0 allocs/op` é um objetivo comum em código muito quente (parsers, roteadores HTTP). Para descobrir **por que** algo aloca, use `go build -gcflags=-m` (ver [[Memória e Escape Analysis]]) e o profile de memória (ver [[Profiling e Diagnóstico]]).

---

## 🪜 4. Sub-benchmarks: como o custo cresce

`b.Run` cria variações, ótimo para ver a **complexidade** na prática (ver [[Big O Notation]]). Saída real, ordenando slices de três tamanhos:

```go
func BenchmarkOrdenar(b *testing.B) {
	for _, n := range []int{100, 10_000, 1_000_000} {
		b.Run(fmt.Sprint("n=", n), func(b *testing.B) {
			orig := aleatorios(n)
			d := make([]int, n)
			for b.Loop() {
				copy(d, orig)          // ordena sempre a mesma entrada desordenada
				slices.Sort(d)
			}
		})
	}
}
```

```
BenchmarkOrdenar/n=100-2         	 1234532	       966.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkOrdenar/n=10000-2       	    1806	    660123 ns/op	       0 B/op	       0 allocs/op
BenchmarkOrdenar/n=1000000-2     	      10	 107410693 ns/op	       0 B/op	       0 allocs/op
```

De 10 mil para 1 milhão (100× mais elementos), o tempo subiu **163×**, perto do previsto por O(n log n): 100 × (log 10⁶ / log 10⁴) = 150×. Os detalhes do algoritmo (o `pdqsort` de `slices.Sort`) estão em [[Algoritmos de Ordenação]].

---

## 📊 5. Comparando com rigor: `benchstat`

Uma execução só **não prova nada**: o ruído (outros processos, frequência da CPU, temperatura) facilmente varia 5–10%. O fluxo correto:

```bash
go test -run='^$' -bench=Juntar -count=6 . > antes.txt
# ...muda o código...
go test -run='^$' -bench=Juntar -count=6 . > depois.txt
benchstat antes.txt depois.txt          # go install golang.org/x/perf/cmd/benchstat@latest
```

Saída real (`Juntar` com `+=` em `antes.txt`, com `strings.Builder` em `depois.txt`):

```
         │   antes.txt   │             depois.txt             │
         │    sec/op     │   sec/op     vs base               │
Juntar-2   16052.5n ± 6%   613.5n ± 8%  -96.18% (p=0.002 n=6)

         │  antes.txt   │            depois.txt             │
         │     B/op     │    B/op     vs base               │
Juntar-2   37000.0 ± 0%   704.0 ± 0%  -98.10% (p=0.002 n=6)

         │  antes.txt  │            depois.txt             │
         │  allocs/op  │ allocs/op   vs base               │
Juntar-2   99.000 ± 0%   1.000 ± 0%  -98.99% (p=0.002 n=6)
```

| Campo | Significado |
| --- | --- |
| `16052.5n` | **Mediana** das 6 execuções (robusta a valores extremos) |
| `± 6%` | Intervalo de confiança: a variação entre execuções |
| `-96.18%` | Mudança estimada |
| `p=0.002` | Valor-p do teste de Mann-Whitney: probabilidade de uma diferença assim surgir por acaso. Abaixo de 0,05, a diferença é considerada **real** |
| `~` no lugar do percentual | "Sem diferença estatisticamente significativa" |

> ✅ Para afirmar "ficou X% mais rápido", use `-count` de pelo menos 6, rode as duas versões **na mesma máquina e nas mesmas condições**, e deixe o `benchstat` decidir se a diferença existe.

---

## 🧵 6. Benchmarks paralelos

`b.RunParallel` executa o corpo em várias goroutines (`GOMAXPROCS` por padrão), dividindo as iterações entre elas. Serve para medir código que será usado **concorrentemente**. Saída real: um cache de leitura protegido por `Mutex` vs `RWMutex`, com 1 e 2 CPUs:

```go
func BenchmarkCacheRWMutex(b *testing.B) {
	c := &cacheRW{m: dados()}               // Get usa RLock/RUnlock
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			c.Get(strconv.Itoa(i % 1000))
			i++
		}
	})
}
```

```
$ go test -run='^$' -bench=Cache -cpu=1,2 .
BenchmarkCacheMutex       	19214811	        53.43 ns/op
BenchmarkCacheMutex-2     	23034556	        59.81 ns/op
BenchmarkCacheRWMutex     	23058397	        52.97 ns/op
BenchmarkCacheRWMutex-2   	 8912002	       127.2 ns/op      ← 2× PIOR que o Mutex
```

Contra a intuição, o `RWMutex`, feito para leitores simultâneos, ficou **duas vezes mais lento** que o `Mutex` com 2 CPUs. O motivo: cada `RLock`/`RUnlock` faz uma operação atômica num **contador compartilhado de leitores**, e com uma seção crítica de poucos nanossegundos (uma busca num map), o custo dominante passa a ser a **linha de cache** desse contador pulando de um núcleo para o outro. Era o aviso de [[sync.WaitGroup e sync.Mutex]] ("meça antes"), agora com números.

> 💡 Num benchmark paralelo, `ns/op` é **tempo de parede** dividido pelo total de operações de todas as goroutines. Se o código escalasse perfeitamente, o `ns/op` cairia pela metade ao dobrar as CPUs.

---

## 🧠 7. O hardware também está no benchmark

### 7.1 Cache da CPU

O mesmo trabalho (somar 128 milhões de `int64`, 1 GB) com acesso **sequencial** e **aleatório**. Saída real, numa máquina com L3 de 260 MB:

```go
b.Run("sequencial", func(b *testing.B) {
	for b.Loop() {
		var s int64
		for i := range n {
			s += dados[i]
		}
	}
})
b.Run("aleatorio", func(b *testing.B) {
	for b.Loop() {
		var s int64
		for _, i := range ordem {          // ordem = rand.Perm(n)
			s += dados[i]
		}
	}
})
```

```
BenchmarkAcesso/sequencial-2   	2	 214984404 ns/op	         1.602 ns/elemento
BenchmarkAcesso/aleatorio-2    	2	1212728412 ns/op	         9.036 ns/elemento
```

Mesma quantidade de somas, **5,6× mais lento** em ordem aleatória. No acesso sequencial, cada linha de cache de 64 bytes traz 8 elementos de uma vez e o *prefetcher* da CPU adivinha o próximo bloco; no aleatório, quase todo acesso é uma falta de cache (e de TLB) que vai até a RAM. É a hierarquia de memória de [[Memória]] e [[Processadores]] aparecendo num laço de duas linhas. Também é a razão de **arrays e slices** costumarem ganhar de [[Linked Lists]] mesmo quando a complexidade é igual.

### 7.2 Fontes de ruído

| Fonte | Mitigação |
| --- | --- |
| Outros processos na máquina | Máquina ociosa; rode as versões intercaladas |
| Frequência dinâmica da CPU (turbo, economia de energia) | Fixar o governor em `performance`; `-count` alto |
| Dados que cabem no cache vs produção | Tamanhos realistas (seção 7.1) |
| Coletor de lixo disparando no meio | Benchmarks mais longos (`-benchtime`); observe `B/op` |
| Máquinas virtuais e containers com CPU limitada | Compare só resultados da mesma máquina |

---

## 🔬 8. Por dentro

```
go test -bench=Juntar
   │
   ├─ roda os testes? só se o -run casar (por isso -run='^$')
   │
   └─ para cada BenchmarkXxx:
        chama a função UMA vez
          └─ b.Loop() é chamado a cada volta:
               1ª chamada: zera estatísticas, liga o cronômetro
               decide se continua: compara o tempo decorrido com -benchtime
               e estima quantas voltas faltam (crescimento controlado)
               última: para o cronômetro e calcula ns/op
        B/op e allocs/op = diferença de runtime.MemStats (Mallocs, TotalAlloc) / N
```

| Detalhe | Consequência |
| --- | --- |
| O tempo vem de um relógio **monotônico** (`time.Now` com leitura monotônica, via `clock_gettime(CLOCK_MONOTONIC)`/vDSO) | Ajustes do relógio do sistema não afetam a medição |
| Alocações vêm das estatísticas do alocador | Alocações na **stack** não aparecem (por isso escape analysis importa) |
| `-cpu=1,2,4` | Muda o `GOMAXPROCS` (ver [[Goroutines]]) entre execuções |
| Profiles direto do benchmark | `go test -bench=X -cpuprofile=cpu.out -memprofile=mem.out` (ver [[Profiling e Diagnóstico]]) |

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Resultado descartado com `b.N` | ns/op absurdamente baixo (< 1 ciclo) | `b.Loop()` |
| Setup caro dentro da medição | Números inflados | Setup antes do `b.Loop` |
| `StopTimer`/`StartTimer` em operações minúsculas | Distorção pelo custo do próprio cronômetro | Prepare as entradas antes |
| Uma execução só | "Melhorou 7%" que é ruído | `-count=6` + `benchstat` |
| Comparar máquinas ou momentos diferentes | Conclusões falsas | Mesma máquina, intercalado |
| Dados pequenos demais | Tudo no cache L1; produção é mais lenta | Tamanhos realistas |
| Benchmark de código que nunca é gargalo | Tempo perdido | Profile primeiro, benchmark depois |
| Esquecer `-run='^$'` | Testes lentos rodam junto | `-run='^$'` |
| Supor que `RWMutex` (ou qualquer "otimização") é mais rápido | Pode ser 2× pior | Meça com `RunParallel` e `-cpu` |
| Benchmarkar o compilador (constantes) | Operação calculada em tempo de compilação | Entradas que o compilador não conhece |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Estrutura | `BenchmarkXxx(b *testing.B)` + `for b.Loop()`; rode com `-bench` |
| `b.Loop` (1.24+) | Setup uma vez; mantém argumentos e resultados vivos; desde 1.26 não impede inlining |
| `b.N` | Estilo antigo; sujeito a código morto e setup repetido |
| Alocações | `-benchmem` / `ReportAllocs`: B/op e allocs/op (só heap) |
| Sub-benchmarks | `b.Run` para tamanhos e variações; veja a complexidade real |
| `benchstat` | Mediana, intervalo, valor-p; `-count ≥ 6` |
| Paralelo | `RunParallel` + `-cpu`; ns/op é tempo de parede por operação |
| Hardware | Cache, prefetch e TLB podem valer mais que o algoritmo |
| Métricas extras | `b.ReportMetric(valor, "unidade")`, `b.SetBytes` (MB/s) |

---

## 🔗 Conexões

**Dentro de Go**
- O pacote `testing` e o binário de teste → [[Testes com go test]]
- Do benchmark ao profile → [[Profiling e Diagnóstico]]
- Por que algo aloca → [[Memória e Escape Analysis]], [[Garbage Collector]]
- Inlining e código morto → [[Funções — Fundamentos]], [[Como Go Funciona]]
- Concatenação e imutabilidade → [[Strings em Profundidade]]
- Pré-alocação → [[Arrays e Slices]]
- Struct vs ponteiro → [[Structs]], [[Ponteiros]]
- `Mutex` vs `RWMutex` vs atômicos → [[sync.WaitGroup e sync.Mutex]]
- Custo de channels e de iteradores → [[Channels]], [[Padrões de Concorrência]]
- `GOMAXPROCS` → [[Goroutines]]

**Sistemas Operacionais**
- [[Memória]] e [[Processadores]] — hierarquia de cache, linhas de 64 bytes, prefetch
- [[Memória Virtual]] e [[Paginação]] — o TLB e as faltas que tornam o acesso aleatório caro
- [[Escalonamento em Sistemas Interativos]] — outros processos competindo pela CPU durante a medição

**Algoritmos e Estruturas de Dados**
- [[Big O Notation]] e [[Complexidade de Algoritmos]] — a teoria que os sub-benchmarks confirmam (ou não)
- [[Algoritmos de Ordenação]] — o que `slices.Sort` faz
- [[Arrays]] e [[Linked Lists]] — localidade de memória na prática

---

⬅️ **Anterior:** [[Testes com go test]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Profiling e Diagnóstico]] ➡️
