---
tags:
  - go
  - go/concorrencia
aliases:
  - context
  - Context
  - Cancelamento em Go
  - context.WithTimeout
  - context.WithCancel
  - Graceful shutdown
ordem: 41
modulo: "09 - Concorrência"
---
# context.Context

> **Nota 41 da trilha de Go.** Em [[Goroutines]] vimos que não dá para "matar" uma goroutine: ela precisa ser **avisada** e encerrar sozinha. O pacote **`context`** é o jeito padrão de dar esse aviso, e de carregar junto um **prazo** e alguns **valores** da requisição, atravessando todas as camadas de chamadas e todas as goroutines envolvidas num trabalho. Nesta nota veremos o problema, a interface `Context`, a **árvore** de contextos e como o cancelamento se propaga, timeouts e prazos, **causas** de cancelamento, `AfterFunc` e `WithoutCancel`, valores (e quando **não** usá-los), as regras de uso, o desligamento gracioso com `signal.NotifyContext` e como tudo isso é implementado.
>
> **Pré-requisitos:** [[Goroutines]], [[Channels]], [[Select]], [[Wrapping e Unwrapping de Erros]].

---

## 🧭 De onde viemos

Em [[Channels]] vimos que fechar um channel acorda **todos** os receptores (broadcast) e, em [[Select]], que um case `<-ctx.Done()` aparece em quase todo laço de eventos. Em [[O Padrão error em Go]] e [[Wrapping e Unwrapping de Erros]] encontramos `context.Canceled` e `context.DeadlineExceeded` embrulhados com `%w`. Em [[Convenções da linguagem]] vimos a convenção de que `ctx` é sempre o primeiro parâmetro, e em [[Defer, Panic e Recover]] prometemos o `signal.NotifyContext`. No SO, [[Syscalls Diversas]] apresentou sinais (`SIGTERM`, `SIGALRM`) e [[Hierarquia de Processos]] os grupos de processos: o `context` é a versão "dentro do processo" dessas ideias.

> 💡 **Context:** valor que acompanha uma unidade de trabalho (uma requisição, um job) e carrega três coisas: um **sinal de cancelamento** (um channel que fecha), opcionalmente um **prazo** (deadline) e opcionalmente **valores** com escopo daquela requisição. É imutável: para mudar algo, cria-se um contexto **filho**.

---

## 😩 1. O problema

Um handler HTTP recebe uma requisição e, para responder, consulta o banco, chama uma API externa e dispara três goroutines. O cliente **fecha a aba** depois de 200 ms. O que acontece com todo esse trabalho?

```
                    cliente desconecta aqui ✂
requisição ──► handler ──┬──► consulta ao banco (8 s)          ← continua?
                         ├──► chamada à API externa (3 s)       ← continua?
                         └──► 3 goroutines de processamento    ← continuam?
```

Sem coordenação, **tudo continua**, gastando CPU, conexões de banco e memória para produzir uma resposta que ninguém vai ler. Sob carga, isso derruba servidores: requisições abandonadas se acumulam. O que falta é um jeito de dizer, de uma vez, para **toda a árvore de trabalho**: "pare".

---

