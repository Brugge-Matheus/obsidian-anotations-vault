---
tags:
  - go
  - go/concorrência
---
# Channels

> Channels são o mecanismo de comunicação entre goroutines em Go. Em vez de goroutines compartilharem memória e brigarem por acesso (com locks), elas se comunicam enviando e recebendo valores através de channels. Pense em um channel como um **cano tipado** que conecta duas goroutines — uma goroutine coloca dados de um lado, outra pega do outro lado.
> 

---

## 1. O Problema que Channels Resolvem

Imagine duas goroutines: uma que calcula um resultado e outra que precisa usar esse resultado. Como a segunda goroutine recebe o valor?

**Abordagem ruim — memória compartilhada com lock:**

```go
var resultado int
var pronto bool
var mu sync.Mutex

go func() {
	r := calcularAlgo()
	mu.Lock()
	resultado = r
	pronto = true
	mu.Unlock()
}()

// Precisa ficar "girando" verificando se terminou — CPU wasted
for {
	mu.Lock()
	if pronto {
		usar(resultado)
		mu.Unlock()
		break
	}
	mu.Unlock()
}
```

**Abordagem Go — channel:**

```go
ch := make(chan int)   // cria o canal

go func() {
	r := calcularAlgo()
	ch <- r   // envia o resultado pelo canal — bloqueia até alguém receber
}()

resultado := <-ch   // recebe do canal — bloqueia até alguém enviar
usar(resultado)
// Simples, seguro, sem locks, sem CPU wasted
```

A filosofia de Go: **"não comunique compartilhando memória — compartilhe memória comunicando".**

---

## 2. O Que É um Channel Internamente

Um channel não é uma conexão direta entre duas goroutines. É uma **fila circular** gerenciada pelo runtime, com um lock interno e duas listas de goroutines esperando:

```
Estrutura interna (hchan):
┌─────────────────────────────────────────────────────┐
│ buf       → fila circular de elementos              │
│ sendx     → próximo índice para escrita             │
│ recvx     → próximo índice para leitura             │
│ qcount    → elementos atualmente no buffer          │
│ dataqsiz  → capacidade total do buffer              │
│ sendq     → lista de goroutines ESPERANDO enviar    │
│ recvq     → lista de goroutines ESPERANDO receber   │
│ lock      → mutex que protege toda a struct         │
└─────────────────────────────────────────────────────┘

Quando você faz ch <- v:
  1. Trava o lock
  2. Se há alguém esperando receber (recvq) → entrega direto, acorda a goroutine
  3. Se há espaço no buffer → copia v para buf, incrementa sendx
  4. Se não → coloca esta goroutine na sendq e suspende ela
  5. Destrava o lock

Quando você faz v := <-ch:
  O oposto: lê do buffer, ou acorda quem está esperando enviar
```

Essa estrutura é por que channels são seguros para uso concorrente — o lock interno protege tudo, mas é muito mais ergonômico do que gerenciar locks manualmente.

---

## 3. Criando Channels

```go
// make(chan Tipo) — channel sem buffer (síncrono)
ch := make(chan int)
ch2 := make(chan string)
ch3 := make(chan *Usuario)

// make(chan Tipo, capacidade) — channel com buffer (assíncrono até encher)
chBuf := make(chan int, 10)
chStr := make(chan string, 100)

// Verificar propriedades
fmt.Println(len(ch))    // 0 — elementos no buffer agora
fmt.Println(cap(ch))    // 0 — capacidade do buffer (0 = sem buffer)
fmt.Println(ch == nil)  // false — channel inicializado

// Channel nil — declarado mas não inicializado
var chNil chan int
fmt.Println(chNil == nil)  // true
```

---

## 4. Canal Sem Buffer vs Canal Com Buffer — A Diferença Fundamental

Esta é a distinção mais importante. O comportamento muda completamente.

### Canal Sem Buffer — Síncrono (Rendezvous)

Envio e recebimento **acontecem ao mesmo tempo**. O remetente bloqueia até o receptor estar pronto, e vice-versa. É um ponto de sincronização obrigatório:

