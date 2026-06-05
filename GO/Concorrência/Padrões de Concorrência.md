---
tags:
  - go
  - go/concorrência
---
# Padrões de Concorrência

> Os padrões de concorrência em Go combinam goroutines, channels e context para construir sistemas escaláveis. São soluções estabelecidas para problemas recorrentes — conhecê-los economiza tempo e evita bugs sutis.
> 

---

## 1. Pipeline — Processamento em Estágios

Cada estágio recebe de um canal, processa e envia para outro. A cada estágio pode rodar em goroutines independentes:

```go
// Funções dos estágios retornam <-chan (somente recepção)
// Isso documenta que o estágio "produz" — não consome

func gerar(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int, len(nums))
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return   // cancelamento respeita o contexto
			}
		}
	}()
	return out
}

func mapear[T, R any](ctx context.Context, in <-chan T, fn func(T) R) <-chan R {
	out := make(chan R)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case out <- fn(v):
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func filtrar[T any](ctx context.Context, in <-chan T, pred func(T) bool) <-chan T {
	out := make(chan T)
	go func() {
		defer close(out)
		for v := range in {
			if pred(v) {
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		}
	}()
	return out
}

// Uso
ctx := context.Background()
nums := gerar(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
pares := filtrar(ctx, nums, func(n int) bool { return n%2 == 0 })
quadrados := mapear(ctx, pares, func(n int) int { return n * n })

for q := range quadrados {
	fmt.Println(q)   // 4, 16, 36, 64, 100
}
```

---

## 2. Fan-Out / Fan-In

**Fan-out:** um canal de entrada alimenta N workers em paralelo.

**Fan-in:** N canais de saída são mesclados em um único canal.

```go
// Fan-out: distribui trabalho para N workers
func fanOut[T, R any](ctx context.Context, in <-chan T, n int, fn func(T) R) []<-chan R {
	saidas := make([]<-chan R, n)
	for i := range n {   // Go 1.22+: range sobre inteiro
		out := make(chan R)
		saidas[i] = out
		go func() {
			defer close(out)
			for v := range in {
				select {
				case out <- fn(v):
				case <-ctx.Done():
					return
				}
			}
		}()
	}
	return saidas
}

// Fan-in: mescla N canais em um
func fanIn[T any](ctx context.Context, canais ...<-chan T) <-chan T {
	merged := make(chan T)
	var wg sync.WaitGroup

	encaminhar := func(ch <-chan T) {
		defer wg.Done()
		for {
			select {
			case v, ok := <-ch:
				if !ok {
					return
				}
				select {
				case merged <- v:
				case <-ctx.Done():
					return
				}
			case <-ctx.Done():
				return
			}
		}
	}

	wg.Add(len(canais))
	for _, ch := range canais {
		go encaminhar(ch)
	}

	go func() {
		wg.Wait()
		close(merged)
	}()

	return merged
}

// Uso combinado
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

jobs := gerar(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 4 workers processando em paralelo
saidas := fanOut(ctx, jobs, 4, func(n int) int {
	time.Sleep(100 * time.Millisecond)   // simulando trabalho
	return n * n
})

// Mesclar resultados
for resultado := range fanIn(ctx, saidas...) {
	fmt.Println(resultado)
}
```

---

## 3. Worker Pool

Limita o número de goroutines ativas — evita criar N goroutines para N jobs:

```go
type Job struct {
	ID    int
	Input string
}

type Resultado struct {
	JobID  int
	Output string
	Erro   error
}

func WorkerPool(
	ctx context.Context,
	numWorkers int,
	jobs <-chan Job,
) <-chan Resultado {
	resultados := make(chan Resultado, numWorkers)
	var wg sync.WaitGroup

	for range numWorkers {   // Go 1.22+
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case job, ok := <-jobs:
					if !ok {
						return
					}
					r := Resultado{JobID: job.ID}
					r.Output, r.Erro = processar(ctx, job)
					select {
					case resultados <- r:
					case <-ctx.Done():
						return
					}
				case <-ctx.Done():
					return
				}
			}
		}()
	}

	go func() {
		wg.Wait()
		close(resultados)
	}()

	return resultados
}

// Uso
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

jobs := make(chan Job, 100)
resultados := WorkerPool(ctx, 5, jobs)

// Enviar jobs
go func() {
	defer close(jobs)
	for i := 0; i < 50; i++ {
		jobs <- Job{ID: i, Input: fmt.Sprintf("dado %d", i)}
	}
}()

// Coletar resultados
for r := range resultados {
	if r.Erro != nil {
		slog.Error("job falhou", "id", r.JobID, "err", r.Erro)
		continue
	}
	fmt.Printf("job %d: %s\n", r.JobID, r.Output)
}
```