## 🧩 2. A interface

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool) // quando expira (ok=false: sem prazo)
	Done() <-chan struct{}                   // FECHADO quando o contexto é cancelado
	Err() error                              // nil enquanto ativo; depois Canceled ou DeadlineExceeded
	Value(key any) any                       // valor associado à chave (ou nil)
}
```

O centro de tudo é o **`Done()`**: um channel que **nunca recebe valores** e que é **fechado** no cancelamento. Como receber de um channel fechado retorna na hora, e fechar acorda todos os receptores (ver [[Channels]]), qualquer número de goroutines pode esperar pelo mesmo cancelamento:

```go
select {
case <-ctx.Done():
	return ctx.Err()           // cancelaram ou o prazo venceu
case r := <-resultado:
	return usar(r)
}
```

| `ctx.Err()` | Significa |
| --- | --- |
| `nil` | Ainda ativo |
| `context.Canceled` | Alguém chamou `cancel()` (ou um ancestral foi cancelado) |
| `context.DeadlineExceeded` | O prazo passou |

---

## 🌳 3. Criando contextos: a árvore

Contextos formam uma **árvore**. Toda função construtora recebe um **pai** e devolve um **filho**:

| Construtor | Cria um filho que... |
| --- | --- |
| `context.Background()` | É a **raiz**: nunca cancela, sem prazo, sem valores. Use em `main`, inicialização, testes |
| `context.TODO()` | Igual ao `Background`, mas sinaliza "ainda não sei qual contexto usar aqui" |
| `WithCancel(pai)` | Cancela quando você chama `cancel()` |
| `WithTimeout(pai, d)` | Cancela sozinho depois da duração `d` |
| `WithDeadline(pai, t)` | Cancela sozinho no instante `t` |
| `WithCancelCause(pai)` | Como `WithCancel`, mas `cancel(err)` registra uma **causa** |
| `WithTimeoutCause` / `WithDeadlineCause` | Prazo com uma causa personalizada |
| `WithValue(pai, k, v)` | Carrega um valor |
| `WithoutCancel(pai)` | Herda os valores, mas **ignora** o cancelamento do pai |

### 3.1 O cancelamento desce, nunca sobe

Cancelar um contexto cancela **todos os descendentes**, mas não afeta o pai nem os irmãos. Saída real:

```go
raiz, cancelRaiz := context.WithCancel(context.Background())
banco, cancelBanco := context.WithCancel(raiz)
api, cancelAPI := context.WithCancel(raiz)
defer cancelAPI()

cancelBanco()
// depois de cancelar só o filho 'banco':
//   raiz   err=<nil>
//   banco  err=context canceled
//   api    err=<nil>

cancelRaiz()
// depois de cancelar a raiz:
//   raiz   err=context canceled
//   api    err=context canceled
```

```
Background
   └── raiz (WithCancel)            ← cancelRaiz() derruba tudo abaixo
         ├── banco (WithCancel)     ← cancelBanco() derruba só este ramo
         └── api (WithCancel)
               └── ... (timeouts, valores, outras goroutines)
```

### 3.2 Sempre chame o `cancel`

Toda função `With...` que devolve um `cancel` **exige** que ele seja chamado, mesmo que o trabalho termine normalmente: é o `cancel` que **desliga o filho do pai** e para o timer interno. Esquecer causa um vazamento (o pai guarda uma referência para o filho até ser cancelado, o que, para um `Background`, é nunca). O `go vet` detecta. Saída real:

```go
func buscar(ctx context.Context) error {
	ctx, _ = context.WithTimeout(ctx, time.Second)   // ❌ cancel descartado
	return consultar(ctx)
}
```

```
main.go:9:7: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

```go
ctx, cancel := context.WithTimeout(ctx, time.Second)
defer cancel()                                       // ✅ sempre, logo na linha seguinte
```

---

## ⏰ 4. Prazos: `WithTimeout` e `WithDeadline`

Saída real: uma operação de 200 ms com prazo de 50 ms.

```go
func operacaoLenta(ctx context.Context) error {
	select {
	case <-time.After(200 * time.Millisecond): // o trabalho "de verdade"
		return nil
	case <-ctx.Done():
		return fmt.Errorf("operacaoLenta: %w", ctx.Err())
	}
}

ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()
prazo, _ := ctx.Deadline()
fmt.Println("prazo daqui a:", time.Until(prazo).Round(10*time.Millisecond))

err := operacaoLenta(ctx)
fmt.Println("erro:", err, "após", ...)
fmt.Println("é DeadlineExceeded?", errors.Is(err, context.DeadlineExceeded))
fmt.Println("é Canceled?        ", errors.Is(err, context.Canceled))
```

```
prazo daqui a: 50ms
erro: operacaoLenta: context deadline exceeded após 50ms
é DeadlineExceeded? true
é Canceled?         false
```

Três detalhes importantes:

- **O prazo efetivo é o menor da cadeia.** Um filho com `WithTimeout(ctx, 10*time.Second)` de um pai que expira em 2 s expira em 2 s. Um filho nunca "estende" o prazo do pai
- **Embrulhe com `%w`** (ver [[Wrapping e Unwrapping de Erros]]): quem chama precisa distinguir "cancelado/expirou" de um erro de negócio com `errors.Is`
- O prazo é **informação** além de gatilho: um cliente pode olhar `ctx.Deadline()` e enviar o tempo restante para o servidor remoto (o gRPC faz isso), ou desistir antes de começar algo que sabidamente não cabe no tempo

---

## 🏷️ 5. Causas: por que foi cancelado?

`Err()` só diz **que** foi cancelado. Desde o Go 1.20/1.21, dá para registrar **por quê**, e lê-lo com `context.Cause`. Saída real:

```go
var ErrClienteSaiu = errors.New("cliente desconectou")

ctx, cancel := context.WithCancelCause(context.Background())
cancel(ErrClienteSaiu)
fmt.Println("Err:  ", ctx.Err())             // Err:   context canceled
fmt.Println("Cause:", context.Cause(ctx))    // Cause: cliente desconectou

ctx2, cancel2 := context.WithTimeoutCause(context.Background(), 10*time.Millisecond,
	errors.New("consulta ao estoque passou de 10ms"))
defer cancel2()
<-ctx2.Done()
fmt.Println("Err:  ", ctx2.Err())            // Err:   context deadline exceeded
fmt.Println("Cause:", context.Cause(ctx2))   // Cause: consulta ao estoque passou de 10ms

ctx3, cancel3 := context.WithCancel(context.Background())
cancel3()
fmt.Println("sem causa explícita, Cause =", context.Cause(ctx3))
// sem causa explícita, Cause = context canceled
```

Repare que `Err()` continua devolvendo os valores padrão (para não quebrar quem compara com `context.Canceled`), e que `Cause` sem causa explícita devolve o próprio `Err()`. Em logs, prefira `context.Cause(ctx)`: "consulta ao estoque passou de 10ms" é muito mais útil que "context deadline exceeded".

---

## 🪝 6. `AfterFunc` e `WithoutCancel`

### 6.1 `context.AfterFunc` (Go 1.21+)

Registra uma função para rodar (numa goroutine própria) **quando o contexto for cancelado**. Útil para integrar com APIs que **não** conhecem `context`: fechar uma conexão, acordar um `sync.Cond`. Saída real:

```go
ctx, cancel := context.WithCancel(context.Background())
context.AfterFunc(ctx, func() {
	fmt.Println("AfterFunc: contexto cancelado, fechando a conexão")
})
fmt.Println("cancelando...")
cancel()
// cancelando...
// AfterFunc: contexto cancelado, fechando a conexão

ctx2, cancel2 := context.WithCancel(context.Background())
stop2 := context.AfterFunc(ctx2, func() { fmt.Println("isto não deve aparecer") })
fmt.Println("stop() antes do cancelamento:", stop2())   // true: desregistrou a tempo
cancel2()                                               // nada é impresso
```

### 6.2 `context.WithoutCancel` (Go 1.21+)

Às vezes um trabalho **precisa** terminar mesmo que a requisição seja cancelada: gravar auditoria, publicar uma métrica, desfazer algo. `WithoutCancel` cria um filho que **herda os valores** mas **não** o cancelamento. Saída real:

```go
pai, cancel := context.WithCancel(context.WithValue(context.Background(), chaveReq{}, "req-42"))
auditoria := context.WithoutCancel(pai)
cancel()
fmt.Println("pai:      ", pai.Err())
fmt.Println("auditoria:", auditoria.Err(), "| valor herdado:", auditoria.Value(chaveReq{}))
// pai:       context canceled
// auditoria: <nil> | valor herdado: req-42
```

> ⚠️ Um contexto sem cancelamento também não tem prazo: dê um novo limite a ele (`WithTimeout(auditoria, 5*time.Second)`), senão o "trabalho que precisa terminar" pode ficar pendurado para sempre.

---

## 🎒 7. Valores no contexto

### 7.1 O padrão correto

```go
type chaveUsuario struct{}   // ✅ tipo NÃO exportado e vazio: nenhum outro pacote colide

func ComUsuario(ctx context.Context, id int) context.Context {
	return context.WithValue(ctx, chaveUsuario{}, id)
}

func Usuario(ctx context.Context) (int, bool) {
	id, ok := ctx.Value(chaveUsuario{}).(int)   // type assertion com vírgula ok
	return id, ok
}
```

