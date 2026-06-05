---
tags:
  - go
  - go/concorrência
---
# Select

> `select` é uma estrutura de controle feita exclusivamente para trabalhar com channels. Se você já precisou esperar pela resposta de uma API mas também queria um timeout, ou queria processar eventos de múltiplas fontes sem saber qual chegaria primeiro — esse é exatamente o problema que `select` resolve.
> 

---

## 1. O Problema que `select` Resolve

Imagine que você tem duas goroutines enviando dados em channels separados. Você quer processar o que chegar primeiro — sem bloquear no primeiro channel enquanto o segundo já tem dados prontos:

```go
// ❌ Sem select — você fica preso esperando ch1, mesmo se ch2 tiver dados
v1 := <-ch1   // bloqueia aqui até ch1 ter dados
v2 := <-ch2   // só chega aqui depois

// E se ch2 tiver dados mas ch1 não? Você espera ch1 desnecessariamente.
```

```go
// ✅ Com select — aguarda qualquer um dos channels estar pronto
select {
case v1 := <-ch1:
	fmt.Println("chegou do ch1:", v1)
case v2 := <-ch2:
	fmt.Println("chegou do ch2:", v2)
}
// Executa o case do channel que tiver dados primeiro
// Se ambos tiverem: escolhe um aleatoriamente (garantido pela spec)
```

`select` é como um `switch`, mas em vez de comparar valores, ele **espera em múltiplos channels simultaneamente** e executa o case do que estiver pronto.

---

## 2. Como `select` Funciona Internamente

Quando o runtime encontra um `select`, ele:

1. **Embaralha a ordem dos cases** — para evitar que um case seja sempre preferido (starvation)
2. **Verifica todos os cases** de uma vez — quais channels têm dados para receber ou espaço para enviar?
3. **Se nenhum estiver pronto:** suspende a goroutine e adiciona ela às filas de espera (`sendq`/`recvq`) de **todos** os channels envolvidos simultaneamente
4. **Quando um channel fica pronto:** acorda a goroutine, remove ela das filas dos outros channels, e executa o case correspondente

Isso é diferente de verificar os channels em sequência — o `select` registra interesse em **todos** ao mesmo tempo e espera por **qualquer um**.

```
Goroutine aguardando select {ch1, ch2, ch3}:

ch1.recvq: [esta goroutine] ←───┐
ch2.recvq: [esta goroutine] ←───┼── goroutine registrada em todos
ch3.recvq: [esta goroutine] ←───┘

Quando ch2 recebe dados:
→ acorda a goroutine
→ remove ela de ch1.recvq e ch3.recvq
→ executa o case do ch2
```

---

## 3. Sintaxe Básica

```go
select {
case v := <-ch1:           // receber de ch1
	usar(v)

case v := <-ch2:           // receber de ch2
	usar(v)

case ch3 <- valor:         // enviar para ch3
	fmt.Println("enviou")

case v, ok := <-ch4:      // receber com ok (detectar canal fechado)
	if !ok {
		fmt.Println("ch4 foi fechado")
		return
	}
	usar(v)
}
```

**Regras:**

- Se **nenhum** case estiver pronto: bloqueia até um ficar pronto
- Se **múltiplos** cases estiverem prontos: escolhe **aleatoriamente** (não o primeiro!)
- Se houver `default`: executa imediatamente se nenhum case estiver pronto

---

## 4. O Case `default` — Select Não Bloqueante

Sem `default`, `select` sempre bloqueia se nenhum channel estiver pronto. Com `default`, ele nunca bloqueia:

```go
// Verificar se há dados disponíveis SEM bloquear
select {
case v := <-ch:
	fmt.Println("tinha dado:", v)
default:
	fmt.Println("canal vazio — continuando sem esperar")
}
```

### Caso de uso: envio não bloqueante

