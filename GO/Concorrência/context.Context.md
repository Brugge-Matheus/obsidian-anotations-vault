---
tags:
  - go
  - go/concorrência
---
# context.Context

> `context.Context` é o mecanismo de Go para propagar **cancelamento**, **timeout** e **valores** entre goroutines ao longo de uma cadeia de chamadas. Se goroutines são os trabalhadores e channels são os canos de comunicação, o context é o **painel de controle** que permite parar tudo de forma coordenada quando necessário.
> 

---

## 1. O Problema que Context Resolve

Imagine um servidor HTTP que recebe uma requisição. Para responder, ele:

1. Consulta o banco de dados
2. Chama uma API externa
3. Processa os dados em paralelo com várias goroutines

O que acontece se o cliente desconectar antes da resposta? Sem coordenação, todas essas operações continuam rodando — consumindo CPU, memória, conexões de banco — para nada. Com `context.Context`, você pode **cancelar toda a cadeia** com uma única chamada:

```go
// Sem context — operações continuam mesmo após o cliente ir embora
func handleRequest(w http.ResponseWriter, r *http.Request) {
	dados := consultarBanco()     // E se o cliente foi embora?
	resultado := chamarAPI(dados)  // Isso ainda roda — desperdício
	responder(w, resultado)
}

// Com context — tudo para quando o cliente desconecta
func handleRequest(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()   // context da requisição HTTP — cancela se o cliente sair

	dados, err := consultarBanco(ctx)     // para se ctx cancelar
	if err != nil {
		return   // cliente foi embora ou timeout — para aqui
	}

	resultado, err := chamarAPI(ctx, dados)   // idem
	if err != nil {
		return
	}

	responder(w, resultado)
}
```

---

## 2. A Interface `context.Context`

`context.Context` é uma interface com quatro métodos:

```go
type Context interface {
	// Deadline retorna quando o context vai expirar automaticamente.
	// ok = false se não há deadline definido.
	Deadline() (deadline time.Time, ok bool)

	// Done retorna um channel que é FECHADO quando o context é cancelado.
	// Receber de um channel fechado retorna imediatamente — é assim que
	// goroutines detectam o cancelamento.
	Done() <-chan struct{}

	// Err retorna nil enquanto o context está ativo.
	// Depois de cancelado: context.Canceled
	// Depois de expirar:   context.DeadlineExceeded
	Err() error

	// Value retorna o valor associado a uma chave — para dados transversais.
	Value(key any) any
}
```

O ponto central é `Done()`: ele retorna um channel que **fica fechado** quando o context é cancelado. Um channel fechado pode ser recebido imediatamente por qualquer goroutine — esse é o mecanismo de broadcast de cancelamento.

---

## 3. Os Tipos de Context

### `context.Background()` — A Raiz

O context raiz de todo programa. Nunca cancela, nunca tem deadline, nunca tem valores. É o ponto de partida:

```go
ctx := context.Background()
// Use em: main(), inicializações, testes, ponto de entrada de requisições
```

### `context.TODO()` — Placeholder

Idêntico ao `Background()`, mas semanticamente significa *"preciso colocar um context aqui mas ainda não sei qual"*. Útil durante desenvolvimento:

```go
ctx := context.TODO()
// Use quando: você sabe que vai precisar de context mas ainda está refatorando
// Ferramentas de análise estática podem avisá-lo sobre TODOs
```

### `context.WithCancel()` — Cancelamento Manual

Cria um context filho que pode ser cancelado chamando `cancel()`:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()   // SEMPRE chame cancel — libera recursos mesmo sem usar

go func() {
	// Esta goroutine para quando cancel() for chamado
	for {
		select {
		case <-ctx.Done():
			fmt.Println("cancelado:", ctx.Err())   // context.Canceled
			return
		default:
			trabalhar()
		}
	}
}()

time.Sleep(2 * time.Second)
cancel()   // Para a goroutine
```

### `context.WithTimeout()` — Cancelamento por Tempo

Cria um context que cancela automaticamente após a duração especificada:

```go
// O context cancela após 5 segundos OU quando cancel() for chamado
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()   // sempre — libera o timer se terminar antes do timeout

