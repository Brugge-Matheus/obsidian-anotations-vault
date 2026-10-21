---
tags:
  - go
  - go/concorrencia
aliases:
  - Padrões de concorrência
  - Pipeline
  - Fan-out
  - Fan-in
  - Worker pool
  - errgroup
  - singleflight
  - Rate limiting
ordem: 42
modulo: "09 - Concorrência"
---
# Padrões de Concorrência

> **Nota 42 da trilha de Go.** Fechamos o módulo juntando as peças (goroutines, channels, `select`, `sync`, `context`) em **padrões** que resolvem problemas recorrentes: **pipelines**, **fan-out/fan-in**, **worker pools**, **limitar a concorrência** (semáforos e `errgroup.SetLimit`), **esperar com erro e cancelamento** (`errgroup`), **deduplicar** chamadas (`singleflight`), **limitar a taxa** (token bucket) e **retry com backoff**. Cada padrão vem com uma saída real, a lista de cuidados e o equivalente no mundo dos sistemas operacionais: thread pools, pipes do shell, semáforos de Dijkstra.
>
> **Pré-requisitos:** [[Goroutines]], [[Channels]], [[Select]], [[sync.WaitGroup e sync.Mutex]], [[context.Context]].

---

## 🧭 De onde viemos

As notas anteriores apresentaram as primitivas uma a uma. Em [[Goroutines]] vimos o vazamento por retorno antecipado e prometemos o `errgroup`; em [[Channels]], a regra de **posse** e a função que devolve `<-chan T`; em [[Select]], o `default`, o timeout e o channel nil; em [[sync.WaitGroup e sync.Mutex]], o `wg.Go`; em [[context.Context]], o cancelamento em árvore. Do lado do SO, [[Utilização de Threads]] e [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] discutiram **pools de threads** e [[Semáforos]] mostrou o produtor-consumidor.

> 💡 **Padrão de concorrência:** uma forma reconhecível e testada de organizar goroutines e a comunicação entre elas para resolver um tipo de problema. Conhecê-los evita reinventar (e reintroduzir os mesmos bugs).

Três regras valem para **todos** os padrões abaixo:

| Regra | Por quê |
| --- | --- |
| Toda goroutine tem um **caminho de saída** (fim do trabalho, channel fechado ou `ctx.Done()`) | Senão é vazamento |
| Quem cria um channel de saída é quem o **fecha** | Senão: panic ou `range` eterno |
| A concorrência é **limitada** quando há recursos finitos (conexões, arquivos, CPU, memória) | "Uma goroutine por item" com um milhão de itens esgota o sistema |

---

## 🏭 1. Pipeline: estágios ligados por channels

Cada **estágio** recebe de um channel, faz uma transformação e envia para outro. Todos rodam ao mesmo tempo, cada um numa goroutine. Saída real:

```go
// Cada estágio: recebe um <-chan, devolve outro <-chan, e para se ctx for cancelado.
func gerar(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func estagio[T, R any](ctx context.Context, in <-chan T, f func(T) (R, bool)) <-chan R {
	out := make(chan R)
	go func() {
		defer close(out)
		for v := range in {
			r, ok := f(v)
			if !ok {
				continue                  // f pode filtrar
			}
			select {
			case out <- r:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel() // ✅ se o consumidor parar cedo, os estágios encerram

nums := gerar(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
pares := estagio(ctx, nums, func(n int) (int, bool) { return n, n%2 == 0 })
quadrados := estagio(ctx, pares, func(n int) (string, bool) { return fmt.Sprintf("%d²=%d", n, n*n), true })

for q := range quadrados {
	fmt.Print(q, " ")
}
// 2²=4 4²=16 6²=36 8²=64 10²=100
```

```
gerar ──[chan int]──► filtrar pares ──[chan int]──► elevar ao quadrado ──[chan string]──► consumidor
(goroutine)           (goroutine)                    (goroutine)                           (main)
```

