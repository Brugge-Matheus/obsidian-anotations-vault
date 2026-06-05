---
tags:
  - go
  - go/tooling
---
# Benchmarks

> Benchmarks medem performance de forma reproduzível. Em Go, são integrados ao `go test` e permitem medir tempo, alocações de memória e comparar implementações com precisão estatística.
> 

---

## 1. Como o Framework de Benchmark Funciona

O runtime de benchmark determina automaticamente `b.N` — o número de iterações necessário para obter uma medição estável:

```
Fase de calibração:
1. Roda com b.N = 1
2. Se durou < benchtime/b.N: aumenta b.N exponencialmente
3. Repete até durar pelo menos benchtime (padrão: 1 segundo)
4. Reporta: ns/op, B/op, allocs/op
```

```go
// Estrutura básica
func BenchmarkSomar(b *testing.B) {
	for b.Loop() {   // Go 1.24+: b.Loop() substitui i < b.N
		Somar(100, 200)
	}
}

// Antes do Go 1.24 (ainda válido):
func BenchmarkSomarAntigo(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Somar(100, 200)
	}
}
```

> 💡 Go 1.24 introduziu `b.Loop()` que substitui `for i := 0; i < b.N; i++`. É mais ergonômico e permite que o framework faça otimizações internas.
> 

---

## 2. Executar Benchmarks

```bash
# Benchmarks não rodam por padrão — precisa do flag -bench
go test -bench=.            # todos os benchmarks
go test -bench=BenchmarkX   # benchmark específico
go test -bench=Bench.*      # por regex

# Flags essenciais
go test -bench=. -benchmem  # inclui alocações de memória
go test -bench=. -benchtime=5s     # tempo mínimo por benchmark
go test -bench=. -benchtime=1000x  # número fixo de iterações
go test -bench=. -count=5          # executa 5 vezes (para análise estatística)
go test -bench=. -cpu=1,2,4,8      # testa com 1, 2, 4 e 8 CPUs

# Combinando
go test -bench=. -benchmem -count=5 | tee resultado.txt
```

### Interpretar a Saída

```
BenchmarkCriarSlice-8    5000000    234 ns/op    96 B/op    2 allocs/op
         │              │          │              │           │
         │              │          │              │           └─ alocações por op
         │              │          │              └─ bytes alocados por op
         │              │          └─ tempo por operação
         │              └─ iterações executadas
         └─ GOMAXPROCS = 8 CPUs
```

---

## 3. `b.ResetTimer` e `b.StopTimer`

```go
func BenchmarkComSetup(b *testing.B) {
	// Setup NÃO contabilizado no benchmark
	dados := gerarDados(10000)
	db := iniciarDB()
	defer db.Close()

	b.ResetTimer()   // zera o timer aqui — setup não conta

	for b.Loop() {
		buscarDados(db, dados[0])
	}
}

func BenchmarkComSetupPorIteracao(b *testing.B) {
	for b.Loop() {
		b.StopTimer()
		dadosAleatorios := gerarAleatorio()   // não conta
		b.StartTimer()

		processar(dadosAleatorios)   // só isso é medido
	}
}
```

---

## 4. `b.ReportAllocs` e Métricas Customizadas

```go
func BenchmarkAllocacoes(b *testing.B) {
	b.ReportAllocs()   // equivale a -benchmem, mas por benchmark

	for b.Loop() {
		s := make([]byte, 1024)
		processar(s)
	}
}

// Métricas customizadas
func BenchmarkThroughput(b *testing.B) {
	dados := gerarDados(1 << 20)   // 1MB

	b.SetBytes(int64(len(dados)))   // reporta MB/s além de ns/op
	b.ResetTimer()

	for b.Loop() {
		processar(dados)
	}
}
// Saída: BenchmarkThroughput-8  500  2345678 ns/op  447.12 MB/s  0 B/op
```

---

## 5. Sub-benchmarks

```go
func BenchmarkOrdenar(b *testing.B) {
	tamanhos := []struct {
		nome string
		n    int
	}{
		{"100", 100},
		{"1K", 1000},
		{"10K", 10000},
		{"1M", 1000000},
	}

	for _, tc := range tamanhos {
		b.Run(tc.nome, func(b *testing.B) {
			dados := gerarSliceAleatorio(tc.n)

			b.ResetTimer()
			for b.Loop() {
				copia := make([]int, len(dados))
				copy(copia, dados)
				slices.Sort(copia)
			}
		})
	}
}
```

