---
tags:
  - go
  - go/tooling
aliases:
  - pprof
  - Profiling em Go
  - go tool pprof
  - net/http/pprof
  - runtime/trace
  - Flight recorder
  - PGO
  - GODEBUG
ordem: 47
modulo: "11 - Tooling e Testes"
---
# Profiling e Diagnóstico

> **Nota 47 da trilha de Go.** Benchmarks respondem "quanto custa **isto**?". Profiling responde a pergunta anterior: "**onde** o programa gasta tempo e memória?". Go traz ferramentas de diagnóstico de nível profissional na biblioteca padrão: **profiles** de CPU, memória, goroutines, bloqueio e disputa de mutex (`runtime/pprof` e `go tool pprof`), o endpoint **`/debug/pprof`** para servidores em produção, **traces de execução** e o **flight recorder** (Go 1.25), dumps de goroutines com `SIGQUIT`, **labels**, `runtime/metrics`, variáveis `GODEBUG` e **PGO** (otimização guiada por profile). Vamos usar tudo isso num programa real, encontrar o gargalo e corrigi-lo.
>
> **Pré-requisitos:** [[Benchmarks]], [[Goroutines]], [[HTTP (net-http)]].

---

## 🧭 De onde viemos

Em [[Benchmarks]] medimos operações isoladas e vimos que intuição engana. Várias notas deixaram ferramentas prometidas para cá: o profile `goroutineleak` em [[Goroutines]], dumps de goroutines e o profile de mutex em [[sync.WaitGroup e sync.Mutex]], labels carregados no contexto em [[context.Context]], o `net/http/pprof` em [[HTTP (net-http)]] e o PGO em [[Funções — Fundamentos]] e [[Como Go Funciona]]. Do lado do SO, sinais (`SIGPROF`, `SIGQUIT`) apareceram em [[Syscalls Diversas]].

> 💡 **Profile (perfil de execução):** um conjunto de **amostras** que associa um custo (tempo de CPU, bytes alocados, tempo bloqueado) às **pilhas de chamadas** (*stack traces*) onde ele ocorreu. Somando as amostras por função, descobre-se quem é responsável pelo custo.

> 💡 **Profiler por amostragem:** em vez de medir cada instrução (caro), interrompe o programa periodicamente e anota "o que estava executando agora". Com amostras suficientes, a proporção estatística revela onde o tempo vai, com custo baixo o bastante para usar em produção.

---

## 🗺️ 1. As ferramentas

| Ferramenta | Pergunta que responde | Custo |
| --- | --- | --- |
| Profile de **CPU** | Onde o tempo de processador é gasto? | Baixo (~100 amostras/s) |
| Profile de **heap** / **allocs** | Quem aloca memória? O que está vivo? | Baixo (amostra a cada ~512 KB alocados) |
| Profile de **goroutine** | Quantas goroutines existem e onde estão paradas? | Pausa breve para coletar |
| Profile **goroutineleak** (1.27) | Quais goroutines estão presas para sempre? | Só quando coletado |
| Profile de **block** | Onde goroutines esperam (channels, `select`, mutex)? | Desligado por padrão |
| Profile de **mutex** | Onde há **disputa** por locks? | Desligado por padrão |
| Profile **threadcreate** | Onde threads do SO foram criadas? | Baixo |
| **Trace** de execução | O que cada goroutine/P/thread fez, no tempo? | Médio (alguns %) |
| **Flight recorder** (1.25) | O que aconteceu nos últimos segundos antes do problema? | Baixo, contínuo |
| `runtime/metrics`, `GODEBUG` | Números e eventos do runtime (GC, escalonador) | Mínimo |

---

## 🔥 2. Profile de CPU, do início ao fim

### 2.1 O programa

Um processador de pedidos: para cada pedido, valida o e-mail com uma expressão regular e calcula uma assinatura (JSON + SHA-256).