```go
ch := make(chan string)   // sem buffer

go func() {
	fmt.Println("antes de enviar")
	ch <- "mensagem"        // BLOQUEIA aqui até alguém receber
	fmt.Println("depois de enviar")   // só executa depois que alguém recebeu
}()

time.Sleep(1 * time.Second)   // simula demora do receptor
msg := <-ch                   // desbloqueia o remetente
fmt.Println("recebeu:", msg)

// Saída garantida nesta ordem:
// antes de enviar
// recebeu: mensagem
// depois de enviar
```

O canal sem buffer garante uma propriedade poderosa: **tudo que a goroutine fez antes de `ch <- x` é visível para quem fez `x := <-ch`**. Isso é uma barreira de memória (happens-before).

### Canal Com Buffer — Assíncrono até Encher

O remetente só bloqueia quando o buffer está **cheio**. O receptor só bloqueia quando o buffer está **vazio**:

```go
ch := make(chan int, 3)   // buffer de 3 elementos

// Esses três envios NÃO bloqueiam — há espaço no buffer
ch <- 1
ch <- 2
ch <- 3

// Este bloquearia — buffer cheio
// ch <- 4   // bloquearia até alguém receber

fmt.Println(len(ch))   // 3 — elementos esperando

// Receber não bloqueia — há elementos no buffer
fmt.Println(<-ch)   // 1
fmt.Println(<-ch)   // 2
fmt.Println(<-ch)   // 3

// Isso bloquearia — buffer vazio
// <-ch   // bloquearia até alguém enviar
```

**Quando usar cada um:**

```
Canal sem buffer:
→ Sincronização entre goroutines ("avise quando terminar")
→ Handoff de um valor específico
→ Quando a confirmação de entrega importa

Canal com buffer:
→ Produtores mais rápidos que consumidores (absorve picos)
→ Evitar goroutine leaks (a goroutine pode enviar sem bloquear)
→ Comunicação assíncrona onde pequenas filas são aceitáveis
```

---

## 5. Enviar, Receber e Fechar

```go
ch := make(chan int, 2)

// Enviar — coloca um valor no canal
ch <- 10
ch <- 20

// Receber — remove e retorna um valor do canal
v1 := <-ch          // 10
fmt.Println(v1)

// Receber com ok — detectar se o canal foi fechado
v2, ok := <-ch
fmt.Println(v2, ok)   // 20 true

// Fechar — sinaliza que não haverá mais envios
close(ch)

// Receber de canal fechado e vazio — retorna zero value com ok=false
v3, ok := <-ch
fmt.Println(v3, ok)   // 0 false

// Tentar receber de canal fechado vazio sem ok:
v4 := <-ch
fmt.Println(v4)   // 0 — zero value, sem panic, sem erro
```

### O Que Fechar um Canal Significa

Fechar um canal é um sinal de "acabou, não virão mais valores". É como fechar uma torneira:

```go
// Padrão clássico: produtor fecha o canal ao terminar
func produtor(ch chan<- int, nums []int) {
	for _, n := range nums {
		ch <- n
	}
	close(ch)   // sinaliza: "terminei de enviar"
}

func consumidor(ch <-chan int) {
	// for range drena o canal e para quando ele for fechado e vazio
	for v := range ch {
		fmt.Println(v)
	}
	// chegou aqui: canal fechado E vazio
	fmt.Println("terminou")
}

ch := make(chan int, 10)
go produtor(ch, []int{1, 2, 3, 4, 5})
consumidor(ch)
// imprime: 1 2 3 4 5 terminou
```

### Regras de Fechamento — O Que NUNCA Fazer

```go
// ❌ Enviar para canal fechado → PANIC
ch := make(chan int, 1)
close(ch)
ch <- 1   // panic: send on closed channel

// ❌ Fechar canal já fechado → PANIC
close(ch)   // panic: close of closed channel

// ❌ Fechar canal nil → PANIC
var chNil chan int
close(chNil)   // panic: close of nil channel

// ✅ Regra de ouro: apenas QUEM ENVIA deve fechar
// O receptor nunca fecha — ele não sabe se haverá mais envios
```