Saída real (repare como o `String()` de um contexto mostra a cadeia):

```go
ctx := context.Background()
fmt.Println(Usuario(ctx))         // 0 false
ctx = ComUsuario(ctx, 7)
ctx, cancel := context.WithCancel(ctx) // valores atravessam outros contextos
defer cancel()
fmt.Println(Usuario(ctx))         // 7 true
fmt.Println(ctx)                  // context.Background.WithValue(main.chaveUsuario, int).WithCancel
```

| Regra | Por quê |
| --- | --- |
| Chave de **tipo próprio não exportado** | Com `string` como chave, dois pacotes que usam `"userID"` se sobrescrevem |
| Funções de acesso tipadas (`ComUsuario`/`Usuario`) | Esconde a chave e a type assertion |
| Só dados **de escopo da requisição** | IDs de requisição e de trace, usuário autenticado, tenant |
| **Nunca** parâmetros de negócio | Uma função que precisa de `limite` deve recebê-lo como parâmetro; escondido no contexto, vira dependência invisível |
| Nunca dependências (banco, logger configurado) | Passe explicitamente ou guarde na struct do serviço |

### 7.2 Custo

Cada `WithValue` cria um nó que aponta para o pai. `Value(k)` **sobe a cadeia** comparando chaves até achar ou chegar à raiz: é uma lista ligada, O(profundidade). Com poucos valores é irrelevante, mas é mais um motivo para não usar o contexto como "mochila" de tudo.

---

## 📏 8. As regras de uso

```go
// 1. Primeiro parâmetro, chamado ctx
func Buscar(ctx context.Context, id int) (*Item, error)

// 2. Nunca guardado em struct (passe a cada chamada)
type Servico struct {
	ctx context.Context     // ❌ o contexto é de UMA requisição; a struct vive mais
}

// 3. Nunca nil: na dúvida, TODO
Buscar(nil, 1)              // ❌ panic dentro de quem chamar ctx.Done()
Buscar(context.TODO(), 1)   // ✅

// 4. Sempre cancelar o que você criou
ctx, cancel := context.WithTimeout(ctx, d)
defer cancel()

// 5. Propagar, não recriar: use o ctx que você RECEBEU como pai
ctx, cancel := context.WithTimeout(context.Background(), d) // ❌ perde o cancelamento de cima
ctx, cancel := context.WithTimeout(ctx, d)                  // ✅

// 6. Respeitar: funções longas verificam ctx e devolvem ctx.Err() embrulhado
```

> 💡 **Exceção à regra 2:** `http.Request` guarda um contexto (`r.Context()`), porque a própria struct **é** a requisição. A regra é sobre structs de vida longa.

---

## 🛑 9. Respeitando o cancelamento

Receber um `ctx` não basta: a função precisa **verificar**. Há três formas:

```go
// 1) Repassar: a forma mais comum. Bibliotecas bem escritas já verificam.
rows, err := db.QueryContext(ctx, "SELECT ...")
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)

// 2) Esperar com select: quando você mesmo bloqueia num channel
select {
case job := <-fila:
	processar(job)
case <-ctx.Done():
	return context.Cause(ctx)
}

// 3) Verificar periodicamente: laços de CPU que não bloqueiam em nada
for i, linha := range matriz {
	if i%1000 == 0 {                       // não precisa ser a cada volta
		if err := ctx.Err(); err != nil {
			return err
		}
	}
	calcular(linha)
}
```

> ⚠️ Código que **não** verifica o contexto (um laço de CPU puro, uma syscall bloqueante de arquivo) não é interrompido. O cancelamento é **cooperativo**: o contexto só **pede**.

---

## 🧯 10. Desligamento gracioso com sinais