---

## 6. Benchmarks em Paralelo

```go
func BenchmarkCacheParalelo(b *testing.B) {
	cache := NewCache(1000)

	// Pré-popular
	for i := 0; i < 1000; i++ {
		cache.Set(fmt.Sprintf("key%d", i), i)
	}

	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		// Cada goroutine executa seu próprio loop
		i := 0
		for pb.Next() {
			cache.Get(fmt.Sprintf("key%d", i%1000))
			i++
		}
	})
}

// Testar escalabilidade com -cpu=1,2,4,8
// go test -bench=BenchmarkCacheParalelo -cpu=1,2,4,8
```

---

## 7. Comparar Implementações

```go
// Comparar concatenação de strings
var resultado string   // variável global evita dead code elimination

func BenchmarkConcatPlus(b *testing.B) {
	palavras := []string{"hello", " ", "world", "!"}
	for b.Loop() {
		s := ""
		for _, p := range palavras {
			s += p
		}
		resultado = s   // impede otimização
	}
}

func BenchmarkConcatBuilder(b *testing.B) {
	palavras := []string{"hello", " ", "world", "!"}
	for b.Loop() {
		var sb strings.Builder
		sb.Grow(20)
		for _, p := range palavras {
			sb.WriteString(p)
		}
		resultado = sb.String()
	}
}

func BenchmarkConcatJoin(b *testing.B) {
	palavras := []string{"hello", " ", "world", "!"}
	for b.Loop() {
		resultado = strings.Join(palavras, "")
	}
}
```

---

## 8. `benchstat` — Análise Estatística

```bash
go install golang.org/x/perf/cmd/benchstat@latest

# Capturar resultados (count=5 para estabilidade estatística)
go test -bench=. -benchmem -count=5 > antes.txt

# Modificar o código...

go test -bench=. -benchmem -count=5 > depois.txt

# Comparar com análise estatística
benchstat antes.txt depois.txt
```

Saída:

```
                 │  antes.txt  │           depois.txt           │
                 │   sec/op    │   sec/op     vs base           │
ConcatPlus-8     │  1.234µ ± 2%│  1.201µ ± 1%  -2.67% (p=0.008)│
ConcatBuilder-8  │  234.5n ± 1%│  198.3n ± 2%  -15.42% (p=0.001)│

p < 0.05 indica diferença estatisticamente significativa
```

---

## 9. Profiling a Partir de Benchmarks

```bash
# CPU profile
go test -bench=BenchmarkSomar -cpuprofile=cpu.prof
go tool pprof cpu.prof
# Dentro do pprof:
# (pprof) top10         — top 10 funções mais custosas
# (pprof) web           — grafo no browser (requer graphviz)
# (pprof) list Somar    — anotação do código fonte

# Memory profile
go test -bench=BenchmarkSomar -memprofile=mem.prof
go tool pprof mem.prof

# Visualização interativa
go tool pprof -http=:8080 cpu.prof
# Acesse http://localhost:8080 para grafo, flamegraph, etc.

# Trace — para análise de goroutines e GC
go test -bench=BenchmarkSomar -trace=trace.out
go tool trace trace.out
```

---

## 10. Evitar Armadilhas

```go
// ⚠️ Dead code elimination — o compilador pode eliminar o cálculo
func BenchmarkErrado(b *testing.B) {
	for b.Loop() {
		_ = Calcular(42)   // pode ser eliminado se Calcular não tem side effects
	}
}

// ✅ Use variável global para forçar que o resultado seja "usado"
var globalResultado int

func BenchmarkCorreto(b *testing.B) {
	var r int
	for b.Loop() {
		r = Calcular(42)
	}
	globalResultado = r   // atribuição global evita eliminação
}

// ⚠️ Dados de teste idênticos — o cache de CPU mascara a realidade
func BenchmarkCacheQuente(b *testing.B) {
	dados := []int{1, 2, 3}   // sempre os mesmos dados — cache quente
	for b.Loop() {
		processar(dados)
	}
}

// ✅ Dados variados para simular cache frio
func BenchmarkCacheFrio(b *testing.B) {
	datasets := make([][]int, b.N)
	for i := range datasets {
		datasets[i] = gerarAleatorio(1000)
	}
	b.ResetTimer()
	for i := range b.N {
		processar(datasets[i])
	}
}
```