É o mesmo desenho de um pipeline de shell, `cat log | grep erro | sort | uniq -c`, em que cada programa é um processo e cada `|` é um pipe do kernel (ver [[Interpretador de Comandos Shell]]).

| Cuidado | Detalhe |
| --- | --- |
| Todo envio dentro de `select` com `ctx.Done()` | Sem isso, se o consumidor desistir, os estágios ficam presos no envio para sempre |
| Cada estágio fecha o **seu** channel de saída | O `range` do estágio seguinte termina em cascata |
| Pipeline **sequencial** (sem paralelismo) e barato | Prefira **iteradores** (`iter.Seq`): sem goroutines, sem channels. Medido no Go 1.27, somando 1 milhão de inteiros: **0,34 ns** por valor com iterador contra **230 ns** com channel (ver [[Iteradores]]) |
| Estágios com custos muito diferentes | O mais lento dita o ritmo; paralelize só ele com fan-out |

---

## 🌬️ 2. Fan-out e fan-in

> 💡 **Fan-out:** várias goroutines lendo do **mesmo** channel de entrada, dividindo o trabalho. **Fan-in:** juntar vários channels num só.

```
                  ┌──► worker 1 ──┐
entrada ──[ch]────┼──► worker 2 ──┼──[ch]──► saída     (fan-out e depois fan-in)
                  └──► worker 3 ──┘
```

O fan-out vem de graça: vários receptores no mesmo channel recebem valores **diferentes** (cada valor vai para um só). O fan-in genérico, com cancelamento:

```go
func juntar[T any](ctx context.Context, entradas ...<-chan T) <-chan T {
	out := make(chan T)
	var wg sync.WaitGroup
	for _, in := range entradas {
		wg.Go(func() {
			for v := range in {
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		})
	}
	go func() {
		wg.Wait()       // quando TODAS as entradas acabaram...
		close(out)      // ...fecha a saída (um único coordenador fecha)
	}()
	return out
}
```

(Para exatamente dois channels, o `select` com channel nil de [[Select]] também resolve, sem goroutines extras.)

> ⚠️ Fan-in **não preserva a ordem**: os resultados saem na ordem em que ficam prontos. Se a ordem importa, veja a seção 3.2.

---

## 👷 3. Worker pool: N trabalhadores, uma fila

Um número **fixo** de goroutines consome uma fila de jobs. É o fan-out com a quantidade de workers sob controle. Saída real, com 9 jobs de 100 ms e 3 workers:

```go
jobs := make(chan Job)
resultados := make(chan Resultado)

const workers = 3
var wg sync.WaitGroup
for w := range workers {
	wg.Go(func() {
		for j := range jobs { // cada worker pega o próximo job livre
			time.Sleep(100 * time.Millisecond) // "processar"
			resultados <- Resultado{JobID: j.ID, Worker: w}
		}
	})
}

go func() { // produtor
	defer close(jobs)
	for i := range 9 {
		jobs <- Job{ID: i}
	}
}()

go func() { // fecha resultados quando TODOS os workers terminarem
	wg.Wait()
	close(resultados)
}()

for r := range resultados { ... }
```

```
jobs por worker: map[0:3 1:3 2:3]
9 jobs de 100ms com 3 workers: 300ms
```

```
produtor ──► [ jobs ] ──┬──► worker 0 ──┐
                        ├──► worker 1 ──┼──► [ resultados ] ──► consumidor
                        └──► worker 2 ──┘
           close(jobs) faz os workers saírem do range; wg.Wait() → close(resultados)
```

A sequência de fechamentos é o ponto delicado: **o produtor fecha `jobs`** → os workers terminam o `range` → **o coordenador fecha `resultados`** depois do `wg.Wait()` → o consumidor termina o seu `range`.

### 3.1 Quantos workers?

| Tipo de trabalho | Ponto de partida |
| --- | --- |
| CPU (*CPU-bound*) | `runtime.GOMAXPROCS(0)`: mais que isso só disputa núcleo |
| E/S (rede, banco) | Limitado pelo recurso do outro lado: o tamanho do pool de conexões, o limite da API |
| Misto | Meça (ver [[Benchmarks]] e [[Profiling e Diagnóstico]]) |

