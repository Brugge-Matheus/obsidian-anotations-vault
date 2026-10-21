---
tags:
  - go
  - go/concorrencia
aliases:
  - select
  - Instrução select
  - Timeout em Go
  - time.After
  - time.Ticker
ordem: 39
modulo: "09 - Concorrência"
---
# Select

> **Nota 39 da trilha de Go.** O `select` é o "`switch` dos channels": ele espera em **vários channels ao mesmo tempo** e executa o primeiro caso que puder prosseguir. Com ele resolvemos timeouts, operações não bloqueantes, cancelamento, multiplexação de várias fontes de eventos e prioridades. Nesta nota veremos as regras exatas (inclusive a **escolha aleatória** e a ordem de avaliação), o `default`, timeouts com `time.After`, laços de eventos, o truque do **channel nil**, prioridade, tickers, as mudanças nos timers do Go 1.23 em diante e como o runtime implementa a espera em vários channels. Do lado do SO, é o primo das syscalls `select`/`poll`/`epoll`.
>
> **Pré-requisitos:** [[Channels]], [[Goroutines]].

---

## 🧭 De onde viemos

Em [[Channels]] cada operação esperava em **um** channel: `v := <-ch` bloqueia até `ch` ter algo. Mas muitas situações reais são "o que vier primeiro": a resposta **ou** o timeout, o próximo trabalho **ou** o pedido de cancelamento, eventos de **duas** filas. Em [[Controle de Fluxo]] vimos o `switch`, e o `select` empresta a sintaxe dele. No SO, em [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] e [[Dispositivos de IO]], vimos o mesmo problema com descritores de arquivo, resolvido por `select()`, `poll()` e `epoll`.

> 💡 **`select`:** instrução que espera simultaneamente por várias operações de channel (envios e recebimentos) e executa **exatamente uma**: a primeira que ficar pronta. Se várias estiverem prontas ao mesmo tempo, escolhe uma **aleatoriamente**.

---

## 🔀 1. Sintaxe e regras

```go
select {
case v := <-entrada:           // recebimento
	usar(v)
case v, ok := <-eventos:       // recebimento com "vírgula ok"
	if !ok { ... }             // eventos foi fechado
case saida <- resultado:       // envio
	fmt.Println("enviado")
case <-ctx.Done():             // recebimento descartando o valor
	return ctx.Err()
default:                       // opcional: se nenhum case estiver pronto AGORA
	fazerOutraCoisa()
}
```

As regras, na ordem em que o runtime as aplica:

| # | Regra |
| --- | --- |
| 1 | **Todas** as expressões de channel e os **valores a enviar** são avaliados **uma vez**, na ordem do código, ao entrar no `select` |
| 2 | Se **um ou mais** cases podem prosseguir, um deles é escolhido por **sorteio uniforme** |
| 3 | Se nenhum pode prosseguir e há `default`, o `default` roda (o `select` **não bloqueia**) |
| 4 | Se nenhum pode prosseguir e não há `default`, a goroutine **bloqueia** até algum ficar pronto |
| 5 | Cases com channel **nil** nunca ficam prontos (são ignorados) |
| 6 | Um recebimento de channel **fechado** está sempre pronto (devolve zero value, `ok == false`) |

### 1.1 Regra 1 na prática: tudo é avaliado, mesmo o que não for escolhido

Saída real: o channel `b` não tem receptor, então só o case de `a` pode prosseguir, mas **as duas** expressões foram calculadas antes:

```go
func canal(nome string, c chan int) chan int { fmt.Println("avaliou o channel", nome); return c }
func valor(v int) int                      { fmt.Println("avaliou o valor", v); return v }

a := make(chan int, 1)
b := make(chan int) // ninguém recebe de b
select {
case canal("a", a) <- valor(1):
	fmt.Println("enviou em a")
case canal("b", b) <- valor(2):
	fmt.Println("enviou em b")
}
```

```
avaliou o channel a
avaliou o valor 1
avaliou o channel b
avaliou o valor 2
enviou em a
```

> ⚠️ Consequência: `case ch <- calcularAlgoCaro():` **sempre** paga o cálculo, mesmo se outro case vencer. Calcule antes, ou envie de dentro do corpo de um case.

### 1.2 Regra 2 na prática: o sorteio

Dois channels sempre prontos ao mesmo tempo, 10 mil vezes. Saída real:

```go
for range 10_000 {
	a <- 1
	b <- 1 // os dois prontos ao mesmo tempo
	select {
	case <-a:
		cont["a"]++
		<-b
	case <-b:
		cont["b"]++
		<-a
	}
}
fmt.Println(cont)
// map[a:4963 b:5037]
```

A ordem dos cases no código **não** é prioridade. O sorteio é proposital: impede que um channel muito ativo deixe outro **faminto** (*starvation*), o mesmo cuidado que os escalonadores do SO têm com processos ([[Escalonamento em Sistemas Interativos]]).

### 1.3 Casos especiais

```go
select {}                // sem cases: bloqueia PARA SEMPRE (às vezes usado no fim do main de um servidor)

select {                 // um case só: é exatamente igual a  v := <-ch
case v := <-ch:
	...
}
```

> ⚠️ **`break` dentro de um `select` sai do `select`, não do `for` em volta.** Para sair do laço, use um rótulo (saída real):
> ```go
> laco:
> 	for {
> 		select {
> 		case v, ok := <-ch:
> 			if !ok {
> 				break laco // sem o rótulo, o break sairia só do select
> 			}
> 			fmt.Print(v, " ")
> 		}
> 	}
> 	fmt.Println("fim")
> // 0 1 2 fim
> ```
> (Rótulos foram vistos em [[Controle de Fluxo]]; um `return` também resolve quando o laço está numa função própria.)

---

## 🚦 2. `default`: operações que não bloqueiam

Com `default`, o `select` vira uma **tentativa**: faz a operação se der, senão segue em frente.

```go
// Tentar receber sem esperar
select {
case msg := <-mensagens:
	processar(msg)
default:
	// nada agora; segue a vida
}
```

### 2.1 Envio que descarta quando a fila está cheia

Útil para métricas, logs e notificações, quando perder um evento é melhor do que travar quem produz. Saída real com uma fila de capacidade 2:

```go
eventos := make(chan string, 2)
for i := range 4 {
	select {
	case eventos <- fmt.Sprint("evento", i):
		fmt.Println("enfileirou evento", i)
	default:
		fmt.Println("fila cheia, descartou evento", i)
	}
}
```

```
enfileirou evento 0
enfileirou evento 1
fila cheia, descartou evento 2
fila cheia, descartou evento 3
```

### 2.2 Verificar cancelamento num laço de CPU

```go
for i, item := range itens {
	select {
	case <-ctx.Done():
		return ctx.Err()      // pediram para parar
	default:                  // não pediram: continua
	}
	processar(item)
}
```

(Para esse caso específico, `if err := ctx.Err(); err != nil { return err }` faz o mesmo e é mais curto; ver [[context.Context]].)

> ⚠️ **Nunca** coloque um `select` com `default` sozinho num laço sem mais nada: ele vira **espera ocupada**, girando a CPU a 100% (é o problema de [[Exclusão mútua com espera ocupada]]). Se é para esperar, tire o `default` e deixe o `select` bloquear.

---

## ⏰ 3. Timeouts

### 3.1 Com `time.After`

`time.After(d)` devolve um `<-chan time.Time` que recebe um valor depois de `d`. Num `select`, vira um timeout. Saída real, com uma busca que leva 50 ms e outra que leva 300 ms, ambas com limite de 200 ms:

```go
func buscar(atraso time.Duration) <-chan string {
	ch := make(chan string, 1) // buffer 1: a goroutine não vaza se ninguém ler
	go func() {
		time.Sleep(atraso)
		ch <- "dados"
	}()
	return ch
}

select {
case d := <-buscar(atraso):
	fmt.Println("chegou:", d, "em", time.Since(inicio).Round(10*time.Millisecond))
case <-time.After(200 * time.Millisecond):
	fmt.Println("timeout depois de", time.Since(inicio).Round(10*time.Millisecond))
}
```

```
chegou: dados em 50ms
timeout depois de 200ms
```

> ⚠️ Repare no **buffer 1** do channel de resultado. Sem ele, quando o timeout vence, a goroutine de `buscar` fica presa para sempre tentando enviar para ninguém: um vazamento (ver [[Goroutines]]).

> ⚠️ O timeout só faz quem **espera** desistir. A busca em si **continua rodando** até terminar. Para realmente interromper o trabalho, ele precisa receber e respeitar um `context` (ver [[context.Context]]).

### 3.2 O que mudou nos timers (Go 1.23 → 1.27)