---

## 4. Semáforo — Limitar Concorrência

```go
// Channel com buffer como semáforo — simples e idiomático
type Semaforo chan struct{}

func NewSemaforo(n int) Semaforo {
	return make(Semaforo, n)
}

func (s Semaforo) Adquirir(ctx context.Context) error {
	select {
	case s <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (s Semaforo) Liberar() {
	<-s
}

// Uso — limitar chamadas a API externa
sem := NewSemaforo(10)   // máximo 10 requisições simultâneas

var wg sync.WaitGroup
for _, url := range urls {
	wg.Add(1)
	go func(u string) {
		defer wg.Done()

		if err := sem.Adquirir(ctx); err != nil {
			return
		}
		defer sem.Liberar()

		requisitarAPI(u)
	}(url)
}
wg.Wait()

// golang.org/x/sync/semaphore — implementação mais robusta da stdlib estendida
import "golang.org/x/sync/semaphore"

sem2 := semaphore.NewWeighted(10)
sem2.Acquire(ctx, 1)   // pesa 1 unidade
defer sem2.Release(1)
```

---

## 5. `errgroup` — WaitGroup com Propagação de Erro

```go
import "golang.org/x/sync/errgroup"

// errgroup.Group = WaitGroup + propagação do primeiro erro
g, ctx := errgroup.WithContext(context.Background())

// Cancelamento automático: se qualquer goroutine retornar erro,
// ctx é cancelado — as outras devem verificar e parar

for _, url := range urls {
	url := url   // capture (necessário antes do Go 1.22)
	g.Go(func() error {
		resp, err := http.Get(url)
		if err != nil {
			return fmt.Errorf("fetch %s: %w", url, err)
		}
		defer resp.Body.Close()

		// Verificar cancelamento durante processamento longo
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		return processar(resp)
	})
}

// Wait aguarda todos e retorna o primeiro erro
if err := g.Wait(); err != nil {
	log.Fatal(err)
}

// Limitar concorrência com errgroup
g2, ctx2 := errgroup.WithContext(ctx)
g2.SetLimit(5)   // máximo 5 goroutines simultâneas
for _, item := range itens {
	g2.Go(func() error {
		return processarItem(ctx2, item)
	})
}
g2.Wait()
```

---

## 6. Context — Propagação de Cancelamento e Valores

```go
// Context deve ser o PRIMEIRO parâmetro de qualquer função que pode ser cancelada
func buscarProdutos(ctx context.Context, categoria string) ([]Produto, error) {
	// Verificar cancelamento antes de trabalho caro
	if err := ctx.Err(); err != nil {
		return nil, err
	}

	// Criar sub-context com timeout mais curto para a query
	queryCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	rows, err := db.QueryContext(queryCtx, "SELECT ...", categoria)
	if err != nil {
		if errors.Is(err, context.DeadlineExceeded) {
			return nil, fmt.Errorf("query timeout: %w", err)
		}
		return nil, fmt.Errorf("query: %w", err)
	}
	defer rows.Close()

	// ...
}

// Valores no context — use com moderação, apenas para dados transversais
type contextKey string
const (
	keyRequestID contextKey = "requestID"
	keyUserID    contextKey = "userID"
)

// Helpers para type-safety
func ComRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, keyRequestID, id)
}

func RequestID(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(keyRequestID).(string)
	return id, ok
}

// No middleware HTTP
func middlewareTracing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := ComRequestID(r.Context(), uuid.New().String())
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

---

## 7. Ticker e Timer — Ações Periódicas

```go
// Executar periodicamente
func executarPeriodicamente(ctx context.Context, intervalo time.Duration, fn func()) {
	ticker := time.NewTicker(intervalo)
	defer ticker.Stop()   // SEMPRE pare o ticker — evita goroutine leak

	// Executar imediatamente na primeira vez
	fn()

	for {
		select {
		case <-ticker.C:
			fn()
		case <-ctx.Done():
			return
		}
	}
}

// Retry com backoff exponencial
func comRetry(ctx context.Context, maxTentativas int, fn func() error) error {
	var err error
	for i := range maxTentativas {
		if err = fn(); err == nil {
			return nil
		}

		if i == maxTentativas-1 {
			break
		}

		// Backoff exponencial: 100ms, 200ms, 400ms, 800ms...
		espera := time.Duration(100*(1<<i)) * time.Millisecond

		select {
		case <-time.After(espera):
		case <-ctx.Done():
			return fmt.Errorf("cancelado após %d tentativas: %w", i+1, ctx.Err())
		}
	}
	return fmt.Errorf("falhou após %d tentativas: %w", maxTentativas, err)
}
```

---

## 8. Once — Inicialização Lazy

```go
type ConexaoDB struct {
	once sync.Once
	db   *sql.DB
	err  error
}

