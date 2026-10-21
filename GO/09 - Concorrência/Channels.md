---
tags:
  - go
  - go/concorrencia
aliases:
  - Channel
  - Canais
  - chan
  - Canal sem buffer
  - Canal com buffer
ordem: 38
modulo: "09 - Concorrência"
---
# Channels

> **Nota 38 da trilha de Go.** Um **channel** é um cano tipado que liga goroutines: uma envia valores de um lado, outra recebe do outro, e a própria operação de enviar/receber **sincroniza** as duas. É a implementação de Go da [[Troca de Mensagens]] do Tanenbaum e o coração do modelo CSP. Nesta nota veremos como criar e usar channels, a diferença fundamental entre **sem buffer** (encontro) e **com buffer** (fila), o significado de **fechar** um channel e o `for range`, channels **direcionais**, o comportamento de channels **nil**, a tabela completa de operações, a regra de **posse**, as garantias de memória e a estrutura `hchan` que o runtime usa por baixo.
>
> **Pré-requisitos:** [[Goroutines]], [[Fundamentos de Concorrência]].

---

## 🧭 De onde viemos

Em [[Goroutines]] vimos que `go f()` não devolve resultado e que goroutines não têm "alça" para esperar ou cancelar. Precisamos de uma forma de **passar dados** entre elas e de **avisar** quando algo terminou. Em [[Fundamentos de Concorrência]] vimos o lema "compartilhe memória comunicando". Channels são a ferramenta desse lema. No Tanenbaum, [[Troca de Mensagens]] apresentou `send`/`receive`, *mailboxes* com buffer e o *rendezvous* sem buffer: as duas variantes existem em Go. Em [[Controle de Fluxo]] e [[Iteradores]] já vimos o `range` sobre channels.

> 💡 **Channel:** valor do tipo `chan T` que permite a goroutines **enviar** (`ch <- v`) e **receber** (`v := <-ch`) valores do tipo `T`, com segurança para uso concorrente. Pode ter um **buffer** (uma fila de capacidade fixa) ou não ter nenhum.

---

## 🛠️ 1. Criando e usando

```go
ch := make(chan int)          // sem buffer (capacidade 0)
fila := make(chan string, 10) // com buffer de 10 elementos
var nada chan int             // nil: declarado, mas não criado (veremos na seção 6)

ch <- 42        // ENVIAR: coloca 42 no channel
v := <-ch       // RECEBER: tira um valor
<-ch            // receber e descartar (só para sincronizar)
v, ok := <-ch   // receber com "vírgula ok": ok == false se fechado e vazio
close(ch)       // FECHAR: "não virão mais valores"
len(fila)       // elementos no buffer agora
cap(fila)       // capacidade do buffer
```

Um channel é um **tipo de referência**, como maps e slices: `make` devolve um ponteiro para uma estrutura do runtime, então passar `ch` para uma função **não copia** o channel; as duas pontas falam com a mesma fila. O zero value é `nil`.

> 💡 O tipo do elemento pode ser qualquer coisa, inclusive outro channel (`chan chan int`) ou uma função. Só há um limite: o elemento não pode ocupar mais de 64 KB. Saída real do compilador com `make(chan [100_000]byte)`:
> ```
> ./main.go:4:2: channel element type too large (>64kB)
> ```
> Para dados grandes, envie um **ponteiro** ou um slice.

---

## 🤝 2. Sem buffer vs com buffer: a diferença que importa

### 2.1 Sem buffer: o encontro (*rendezvous*)

Num channel **sem buffer**, o envio só se completa quando **alguém recebe**, e o recebimento só se completa quando **alguém envia**. As duas goroutines precisam se encontrar. Saída real:

```go
ch := make(chan string) // sem buffer
go func() {
	inicio := time.Now()
	ch <- "mensagem" // bloqueia até alguém receber
	fmt.Println("envio bloqueou por", time.Since(inicio).Round(10*time.Millisecond))
}()
time.Sleep(500 * time.Millisecond) // o receptor demora
fmt.Println("recebeu:", <-ch)
```