---

## 11. Resumo dos Flags

| Flag | O que faz |
| --- | --- |
| `-bench=.` | Executa todos os benchmarks |
| `-bench=Nome` | Benchmark específico (regex) |
| `-benchmem` | Inclui B/op e allocs/op |
| `-benchtime=Xs` | Duração mínima (padrão: 1s) |
| `-benchtime=Nx` | Número fixo de iterações |
| `-count=N` | Executa N vezes (para benchstat) |
| `-cpu=1,2,4` | Testa com diferentes GOMAXPROCS |
| `-cpuprofile=f` | Gera CPU profile |
| `-memprofile=f` | Gera memory profile |
| `-trace=f` | Gera execution trace |

---

## 12. Por Dentro: Benchmarks, CPU e Memória

### Como `b.N` é calibrado: medindo tempo de CPU

```
Fase de calibração do framework:

Rodada 1: b.N = 1        → durou 50ns   (muito rápido — aumenta)
Rodada 2: b.N = 100      → durou 5μs    (ainda < 1s — aumenta)
Rodada 3: b.N = 10000    → durou 500μs  (ainda < 1s — aumenta)
Rodada 4: b.N = 2000000  → durou 1.02s  (≥ 1s — reporta resultado)

Resultado: 1.02s / 2000000 = 510 ns/op

Por que 1 segundo?
  - Estabiliza leituras do contador de ciclos (TSC — Time Stamp Counter)
  - Amortiza overhead de JIT warmup e cache warmup
  - Permite que o OS escalone e descale o processo sem distorcer muito

No hardware:
  time.Now() usa instrução RDTSC (Read Time Stamp Counter)
  ou clock_gettime(CLOCK_MONOTONIC) via syscall
  TSC é um contador de ciclos de CPU (incrementa a cada ciclo)
  frequência moderna: ~3-5 GHz → resolução de ~0.2-0.3ns
```

Conecta com [[Processadores]] — RDTSC é uma instrução de CPU para ler o contador de ciclos. Benchmarks medem ciclos de CPU convertidos para nanosegundos.

### `b.ResetTimer` / `b.StopTimer`: contadores de performance

```go
func BenchmarkComSetup(b *testing.B) {
    dados := gerarDados(10000)   // não deve ser medido
    db := iniciarDB()
    defer db.Close()

    b.ResetTimer()   // zera: time.now() interno, bytes alocados, allocs

    for b.Loop() {
        buscarDados(db, dados[0])   // apenas isto é medido
    }
}

// Internamente b.ResetTimer faz:
//   b.duration = 0      (zeroa tempo acumulado)
//   b.start = time.Now() (reinicia o timer)
//   b.netAllocs = 0     (zeroa contagem de allocs)
//   b.netBytes = 0      (zeroa contagem de bytes)
```

O mecanismo usa contadores similares a **hardware performance counters** (PMC — Performance Monitoring Counters no x86) para allocs, mas time é medido via RDTSC. Conecta com [[Processadores]] (performance counters, profiling hardware).

### `-benchmem`: contando alocações no heap

```
-benchmem reporta:
  B/op     = bytes alocados no heap por operação
  allocs/op = número de alocações (chamadas ao allocator) por operação

Como é medido:
  runtime.ReadMemStats() antes e depois do loop
  MemStats.Mallocs (total de alocações)
  MemStats.TotalAlloc (total de bytes alocados — inclui alocações já liberadas pelo GC)

  B/op     = (TotalAlloc_depois - TotalAlloc_antes) / b.N
  allocs/op = (Mallocs_depois - Mallocs_antes) / b.N

Por que alocações importam para performance?
  1. cada alloc → runtime procura bloco livre no heap
  2. GC precisa rastrear cada objeto alocado
  3. mais allocs → GC mais frequente → latência spike
  4. zero alloc = código que não pressiona o GC
```

Conecta com [[Gerenciamento de Memória]] — `benchmem` mede diretamente a pressão no heap e no GC. Código com `0 allocs/op` não gera trabalho para o GC.

### CPU profiling com SIGPROF: sampling profiler