resultado, err := fazerRequisicaoHTTP(ctx)
if err != nil {
	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("operação demorou mais de 5 segundos")
	}
	return
}
```

### `context.WithDeadline()` — Cancelamento em Instante Específico

Como `WithTimeout`, mas você especifica o momento exato de expiração:

```go
// Cancela às meia-noite
meianoite := time.Now().Truncate(24 * time.Hour).Add(24 * time.Hour)
ctx, cancel := context.WithDeadline(context.Background(), meianoite)
defer cancel()

// Cancela no fim do turno de trabalho
fimTurno := time.Date(2024, 3, 15, 18, 0, 0, 0, time.Local)
ctx2, cancel2 := context.WithDeadline(context.Background(), fimTurno)
defer cancel2()
```

### `context.WithValue()` — Valores no Context

Armazena um par chave-valor que pode ser recuperado mais adiante na cadeia:

```go
type ctxKey string   // tipo próprio evita colisão com outros pacotes

const keyRequestID ctxKey = "requestID"

// Armazenar
ctx := context.WithValue(r.Context(), keyRequestID, "req-abc-123")

// Recuperar
if id, ok := ctx.Value(keyRequestID).(string); ok {
	fmt.Println("request ID:", id)
}
```

---

## 4. Como Contexts se Relacionam — A Árvore

Contexts formam uma **árvore hierárquica**. Quando um pai é cancelado, todos os filhos são cancelados automaticamente. Mas cancelar um filho não afeta o pai:

```
context.Background()
    └── WithCancel()       ← ctx raiz da requisição HTTP
           ├── WithTimeout(5s)  ← para a query do banco
           │      └── WithValue(userID)  ← propaga o ID do usuário
           └── WithTimeout(3s)  ← para a chamada de API externa

Se o ctx raiz for cancelado:
→ O filho WithTimeout(5s) cancela
→ O filho WithTimeout(3s) cancela
→ O neto WithValue(userID) cancela
→ Tudo para

Se apenas o filho WithTimeout(5s) expirar:
→ O neto WithValue cancela
→ O ctx raiz continua normal
→ O filho WithTimeout(3s) continua normal
```

```go
func processarRequisicao(ctx context.Context, userID int) error {
	// Sub-context para a query — timeout mais apertado
	dbCtx, dbCancel := context.WithTimeout(ctx, 3*time.Second)
	defer dbCancel()

	dados, err := queryBanco(dbCtx, userID)
	if err != nil {
		return fmt.Errorf("banco: %w", err)
	}

	// Sub-context para API — timeout diferente
	apiCtx, apiCancel := context.WithTimeout(ctx, 10*time.Second)
	defer apiCancel()

	resultado, err := chamarAPI(apiCtx, dados)
	if err != nil {
		return fmt.Errorf("api: %w", err)
	}

	return salvar(ctx, resultado)
}
```

---

## 5. Detectar Cancelamento em Goroutines

O padrão mais comum: `select` esperando tanto pelo trabalho quanto pelo cancelamento:

```go
func processarItens(ctx context.Context, itens <-chan Item) error {
	for {
		select {
		case item, ok := <-itens:
			if !ok {
				return nil   // channel fechado — terminou normalmente
			}
			if err := processar(ctx, item); err != nil {
				return err
			}

		case <-ctx.Done():
			// Context cancelado — parar imediatamente
			return fmt.Errorf("processamento cancelado: %w", ctx.Err())
		}
	}
}
```

### Verificar cancelamento em loops síncronos

Quando o loop não usa channels (processamento puro de CPU), verifique periodicamente:

```go
func calcularMatriz(ctx context.Context, dados [][]float64) ([]float64, error) {
	resultado := make([]float64, len(dados))

	for i, linha := range dados {
		// Verificar a cada iteração — sem bloquear
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		default:
		}

		resultado[i] = calcularLinha(linha)   // operação pesada
	}

	return resultado, nil
}
```

---

## 6. Context com Valores — Boas Práticas

`WithValue` deve ser usado **apenas para dados transversais** — informações que pertencem à requisição, não à lógica de negócio:

```go
// ✅ Bom uso de valores no context
// - Request ID (para logging e tracing)
// - User ID autenticado
// - Informações de tenant (multi-tenancy)
// - Trace/span de distributed tracing

// ❌ Mau uso de valores no context
// - Parâmetros de funções (use parâmetros normais)
// - Configurações (passe explicitamente)
// - Resultados de funções (use valores de retorno)

// Padrão correto: tipo próprio como chave (evita colisão entre pacotes)
type authKey struct{}
type traceKey struct{}