func (c *ConexaoDB) Get() (*sql.DB, error) {
	c.once.Do(func() {
		c.db, c.err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
		if c.err == nil {
			c.err = c.db.Ping()
		}
	})
	return c.db, c.err
}

// Singleton thread-safe
var conexao ConexaoDB

func GetDB() (*sql.DB, error) {
	return conexao.Get()
}
```

---

## 9. Regras de Ouro para Concorrência em Go

```
1. "Share memory by communicating, don't communicate by sharing memory"
   — Use channels para transferir dados entre goroutines

2. Se precisar de estado compartilhado:
   - Simples (contador, flag): sync/atomic
   - Estrutura de dados: sync.Mutex / sync.RWMutex
   - Read-heavy, chaves estáveis: sync.Map

3. Sempre propague context.Context como primeiro parâmetro

4. Sempre pare goroutines:
   - Feche channels do lado do produtor
   - Use ctx.Done() para goroutines de longa duração
   - Use WaitGroup para esperar todas terminarem

5. Detecte races: go test -race ./... em CI/CD

6. Nunca copie: sync.Mutex, sync.WaitGroup, sync.Once

7. Goroutine leaks matam:
   - Toda goroutine deve ter um caminho para terminar
   - Use context para cancelamento

8. Prefira errgroup a WaitGroup quando há erros
```

---

## Conexão com Sistemas Operacionais

### Worker Pool — Thread Pools de SO ([[Utilização de Threads]])

[[Utilização de Threads]] explica que criar e destruir threads do SO é caro (~10 µs por thread, envolve syscall). A solução canônica de SO para servidores é o **thread pool**: um número fixo de threads criadas no startup, que ficam esperando trabalho em uma fila. Quando um job chega, uma thread livre o pega, processa e volta para esperar — eliminando o overhead de criação e destruição.

O Worker Pool de Go é exatamente esse padrão, implementado com goroutines em vez de threads de SO:

```
Thread Pool clássico (nível de SO):
  Startup:
    pthread_create(worker1, NULL, worker_fn, queue)  <- cria threads fixas
    pthread_create(worker2, NULL, worker_fn, queue)
    pthread_create(worker3, NULL, worker_fn, queue)

  Runtime:
    job chegou -> pthread_mutex_lock(queue_mutex)
               -> enfileira job
               -> pthread_cond_signal(queue_cond)  <- acorda um worker
               -> pthread_mutex_unlock(queue_mutex)

Worker Pool em Go:
  Startup:
    for range numWorkers { go worker(jobs) }  <- cria goroutines fixas

  Runtime:
    job chegou -> jobs <- job  <- channel faz o trabalho de mutex + cond variable
    worker goroutine desperta -> processa -> volta a esperar em jobs
```

A diferença de custo é significativa: um thread pool de SO precisa de N × 1-8 MB de stack. Um worker pool Go com as mesmas N goroutines usa N × 2 KB iniciais. Para N=100, isso é 800 MB vs 200 KB — diferença de 4000x.

A filosofia é a mesma ([[Utilização de Threads]]: pools evitam thrashing de criação/destruição), mas Go pode ter pools muito maiores porque o custo por worker é drasticamente menor.

### Pipeline — O Modelo de Pipes UNIX

O padrão Pipeline de Go tem uma genealogia direta nos pipes UNIX. Em POSIX, o operador `|` no shell conecta a saída padrão de um processo à entrada padrão do seguinte via um buffer circular no kernel:

```
Shell UNIX pipeline:
  cat arquivo.txt | grep "erro" | sort | uniq -c

  cat -> [pipe1] -> grep -> [pipe2] -> sort -> [pipe3] -> uniq
  cada processo: lê de stdin, processa, escreve em stdout
  processos rodam em paralelo, síncronos pelos pipes

Go pipeline:
  numeros := gerar(ctx, dados...)
  filtrados := filtrar(ctx, numeros, predicate)
  resultados := mapear(ctx, filtrados, transform)

  gerar -> [channel] -> filtrar -> [channel] -> mapear -> consumidor
  cada goroutine: lê do channel, processa, escreve no channel
  goroutines rodam em paralelo, síncronas pelos channels