Durante anos, a recomendação era "não use `time.After` em laços": cada chamada criava um timer que **não podia ser coletado** até disparar, então um `time.After(time.Hour)` num laço de requisições acumulava memória por uma hora. O **Go 1.23** mudou duas coisas:

| Mudança | Efeito |
| --- | --- |
| Timers e tickers sem referências podem ser **coletados** pelo GC mesmo sem `Stop` | `time.After` e `time.Tick` em laços deixaram de vazar |
| O channel do timer passou a ser **síncrono** (sem buffer) | `Reset`/`Stop` não deixam mais um valor "velho" esperando no channel |

Medição real no Go 1.27: um milhão de `time.After(time.Hour)` criados e abandonados num laço:

```
1 milhão de time.After(1h) abandonados → heap em uso: 92 KB
```

No **Go 1.27**, a opção de compatibilidade (`GODEBUG=asynctimerchan=1`) foi **removida**: os channels de timer são sempre síncronos, qualquer que seja a versão no `go.mod`.

> 💡 `time.After` num laço muito quente ainda **aloca** um timer por volta. Quando isso importa, reutilize um `time.Timer` com `Reset`, ou use um `context.WithTimeout` criado uma vez para a operação inteira.

### 3.3 Com `context`

Em código real, o prazo normalmente vem de cima (de uma requisição HTTP, por exemplo), então o case de timeout é o `ctx.Done()`:

```go
func buscarComPrazo(ctx context.Context, id int) (string, error) {
	resultado := make(chan string, 1)
	go func() { resultado <- consultar(ctx, id) }()

	select {
	case r := <-resultado:
		return r, nil
	case <-ctx.Done():
		return "", fmt.Errorf("buscar %d: %w", id, ctx.Err())   // Canceled ou DeadlineExceeded
	}
}
```

---

## 🔁 4. O laço de eventos: `for` + `select`

A forma mais comum de `select` é dentro de um `for`, numa goroutine que reage a várias fontes até ser cancelada. É o "laço de eventos" de [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]], só que sem callbacks:

```go
func processarEventos(ctx context.Context, pedidos <-chan Pedido, pagamentos <-chan Pagamento) {
	for {
		select {
		case p, ok := <-pedidos:
			if !ok {
				return                 // a fonte de pedidos acabou
			}
			registrarPedido(p)
		case pg := <-pagamentos:
			conciliar(pg)
		case <-ctx.Done():
			slog.Info("encerrando", "motivo", context.Cause(ctx))
			return
		}
	}
}
```

---

## 🚫 5. O truque do channel nil

Como um case com channel **nil** nunca é escolhido, atribuir `nil` a uma variável de channel **desliga** aquele case. O exemplo clássico é **mesclar** channels que fecham em momentos diferentes. Saída real:

```go
func mesclar(a, b <-chan string) <-chan string {
	out := make(chan string)
	go func() {
		defer close(out)
		for a != nil || b != nil {         // até os dois estarem desligados
			select {
			case v, ok := <-a:
				if !ok {
					a = nil                // a fechou: desliga este case
					fmt.Println("(a fechou)")
					continue
				}
				out <- v
			case v, ok := <-b:
				if !ok {
					b = nil
					fmt.Println("(b fechou)")
					continue
				}
				out <- v
			}
		}
	}()
	return out
}

// mesclar(gerar("a", 2), gerar("b", 4))
// (a fechou)
// (b fechou)
// recebidos: 6
```

Sem o `nil`, depois que `a` fecha o case dele ficaria **sempre pronto** (recebimento de channel fechado), e o laço giraria recebendo zero values sem parar.

O mesmo truque serve para **pausar** uma fonte: um case de envio só fica "ligado" quando há algo para enviar:

```go
var saida chan<- Item          // nil: case de envio desligado
var proximo Item
if len(pendentes) > 0 {
	saida = destino            // há item: liga o case
	proximo = pendentes[0]
}
select {
case saida <- proximo:
	pendentes = pendentes[1:]
case novo := <-entrada:
	pendentes = append(pendentes, novo)
}
```

---

## 🥇 6. Prioridade

Como o sorteio é uniforme, não dá para dizer "prefira este case" num `select` só. O padrão é **dois selects**: o primeiro tenta só o channel prioritário, sem bloquear; o segundo espera qualquer um. Saída real com três itens em cada fila (ambas já cheias):