func ComUsuario(ctx context.Context, usuario *Usuario) context.Context {
	return context.WithValue(ctx, authKey{}, usuario)
}

func UsuarioDoContext(ctx context.Context) (*Usuario, bool) {
	u, ok := ctx.Value(authKey{}).(*Usuario)
	return u, ok
}

func ComTraceID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, traceKey{}, id)
}

func TraceID(ctx context.Context) string {
	if id, ok := ctx.Value(traceKey{}).(string); ok {
		return id
	}
	return ""
}
```

**Por que usar tipo próprio como chave?**

```go
// ❌ Chave string — colisão entre pacotes
ctx = context.WithValue(ctx, "userID", 42)   // pacote auth
ctx = context.WithValue(ctx, "userID", 99)   // pacote admin sobrescreve!

// ✅ Chave tipo próprio — impossível colidir
type authUserIDKey struct{}   // em pacote auth
type adminUserIDKey struct{}  // em pacote admin
// Nunca colidem — são tipos diferentes
```

---

## 7. Context em Requisições HTTP

O `net/http` já integra context nativamente:

```go
// O context da request cancela automaticamente quando:
// - O cliente desconecta
// - O servidor é encerrado
// - O timeout do servidor expira
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()   // context já vem pronto com a requisição

	// Passar para TODAS as operações downstream
	dados, err := buscarDados(ctx, r.URL.Query().Get("id"))
	if err != nil {
		if errors.Is(err, context.Canceled) {
			// Cliente foi embora — não precisa responder
			return
		}
		http.Error(w, "erro interno", 500)
		return
	}

	json.NewEncoder(w).Encode(dados)
}

// Cliente HTTP — context controla o timeout da chamada
func buscarAPI(ctx context.Context, url string) ([]byte, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err   // pode ser context.DeadlineExceeded ou context.Canceled
	}
	defer resp.Body.Close()

	return io.ReadAll(resp.Body)
}
```

---

## 8. Context com Banco de Dados

Todas as operações de banco de dados em Go aceitam context — use sempre:

```go
func buscarUsuario(ctx context.Context, db *sql.DB, id int) (*Usuario, error) {
	// QueryContext para com o context — sem desperdiçar conexões
	row := db.QueryRowContext(ctx,
		"SELECT id, nome, email FROM usuarios WHERE id = $1", id)

	var u Usuario
	if err := row.Scan(&u.ID, &u.Nome, &u.Email); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrNaoEncontrado
		}
		return nil, fmt.Errorf("buscarUsuario: %w", err)
	}
	return &u, nil
}

func criarPedidoComTransacao(ctx context.Context, db *sql.DB, pedido Pedido) error {
	// BeginTx respeita o context também
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()   // no-op se Commit já foi chamado

	if _, err := tx.ExecContext(ctx,
		"INSERT INTO pedidos (id, valor) VALUES ($1, $2)",
		pedido.ID, pedido.Valor); err != nil {
		return err
	}

	return tx.Commit()
}
```

---

## 9. `context.WithoutCancel` — Operações que Não Devem Ser Canceladas (Go 1.21+)

Às vezes uma operação precisa continuar mesmo que o context pai cancele — por exemplo, salvar logs de auditoria ou enviar métricas mesmo após uma requisição falhar:

```go
func processarRequisicao(ctx context.Context) error {
	defer func() {
		// Auditoria deve acontecer mesmo se a requisição for cancelada
		// Cria um context sem cancelamento baseado no pai
		auditCtx := context.WithoutCancel(ctx)   // Go 1.21+
		registrarAuditoria(auditCtx, "requisicao processada")
	}()

	return executarLogica(ctx)
}
```

---

## 10. Erros de Context — O Que Cada Um Significa

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

err := operacaoLonga(ctx)

switch {
case errors.Is(err, context.Canceled):
	// cancel() foi chamado explicitamente — alguém pediu para parar
	fmt.Println("operação cancelada pelo chamador")

case errors.Is(err, context.DeadlineExceeded):
	// O timeout expirou antes de terminar
	fmt.Println("operação demorou demais")

case err != nil:
	// Erro de negócio normal — não relacionado ao context
	fmt.Println("erro:", err)

default:
	fmt.Println("sucesso")
}
```

**Importante:** quando uma função retorna erro por causa do context, ela deve retornar `ctx.Err()` (ou um erro que o englobe com `%w`) — não um erro genérico. Isso permite que o chamador distinga entre cancelamento e erros de negócio.