Quando o Docker para um container (ou o Kubernetes remove um pod), o processo recebe **`SIGTERM`** e tem alguns segundos para terminar (ver [[Processo raiz (PID 1)]]). `signal.NotifyContext` transforma esse sinal num cancelamento de contexto. Saída real (o programa envia `SIGTERM` a si mesmo para simular o `docker stop`):

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() {
	time.Sleep(50 * time.Millisecond)
	syscall.Kill(os.Getpid(), syscall.SIGTERM) // simula o "docker stop"
}()

fmt.Println("servindo... (esperando sinal)")
<-ctx.Done()
fmt.Println("Err:  ", ctx.Err())
fmt.Println("Cause:", context.Cause(ctx))
fmt.Println("encerrando com elegância")
```

```
servindo... (esperando sinal)
Err:   context canceled
Cause: terminated signal received
encerrando com elegância
```

Desde o **Go 1.26**, a **causa** informa qual sinal chegou (`terminated` é a descrição de `SIGTERM`; Ctrl+C daria `interrupt signal received`). A partir daí, o `ctx` é passado para tudo, e o servidor HTTP é desligado com `srv.Shutdown`, que veremos em [[HTTP (net-http)]].

> 💡 O `stop()` restaura o comportamento padrão do sinal: um **segundo** Ctrl+C mata o processo na hora, o que é útil se o desligamento gracioso travar.

---

## 🔬 11. Por dentro

### 11.1 As structs

Cada construtor devolve uma struct que **embute** o pai (a "árvore" é feita de ponteiros para cima) e acrescenta um pedaço. A real do Go 1.27 (`context/context.go`):

```go
type cancelCtx struct {
	Context                            // o pai (embutido)

	mu       sync.Mutex
	done     atomic.Value              // chan struct{}, criado só quando alguém chama Done()
	children map[canceler]struct{}     // filhos canceláveis; vira nil no cancelamento
	err      atomic.Value              // Canceled / DeadlineExceeded
	cause    error
}

type timerCtx struct {                 // WithTimeout / WithDeadline
	cancelCtx
	timer    *time.Timer               // time.AfterFunc(d, cancelar)
	deadline time.Time
}

type valueCtx struct {                 // WithValue
	Context
	key, val any
}
```

### 11.2 Como o cancelamento se propaga

Ao criar um filho cancelável, `propagateCancel` **pendura** o filho no pai:

```
1. o pai nunca cancela (Done() == nil, ex.: Background)?  → nada a fazer
2. o pai JÁ foi cancelado?                                → cancela o filho na hora
3. o pai é (ou deriva de) um cancelCtx?                   → adiciona o filho em pai.children
4. o pai tem um método AfterFunc?                         → registra lá
5. senão (um Context "estranho", implementado por fora)   → dispara uma goroutine que espera
                                                            pai.Done() ou filho.Done()