```
recebeu: mensagem
envio bloqueou por 500ms
```

O remetente ficou **meio segundo** parado esperando. Isso transforma o channel numa ferramenta de **sincronização**, não só de transporte: quando `ch <- v` retorna, o remetente **sabe** que o valor foi entregue.

### 2.2 Com buffer: a fila (*mailbox*)

Num channel **com buffer** de capacidade N, o envio só bloqueia se o buffer estiver **cheio**, e o recebimento só bloqueia se estiver **vazio**:

```go
buf := make(chan string, 1)
buf <- "mensagem"        // há espaço: retorna na hora (saída real: menos de 1 ms)
```

```
           sem buffer                          com buffer (cap 3)
  remetente ──── encontro ──── receptor     remetente ──► [ a | b | _ ] ──► receptor
  (os dois precisam estar lá)               (só bloqueia se cheio / vazio)
```

### 2.3 Qual usar?

| | Sem buffer | Com buffer |
| --- | --- | --- |
| Modelo do Tanenbaum | *Rendezvous* | *Mailbox* com N vagas |
| Garantia ao enviar | O valor **foi recebido** | O valor **está na fila** (talvez ninguém leia) |
| Uso típico | Sincronizar, entregar um resultado, "terminei" | Absorver picos entre produtor e consumidor; semáforo; evitar vazamento quando se sabe quantos envios haverá |
| Risco | Deadlock se ninguém receber | Esconder um problema de vazão; a fila enche e o bug só aparece sob carga |

> ✅ **Comece sem buffer.** Adicione buffer quando tiver um motivo claro e souber dizer **por que aquela capacidade**. Um buffer "bem grande, só por garantia" quase sempre esconde um deadlock ou um consumidor lento que vai aparecer em produção.

> 💡 Um uso legítimo e muito comum de buffer é **exatamente o número de envios**: se N goroutines vão enviar um resultado cada, `make(chan T, N)` garante que nenhuma bloqueia mesmo que o receptor desista cedo (foi a correção do vazamento em [[Goroutines]]).

---

## 🚪 3. Fechar um channel e o `for range`

### 3.1 O que `close` significa

`close(ch)` não destrói nada: é um **aviso** de que **não haverá mais envios**. Depois de fechado:

- Valores que já estavam no buffer **continuam** podendo ser recebidos
- Quando o buffer esvazia, todo recebimento retorna **imediatamente** com o **zero value** e `ok == false`
- **Todas** as goroutines bloqueadas recebendo são acordadas (é um **broadcast**)

Saída real:

```go
ch := make(chan int, 3)
ch <- 10
ch <- 20
fmt.Println("len:", len(ch), "cap:", cap(ch))   // len: 2 cap: 3
fmt.Println(<-ch)                              // 10
close(ch)
v, ok := <-ch
fmt.Println(v, ok)    // 20 true   ← ainda havia um valor no buffer
v, ok = <-ch
fmt.Println(v, ok)    // 0 false   ← fechado e vazio
fmt.Println(<-ch)     // 0         ← sempre o zero value, sem bloquear
```

### 3.2 `for range`: consumir até fechar

`for v := range ch` recebe valores até o channel ser **fechado e esvaziado**. É a forma idiomática de consumir. Saída real:

```go
nums := make(chan int)
go func() {
	defer close(nums)          // ✅ quem envia fecha, ao terminar
	for i := range 3 {
		nums <- i * i
	}
}()
for n := range nums {
	fmt.Print(n, " ")
}
fmt.Println("| range terminou porque o channel foi fechado")
// 0 1 4 | range terminou porque o channel foi fechado
```

> ⚠️ Sem o `close`, o `for range` esperaria para sempre por um próximo valor. Se for a última goroutine viva, o runtime acusa deadlock; se não for, é um vazamento silencioso.

### 3.3 As regras de fechamento

Saídas reais das três operações proibidas (todas causam **panic**):