```go
func minhafuncao(ctx context.Context) error {
	select {
	case resultado := <-processar():
		return usar(resultado)
	case <-ctx.Done():
		return fmt.Errorf("minhafuncao: %w", ctx.Err())   // ✅ propaga o erro de context
	}
}
```

---

## 11. Regras de Ouro do Context

```
1. SEMPRE passe context como PRIMEIRO parâmetro
   func Buscar(ctx context.Context, id int) (*Item, error)

2. NUNCA armazene context em structs
   type Servico struct { ctx context.Context }   ❌
   Passe sempre como parâmetro de função

3. SEMPRE chame cancel() — use defer
   ctx, cancel := context.WithTimeout(...)
   defer cancel()   ← mesmo que o timeout expire antes

4. NUNCA passe nil como context
   buscar(nil, id)   ❌
   buscar(context.Background(), id)   ✅
   buscar(context.TODO(), id)         ✅ (se ainda não sabe qual)

5. Use context.Background() na raiz
   Em main(), testes, e ponto de entrada de goroutines longas

6. Propague o erro de context com %w
   return fmt.Errorf("operacao: %w", ctx.Err())

7. Valores no context: apenas dados transversais
   request ID, user ID, trace ID — nunca parâmetros de negócio

8. Use tipo próprio como chave de context.WithValue
   type minhaChave struct{}   ← evita colisão com outros pacotes
```

---

## 12. Resumo

```
O que é:    Mecanismo para propagar cancelamento, timeout e valores
            entre goroutines ao longo de uma cadeia de chamadas

Quatro construtores:
  context.Background()          → raiz, nunca cancela
  context.WithCancel(pai)       → cancelamento manual via cancel()
  context.WithTimeout(pai, d)   → cancela após duração d
  context.WithDeadline(pai, t)  → cancela em instante t
  context.WithValue(pai, k, v)  → propaga valor
  context.WithoutCancel(pai)    → remove cancelamento (Go 1.21+)

Detectar cancelamento:
  select {
  case <-ctx.Done():   // channel fechado = cancelado
      return ctx.Err()
  }

Erros possíveis:
  context.Canceled          → cancel() foi chamado
  context.DeadlineExceeded  → timeout ou deadline expirou

Regra fundamental:
  Contexts formam árvore — cancelar o pai cancela todos os filhos
  Cancelar um filho não afeta o pai
```

---

## Conexão com Sistemas Operacionais

### WithTimeout e WithDeadline — Sinais de Alarme do SO ([[Processos]])

[[Processos]] descreve que um processo pode pedir ao SO para ser notificado após N segundos usando `alarm(N)` ou `setitimer()`. Quando o tempo expira, o kernel envia `SIGALRM` ao processo, interrompendo sua execução normal. Essa é a mecânica fundamental de timeout em nível de processo.

`context.WithTimeout` e `context.WithDeadline` implementam o mesmo conceito, mas dentro do runtime Go, sem envolver o kernel para cada timeout individual:

```
Nível de SO — alarm()/SIGALRM:
  processo chama alarm(5)
  kernel inicia timer de 5 segundos
  após 5s: kernel entrega SIGALRM ao processo
  processo executa o signal handler

Nível de Go — context.WithTimeout:
  ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
  runtime Go inicia timer interno (time.AfterFunc)
  após 5s: runtime chama cancel() internamente
  ctx.Done() é fechado → todas as goroutines que esperam são acordadas
```

A abstração de Go é mais rica: um único timeout pode propagar cancelamento para toda uma árvore de goroutines simultâneas (banco de dados + API externa + processamento paralelo), enquanto `SIGALRM` afeta apenas o processo inteiro com um único handler.

`context.WithDeadline` é o análogo exato de `timer_create()` com tempo absoluto: em vez de "daqui a N segundos", é "às 14h30 em ponto" — o runtime converte internamente o deadline absoluto em uma duração relativa para o timer.

### Propagação de Cancelamento — Sinais para Grupos de Processos ([[Processos]])

Em [[Processos]], quando um processo pai termina ou é morto com `kill(-pgid, SIGTERM)`, o sinal é entregue a todo o processo group — todos os filhos que foram criados com o mesmo PGID recebem o sinal simultaneamente. Isso é um mecanismo de cancelamento em cascata: uma ação no pai se propaga para toda a subárvore de processos.