```

E o `cancel()`:

```
1. trava o mutex; se já cancelado, sai (cancelar duas vezes não faz nada)
2. grava err e cause
3. FECHA o channel done          ← acorda todas as goroutines em <-ctx.Done()
4. cancela cada filho em children (recursivamente: a árvore inteira abaixo)
5. children = nil
6. se pedido, remove-se do children do pai   ← por isso esquecer o cancel vaza
```

O channel `done` é criado **preguiçosamente**: um contexto que ninguém espera nunca aloca o channel.

### 11.3 Paralelo com o SO

| SO | `context` |
| --- | --- |
| `alarm(n)` / `setitimer`: o kernel entrega `SIGALRM` depois de `n` segundos | `WithTimeout`: o runtime chama `cancel` via `time.AfterFunc` |
| `timer_create` com tempo absoluto | `WithDeadline` |
| `kill(-pgid, SIGTERM)`: sinal para um **grupo** de processos ([[Hierarquia de Processos]]) | `cancel()` na raiz: cancela a subárvore inteira |
| Grupo herdado no `fork` | Árvore montada **explicitamente** ao passar o pai |
| `setsid()`: o filho sai do grupo e sobrevive ao sinal do pai | `WithoutCancel` |
| Tratador de sinal + flag verificada no laço principal | `select` com `<-ctx.Done()` / `ctx.Err()` |
| `SIGTERM` do `docker stop` ([[Processo raiz (PID 1)]]) | `signal.NotifyContext` |

A grande diferença: um sinal **interrompe** o processo onde quer que ele esteja (por isso tratadores de sinal são tão restritos), enquanto o contexto só é **observado** quando o código decide olhar. Cooperativo, sem surpresas no meio de uma seção crítica.

---

## ⚠️ 12. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Descartar o `cancel` | Vazamento de contexto e de timer; `go vet` reclama | `defer cancel()` |
| Criar `context.Background()` no meio do código | Perde o cancelamento e o prazo de cima | Derive do `ctx` recebido |
| Guardar `ctx` numa struct de vida longa | Usa um contexto já cancelado ou de outra requisição | Passe como parâmetro |
| Passar `nil` | Panic | `context.TODO()` |
| Retornar erro genérico ao cancelar | Chamador não distingue cancelamento | `fmt.Errorf("...: %w", ctx.Err())` |
| Chave `string` em `WithValue` | Colisão entre pacotes | `type chave struct{}` |
| Parâmetros de negócio no contexto | Dependências invisíveis | Parâmetros explícitos |
| Laço de CPU sem verificar `ctx` | Cancelamento ignorado | `ctx.Err()` periodicamente |
| `WithoutCancel` sem prazo novo | Trabalho pendurado para sempre | `WithTimeout` sobre ele |
| Logar só `ctx.Err()` | "context canceled" sem pista | `context.Cause(ctx)` + causas explícitas |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Para quê | Cancelamento, prazo e valores de uma unidade de trabalho |
| `Done()` | Channel fechado no cancelamento: broadcast para todas as goroutines |
| `Err()` | `nil`, `Canceled` ou `DeadlineExceeded` |
| Árvore | Cancelar desce para os filhos, nunca sobe |
| `cancel` | Sempre chamar (`defer`); senão vaza |
| Prazos | O menor da cadeia vale; embrulhe o erro com `%w` |
| Causas | `WithCancelCause`, `WithTimeoutCause`, `context.Cause` |
| `AfterFunc` | Roda uma função no cancelamento; integra APIs sem contexto |
| `WithoutCancel` | Herda valores, ignora cancelamento; dê um prazo novo |
| Valores | Só dados da requisição; chave de tipo não exportado; busca O(profundidade) |
| Regras | Primeiro parâmetro, nunca em struct, nunca nil, propague |
| Sinais | `signal.NotifyContext`; causa diz o sinal (1.26+) |
| Implementação | `cancelCtx` com mapa de filhos; `done` criado sob demanda; `timerCtx` usa `time.AfterFunc` |

---

## 🔗 Conexões

**Dentro de Go**
- Por que não dá para matar goroutines → [[Goroutines]]
- Fechar um channel como broadcast → [[Channels]]
- `case <-ctx.Done()` → [[Select]]
- errgroup, pipelines e workers canceláveis → [[Padrões de Concorrência]]
- `r.Context()`, `NewRequestWithContext`, `Shutdown` → [[HTTP (net-http)]]
- `Dialer.DialContext` → [[Sockets e o Pacote net]]
- `Canceled`/`DeadlineExceeded` embrulhados → [[Wrapping e Unwrapping de Erros]], [[O Padrão error em Go]]
- `ctx` como primeiro parâmetro → [[Convenções da linguagem]]
- `defer cancel()` → [[Defer, Panic e Recover]]
- Type assertion na leitura de valores → [[Type Assertion e Type Switch]]
- Labels de profiling (`pprof.Do`) carregados no contexto → [[Profiling e Diagnóstico]]

**Sistemas Operacionais**
- [[Syscalls Diversas]] — sinais, `kill`, `alarm`
- [[Hierarquia de Processos]] — grupos de processos e sinais para o grupo
- [[Término de Processos]] — término voluntário (limpo) vs forçado
- [[Processos]] — sinais como interrupções de software
- [[Política versus Mecanismo]] — o contexto é o mecanismo; quando e por que cancelar é política de quem chama

**Docker**
- [[Processo raiz (PID 1)]] — o `SIGTERM` do `docker stop` e o prazo antes do `SIGKILL`

---

⬅️ **Anterior:** [[sync.WaitGroup e sync.Mutex]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Padrões de Concorrência]] ➡️