```

Um channel sem buffer em Go é funcionalmente idêntico a um pipe UNIX sem buffer: o produtor bloqueia até o consumidor estar pronto para receber. Um channel com buffer corresponde a um pipe com buffer — o produtor pode enviar até N itens sem bloquear.

A diferença fundamental: pipes UNIX são IPC (Inter-Process Communication), comunicação entre processos distintos com espaços de endereçamento separados. Channels Go são comunicação entre goroutines dentro do mesmo processo. Pipes são tipados como bytes; channels são tipados em Go.

### Fan-Out — Fork Paralelo ([[O Modelo de Processos]])

Fan-out em Go imita o padrão de fork paralelo de sistemas operacionais. Em [[O Modelo de Processos]], `fork()` cria um processo filho que executa em paralelo com o pai. O padrão clássico de servidores UNIX é `fork()` para cada requisição recebida:

```
Fork paralelo (modelo de processos):
  while (1) {
      fd = accept(server_fd, ...);  // espera conexão
      if (fork() == 0) {
          // filho: processa esta conexão
          processar(fd);
          exit(0);
      }
      // pai: volta a esperar por mais conexões
  }

Fan-out em Go:
  for job := range jobs {
      go processar(job)   // goroutine análoga ao fork() do filho
  }
  // goroutine principal: continua criando mais goroutines
```

A diferença de custo é extrema: `fork()` cria um processo completo com cópia do espaço de endereçamento (mesmo com copy-on-write, há custo de criação de tabela de páginas). Criar uma goroutine custa ~100 ns e 2 KB — ordens de magnitude mais barato.

Por isso, onde sistemas Unix usam processos separados (Apache com MPM prefork, CGI), servidores Go usam goroutines por requisição — sem pool! O `net/http` do Go cria uma goroutine por conexão aceita porque o custo é baixo o suficiente.

### Cancelamento via Context — Sinais para Grupo de Processos ([[Processos]])

[[Processos]] descreve sinais como mecanismo de notificação assíncrona entre processos. O padrão de cancelamento via `context.Context` em Padrões de Concorrência é o análogo para goroutines:

```
Cancelamento via SIGTERM para process group:
  servidor recebe SIGTERM
    -> envia SIGTERM para process group inteiro: kill(-pgid, SIGTERM)
    -> todos os processos filhos recebem o sinal
    -> cada um executa seu signal handler e termina gracefully

Cancelamento via context em Go:
  ctx raiz cancelado (ex: SIGTERM recebido pelo programa)
    -> cancel() chamado
    -> ctx.Done() fechado em todos os contexts filhos na árvore
    -> cada goroutine em select { case <-ctx.Done() } acorda
    -> cada goroutine limpa recursos e retorna

Implementação típica para SIGTERM:
  ctx, cancel := context.WithCancel(context.Background())

  sigs := make(chan os.Signal, 1)
  signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT)
  go func() {
      <-sigs    // espera sinal do kernel
      cancel()  // converte em cancelamento de context
  }()
```

Essa é a ponte entre o modelo de sinais de [[Processos]] e o modelo de concorrência de Go: o programa recebe um sinal UNIX (interface do SO com o processo), converte para cancelamento de context (interface Go com goroutines), e propaga para toda a árvore de goroutines.

### Semáforo — Controle de Recursos Compartilhados ([[Threads POSIX]])

O Semáforo implementado com channel com buffer em Go corresponde ao semáforo clássico de Dijkstra (P/V), que é a base de controle de concorrência em SO. [[Threads POSIX]] define `sem_wait()` e `sem_post()` como a API POSIX para semáforos. Em sistemas operacionais, semáforos controlam acesso a recursos limitados:

```
Semáforo POSIX:
  sem_t sem;
  sem_init(&sem, 0, 10);  // 10 slots disponíveis
  sem_wait(&sem);         // P(): decrementa ou bloqueia
    /* usa recurso */
  sem_post(&sem);         // V(): incrementa, acorda esperando

Channel com buffer como semáforo Go:
  sem := make(chan struct{}, 10)  // 10 slots disponíveis
  sem <- struct{}{}               // P(): ocupa slot ou bloqueia
    /* usa recurso */
  <-sem                           // V(): libera slot

Capacidade do buffer = número de tokens (recursos) disponíveis
```

Esse padrão é idêntico ao que o SO usa internamente para controlar o número máximo de conexões simultâneas, threads em um pool, ou acesso a regiões de memória compartilhada — apenas com a elegante sintaxe de channels Go no lugar das chamadas POSIX.

3. Sempre propague context.Context como primeiro parâmetro

4. Sempre pare goroutines:
   - Feche channels do lado do produtor
   - Use ctx.Done() para goroutines de longa duração
   - Use WaitGroup para esperar todas terminarem

5. Detecte races: go test -race ./... em CI/CD

6. Nunca copie: sync.Mutex, sync.WaitGroup, sync.Once

7. Goroutine leaks matam:
   - Toda goroutine deve ter um caminho para terminar
   - Use context para cancelamento

8. Prefira errgroup a WaitGroup quando há erros
```