```go
for alta != nil || normal != nil {
	select { // 1º: só a fila de alta prioridade, sem bloquear
	case t, ok := <-alta:
		if !ok {
			alta = nil
			continue
		}
		fmt.Print(t, " ")
		continue
	default:
	}
	select { // 2º: nada urgente; espera qualquer uma
	case t, ok := <-alta:
		if !ok {
			alta = nil
			continue
		}
		fmt.Print(t, " ")
	case t, ok := <-normal:
		if !ok {
			normal = nil
			continue
		}
		fmt.Print(t, " ")
	}
}
// ALTA0 ALTA1 ALTA2 normal0 normal1 normal2
```

> ⚠️ Prioridade estrita pode **matar de fome** a fila normal se a alta nunca esvaziar. É o mesmo dilema do escalonamento por prioridades estáticas do Tanenbaum ([[Escalonamento em Sistemas Interativos]]).

---

## ⏲️ 7. Tarefas periódicas: `time.Ticker`

Um `Ticker` envia no seu channel `C` a cada intervalo. Com `select`, vira uma tarefa periódica cancelável:

```go
func monitorar(ctx context.Context, intervalo time.Duration) {
	t := time.NewTicker(intervalo)
	defer t.Stop()               // boa prática: para de gerar ticks (desde o 1.23 o GC também recolhe)

	for {
		select {
		case agora := <-t.C:
			coletarMetricas(agora)
		case <-ctx.Done():
			return
		}
	}
}
```

| Detalhe do Ticker | Comportamento |
| --- | --- |
| Consumidor lento | Ticks perdidos são **descartados**, não acumulados (o ticker ajusta o intervalo) |
| `t.Reset(d)` | Muda o intervalo |
| `time.Tick(d)` | Atalho sem `Stop`; desde o 1.23 é seguro, porque o ticker é coletado quando ninguém mais o referencia |

Veremos *rate limiting* e *retry* com backoff, que usam timers e tickers, em [[Padrões de Concorrência]].

---

## 🔬 8. Por dentro: como o runtime espera em vários channels

### 8.1 O compilador simplifica os casos fáceis

| Forma no código | Vira |
| --- | --- |
| `select {}` | `runtime.block()`: estaciona para sempre |
| Um único case | Uma operação de channel comum |
| Um case + `default` | `selectnbsend` / `selectnbrecv`: tentativa sem bloqueio, que nem pega o lock quando o channel está obviamente vazio/cheio |
| Qualquer outro | `runtime.selectgo` |

### 8.2 O algoritmo de `selectgo`

```
0. monta a lista de cases (cases nil são removidos)
1. POLLORDER: embaralha a ordem de verificação (sorteio uniforme)
2. LOCKORDER: ordena os channels por ENDEREÇO de memória (heap sort)
3. trava os locks de todos os channels, NA ORDEM DE ENDEREÇO
4. passada 1: percorre os cases na pollorder; o primeiro que pode prosseguir
   é executado → destrava tudo → retorna
5. se nada estava pronto e há default → destrava tudo → default
6. passada 2: cria um sudog para ESTA goroutine em CADA channel
   (na recvq ou na sendq de cada um) e ESTACIONA (gopark)
7. ...outra goroutine opera num dos channels e acorda esta;
   no sudog fica registrado qual case venceu
8. passada 3: trava tudo de novo e REMOVE os sudogs dos outros channels
```

```
goroutine G esperando em select { <-c1, <-c2, c3 <- v }

c1.recvq: [... sudog(G)]  ─┐
c2.recvq: [... sudog(G)]  ─┼── G registrada em TODOS ao mesmo tempo
c3.sendq: [... sudog(G)]  ─┘

alguém envia em c2 → acorda G → G se remove de c1.recvq e c3.sendq
```

Dois detalhes importantes:

- **Travar em ordem de endereço** evita deadlock entre dois `select` concorrentes que usam os mesmos channels em ordens diferentes. É a solução clássica de "ordenar os recursos" para quebrar a **espera circular**, uma das condições de deadlock.
- Como a goroutine está na fila de vários channels, dois remetentes podem tentar acordá-la ao mesmo tempo. Um campo atômico (`selectDone`) garante que **só um** vence; o perdedor segue para o próximo da fila.

### 8.3 Paralelo com o SO