```go
func validarEmail(s string) bool {
	re := regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`)
	return re.MatchString(s)
}

func assinatura(p Pedido) string {
	b, _ := json.Marshal(p)
	h := sha256.Sum256(b)
	return fmt.Sprintf("%x", h[:8])
}

func main() {
	f, _ := os.Create("cpu.out")
	pprof.StartCPUProfile(f)          // ← começa a amostrar
	defer pprof.StopCPUProfile()

	fmt.Println("válidos:", processar(200_000))
}
```

Outras formas de obter o mesmo profile, sem mudar o código:

```bash
go test -bench=. -cpuprofile=cpu.out                        # de um benchmark (ver Benchmarks)
curl -o cpu.out 'http://localhost:6060/debug/pprof/profile?seconds=30'   # de um servidor (seção 4)
```

### 2.2 `top`: quem gasta mais

Saída real (`go tool pprof -top -cum app cpu.out`, linhas selecionadas):

```
Duration: 1.72s, Total samples = 1940ms (112.67%)
      flat  flat%   sum%        cum   cum%
         0     0%     0%      1.65s 85.05%  main.main
         0     0%     0%      1.65s 85.05%  main.processar
         0     0%     0%      1.34s 69.07%  main.validarEmail
     0.01s  0.52%  0.52%      1.15s 59.28%  regexp.MustCompile
         0     0%  0.52%      1.14s 58.76%  regexp.Compile (inline)
         0     0%  7.22%      0.26s 13.40%  main.assinatura
     0.01s  0.52% 15.46%      0.17s  8.76%  encoding/json.Marshal (inline)
```

| Coluna | Significado |
| --- | --- |
| `flat` | Tempo gasto **dentro** da própria função |
| `cum` | Tempo da função **mais tudo que ela chamou** |
| `Total samples = 1940ms (112.67%)` | Mais de 100% da duração: houve trabalho em **paralelo** (o GC, em outro núcleo) |

A leitura: `validarEmail` responde por **69%** do tempo, e quase tudo é `regexp.MustCompile`, **compilar** a expressão regular, não usá-la. O SHA-256, que intuitivamente pareceria o mais caro, fica com uma fração pequena.

### 2.3 `list`: qual linha

Saída real (`go tool pprof -list main.validarEmail app cpu.out`):

```
ROUTINE ======================== main.validarEmail in /tmp/gt/ex/n47/cpu/main.go
         0      1.34s (flat, cum) 69.07% of Total
         .          .     20:func validarEmail(s string) bool {
         .      1.15s     21:	re := regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`)
         .      190ms     22:	return re.MatchString(s)
         .          .     23:}
```

### 2.4 Corrigir e medir

```go
var reEmail = regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`) // ✅ compila uma vez

func validarEmail(s string) bool {
	return reEmail.MatchString(s)
}
```

Tempo total real: **1,754 s → 0,359 s** (4,9× mais rápido) com uma mudança de duas linhas que nenhum palpite teria priorizado. É o ciclo: **profile → hipótese → correção → medição** (com [[Benchmarks]] e `benchstat` para confirmar).

### 2.5 Os comandos do `pprof`

```bash
go tool pprof app cpu.out               # modo interativo
(pprof) top                             # top por flat
(pprof) top -cum                        # top por cum
(pprof) list validarEmail               # código anotado
(pprof) peek MustCompile                # quem chama e quem é chamado
(pprof) web                             # grafo de chamadas (precisa do graphviz)