```go
// Tentar enviar sem bloquear — útil para logs, métricas, notificações
func notificar(ch chan<- string, msg string) {
	select {
	case ch <- msg:
		// enviado com sucesso
	default:
		// canal cheio — descarta a notificação em vez de bloquear
		fmt.Println("aviso: canal cheio, notificação descartada")
	}
}
```

### Caso de uso: verificar cancelamento sem bloquear

```go
func processarItens(ctx context.Context, itens []Item) error {
	for _, item := range itens {
		// Verificar cancelamento a cada iteração, SEM bloquear
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		// Processar o item normalmente
		if err := processar(item); err != nil {
			return err
		}
	}
	return nil
}
```

---

## 5. Timeout — O Caso de Uso Mais Comum

Essa é provavelmente a aplicação mais frequente de `select` no dia a dia: você quer esperar por um resultado, mas não para sempre:

```go
func buscarComTimeout(id int) (string, error) {
	resultado := make(chan string, 1)   // buffer=1 evita goroutine leak

	go func() {
		dados := buscarNoBanco(id)   // operação potencialmente lenta
		resultado <- dados
	}()

	select {
	case dados := <-resultado:
		return dados, nil   // chegou a tempo

	case <-time.After(3 * time.Second):
		return "", fmt.Errorf("timeout: busca do id %d excedeu 3s", id)
	}
}
```

`time.After(d)` retorna um channel que recebe um valor após a duração `d`. Quando esse channel dispara, o case de timeout é selecionado.

### Preferir `context.WithTimeout` em produção

`time.After` cria um timer que nunca é cancelado se o case principal for selecionado antes — em código de alta frequência isso pode vazar timers. O `context.WithTimeout` é mais limpo:

```go
func buscarComContexto(ctx context.Context, id int) (string, error) {
	resultado := make(chan string, 1)

	go func() {
		resultado <- buscarNoBanco(id)
	}()

	select {
	case dados := <-resultado:
		return dados, nil

	case <-ctx.Done():
		// ctx.Done() fecha quando o timeout expirar ou quando cancel() for chamado
		return "", fmt.Errorf("busca cancelada: %w", ctx.Err())
	}
}

// Uso — o timer é limpo quando cancel() é chamado
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()   // sempre chame cancel!

dados, err := buscarComContexto(ctx, 42)
```

---

## 6. Select em Loop — Processar Continuamente

Na prática, `select` quase sempre aparece dentro de um `for` loop para processar eventos continuamente:

```go
func processarEventos(
	ctx    context.Context,
	pedidos <-chan Pedido,
	pagamentos <-chan Pagamento,
	logger *slog.Logger,
) {
	for {
		select {
		case pedido, ok := <-pedidos:
			if !ok {
				logger.Info("canal de pedidos fechado")
				return
			}
			logger.Info("novo pedido", "id", pedido.ID)
			processarPedido(pedido)

		case pagamento, ok := <-pagamentos:
			if !ok {
				logger.Info("canal de pagamentos fechado")
				return
			}
			logger.Info("pagamento recebido", "valor", pagamento.Valor)
			registrarPagamento(pagamento)

		case <-ctx.Done():
			logger.Info("encerrando processamento", "motivo", ctx.Err())
			return
		}
	}
}
```

Esse padrão é o coração de muitos sistemas Go: um loop infinito com `select` que responde a múltiplas fontes de eventos e para quando o contexto é cancelado.

---

## 7. Canal Nil — Desabilitar um Case Dinamicamente

Uma propriedade pouco conhecida mas muito útil: **um case com canal nil nunca é selecionado**. Ele bloqueia para sempre — como se não existisse. Isso permite desabilitar casos dinamicamente:

```go
func mesclarCanais(ch1, ch2 <-chan int) <-chan int {
	saida := make(chan int)

	go func() {
		defer close(saida)

		for ch1 != nil || ch2 != nil {
			select {
			case v, ok := <-ch1:
				if !ok {
					ch1 = nil   // ch1 fechou → este case nunca mais será selecionado
					continue
				}
				saida <- v

			case v, ok := <-ch2:
				if !ok {
					ch2 = nil   // ch2 fechou → este case nunca mais será selecionado
					continue
				}
				saida <- v
			}
		}
	}()

	return saida
}
```