```
enviar em channel fechado    → send on closed channel
fechar duas vezes            → close of closed channel
fechar channel nil           → close of nil channel
```

Daí as regras:

| Regra | Por quê |
| --- | --- |
| **Só quem envia fecha** | Só o remetente sabe que não enviará mais; o receptor fechando pode causar panic num envio |
| Com **vários remetentes**, ninguém fecha "no meio": um coordenador espera todos (WaitGroup) e fecha | Evita "enviar em channel fechado" |
| Fechar é **opcional** | Um channel não fechado é coletado pelo GC normalmente quando ninguém mais o referencia; feche só quando o receptor precisa saber que acabou |
| Não use `close` para "liberar recursos" | Não há recurso a liberar; não é um arquivo |

```go
// N remetentes, um coordenador fecha depois que todos terminam
var wg sync.WaitGroup
resultados := make(chan int)
for _, item := range itens {
	wg.Add(1)
	go func() {
		defer wg.Done()
		resultados <- processar(item)
	}()
}
go func() {
	wg.Wait()
	close(resultados)          // ✅ só depois que todos os envios acabaram
}()
for r := range resultados { ... }
```

---

## ➡️ 4. Channels direcionais

Um parâmetro pode declarar que só **envia** ou só **recebe**:

| Tipo | Pode | Não pode |
| --- | --- | --- |
| `chan T` | Enviar, receber, fechar | — |
| `chan<- T` (só envio) | Enviar, fechar | Receber |
| `<-chan T` (só recebimento) | Receber | Enviar, fechar |

Um `chan T` é convertido **automaticamente** para qualquer das direções ao ser passado. O compilador garante o resto. Saída real ao violar as direções:

```go
func produtor(ch chan<- int) {
	ch <- 1
	<-ch            // ❌
}

func consumidor(ch <-chan int) {
	<-ch
	ch <- 1         // ❌
	close(ch)       // ❌
}
```

```
./main.go:5:4: invalid operation: cannot receive from send-only channel chan<- int ch (variable of type chan<- int)
./main.go:10:2: invalid operation: cannot send to receive-only channel <-chan int ch (variable of type <-chan int)
./main.go:11:8: invalid operation: cannot close receive-only channel ch (variable of type <-chan int)
```

O padrão mais comum é uma função que **cria** o channel, dispara a goroutine que o alimenta e devolve só a ponta de leitura. Assim, quem recebe não tem como fechar nem enviar por engano:

```go
func gerar(n int) <-chan int {           // devolve só a ponta de leitura
	out := make(chan int)
	go func() {
		defer close(out)                 // a goroutine dona fecha
		for i := range n {
			out <- i
		}
	}()
	return out
}
```

> 💡 Essa é a base de **pipelines**: cada estágio recebe um `<-chan` e devolve outro `<-chan` (ver [[Padrões de Concorrência]]). Para uma sequência simples, sem concorrência, prefira um **iterador** (`iter.Seq`): é muito mais barato e não arrisca vazar goroutines (ver [[Iteradores]]).

---

## 🕳️ 5. Posse: quem é dono do channel

A maioria dos bugs com channels some quando cada channel tem um **dono** claro:

| Responsabilidade | Dono (quem envia) | Usuário (quem recebe) |
| --- | --- | --- |
| Criar (`make`) | ✅ | |
| Enviar | ✅ | |
| Fechar | ✅ | Nunca |
| Expor | Como `<-chan T` | Recebe com `range` ou `v, ok` |
| Saber quando acabou | Decide | Descobre pelo `close` |

O mesmo vale para os **dados**: ao enviar um ponteiro ou slice por um channel, o remetente **entrega a posse** e não deve mais modificá-lo. Se continuar mexendo, há um data race, porque o channel copia o **ponteiro**, não o conteúdo:

```go
buf := make([]byte, 1024)
ch <- buf          // entregou o slice...
buf[0] = 'x'       // ❌ data race: o receptor pode estar lendo o mesmo array
```

---

## 🚫 6. Channels nil