go tool pprof -http=:8081 app cpu.out   # interface web
go tool pprof -diff_base=antes.out app depois.out   # comparar dois profiles
```

Na interface web (`-http`), o padrão desde o Go 1.26 é o **flame graph**: cada retângulo é uma função, a largura é proporcional ao tempo, e as funções chamadas ficam empilhadas acima de quem as chamou. A largura de `regexp.MustCompile` saltaria aos olhos.

---

## 💾 3. Profiles de memória

O mesmo programa, profile `allocs`, ordenado por **bytes alocados no total** (`-sample_index=alloc_space`). Saída real:

```
      flat  flat%   sum%        cum   cum%
  271.12MB 22.35% 22.35%   271.12MB 22.35%  regexp/syntax.(*compiler).inst
  241.53MB 19.91% 42.25%   241.53MB 19.91%  regexp/syntax.(*parser).newRegexp
  190.15MB 15.67% 57.93%   190.15MB 15.67%  regexp.onePassCopy
   67.51MB  5.56% 63.49%    94.01MB  7.75%  regexp/syntax.(*Regexp).Simplify
   62.02MB  5.11% 68.60%   188.02MB 15.50%  regexp.makeOnePass
```

O mesmo culpado, visto pelo lado da memória: compilar a regex 200 mil vezes alocou **mais de 1 GB** no total. Cada byte alocado é trabalho a mais para o coletor de lixo (ver [[Garbage Collector]]), e isso aparece no profile de CPU como `runtime.mallocgc` e `runtime.growslice`.

| `-sample_index` | Mostra | Responde |
| --- | --- | --- |
| `alloc_space` | Bytes alocados desde o início | "Quem gera mais trabalho para o GC?" |
| `alloc_objects` | Número de alocações | "Quem aloca mais vezes?" |
| `inuse_space` | Bytes **vivos agora** | "Quem está segurando memória?" (vazamentos) |
| `inuse_objects` | Objetos vivos agora | Idem, por contagem |

> 💡 O profile `heap` e o `allocs` têm os mesmos dados; mudam só o índice padrão (`inuse_space` vs `alloc_space`). Para entender **por que** algo foi para a heap, combine com `go build -gcflags=-m` (ver [[Memória e Escape Analysis]]).

---

## 🌐 4. Em produção: `net/http/pprof`

Importar o pacote `net/http/pprof` registra handlers em `/debug/pprof/` no **`http.DefaultServeMux`**:

```go
import _ "net/http/pprof" // registra /debug/pprof/ no http.DefaultServeMux

func main() {
	go func() { // servidor de diagnóstico separado, só em localhost
		log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
	}()
	app := http.NewServeMux() // a aplicação usa o SEU mux
	...
	log.Fatal(http.ListenAndServe(":8080", app))
}
```

O índice `/debug/pprof/` lista os profiles disponíveis. Saída real (contagem e nome):

```
2 allocs          0 block          0 cmdline        4 goroutine
0 goroutineleak   2 heap           0 mutex          0 profile
0 symbol          6 threadcreate   0 trace
```

| Endpoint | Uso |
| --- | --- |
| `/debug/pprof/profile?seconds=30` | Profile de CPU de 30 s |
| `/debug/pprof/heap`, `/allocs` | Memória |
| `/debug/pprof/goroutine?debug=2` | Pilha **completa** de cada goroutine (texto) |
| `/debug/pprof/goroutineleak` | Goroutines presas para sempre (Go 1.27) |
| `/debug/pprof/block`, `/mutex` | Espera e disputa (ligar antes; seção 5) |
| `/debug/pprof/trace?seconds=5` | Trace de execução |

```bash
go tool pprof -http=:8081 http://servidor:6060/debug/pprof/heap   # baixa e abre direto
```

> ⚠️ **Segurança:** os endpoints expõem detalhes internos (código, memória) e podem custar CPU. Nunca os exponha na porta pública: use um listener separado em `127.0.0.1` (acessado por túnel SSH ou pelo `kubectl port-forward`) ou proteja com autenticação. É mais um motivo para **não** registrar a aplicação no `DefaultServeMux` (ver [[HTTP (net-http)]]).

---

## 🔒 5. Disputa e bloqueio: profiles de mutex e block

Desligados por padrão (têm custo), são ligados com uma taxa de amostragem:

```go
runtime.SetMutexProfileFraction(1)   // 1 = registra toda disputa; 100 = 1 em 100
runtime.SetBlockProfileRate(1)       // registra todo bloqueio com duração >= 1 ns
```

Um "banco" em que oito goroutines depositam, mas o lock é segurado durante uma "E/S" de 100 µs. Saída real do profile de mutex:

```go
func depositar() {
	mu.Lock()
	time.Sleep(100 * time.Microsecond) // ❌ "E/S" segurando o lock
	saldo++
	mu.Unlock()
}
```

```
Type: delay
Showing nodes accounting for 12.29s, 100% of 12.29s total
      flat  flat%   sum%        cum   cum%
    12.29s   100%   100%     12.29s   100%  sync.(*Mutex).Unlock (inline)
         0     0%   100%     12.29s   100%  main.depositar
         0     0%   100%     12.29s   100%  main.main.func1