Sem esse truque, você precisaria de lógica complexa para saber quais channels ainda estão ativos. Com canal nil, o case simplesmente some do `select`.

---

## 8. Múltiplos Cases Prontos — A Escolha Aleatória

É importante entender que quando múltiplos cases estão prontos, Go escolhe **aleatoriamente** — não o primeiro da lista, não o último:

```go
ch1 := make(chan string, 1)
ch2 := make(chan string, 1)

// Ambos já têm dados
ch1 <- "um"
ch2 <- "dois"

// Qual será selecionado? Depende do runtime — não é determinístico
select {
case v := <-ch1:
	fmt.Println(v)   // pode ser "um"
case v := <-ch2:
	fmt.Println(v)   // ou pode ser "dois"
}
```

Isso é intencional — evita que um channel "faminto" seja sempre preterido. Mas significa que **não dá para implementar prioridade diretamente com select**.

### Como implementar prioridade

Se você precisa que `ch1` seja sempre processado antes de `ch2`, use dois selects aninhados:

```go
func processarComPrioridade(alta, normal <-chan Tarefa, ctx context.Context) {
	for {
		// Primeiro: verificar canal de alta prioridade sem bloquear
		select {
		case t := <-alta:
			processar(t)
			continue   // voltar ao topo para verificar alta prioridade de novo
		default:
		}

		// Segundo: se não há nada de alta prioridade, esperar em ambos
		select {
		case t := <-alta:
			processar(t)
		case t := <-normal:
			processar(t)
		case <-ctx.Done():
			return
		}
	}
}
```

O primeiro `select` com `default` só seleciona `alta` se houver dados imediatamente — sem esperar. Se não houver, cai no segundo `select` que espera em ambos.

---

## 9. Ticker — Executar Ações Periódicas

`time.Ticker` cria um channel que recebe a cada intervalo definido. Combinado com `select`, é a forma idiomática de executar tarefas periodicamente:

```go
func monitorar(ctx context.Context, db *sql.DB) {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()   // SEMPRE pare o ticker — evita goroutine leak de timer

	for {
		select {
		case <-ticker.C:
			// Executa a cada 30 segundos
			stats := coletarEstatisticas(db)
			slog.Info("estatísticas", "conexoes", stats.Conexoes, "queries", stats.Queries)

		case <-ctx.Done():
			slog.Info("monitor encerrado")
			return
		}
	}
}
```

---

## 10. Padrão Completo — Rate Limiter

Um caso de uso real que combina tudo que vimos: `select`, `time.Ticker`, `default`, e canal com buffer:

```go
// RateLimiter permite no máximo N operações por segundo
type RateLimiter struct {
	tokens chan struct{}
	ticker *time.Ticker
	done   chan struct{}
}

func NovoRateLimiter(operacoesPorSegundo int) *RateLimiter {
	rl := &RateLimiter{
		tokens: make(chan struct{}, operacoesPorSegundo),
		ticker: time.NewTicker(time.Second / time.Duration(operacoesPorSegundo)),
		done:   make(chan struct{}),
	}

	// Goroutine que adiciona tokens periodicamente
	go func() {
		for {
			select {
			case <-rl.ticker.C:
				// Adicionar um token — sem bloquear se já estiver cheio
				select {
				case rl.tokens <- struct{}{}:
				default:   // bucket cheio — descarta o token extra
				}

			case <-rl.done:
				return
			}
		}
	}()

	return rl
}

// Aguardar um token — bloqueia se o rate limit foi atingido
func (rl *RateLimiter) Aguardar(ctx context.Context) error {
	select {
	case <-rl.tokens:
		return nil   // conseguiu um token — pode prosseguir
	case <-ctx.Done():
		return ctx.Err()   // timeout ou cancelamento
	}
}

func (rl *RateLimiter) Parar() {
	rl.ticker.Stop()
	close(rl.done)
}

// Uso
limiter := NovoRateLimiter(10)   // máximo 10 operações por segundo
defer limiter.Parar()

for _, req := range requisicoes {
	if err := limiter.Aguardar(ctx); err != nil {
		return err
	}
	go processarRequisicao(req)
}
```