O zero value de um channel é `nil`. Ele é útil (dentro de `select`), mas perigoso fora:

| Operação em channel nil | Resultado |
| --- | --- |
| Enviar | Bloqueia **para sempre** |
| Receber | Bloqueia **para sempre** |
| Fechar | Panic: `close of nil channel` |
| `len`, `cap` | 0 |
| Num `case` de `select` | O case é **ignorado** (nunca é escolhido) |

Saída real de um recebimento num channel nil (repare no motivo entre colchetes):

```go
var ch chan int // nil: declarado, mas nunca criado com make
<-ch
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
	/tmp/gt/ex/n38/nilch/main.go:5 +0x17
```

A propriedade "um case nil é ignorado" permite **desligar** cases de um `select` dinamicamente. Veremos em [[Select]].

---

## 📊 7. Tabela completa de operações

| Operação | nil | Aberto, com espaço / com dados | Aberto, cheio / vazio | Fechado |
| --- | --- | --- | --- | --- |
| `ch <- v` | Bloqueia ∞ | Envia | Bloqueia | **Panic** |
| `<-ch` | Bloqueia ∞ | Recebe | Bloqueia | Valores restantes; depois zero value |
| `v, ok := <-ch` | Bloqueia ∞ | `v, true` | Bloqueia | Restantes com `true`; depois `zero, false` |
| `close(ch)` | **Panic** | Fecha | Fecha (acorda quem espera) | **Panic** |
| `len(ch)` | 0 | Elementos no buffer | 0 ou cap | Elementos restantes |
| `for range ch` | Bloqueia ∞ | Itera | Espera | Esvazia e termina |

---

## 🔗 8. Channels também ordenam a memória

Um channel não só transporta o valor enviado: ele cria uma relação **happens-before** (ver [[Fundamentos de Concorrência]]), então **tudo** que o remetente escreveu antes de enviar fica visível para o receptor depois de receber. Saída real:

```go
var dado string

pronto := make(chan struct{})
go func() {
	dado = "olá"      // 1) escreve
	close(pronto)     // 2) sinaliza
}()
<-pronto              // 3) espera o sinal
fmt.Println(dado)     // 4) garantido: "olá"
```

As regras exatas do modelo de memória de Go:

| Regra | Significado |
| --- | --- |
| Um envio acontece antes de o recebimento correspondente **completar** | O receptor vê tudo que o remetente fez antes do envio |
| O `close` acontece antes de um recebimento que retorna por causa do fechamento | Fechar é um sinal com garantia de memória para **todos** |
| Num channel **sem buffer**, o recebimento acontece antes de o envio **completar** | O remetente também sabe que o receptor chegou ("encontro" nos dois sentidos) |
| Num channel com capacidade C, o k-ésimo recebimento acontece antes de o (k+C)-ésimo envio completar | É o que permite usar um channel com buffer como **semáforo** |

> 💡 `chan struct{}` é o tipo idiomático para channels de **sinal**: `struct{}` ocupa zero bytes, então só a sincronização importa, não o valor.

---

## 🔬 9. Por dentro: a estrutura `hchan`

`make(chan T, n)` chama `runtime.makechan`, que aloca uma `hchan` (e, logo depois dela, o buffer). A estrutura real do Go 1.27 (`runtime/chan.go`):

```go
type hchan struct {
	qcount   uint           // quantos elementos há no buffer
	dataqsiz uint           // capacidade do buffer circular
	buf      unsafe.Pointer // o array circular de dataqsiz elementos
	elemsize uint16         // tamanho do elemento (por isso o limite de 64 KB)
	closed   uint32
	timer    *timer         // timer que alimenta este channel (time.After, Ticker)
	elemtype *_type
	sendx    uint           // próxima posição de escrita no buffer
	recvx    uint           // próxima posição de leitura
	recvq    waitq          // fila de goroutines esperando para RECEBER
	sendq    waitq          // fila de goroutines esperando para ENVIAR
	bubble   *synctestBubble
	lock     mutex          // protege tudo acima
}
```