```

**12,29 segundos** de espera somada, num programa que rodou por menos de meio segundo: oito goroutines passaram quase todo o tempo esperando umas às outras. O profile atribui o atraso ao **`Unlock`** (o ponto em que quem segurava o lock finalmente o soltou), apontando a seção crítica responsável, `depositar`. A correção é a de [[sync.WaitGroup e sync.Mutex]]: tirar a E/S de dentro do lock.

---

## 🧵 6. O que cada goroutine está fazendo

### 6.1 Dump com `SIGQUIT`

Mandar `SIGQUIT` (Ctrl+\ no terminal, ou `kill -QUIT <pid>`) a um programa Go faz o runtime **imprimir a pilha de todas as goroutines** e sair. Saída real (resumida):

```
SIGQUIT: quit
PC=0x40d96e m=0 sigcode=0

goroutine 1 gp=0x36b00cf781e0 m=0 mp=0x573240 [running]:
syscall.Kill(0x27?, 0x0?)
main.main()

goroutine 2 gp=0x36b00cf78780 m=nil [force gc (idle)]:
runtime.forcegchelper()

goroutine 3 gp=0x36b00cf78d20 m=nil [GC sweep wait]:
runtime.bgsweep(0x36b00cfe6000)
...
```

É a ferramenta de primeira linha para um processo **travado**: o motivo entre colchetes (`[chan receive]`, `[sync.Mutex.Lock]`, `[IO wait]`, `[select]`) diz onde cada goroutine parou. Repare que o runtime tem goroutines próprias (o "ajudante" do GC forçado, o varredor, o devolvedor de memória). `GOTRACEBACK=all` (padrão no `SIGQUIT`) ou `crash` (gera core dump) controlam o detalhe.

### 6.2 Labels: de qual requisição era esta goroutine?

`pprof.Do` anexa **labels** (pares chave-valor) à goroutine. Eles aparecem nos profiles (dá para filtrar: `go tool pprof -tagfocus=rota=/pedidos`) e, desde o **Go 1.27**, no **cabeçalho do stack trace**. Saída real de uma requisição rotulada que travou:

```go
func atenderRequisicao(ctx context.Context, rota string) {
	pprof.Do(ctx, pprof.Labels("rota", rota, "tenant", "acme"), func(ctx context.Context) {
		mu.Lock() // ❌ o lock já está com outra pessoa: trava para sempre
	})
}
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.Mutex.Lock] {rota: /pedidos, tenant: acme}:
```

Com `GODEBUG=tracebacklabels=0` (útil se os labels carregam dados sensíveis), a linha volta a ser só `goroutine 1 [sync.Mutex.Lock]:`.

> ⚠️ `pprof.Do` restaura os labels anteriores com um `defer` ao sair. Num **panic**, esse `defer` roda antes de o stack trace ser impresso, então o panic **não** mostra os labels; travamentos, `SIGQUIT` e profiles mostram.

### 6.3 Goroutines vazadas

O profile `goroutineleak` (Go 1.27) foi demonstrado em [[Goroutines]]: ele usa o coletor de lixo para provar que uma goroutine bloqueada nunca poderá acordar e aponta a linha exata onde ela está presa.

---

## 🎞️ 7. Traces de execução

Um **profile** agrega ("70% do tempo foi em X"); um **trace** é uma **linha do tempo** com cada evento: goroutine criada, bloqueada, desbloqueada, rodando em qual P, syscalls, ciclos do GC, pausas. É a ferramenta para problemas de **latência** e de **concorrência** ("por que esta requisição levou 300 ms se a CPU estava ociosa?").

```bash
go test -trace=trace.out ...
curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5'
go tool trace trace.out          # abre a interface web: linha do tempo por P, por goroutine,
                                 # latência de escalonamento, bloqueios por rede e por sync