| SO (descritores de arquivo) | Go (channels) |
| --- | --- |
| `select(nfds, &readfds, &writefds, ..., &timeout)` | `select { case <-a: case b <- v: case <-time.After(d): }` |
| `poll`/`epoll_wait`: bloqueia até algum fd ficar pronto | `selectgo`: estaciona até algum channel ficar pronto |
| `O_NONBLOCK` + `EAGAIN` | `select` com `default` |
| Timeout como parâmetro da syscall | Um case com `time.After` ou `ctx.Done()` |
| Processo no estado **bloqueado** ([[Estados de Processos]]) | Goroutine `_Gwaiting` com motivo `[select]` |

A diferença de escala é grande: `select()` do UNIX percorre **todos** os fds a cada chamada (O(n)), e por isso o Linux criou o `epoll`, que só devolve os prontos. O `select` de Go tem poucos cases, então percorrer todos é barato. E, por baixo, o **netpoller** do runtime usa `epoll` para os sockets (ver [[Sockets e o Pacote net]]).

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esperar que o primeiro case tenha prioridade | Ordem aleatória | Dois selects (seção 6) |
| `break` querendo sair do `for` | O laço continua | `break rotulo` ou `return` |
| `select` + `default` num laço sem mais nada | CPU a 100% | Tire o `default` |
| Case de recebimento de channel fechado sem tratar `ok` | Laço gira recebendo zero values | `if !ok { ch = nil }` ou `return` |
| Goroutine de resultado sem buffer, com timeout | Vazamento quando o timeout vence | `make(chan T, 1)` ou `context` |
| Achar que o timeout interrompe o trabalho | Trabalho continua consumindo recursos | Passe `ctx` para o trabalho |
| Cálculo caro no valor de um case de envio | Custo pago mesmo sem enviar | Calcule fora ou no corpo |
| `time.After` em laço muito quente | Alocação por volta | Reutilize um `Timer` ou um `context` |
| Prioridade estrita | Fila normal faminta | Limite quantos itens prioritários seguidos |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `select` | Espera vários channels; executa um case |
| Avaliação | Todas as expressões de channel e valores de envio, uma vez, na entrada |
| Vários prontos | Sorteio uniforme; a ordem no código não é prioridade |
| `default` | Torna o `select` não bloqueante |
| `select {}` | Bloqueia para sempre |
| Timeout | `time.After(d)` ou `ctx.Done()`; não interrompe o trabalho em si |
| Timers (1.23+) | Coletáveis sem `Stop`; channels síncronos; no 1.27 sem opção de voltar |
| Channel nil | Desliga o case; usado para mesclar e pausar fontes |
| Prioridade | Dois selects: o primeiro com `default` |
| Ticker | Tarefas periódicas; ticks perdidos são descartados |
| Implementação | Embaralha, trava por endereço, sudog em todos os channels, estaciona |
| No SO | `select`/`poll`/`epoll` sobre descritores de arquivo |

---

## 🔗 Conexões

**Dentro de Go**
- As operações de channel e o channel nil → [[Channels]]
- Quem bloqueia e acorda → [[Goroutines]]
- `ctx.Done()` como case de cancelamento → [[context.Context]]
- Rate limiting, retry, fan-in → [[Padrões de Concorrência]]
- O `switch` e os rótulos → [[Controle de Fluxo]]
- `time.Timer`, `time.Ticker` e o pacote `time` → [[Pacotes Importantes da Stdlib]]
- Testar código com tempo sem esperar de verdade (`testing/synctest`) → [[Testes com go test]]
- O netpoller (epoll) dos sockets → [[Sockets e o Pacote net]]
- O escalonador por trás do `gopark` → [[Scheduler do Go]]

**Sistemas Operacionais**
- [[Dispositivos de IO]] — E/S bloqueante, não bloqueante e multiplexada
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — laços de eventos e as interfaces `select`/`poll`/`epoll`
- [[Estados de Processos]] — a goroutine num `select` está bloqueada
- [[Escalonamento em Sistemas Interativos]] — escolha aleatória contra inanição; prioridades e fome
- [[Exclusão mútua com espera ocupada]] — por que `default` em laço é espera ocupada
- [[Semáforos]] — a ordem de aquisição importa para evitar deadlock (aqui, travar por endereço)
- [[Syscalls para Gerenciamento de Arquivos]] — descritores de arquivo, o "channel" do kernel

**Nginx**
- [[Conexões, Workers e Keep-Alive]] — `use epoll;` no Nginx é o mesmo mecanismo do netpoller de Go

---

⬅️ **Anterior:** [[Channels]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[sync.WaitGroup e sync.Mutex]] ➡️