```
go test -bench=. -cpuprofile=cpu.prof

Mecanismo interno:
  1. runtime chama setitimer(ITIMER_PROF, 10ms, ...) no início
  2. kernel envia SIGPROF para o processo a cada ~10ms
  3. signal handler do runtime captura o stack atual:
       goroutine 1: [running]
         main.BenchmarkSomar at bench_test.go:12
         testing.(*B).runN at testing.go:1234
         ...
  4. stack amostrado é escrito em cpu.prof (formato pprof)
  
  Após N amostras:
    "função X aparece em 60% das amostras" = X usa 60% do CPU
    (amostragem estatística, não contagem exata de ciclos)

go tool pprof cpu.prof
  > top10       → top 10 funções por %CPU
  > web         → grafo de chamadas no browser (requer graphviz)
  > list Somar  → anotação fonte com % por linha
```

Conecta com [[Processos]] — `SIGPROF` é um sinal Unix; `setitimer(ITIMER_PROF)` configura envio periódico baseado em CPU time gasto pelo processo. O runtime usa este mecanismo para sampling profiling.

### Memory profiling: heap profiler

```
go test -bench=. -memprofile=mem.prof

Mecanismo:
  - runtime interceta cada chamada ao allocator (malloc interno do Go)
  - a cada N bytes alocados (default: 512KB), registra o stack atual
  - grava em mem.prof: {stack_trace → bytes_alocados}

go tool pprof mem.prof
  > top10         → funções que mais alocam
  > list Processo → anotação por linha de código

Tipos de profile:
  -alloc_space   = total histórico alocado (inclui já liberado pelo GC)
  -inuse_space   = objetos vivos agora (o que está no heap)
```

Conecta com [[Gerenciamento de Memória]] — heap profiling rastreia sites de alocação. `inuse_space` mostra o que está consumindo heap agora.

### Efeitos de cache em benchmarks

```
L1 cache: ~4KB-64KB, latência ~1ns  (dentro do core)
L2 cache: ~256KB-1MB, latência ~4ns  (por core)
L3 cache: ~4MB-64MB, latência ~10ns  (compartilhado)
RAM:      ilimitada,  latência ~60-100ns

Benchmark com dados pequenos (cache quente):
  dados := []int{1, 2, 3}  // 24 bytes → sempre em L1
  → mede velocidade COM cache quente (melhor caso)

Benchmark com dados grandes (cache frio):
  dados := make([]int, 10_000_000)  // 80MB → não cabe em L3!
  → cada acesso → cache miss → vai para RAM → ~60ns extra por acesso
  → mede comportamento REAL com grandes volumes de dados

Cache line = 64 bytes (tamanho mínimo de transferência L1↔L2)
Acessar dados sequencialmente (stride=1) é ~10x mais rápido
que acesso aleatório (pointer chasing) por causa de prefetcher de CPU.
```

Conecta com [[Processadores]] — L1/L2/L3 cache, cache line size (64 bytes), cache miss penalty, hardware prefetcher.

---

## 13. Conexão com Sistemas Operacionais

- **[[Processadores]]**: `b.N` é calibrado medindo tempo via RDTSC (Read Time Stamp Counter) ou `clock_gettime(CLOCK_MONOTONIC)`. O framework roda até que a medição seja estável por ~1 segundo. `b.ResetTimer`/`b.StopTimer` controlam exatamente quais ciclos são contabilizados.

- **[[Processadores]]** (performance counters): CPU profiling usa `setitimer(ITIMER_PROF)` para receber `SIGPROF` a cada ~10ms. O signal handler amostras o stack — profiling por amostragem estatística, não instrução por instrução. `b.SetBytes` reporta throughput em MB/s calculado dos ciclos medidos.

- **[[Gerenciamento de Memória]]**: `-benchmem` usa `runtime.ReadMemStats()` para contar alocações no heap (`allocs/op`) e bytes (`B/op`). Zero allocs = código que não pressiona o GC. Memory profiling (`-memprofile`) rastreia sites de alocação — fundamental para otimizar código que roda em hot paths.

- **[[Processos]]**: `SIGPROF` é um sinal Unix gerado pelo kernel via `setitimer(ITIMER_PROF, interval)` baseado em CPU time consumido pelo processo. O runtime Go usa este mecanismo para sampling profiler sem overhead de instrumentação total.

- **[[Processadores]]** (cache): Benchmarks com dados pequenos medem desempenho com L1/L2 cache quente. Para resultados realistas, use volumes de dados que excedem o L3 (tipicamente 32-64MB). Cache lines de 64 bytes e acesso sequencial vs aleatório podem causar diferença de 10-100x nos resultados.