### 3.2 Preservando a ordem

Se cada resultado tem um índice, escreva direto na posição dele num slice pré-alocado. Posições diferentes não disputam memória:

```go
resultados := make([]Resultado, len(itens))
var wg sync.WaitGroup
for i, item := range itens {
	wg.Go(func() {
		resultados[i] = processar(item)   // cada goroutine escreve SÓ no seu índice
	})
}
wg.Wait()                                 // depois do Wait, ler é seguro
```

### 3.3 Paralelo com o SO

O worker pool é o **pool de threads** de [[Utilização de Threads]]: criar threads sob demanda é caro, então o servidor cria N threads no início e elas ficam esperando trabalho numa fila protegida por mutex + variável de condição. Em Go, a fila é o channel. A diferença é o custo: como uma goroutine custa ~2 KB, o próprio `net/http` dispensa o pool e cria **uma goroutine por conexão** (ver [[HTTP (net-http)]]). O pool continua útil para **limitar** o uso de um recurso, não para economizar criação.

---

## 🚦 4. Limitar a concorrência

### 4.1 Semáforo com channel

Um channel com buffer de capacidade N funciona como um **semáforo contador** de Dijkstra ([[Semáforos]]): enviar ocupa uma vaga (`down`), receber libera (`up`):

```go
sem := make(chan struct{}, 10)        // no máximo 10 ao mesmo tempo

for _, url := range urls {
	wg.Go(func() {
		sem <- struct{}{}             // down: ocupa uma vaga (bloqueia se as 10 estão ocupadas)
		defer func() { <-sem }()      // up: libera
		baixar(url)
	})
}
wg.Wait()
```

Funciona graças à regra do modelo de memória vista em [[Channels]]: num channel de capacidade C, o k-ésimo recebimento acontece antes do (k+C)-ésimo envio completar.

> ⚠️ Esse formato ainda cria **uma goroutine por item** (as que excedem o limite ficam bloqueadas no `sem <-`). Com um milhão de itens, é um milhão de goroutines paradas. Para limitar também a **criação**, adquira a vaga **antes** do `go`, ou use o `SetLimit` abaixo.

Para "pesos" diferentes (uma tarefa ocupa 3 vagas, outra 1), existe `golang.org/x/sync/semaphore.Weighted`, com `Acquire(ctx, n)` cancelável.

### 4.2 `errgroup.SetLimit`

O `errgroup` (próxima seção) tem um limite embutido, e o `g.Go` **bloqueia** quem chama até haver vaga, então não acumula goroutines. Saída real:

```go
g, _ := errgroup.WithContext(context.Background())
g.SetLimit(3) // no máximo 3 goroutines ao mesmo tempo
for range 12 {
	g.Go(func() error { // bloqueia aqui se já houver 3 rodando
		... // mede o pico de goroutines ativas
		time.Sleep(50 * time.Millisecond)
		return nil
	})
}
g.Wait()
```

```
pico de goroutines simultâneas: 3
12 tarefas de 50ms, limite 3: 200ms
```

Doze tarefas em lotes de três: 4 × 50 ms = 200 ms. Há também `g.TryGo(f)`, que não bloqueia e devolve `false` se não houver vaga.

---

## 🧯 5. `errgroup`: esperar, com erro e cancelamento

`sync.WaitGroup` espera, mas não sabe de erros. O pacote **`golang.org/x/sync/errgroup`** (mantido pela equipe de Go, fora da biblioteca padrão) junta três coisas: esperar um grupo, guardar o **primeiro erro** e **cancelar** as outras quando uma falha. Saída real com quatro downloads, um deles falhando:

```go
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error { return baixar(ctx, "/a", 10*time.Millisecond, false) })
g.Go(func() error { return baixar(ctx, "/b", 30*time.Millisecond, true) })   // falha
g.Go(func() error { return baixar(ctx, "/c", 500*time.Millisecond, false) })
g.Go(func() error { return baixar(ctx, "/d", 500*time.Millisecond, false) })

err := g.Wait()
fmt.Println("Wait:", err, "após", ...)
fmt.Println("Cause do ctx:", context.Cause(ctx))
```

```
  ok: /a
  cancelado: /d
  cancelado: /c
Wait: baixar /b: HTTP 503 após 30ms
Cause do ctx: baixar /b: HTTP 503
```

`/c` e `/d` levariam 500 ms, mas pararam aos 30 ms, quando `/b` falhou. O contexto devolvido por `WithContext` é cancelado com o **erro como causa** (ver [[context.Context]]), e `Wait` devolve esse primeiro erro.

| Método | Faz |
| --- | --- |
| `errgroup.WithContext(ctx)` | Cria o grupo e um `ctx` filho cancelado no primeiro erro (ou quando `Wait` retorna) |
| `g.Go(f)` | Roda `f` numa goroutine (respeitando o limite) |
| `g.TryGo(f)` | Idem, mas só se houver vaga |
| `g.SetLimit(n)` | Limita goroutines simultâneas |
| `g.Wait()` | Espera todas; devolve o primeiro erro |

> ⚠️ O `errgroup` **não** recupera panics: um panic numa goroutine do grupo derruba o processo, como qualquer goroutine (a decisão está comentada no código-fonte: propagar o panic para o `Wait` esconderia o stack trace e poderia causar deadlocks). E as funções **precisam** usar o `ctx` do grupo, senão o cancelamento não as alcança.

> ✅ Este é o substituto direto do código que vazava goroutines em [[Goroutines]]: em vez de retornar no primeiro erro e abandonar as outras, o `errgroup` cancela todas e **espera** cada uma sair.

---

## 🧲 6. `singleflight`: uma consulta para muitos pedidos iguais

Quando um item popular sai do cache, cem requisições simultâneas podem disparar cem consultas idênticas ao banco (o *cache stampede*). O `golang.org/x/sync/singleflight` garante que, para a **mesma chave**, só uma execução acontece por vez; as outras **esperam e recebem o mesmo resultado**. Saída real:

```go
var grupo singleflight.Group
for range 100 { // 100 requisições simultâneas pela MESMA chave
	wg.Go(func() {
		_, _, shared := grupo.Do("produto:42", func() (any, error) {
			return buscarNoBanco("42")   // leva 50 ms
		})
		...
	})
}
```

```
requisições: 100 | consultas ao banco: 1 | receberam resultado compartilhado: 100
```

> ⚠️ Se a execução falhar, **todos** recebem o mesmo erro, e se ela travar, todos travam. Use com um `context` com prazo dentro da função, e `DoChan` se precisar de `select`.

---

## ⏱️ 7. Limitar a taxa (*rate limiting*)

Limitar a **concorrência** é "no máximo N ao mesmo tempo"; limitar a **taxa** é "no máximo N por segundo". O algoritmo clássico é o **token bucket**: um balde com capacidade B recebe R fichas por segundo; cada operação consome uma ficha; sem fichas, espera. Assim se permite uma **rajada** de até B e uma taxa sustentada de R. O pacote `golang.org/x/time/rate` implementa isso. Saída real com 5 por segundo e rajada de 3:

```go
lim := rate.NewLimiter(rate.Limit(5), 3)   // 5 eventos/s, rajada de até 3
for i := range 8 {
	lim.Wait(ctx)                          // bloqueia até haver uma ficha (cancelável)
	fmt.Printf("req %d em %4dms\n", i, time.Since(inicio).Milliseconds())
}
fmt.Println("Allow() agora (sem esperar):", lim.Allow())
```

```
req 0 em    0ms
req 1 em    0ms
req 2 em    0ms      ← a rajada: as 3 fichas iniciais
req 3 em  200ms      ← depois, uma ficha a cada 1/5 s
req 4 em  400ms
req 5 em  600ms
req 6 em  801ms
req 7 em 1000ms
Allow() agora (sem esperar): false
```