`context.Context` implementa o mesmo padrão, mas para goroutines dentro de um único processo:

```
Propagação de sinal em árvore de processos (SO):
  init (PID 1)
    └── servidor (PGID=100)
          ├── worker-1 (PGID=100)  ← recebe SIGTERM
          ├── worker-2 (PGID=100)  ← recebe SIGTERM
          └── worker-3 (PGID=100)  ← recebe SIGTERM
  kill(-100, SIGTERM) → todos recebem

Propagação de cancelamento em árvore de contexts (Go):
  context.Background()
    └── ctx raiz (WithCancel)
          ├── ctx banco (WithTimeout 3s)   ← cancelado
          ├── ctx api   (WithTimeout 10s)  ← cancelado
          └── ctx cache (WithValue)        ← cancelado
  cancel() no ctx raiz → todos os filhos cancelados
```

A diferença arquitetural importante: no modelo de processos, PGID é uma propriedade herdada no momento do `fork()`. No Go, a árvore de contexts é explícita — você constrói a hierarquia passando o ctx pai para `WithCancel/WithTimeout/WithDeadline`. Isso dá controle granular que o modelo de processos não oferece.

### ctx.Done() — Channel Fechado como Broadcast ([[Processos]])

O mecanismo de `ctx.Done()` é elegante: é um channel que permanece aberto enquanto o context está ativo e é **fechado** quando o context é cancelado. Fechar um channel é uma operação de broadcast — todas as goroutines que estão esperando receber desse channel são acordadas simultaneamente.

Isso é o análogo em Go do `kill` para um process group em [[Processos]]:

```
Broadcast via sinal UNIX (nível de processo):
  kill(-pgid, SIGTERM)
  → kernel acorda cada processo do grupo e entrega o sinal
  → cada processo executa seu signal handler
  → N processos, N entregas simultâneas

Broadcast via close(channel) (nível de goroutine):
  cancel()  →  close(ctx.done)
  → runtime acorda cada goroutine bloqueada em <-ctx.Done()
  → cada goroutine lê do canal fechado (retorna imediatamente com zero value)
  → N goroutines, N desbloqueios simultâneos
```

Um channel aberto enviando um valor (`ch <- struct{}{}`) acorda apenas **uma** goroutine (unicast). Um channel **fechado** acorda **todas** as goroutines que esperam nele (broadcast). Esse comportamento é intencional e é exatamente por que o `context.Context` usa fechamento de channel e não envio de valor para sinalizar cancelamento.

### Context HTTP — Ciclo de Vida de Conexão ([[Processos]], [[System Calls]])

Quando um cliente HTTP se conecta ao servidor, o kernel cria um socket e entrega-o para o processo servidor via `accept()` — uma syscall descrita em [[System Calls]]. O `net/http` de Go cria um context para cada requisição e integra o ciclo de vida desse context com o ciclo de vida da conexão TCP.

```
Ciclo de vida de uma requisição HTTP em Go:

  Cliente conecta
       ↓
  kernel: accept() → socket fd
       ↓
  net/http: cria r.Context() ligado ao socket
       ↓
  handler executa:
    db.QueryContext(ctx, ...)     ← respeit o context
    http.NewRequestWithContext(ctx, ...) ← respeita o context
       ↓
  Cliente desconecta (TCP FIN ou RST)
       ↓
  net/http detecta EOF no socket (via epoll/kqueue — [[Dispositivos de IO]])
       ↓
  cancela r.Context() internamente
       ↓
  ctx.Done() fecha → goroutines do handler detectam cancelamento
       ↓
  query de banco cancelada (driver chama QueryContext abort)
  chamadas HTTP externas canceladas (transport fecha conexão)
  → recursos liberados sem completar trabalho desnecessário
```

Essa integração com [[System Calls]] é fundamental para a eficiência de servidores Go: sem context, uma query de banco de 10 segundos continuaria rodando mesmo após o cliente que a solicitou ter desconectado há 9 segundos. Com context, a query é abortada assim que a desconexão é detectada.

O `context.WithoutCancel` (Go 1.21+) é o mecanismo para operações que **não devem** ser canceladas quando o request context cancela — como audit logs ou métricas que precisam ser persistidos mesmo após a requisição falhar. É o análogo de um processo filho que sobrevive ao pai usando `setsid()` para se desligar do process group original.