```
 hchan (cap 4)                         buffer circular
┌──────────────────────┐       ┌─────┬─────┬─────┬─────┐
│ qcount=2 dataqsiz=4  │       │     │  a  │  b  │     │
│ buf ─────────────────┼──────►└─────┴─────┴─────┴─────┘
│ recvx=1 sendx=3      │               ▲recvx      ▲sendx
│ recvq: (vazia)       │
│ sendq: (vazia)       │       goroutines bloqueadas ficam em listas de
│ lock                 │       "sudog" (um registro por goroutine esperando)
└──────────────────────┘
```

O envio `ch <- v` (`runtime.chansend`), em ordem:

```
1. channel nil?                → estaciona a goroutine para sempre
2. trava o lock
3. fechado?                    → destrava e PANIC
4. alguém esperando em recvq?  → COPIA v DIRETO para a stack do receptor,
                                 marca-o como pronto e retorna (nem passa pelo buffer!)
5. há espaço no buffer?        → copia v para buf[sendx], avança sendx, qcount++
6. senão                       → cria um sudog com o endereço de v, entra em sendq,
                                 ESTACIONA a goroutine (gopark) e libera o lock
```

O recebimento é o espelho: se há remetente esperando, pega dele (num channel com buffer cheio, pega da cabeça da fila e coloca o valor do remetente no fim); senão, pega do buffer; senão, estaciona em `recvq`.

> 💡 O passo 4 é um dos raros lugares em que uma goroutine **escreve na stack de outra**: a cópia direta evita passar pelo buffer e economiza uma cópia. É seguro porque o receptor está estacionado e o lock do channel está travado.

### 9.1 Paralelo com o SO

A `hchan` é exatamente o que você montaria em C com as primitivas de [[Threads POSIX]] para resolver o produtor-consumidor de [[Semáforos]] e [[Monitores]]:

| Em C com pthreads | Em `hchan` |
| --- | --- |
| Buffer circular + `head`/`tail`/`count` | `buf`, `recvx`, `sendx`, `qcount` |
| `pthread_mutex_t` protegendo a struct | `lock` |
| `pthread_cond_t not_empty` (receptores esperando) | `recvq` |
| `pthread_cond_t not_full` (remetentes esperando) | `sendq` |
| `pthread_cond_wait` adormece a **thread** (syscall futex) | `gopark` estaciona a **goroutine** (sem syscall) |

E com os mecanismos de IPC de [[Arquivos]] e [[Troca de Mensagens]]:

| SO | Go |
| --- | --- |
| Pipe (bytes, entre processos, buffer no kernel de 64 KB no Linux) | Channel (valores tipados, entre goroutines, buffer na heap do processo) |
| `write` num pipe cheio bloqueia; `read` num pipe vazio bloqueia | Envio num channel cheio bloqueia; recebimento num vazio bloqueia |
| Fechar a ponta de escrita → o leitor recebe EOF (`read` devolve 0) | `close(ch)` → o receptor recebe `ok == false` |
| Escrever num pipe sem leitor → `SIGPIPE` | Enviar em channel fechado → panic |
| *Rendezvous* (troca de mensagens sem buffer) | Channel sem buffer |

### 9.2 Quanto custa

Medição real (Go 1.27, 2 núcleos), um remetente e um receptor em goroutines diferentes:

```
channel sem buffer (envio+recebimento):  228.3 ns/op
channel com buffer (envio+recebimento):   81.5 ns/op
mutex Lock+Unlock sem disputa:            19.4 ns/op
```