```

No código, regiões e tarefas anotam o trace: `trace.WithRegion(ctx, "validar", f)`, `trace.NewTask(ctx, "pedido")`.

### 7.1 Flight recorder (Go 1.25+)

Traces longos são enormes, e o problema raro nunca acontece enquanto você grava. O **flight recorder** ("caixa-preta") grava o trace **continuamente** num buffer circular na memória, guardando só os últimos segundos, e você pede um **snapshot** quando algo estranho acontece. Saída real:

```go
fr := trace.NewFlightRecorder(trace.FlightRecorderConfig{
	MinAge:   2 * time.Second, // manter pelo menos os últimos ~2 s
	MaxBytes: 8 << 20,         // e no máximo ~8 MB
})
fr.Start()
defer fr.Stop()

for i := range 50 {
	inicio := time.Now()
	trabalho(i)                               // a requisição 37 demora 300 ms
	if d := time.Since(inicio); d > 100*time.Millisecond {
		once.Do(func() {                      // só o primeiro evento lento gera um snapshot
			var buf bytes.Buffer
			n, err := fr.WriteTo(&buf)
			os.WriteFile("lento.trace", buf.Bytes(), 0o644)
			fmt.Printf("requisição %d levou %v → snapshot do trace: %d KB (err=%v)\n", ...)
		})
	}
}
```

```
requisição 37 levou 300ms → snapshot do trace: 4 KB (err=<nil>)
```

O arquivo `lento.trace` se abre normalmente no `go tool trace`, mostrando exatamente o que o programa fazia antes e durante a lentidão.

---

## 📈 8. Números do runtime

### 8.1 `runtime/metrics`

API estável com **107 métricas** do runtime no Go 1.27 (GC, heap, escalonador, mutex, CPU). Saída real:

```go
amostras := []metrics.Sample{
	{Name: "/sched/goroutines:goroutines"},
	{Name: "/sched/gomaxprocs:threads"},
	{Name: "/gc/cycles/total:gc-cycles"},
	{Name: "/gc/heap/allocs:bytes"},
	{Name: "/memory/classes/heap/objects:bytes"},
	{Name: "/sched/latencies:seconds"},
}
metrics.Read(amostras)
```

```
/sched/goroutines:goroutines           58
/sched/gomaxprocs:threads              2
/gc/cycles/total:gc-cycles             7
/gc/heap/allocs:bytes                  20689640
/memory/classes/heap/objects:bytes     3172384
/sched/latencies:seconds               histograma com 23 amostras
total de métricas disponíveis: 107
```

É o que exportadores (Prometheus, OpenTelemetry) leem para os painéis de um serviço Go. `/sched/latencies` (quanto tempo goroutines prontas esperam para rodar) é um ótimo indicador de CPU insuficiente ou `GOMAXPROCS` mal ajustado.

### 8.2 `GODEBUG`: o runtime falando

```bash
GODEBUG=gctrace=1 ./app                 # uma linha por ciclo do GC
GODEBUG=schedtrace=1000 ./app           # estado do escalonador a cada 1000 ms
GODEBUG=inittrace=1 ./app               # custo de cada init() de pacote
```

Saída real de `gctrace=1`:

```
gc 1 @0.012s 5%: 0.071+2.0+0.062 ms clock, 0.14+0.44/1.0/0+0.12 ms cpu, 3->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 2 P
gc 2 @0.024s 6%: 0.022+1.4+0.005 ms clock, 0.044+1.2/0.64/0+0.010 ms cpu, 3->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 2 P
```

Cada campo (fases com pausa, heap antes/depois, meta) será dissecado em [[Garbage Collector]]; o `schedtrace`, em [[Scheduler do Go]].

---

## 🎯 9. PGO: otimização guiada por profile (Go 1.21+)

O compilador pode usar um **profile de CPU de produção** para otimizar melhor as partes quentes: inlinar mais agressivamente as funções chamadas com frequência e **desvirtualizar** chamadas de interface (trocar a chamada indireta pela direta quando o profile mostra que quase sempre é o mesmo tipo concreto, ver [[Interfaces]]).

```bash
curl -o default.pgo 'http://producao:6060/debug/pprof/profile?seconds=30'
mv default.pgo ./cmd/servidor/      # na pasta do pacote main
go build ./cmd/servidor             # usa default.pgo automaticamente (-pgo=auto)
go version -m servidor | grep pgo   # confirma
```

Saída real de `go version -m` de um binário compilado assim:

```
	build	-pgo=/tmp/gt/ex/n47/cpu/default.pgo