| Método | Uso |
| --- | --- |
| `Wait(ctx)` | Cliente que deve respeitar o limite de uma API: espera a vez |
| `Allow()` | Servidor protegendo-se: se não há ficha, responde `429 Too Many Requests` na hora |
| `Reserve()` | Saber quanto tempo esperar sem bloquear |

Implementações sem dependência existem (um `time.Ticker` alimentando um channel com buffer), mas o `rate` calcula as fichas **sob demanda** a partir do tempo decorrido, sem goroutine nem ticker. É o mesmo algoritmo que o módulo `limit_req` do Nginx usa (uma variação, o *leaky bucket*).

---

## 🔁 8. Retry com backoff exponencial e jitter

Falhas **temporárias** (rede, 503, timeout) costumam passar se você tentar de novo, mas tentar **imediatamente** e **todos ao mesmo tempo** piora a sobrecarga do outro lado. A receita: esperar cada vez mais (**backoff exponencial**), com um sorteio (**jitter**) para os clientes não sincronizarem, e respeitando o contexto. Saída real:

```go
var ErrTemporario = errors.New("serviço indisponível")

func comRetry(ctx context.Context, tentativas int, base time.Duration, f func() error) error {
	var err error
	for i := range tentativas {
		if err = f(); err == nil || !errors.Is(err, ErrTemporario) {
			return err // sucesso, ou erro que não adianta repetir
		}
		if i == tentativas-1 {
			break
		}
		espera := base << i                  // 100ms, 200ms, 400ms...
		espera = espera/2 + rand.N(espera/2) // jitter: entre 50% e 100%
		fmt.Printf("  tentativa %d falhou; esperando %v\n", i+1, espera.Round(time.Millisecond))
		select {
		case <-time.After(espera):
		case <-ctx.Done():
			return fmt.Errorf("desistiu após %d tentativas: %w", i+1, context.Cause(ctx))
		}
	}
	return fmt.Errorf("falhou após %d tentativas: %w", tentativas, err)
}
```

```
  tentativa 1 falhou; esperando 96ms
  tentativa 2 falhou; esperando 169ms
  tentativa 3 falhou; esperando 254ms
resultado: <nil> | chamadas: 4

  tentativa 1 falhou; esperando 64ms         ← agora com prazo total de 250 ms
  tentativa 2 falhou; esperando 137ms
  tentativa 3 falhou; esperando 325ms
resultado: desistiu após 3 tentativas: context deadline exceeded
```

| Cuidado | Por quê |
| --- | --- |
| Só repita erros **temporários** | Repetir um 400 (requisição inválida) nunca vai funcionar |
| Só repita operações **idempotentes** (ou com chave de idempotência) | Repetir um "debitar R$ 100" que na verdade funcionou debita duas vezes |
| Limite tentativas **e** tempo total (contexto) | Senão o retry vira um travamento |
| Jitter | Sem ele, mil clientes que falharam juntos voltam juntos (*thundering herd*) |

(`rand.N` é do `math/rand/v2`, ver [[Operações Aritméticas]].)

---

## 🧭 9. Qual padrão usar?