O channel sem buffer custa mais porque **toda** operação acorda a outra goroutine (troca de contexto); com buffer, o remetente segue em frente enquanto há espaço. Channels são rápidos o bastante para quase tudo, mas não são de graça: para proteger um contador num laço muito quente, um mutex ou um atômico é uma ordem de grandeza mais barato (ver [[sync.WaitGroup e sync.Mutex]]).

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Enviar sem ninguém para receber (sem buffer) | Deadlock ou goroutine presa | Garanta um receptor, use buffer do tamanho certo, ou `select` com cancelamento |
| Esquecer o `close` antes de um `for range` | O laço nunca termina | O dono fecha (`defer close(out)`) |
| Receptor fechando o channel | `send on closed channel` | Só quem envia fecha |
| Vários remetentes fechando | `close of closed channel` | Um coordenador fecha depois do `wg.Wait()` |
| Channel declarado e não criado (`var ch chan T`) | Bloqueia para sempre | `make(chan T)` |
| Buffer "grande por garantia" | Bug aparece só sob carga | Capacidade com justificativa |
| Modificar um slice/ponteiro depois de enviar | Data race | Envio = transferência de posse |
| Usar `len(ch)` para decidir se pode receber | Corrida: o valor muda logo depois | Use `select` com `default` |
| Channel + goroutine para iterar uma coleção | Lento e vaza se o laço parar cedo | Iterador (`iter.Seq`) |
| Enviar structs enormes por valor | Cópias caras (e limite de 64 KB) | Envie ponteiros (e entregue a posse) |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `chan T` | Tipo de referência; zero value `nil`; criado com `make` |
| Sem buffer | Encontro: envio e recebimento se esperam; sincroniza |
| Com buffer | Fila de N; bloqueia só cheio (envio) ou vazio (recebimento) |
| `close` | "Não haverá mais envios"; broadcast; restantes ainda são lidos; depois zero value + `ok == false` |
| `for range ch` | Consome até fechado e vazio |
| Panics | Enviar em fechado; fechar fechado; fechar nil |
| Direção | `chan<- T` só envia; `<-chan T` só recebe; verificado na compilação |
| Posse | Quem cria, envia e fecha é o dono; enviar um ponteiro entrega a posse |
| nil | Bloqueia para sempre; num `select`, desliga o case |
| Memória | Envio acontece antes do recebimento completar; `close` também sincroniza |
| `hchan` | Buffer circular + lock + filas de sudogs; cópia direta para receptor esperando |
| Custo | ~80–230 ns por mensagem; mutex/atômico são mais baratos para proteger estado |

---

## 🔗 Conexões

**Dentro de Go**
- O lema CSP e happens-before → [[Fundamentos de Concorrência]]
- Quem envia e recebe → [[Goroutines]]
- Esperar em vários channels, timeouts → [[Select]]
- Quando preferir mutex ou atômico → [[sync.WaitGroup e sync.Mutex]]
- `ctx.Done()` é um channel fechado no cancelamento → [[context.Context]]
- Pipelines, fan-out/fan-in, semáforos → [[Padrões de Concorrência]]
- Iterar sem goroutines → [[Iteradores]]
- A "vírgula ok" → [[Múltiplos Retornos e Erros]]
- `range` sobre channels → [[Controle de Fluxo]]
- Channels como tipo de referência → [[Tipos de Dados]]
- `io.Pipe`, o equivalente em bytes → [[io.Reader e io.Writer]]
- Testar código com channels e tempo (`testing/synctest`) → [[Testes com go test]]

**Sistemas Operacionais**
- [[Troca de Mensagens]] — `send`/`receive`, mailbox (com buffer) e rendezvous (sem buffer)
- [[Semáforos]] e [[Monitores]] — o produtor-consumidor que a `hchan` resolve
- [[Threads POSIX]] e [[Mutexes]] — mutex + variáveis de condição, o equivalente em C
- [[Dormir e Despertar]] — `gopark`/`goready` sem a "perda do despertar", porque tudo acontece com o lock travado
- [[Arquivos]] e [[Interpretador de Comandos Shell]] — pipes: o ancestral dos channels
- [[Syscalls Diversas]] — `SIGPIPE`, o equivalente de enviar num channel fechado

**Algoritmos e Estruturas de Dados**
- [[Queue (Fila)]] — o buffer de um channel é uma fila circular FIFO

---

⬅️ **Anterior:** [[Goroutines]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Select]] ➡️