```

E um resultado honesto: no programa desta nota, já corrigido, **não houve ganho mensurável** (391 ms sem PGO, 414 ms com, dentro do ruído), porque o profile tinha só ~0,4 s de amostras de um programa minúsculo. A documentação oficial relata ganhos típicos de **2% a 14%** em serviços reais, com profiles **representativos** de produção (por isso o fluxo recomendado é coletar periodicamente do ambiente real e versionar o `default.pgo` no repositório).

---

## 🔬 10. Por dentro: como o profiler de CPU funciona

```
pprof.StartCPUProfile
   └─ pede ao kernel um timer por THREAD baseado em tempo de CPU
      (timer_create com CLOCK_THREAD_CPUTIME_ID no Linux; setitimer(ITIMER_PROF) em outros)
            │  a cada 10 ms de CPU consumida por uma thread
            ▼
      o kernel entrega SIGPROF àquela thread
            ▼
      o tratador de sinal do runtime percorre a pilha da goroutine atual
      (usando a tabela .gopclntab do binário) e grava a amostra num buffer
            ▼
      uma goroutine de fundo esvazia o buffer no formato pprof (protobuf compactado)
```

| Detalhe | Consequência |
| --- | --- |
| Amostra por tempo de **CPU**, não de parede | Goroutines bloqueadas (esperando rede, lock) **não aparecem** no profile de CPU: para elas, use block/mutex/trace |
| 100 Hz | Poucas amostras em programas curtos (o caso do PGO acima) |
| Pilhas via `.gopclntab` | Funcionam mesmo em binários com `-ldflags=-s -w` (ver [[Build, Flags e Cross-compilation]]) |
| Funções inlinadas aparecem como `(inline)` | O compilador grava a árvore de inlining para reconstruir a pilha lógica |
| Formato pprof | Padrão aberto (protobuf), lido por outras ferramentas e pelos profilers contínuos de nuvem |

O profile de memória funciona de outro jeito: o alocador **amostra** uma alocação a cada ~512 KB alocados (`runtime.MemProfileRate`, ajustável), registrando a pilha e o tamanho, e o GC informa quando o objeto morre (para o `inuse`).

| Ferramenta de SO | Equivalente em Go |
| --- | --- |
| `perf record` / `perf top` (amostragem por contadores de hardware) | Profile de CPU (por `SIGPROF`); o `perf` também funciona com binários Go |
| `strace -c`, `ltrace` | Trace de execução (syscalls, bloqueios) |
| `gdb` com `thread apply all bt` | Dump de `SIGQUIT`; o depurador Delve (`dlv`) |
| `/proc/<pid>/status`, `top` | `runtime/metrics` |
| `valgrind --tool=massif` | Profile de heap |

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Otimizar sem profile | Tempo gasto na parte errada | Profile primeiro |
| Profile de CPU de um programa que espera | Nada aparece | Block/mutex profile, trace |
| Ler só `flat` | Não vê que a função chama algo caro | Olhe `cum` e o flame graph |
| `/debug/pprof` exposto publicamente | Vazamento de informação e DoS | Listener em localhost / autenticação |
| `net/http/pprof` + `DefaultServeMux` na porta da aplicação | Endpoints expostos sem querer | Mux próprio para a aplicação |
| Mutex/block profile sempre em taxa máxima | Custo em produção | Frações maiores (ex.: 1 em 100) |
| Profiles de poucos segundos | Ruído estatístico | 30 s ou mais, sob carga representativa |
| PGO com profile de teste sintético | Otimiza o caminho errado | Profile de produção |
| Esperar labels no stack de um panic | Não aparecem | Labels aparecem em travamentos, `SIGQUIT` e profiles |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Profile | Custo associado a pilhas; por amostragem |
| CPU | `StartCPUProfile`, `-cpuprofile`, `/debug/pprof/profile`; 100 Hz via `SIGPROF` |
| `pprof` | `top`, `top -cum`, `list`, `peek`, `-http` (flame graph), `-diff_base` |
| Memória | `allocs`/`heap`; `alloc_space` vs `inuse_space` |
| Mutex/block | Desligados por padrão; `SetMutexProfileFraction`, `SetBlockProfileRate` |
| Goroutines | `/debug/pprof/goroutine?debug=2`, `SIGQUIT`, `goroutineleak` (1.27) |
| Labels | `pprof.Do`; filtros no pprof; no cabeçalho do traceback (1.27) |
| Trace | Linha do tempo para latência e concorrência; `go tool trace` |
| Flight recorder (1.25) | Buffer circular contínuo; snapshot quando algo dá errado |
| `runtime/metrics` | Métricas estáveis para monitoramento |
| `GODEBUG` | `gctrace`, `schedtrace`, `inittrace`... |
| PGO (1.21) | `default.pgo` de produção; ganhos de 2–14% em serviços reais |

---

## 🔗 Conexões

**Dentro de Go**
- Medir uma operação isolada → [[Benchmarks]]
- Profiles a partir de testes → [[Testes com go test]]
- Escape analysis e alocações → [[Memória e Escape Analysis]]
- O que `gctrace` mostra → [[Garbage Collector]]
- O que `schedtrace` e o trace mostram → [[Scheduler do Go]]
- Goroutines vazadas e `GOMAXPROCS` → [[Goroutines]]
- Disputa por locks → [[sync.WaitGroup e sync.Mutex]]
- Labels no contexto → [[context.Context]]
- `/debug/pprof` e o `DefaultServeMux` → [[HTTP (net-http)]]
- Inlining e desvirtualização (PGO) → [[Funções — Fundamentos]], [[Interfaces]]
- `.gopclntab` e símbolos → [[Como Go Funciona]], [[Build, Flags e Cross-compilation]]
- Expressões regulares e `regexp` → [[Pacotes Importantes da Stdlib]]

**Sistemas Operacionais**
- [[Syscalls Diversas]] — sinais: `SIGPROF` (profiling), `SIGQUIT` (dump)
- [[Processos]] e [[Estados de Processos]] — o que um dump de goroutines revela
- [[Processadores]] — amostragem, contadores de hardware, `perf`
- [[Introdução ao Escalonamento]] — latência de escalonamento, a métrica `/sched/latencies`
- [[Inversão de Prioridade e RCU]] — disputa por locks, o que o profile de mutex mede

---

⬅️ **Anterior:** [[Benchmarks]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Build, Flags e Cross-compilation]] ➡️