| Problema | Padrão |
| --- | --- |
| Transformar um fluxo em etapas, com paralelismo entre etapas | Pipeline |
| Uma etapa lenta precisa de várias goroutines | Fan-out (+ fan-in) |
| Processar uma fila com uso de recursos controlado | Worker pool |
| "N tarefas, no máximo K ao mesmo tempo, quero o primeiro erro" | `errgroup` + `SetLimit` |
| Resultados na ordem original | Slice pré-alocado indexado |
| Muitas requisições iguais ao mesmo tempo | `singleflight` |
| "No máximo N por segundo" | `rate.Limiter` |
| Falhas temporárias | Retry com backoff exponencial + jitter + contexto |
| Percorrer uma sequência, sem concorrência | Iterador, não channel |

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Estágio que envia sem `select` com `ctx.Done()` | Goroutines presas quando o consumidor para | Todo envio com caso de cancelamento |
| Fechar `resultados` dentro de um worker | `send on closed channel` nos outros | Um coordenador: `wg.Wait(); close(...)` |
| Esquecer de fechar `jobs` | Workers nunca saem do `range` | O produtor fecha |
| Goroutine por item sem limite | Arquivos/conexões/memória esgotados | Pool, semáforo ou `SetLimit` |
| Semáforo adquirido **dentro** da goroutine | Milhões de goroutines paradas | Adquira antes do `go` ou use `SetLimit` |
| `errgroup` com funções que ignoram o `ctx` | Cancelamento não acontece | Use o `ctx` do grupo |
| Esperar ordem no fan-in | Resultados embaralhados | Índices |
| Retry sem limite ou sem jitter | Sobrecarga em cascata | Tentativas, prazo, jitter |
| Retry de operação não idempotente | Efeito duplicado | Chave de idempotência |
| Pipeline de channels para tarefa sequencial | Lento e frágil | Iteradores |

---

## 📋 Resumo

| Padrão | Ponto-chave |
| --- | --- |
| Pipeline | Estágios `<-chan` → `<-chan`; cada um fecha a sua saída; envios canceláveis |
| Fan-out / fan-in | Vários leitores no mesmo channel; juntar com WaitGroup + um único `close` |
| Worker pool | N fixo; produtor fecha `jobs`; coordenador fecha `resultados` |
| Ordem | Escreva no índice de um slice pré-alocado |
| Semáforo | Channel com buffer; ou `semaphore.Weighted` |
| `errgroup` | Espera + primeiro erro + cancelamento com causa; `SetLimit`, `TryGo`; não recupera panics |
| `singleflight` | Uma execução por chave; os demais compartilham o resultado |
| `rate.Limiter` | Token bucket: rajada B, taxa R; `Wait` ou `Allow` |
| Retry | Backoff exponencial + jitter + contexto; só temporários e idempotentes |

---

## 🔗 Conexões

**Dentro de Go**
- As primitivas → [[Goroutines]], [[Channels]], [[Select]], [[sync.WaitGroup e sync.Mutex]], [[context.Context]]
- CSP e escolha channel vs mutex → [[Fundamentos de Concorrência]]
- Funções genéricas como `estagio[T, R]` → [[Generics]]
- A alternativa sem goroutines para sequências → [[Iteradores]]
- `errors.Is` para classificar erros temporários → [[Wrapping e Unwrapping de Erros]]
- `math/rand/v2` → [[Operações Aritméticas]]
- Uma goroutine por conexão, sem pool → [[HTTP (net-http)]]
- Módulos fora da stdlib (`golang.org/x/...`) → [[Sistema de Módulos (go mod)]]
- Testar padrões com tempo usando `testing/synctest` → [[Testes com go test]]
- Escolher o número de workers medindo → [[Benchmarks]], [[Profiling e Diagnóstico]]

**Sistemas Operacionais**
- [[Utilização de Threads]] — pools de threads e o servidor com despachante e trabalhadores
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — os modelos de servidor
- [[Semáforos]] — `down`/`up`, o semáforo contador e o produtor-consumidor
- [[Interpretador de Comandos Shell]] — pipelines com `|`
- [[Troca de Mensagens]] — estágios que só se comunicam por mensagens
- [[Processos e Forks no CGI-FastCGI]] — um processo por requisição, o oposto caro do worker pool

**Nginx e Redes**
- [[Conexões, Workers e Keep-Alive]] — `worker_processes` e `worker_connections`, um pool fixo dimensionado pelos núcleos
- [[Load Balance]] e [[Servidores de Backup e Health Check]] — retry e falhas temporárias do lado do proxy

**Algoritmos e Estruturas de Dados**
- [[Queue (Fila)]] — a fila de jobs
- [[Hashmap]] — o `singleflight` é um map de chave → chamada em andamento

---

⬅️ **Anterior:** [[context.Context]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Sockets e o Pacote net]] ➡️