---

## 11. Resumo — Guia de Decisão

```
Preciso esperar em APENAS UM channel?
→ Use recepção direta: v := <-ch

Preciso esperar em MÚLTIPLOS channels?
→ Use select

Quero esperar mas com timeout?
→ select + context.WithTimeout ou time.After

Quero verificar um channel SEM bloquear?
→ select com default

Quero processar eventos continuamente?
→ for { select { ... } }

Quero desabilitar um case dinamicamente?
→ Defina o channel como nil

Quero prioridade entre channels?
→ select aninhado com default no primeiro

Quero executar ações periódicas?
→ time.NewTicker + select

Regras importantes:
→ Múltiplos cases prontos → escolha ALEATÓRIA (não o primeiro!)
→ Canal nil em um case → esse case nunca é selecionado
→ Canal fechado em um case de recepção → sempre pronto (retorna zero + ok=false)
→ default presente → nunca bloqueia
→ Sem cases e sem default → bloqueia para sempre (deadlock)
```

---

## Conexão com Sistemas Operacionais

### select em Múltiplos Channels — I/O Multiplexing ([[Dispositivos de IO]])

[[Dispositivos de IO]] descreve o problema fundamental de I/O multiplexing: um processo que precisa esperar por dados de múltiplas fontes (múltiplos sockets, pipes, arquivos) sem bloquear em nenhuma delas indefinidamente. A solução POSIX são as syscalls `select()`, `poll()` e `epoll()`:

```
Problema: servidor precisa atender múltiplos clientes sem criar thread por cliente

syscall select() (POSIX):
  fd_set readfds;
  FD_SET(socket1, &readfds);
  FD_SET(socket2, &readfds);
  FD_SET(socket3, &readfds);
  select(maxfd+1, &readfds, NULL, NULL, &timeout);
  // kernel bloqueia até qualquer fd estar pronto
  // retorna quais fds têm dados disponíveis

Go select em channels:
  select {
  case v := <-ch1:   // análogo a FD_SET(socket1)
  case v := <-ch2:   // análogo a FD_SET(socket2)
  case v := <-ch3:   // análogo a FD_SET(socket3)
  }
  // runtime bloqueia até qualquer channel ter dados
```

A analogia é direta e intencional. O `select` de Go é inspirado pelo `select()` do UNIX, com a mesma semântica de "esperar em múltiplos pontos de I/O simultaneamente". A diferença é que o `select()` do POSIX opera em file descriptors (abstrações do kernel sobre I/O de hardware), enquanto `select` de Go opera em channels (abstrações do runtime sobre comunicação entre goroutines).

No kernel Linux, `epoll` (a versão moderna e escalável de `select/poll`) é usado internamente pelo runtime Go para gerenciar I/O de rede — quando uma goroutine está esperando por I/O de socket, o runtime registra o fd no epoll e coloca a goroutine em estado bloqueado. Quando o kernel notifica via epoll que o fd está pronto, o runtime acorda a goroutine.

### Goroutine Bloqueada em select — Estado "Blocked" ([[Estados de Processos]])

[[Estados de Processos]] descreve três estados principais de um processo/thread:
- **Running**: executando no CPU
- **Ready**: pronto para executar, esperando CPU
- **Blocked**: esperando por um evento externo (I/O, lock, timer)

Quando uma goroutine entra em `select` e nenhum case está pronto, ela transita para o estado **Blocked** — o análogo exato de um processo que chama `read()` em um socket vazio:

```
Estados de uma Goroutine:

  Running  →  goroutine executa código no CPU via P/M
     ↓
  [select sem cases prontos]
     ↓
  Blocked  →  goroutine suspensa, removida da fila do P
              registrada nas recvq/sendq dos channels
              P livre para executar outras goroutines
     ↓
  [um channel fica pronto]
     ↓
  Ready    →  goroutine colocada de volta na fila do P
     ↓
  Running  →  P escolhe a goroutine e executa o case selecionado
```

A transição Running → Blocked em Go não envolve o kernel (diferente de um processo bloqueando em `read()`, que faz uma syscall). É gerenciada inteiramente pelo runtime em user space — o que torna a troca de contexto ~100x mais rápida que a de processos.

### select com default — Non-Blocking I/O Polling ([[Dispositivos de IO]])

[[Dispositivos de IO]] descreve duas abordagens para verificar se um dispositivo de I/O tem dados:
1. **Blocking I/O**: a chamada bloqueia até dados estarem disponíveis
2. **Non-blocking I/O (polling)**: a chamada retorna imediatamente, indicando se havia dados ou não (com `O_NONBLOCK` em UNIX)

`select` com `default` em Go implementa exatamente non-blocking polling para channels:

```
Non-blocking I/O em POSIX:
  // Configura socket como non-blocking
  fcntl(fd, F_SETFL, O_NONBLOCK);

  // Tenta ler — retorna imediatamente
  n = read(fd, buf, size);
  if (n == -1 && errno == EAGAIN) {
      // sem dados disponíveis — tentar depois
  }

Non-blocking polling em Go (select + default):
  select {
  case v := <-ch:
      // havia dado disponível — processa
  default:
      // sem dados — continua sem bloquear
      // análogo ao errno == EAGAIN
  }
```

Esse padrão é usado no loop de polling de Go para verificar cancelamento de context sem bloquear a goroutine:

```go
// Verificar ctx a cada iteração sem bloquear:
select {
case <-ctx.Done():
    return ctx.Err()
default:
    // ctx ainda ativo — continua processando
}
```

### Implementação Interna — Registro em recvq/sendq ([[Implementando Threads em User Space]])

[[Implementando Threads em User Space]] descreve como um scheduler de user-space precisa gerenciar listas de threads esperando por recursos. O `select` de Go implementa um mecanismo sofisticado para esperar em múltiplos channels simultaneamente:

```
Goroutine G1 entra em: select { case <-ch1; case <-ch2; case <-ch3 }

1. Runtime embaralha a ordem dos cases (evita starvation)
2. Adquire locks de todos os channels na ordem de endereço
   (para evitar deadlock entre múltiplos selects concorrentes)
3. Verifica se algum channel já está pronto:
   → se sim: executa o case, libera locks, retorna
4. Se nenhum está pronto:
   → Cria um sudog (estrutura que representa G1 esperando em um channel)
   → Insere o sudog na recvq de ch1
   → Insere o sudog na recvq de ch2
   → Insere o sudog na recvq de ch3
   → Libera os locks
   → Suspende G1 (park — remove do P, coloca no estado blocked)

ch1.recvq: [..., sudog(G1)]
ch2.recvq: [..., sudog(G1)]   ← G1 registrada em TODOS simultaneamente
ch3.recvq: [..., sudog(G1)]

5. Quando ch2 recebe um dado:
   → Runtime adquire lock de ch2
   → Encontra sudog(G1) na recvq
   → Acorda G1 (unpark — coloca de volta na fila do P)
   → Remove sudog(G1) de ch1.recvq e ch3.recvq
   → G1 executa o case do ch2

```

Esse design é descrito em [[Implementando Threads em User Space]] como o desafio central de implementar bloqueio em múltiplos recursos: a thread precisa registrar interesse em todos simultaneamente e garantir que apenas um "acorde" ela. O Go usa o sudog como o mecanismo de registro e o lock por endereço para garantir atomicidade.