---

## 6. Channels Direcionais — Documentar a Intenção

Ao passar channels para funções, você pode restringir a direção de uso. O compilador garante que a função só usa o canal da forma declarada:

```go
// chan<- T — somente envio (write-only)
func produtor(ch chan<- int) {
	ch <- 42   // ok — pode enviar
	// <-ch    // ❌ erro de compilação: receive from send-only channel
	// close(ch) // ok — quem envia pode fechar
}

// <-chan T — somente recepção (read-only)
func consumidor(ch <-chan int) {
	v := <-ch   // ok — pode receber
	// ch <- 1  // ❌ erro de compilação: send to receive-only channel
	// close(ch) // ❌ erro de compilação: close of receive-only channel
	fmt.Println(v)
}

// chan T — bidirecional (conversível para qualquer direção)
ch := make(chan int, 1)
go produtor(ch)    // chan int → chan<- int (conversão implícita)
go consumidor(ch)  // chan int → <-chan int (conversão implícita)
```

**Por que isso importa?** Torna o código auto-documentado e captura erros em compilação. Quando você vê `ch <-chan int` em uma assinatura, sabe imediatamente que essa função é uma leitora — não pode fechar nem enviar.

---

## 7. `for range` em Channels — Iterar até Fechar

O padrão mais comum para consumir todos os valores de um channel:

```go
ch := make(chan string, 5)

// Preencher o canal
go func() {
	palavras := []string{"go", "é", "incrível"}
	for _, p := range palavras {
		ch <- p
	}
	close(ch)   // FUNDAMENTAL: sem close, o for range bloqueia para sempre!
}()

// Iterar — para automaticamente quando ch for fechado E vazio
for palavra := range ch {
	fmt.Println(palavra)
}
// go
// é
// incrível

// Equivalente manual (sem range):
for {
	v, ok := <-ch
	if !ok {
		break   // canal fechado e vazio
	}
	fmt.Println(v)
}
```

---

## 8. Comportamento de Channels Nil

Canal nil é declarado mas não inicializado. Comportamento especial:

```go
var ch chan int   // nil

// Enviar para nil → bloqueia PARA SEMPRE (goroutine leak!)
// ch <- 1

// Receber de nil → bloqueia PARA SEMPRE (goroutine leak!)
// <-ch

// Fechar nil → PANIC
// close(ch)

// Mas em um select, case com canal nil é IGNORADO (nunca selecionado)
// Isso é útil! Veremos no select.
```

---

## 9. Tabela de Comportamentos

Entender isso evita panics e deadlocks:

| Operação | Canal nil | Aberto (com espaço) | Aberto (cheio/vazio) | Fechado |
| --- | --- | --- | --- | --- |
| Enviar `ch <- v` | Bloqueia ∞ | Sucesso | Bloqueia | **PANIC** |
| Receber `v := <-ch` | Bloqueia ∞ | Sucesso | Bloqueia | Zero value + ok=false |
| `v, ok := <-ch` | Bloqueia ∞ | v, true | Bloqueia | zero, false |
| `close(ch)` | **PANIC** | Fecha | Fecha | **PANIC** |
| `len(ch)` | 0 | Elementos no buffer | Elementos no buffer | Elementos restantes |
| `for range ch` | Bloqueia ∞ | Itera | Itera | Para quando vazio |

---

## 10. Padrão: Pipeline — Encadear Stages de Processamento

Um dos padrões mais poderosos: cada estágio recebe de um canal e emite para outro. Podem rodar em paralelo automaticamente:

```go
// Estágio 1: gera números e os coloca no canal
func gerar(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)   // sinaliza que terminou
	}()
	return out   // retorna <-chan: quem chama só pode ler
}

// Estágio 2: eleva ao quadrado cada número recebido
func elevarQuadrado(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

// Estágio 3: filtra apenas os pares
func filtrarPares(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			if n%2 == 0 {
				out <- n
			}
		}
		close(out)
	}()
	return out
}

// Compor o pipeline
func main() {
	// Números → quadrados → filtrar pares
	numeros  := gerar(1, 2, 3, 4, 5)
	quadrados := elevarQuadrado(numeros)
	pares     := filtrarPares(quadrados)

	for v := range pares {
		fmt.Println(v)   // 4 (2²), 16 (4²)
	}
}
```

Os três estágios rodam em goroutines separadas, em paralelo. O pipeline processa dados conforme chegam — não precisa carregar tudo na memória.

---

## 11. Padrão: Fan-Out — Distribuir Trabalho

Um canal alimentando múltiplos workers simultaneamente:

```go
func worker(id int, jobs <-chan string, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		fmt.Printf("worker %d processando: %s\n", id, job)
		time.Sleep(100 * time.Millisecond)   // simula trabalho
	}
}

func main() {
	jobs := make(chan string, 10)
	var wg sync.WaitGroup

	// Criar 3 workers — todos lendo do mesmo canal
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(i, jobs, &wg)
	}

	// Enviar 9 jobs — distribuídos automaticamente entre os workers
	for _, job := range []string{"a", "b", "c", "d", "e", "f", "g", "h", "i"} {
		jobs <- job
	}
	close(jobs)   // workers param quando o canal fechar e esvaziar

	wg.Wait()
	fmt.Println("todos os jobs concluídos")
}
```

---

## 12. Padrão: Done Channel — Cancelamento via Channel

Antes do `context.Context` existir, o padrão era usar um canal `done` para sinalizar cancelamento:

```go
func trabalhar(done <-chan struct{}) {
	for {
		select {
		case <-done:
			fmt.Println("cancelado — encerrando")
			return
		default:
			// continua trabalhando...
			fmt.Println("trabalhando...")
			time.Sleep(500 * time.Millisecond)
		}
	}
}

func main() {
	done := make(chan struct{})

	go trabalhar(done)

	time.Sleep(2 * time.Second)
	close(done)   // close é "broadcast" — acorda TODAS as goroutines aguardando done
	              // diferente de send (ch <- v) que acorda apenas UMA

	time.Sleep(100 * time.Millisecond)
	fmt.Println("main encerrou")
}
```

**Hoje, use `context.Context`** para cancelamento — é o padrão moderno e mais rico:

```go
func trabalhar(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("cancelado:", ctx.Err())
			return
		default:
			fmt.Println("trabalhando...")
			time.Sleep(500 * time.Millisecond)
		}
	}
}

ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
go trabalhar(ctx)
```

---

## 13. Evitar Goroutine Leaks com Channels

Um goroutine leak acontece quando uma goroutine fica bloqueada em um channel para sempre:

```go
// ❌ Goroutine leak clássico
func calcularResultado() int {
	ch := make(chan int)   // sem buffer

	go func() {
		resultado := calcularAlgo()
		ch <- resultado   // BLOQUEIA se ninguém ler — goroutine vaza!
	}()

	// Se esta função retornar sem ler do ch...
	// a goroutine fica presa tentando enviar para sempre
	return 42   // voltou sem ler — goroutine vazou!
}

// ✅ Solução 1: canal com buffer (goroutine pode enviar sem bloquear)
ch := make(chan int, 1)   // buffer de 1 — envio não bloqueia

// ✅ Solução 2: context para cancelamento
func calcularComContexto(ctx context.Context) (int, error) {
	ch := make(chan int, 1)

	go func() {
		select {
		case ch <- calcularAlgo():   // tenta enviar
		case <-ctx.Done():           // ou cancela
		}
	}()

	select {
	case resultado := <-ch:
		return resultado, nil
	case <-ctx.Done():
		return 0, ctx.Err()
	}
}
```

---

## 14. Resumo — Quando Usar Channels

```
Use channels quando:
→ Transferir dados entre goroutines ("passou a responsabilidade")
→ Sincronizar duas goroutines (canal sem buffer como rendezvous)
→ Distribuir trabalho para N workers (fan-out)
→ Coletar resultados de N goroutines (fan-in)
→ Sinalizar eventos (close para broadcast, send para unicast)
→ Construir pipelines de processamento

Prefira sync.Mutex quando:
→ Proteger estado compartilhado in-place (cache, contador)
→ A lógica é simples e não envolve comunicação de dados

Prefira sync/atomic quando:
→ Incrementar/decrementar um contador simples
→ Trocar um ponteiro atomicamente

Próximo: Select — aguardar em múltiplos channels simultaneamente
```

---

## Conexão com Sistemas Operacionais

### Channel Interno (hchan) — Mutex + Condition Variable ([[Threads POSIX]])

A estrutura interna de um channel em Go (`hchan`) é exatamente o que você construiria manualmente em C com as primitivas de [[Threads POSIX]]: um mutex para proteger o acesso à fila e condition variables para suspender goroutines que precisam esperar. A comparação direta:

```
Implementação manual em C com pthreads:
  typedef struct {
      int     *buf;          // buffer circular
      int      head, tail;   // índices de leitura/escrita
      int      count, cap;   // elementos atuais e capacidade
      pthread_mutex_t lock;  // protege toda a struct
      pthread_cond_t  not_empty;  // acorda leitores bloqueados
      pthread_cond_t  not_full;   // acorda escritores bloqueados
  } Channel;

  // Enviar:
  pthread_mutex_lock(&ch->lock);
  while (ch->count == ch->cap)
      pthread_cond_wait(&ch->not_full, &ch->lock);  // espera espaço
  ch->buf[ch->tail] = valor;
  ch->tail = (ch->tail + 1) % ch->cap;
  ch->count++;
  pthread_cond_signal(&ch->not_empty);   // acorda leitor
  pthread_mutex_unlock(&ch->lock);

Estrutura hchan de Go (equivalente):
  buf       -> buffer circular de elementos
  sendx     -> tail (próximo índice para escrita)
  recvx     -> head (próximo índice para leitura)
  qcount    -> count (elementos no buffer)
  dataqsiz  -> cap (capacidade do buffer)
  lock      -> mutex interno (equivale a pthread_mutex_t)
  sendq     -> lista de goroutines esperando enviar (equiv. a not_full waiters)
  recvq     -> lista de goroutines esperando receber (equiv. a not_empty waiters)
```

A diferença fundamental: o `pthread_cond_wait` suspende uma thread de SO (envolvendo o kernel). O `sendq`/`recvq` de Go suspende uma goroutine em user space — ordens de magnitude mais rápido, pois não faz syscall.

### Canal Sem Buffer — Rendezvous e Barreira de Sincronização ([[Threads POSIX]])

[[Threads POSIX]] descreve barreiras como mecanismo para sincronizar múltiplas threads em um ponto: nenhuma avança até que todas tenham chegado ao ponto de barreira. Um canal sem buffer em Go implementa um **rendez-vous** (encontro): o remetente e o receptor chegam ao canal e precisam encontrar-se para a transferência ocorrer.

```
Barrier POSIX (ponto de encontro para N threads):
  pthread_barrier_wait(&barreira);  // todos esperam até N chegarem

Canal sem buffer Go (ponto de encontro para 2 goroutines):
  ch <- valor   // remetente espera o receptor
  <-ch          // receptor espera o remetente
                // os dois "se encontram" — transferência ocorre
```

Essa propriedade de sincronização é poderosa: `tudo que G1 fez antes de ch <- x é visível para G2 depois de x := <-ch`. Isso é uma **happens-before guarantee** — garantia de ordem de memória. Sem canal (ou sem alguma sincronização), o hardware poderia reordenar escritas na memória e G2 poderia ver valores inconsistentes.

### Channels como IPC Intra-Processo — [[Processos]]

[[Processos]] discute IPC (Inter-Process Communication): mecanismos para processos distintos trocarem dados. Os principais são pipes, message queues, shared memory, e sockets. Channels Go são o equivalente intra-processo desses mecanismos:

```
IPC entre processos (SO):                 Channels entre goroutines (Go):
  Pipe (anônimo, sequencial)         ~=   Canal sem buffer (síncrono)
  Pipe com buffer (PIPE_BUF)         ~=   Canal com buffer (make(chan T, N))
  Message queue (POSIX mq_send)      ~=   Canal com buffer tipado
  Shared memory + mutex              ~=   Variável compartilhada + sync.Mutex

Diferença principal:
  IPC: dados são COPIADOS entre espaços de endereçamento separados
       (custo: copy de bytes entre memória do kernel e do processo)
  Channels Go: dados são COPIADOS dentro do mesmo espaço de endereçamento
               (custo: memcpy dentro do hchan, sem syscall para valores pequenos)
               Para valores grandes, use ponteiros — só o ponteiro é copiado.
```

A filosofia "compartilhe memória comunicando" de Go reflete essa analogia com IPC: assim como processos UNIX não compartilham memória e se comunicam via pipes (cópia explícita), goroutines Go devem preferir enviar dados por channels (cópia ou transferência de ponteiro) em vez de compartilhar memória diretamente.

### Pipes UNIX e Channels — Padrão Produtor/Consumidor

Em sistemas UNIX, o pipe é a abstração primitiva de pipeline: um processo escreve em um fd de escrita, o kernel bufferiza, e outro processo lê de um fd de leitura. O canal sem buffer em Go é o análogo perfeito:

```
UNIX pipe (processo A -> processo B):
  int fds[2];
  pipe(fds);             // cria pipe: fds[0]=leitura, fds[1]=escrita
  // processo A: escreve
  write(fds[1], &dado, sizeof(dado));   // copia para buffer do kernel
  // processo B: lê
  read(fds[0], &dado, sizeof(dado));    // copia do buffer do kernel

  Sem buffer: write() bloqueia se nenhum leitor; read() bloqueia se vazio
  Com buffer: write() só bloqueia se buffer cheio (PIPE_BUF = 64 KB no Linux)

Go channel (goroutine A -> goroutine B):
  ch := make(chan Dado)   // sem buffer
  go func() { ch <- dado }()     // goroutine A: bloqueia até B receber
  dado := <-ch                   // goroutine B: bloqueia até A enviar
```

A diferença mais prática: pipes UNIX são unidirecionais e transportam bytes sem tipo. Channels Go têm tipo — o compilador garante que apenas o tipo correto é enviado/recebido, e a conversão é feita pelo runtime, não manualmente pelo programador.

### close(ch) como Broadcast — Sinais e Kill para Process Group ([[Processos]])

[[Processos]] distingue dois tipos de notificação:
- **Unicast**: sinal enviado a um processo específico (`kill(pid, sig)`)
- **Broadcast**: sinal enviado a um grupo (`kill(-pgid, sig)`)

Channels Go têm a mesma distinção:

```
Unicast: ch <- struct{}{}
  → Apenas UMA goroutine bloqueada em <-ch é acordada
  → As outras continuam esperando

Broadcast: close(ch)
  → TODAS as goroutines bloqueadas em <-ch são acordadas simultaneamente
  → Cada uma recebe o zero value com ok=false
  → Goroutines futuras que tentarem receber de ch também retornam imediatamente

Analogia com sinais UNIX:
  send(ch, msg)  ~=  kill(pid, SIGUSR1)    -- acorda um worker específico
  close(ch)      ~=  kill(-pgid, SIGTERM)  -- cancela todos os workers

// Padrão done channel (pré-context):
done := make(chan struct{})
for range numWorkers {
    go func() {
        select {
        case <-done:   // close(done) acorda TODOS os workers simultaneamente
            return
        case job := <-jobs:
            processar(job)
        }
    }()
}
close(done)  // broadcast: todos os workers param
```

Esse comportamento é o que torna `context.Context` possível: internamente, `ctx.Done()` retorna um channel que é fechado (`close`) quando o context cancela. O fechamento propaga o sinal de cancelamento como broadcast para todas as goroutines que esperam nesse channel, sem precisar saber quantas são.