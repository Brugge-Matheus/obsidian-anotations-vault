---
tags:
  - go
  - go/io
---
# HTTP (net/http)

> O pacote `net/http` cobre cliente e servidor HTTP com alta performance. O servidor usa goroutines — uma por request — e é suficientemente eficiente para a maioria das aplicações sem frameworks externos.
> 

---

## 1. Como o Servidor Funciona Internamente

```go
Quando você chama http.ListenAndServe(":8080", handler):

1. Cria um TCP listener na porta 8080
2. Loop de Accept(): aguarda novas conexões TCP
3. Para cada conexão: go func() { srv.Serve(conn) }()
   — uma goroutine por conexão
4. Dentro da goroutine: lê a request HTTP, chama o handler, escreve a response
5. Se Keep-Alive: reutiliza a goroutine para múltiplas requests

HTTP/2 (quando TLS está ativo):
- Multiplexação: múltiplas streams por conexão
- Uma goroutine por stream (não por conexão)
```

---

## 2. Servidor HTTP Básico

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello, World!")
	})

	mux.HandleFunc("GET /saude", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintln(w, `{"status":"ok"}`)
	})

	log.Println("Servidor em http://localhost:8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		log.Fatal(err)
	}
}
```

---

## 3. Roteamento com `http.ServeMux` (Go 1.22+)

Go 1.22 melhorou significativamente o ServeMux com suporte a método HTTP e path parameters:

```go
mux := http.NewServeMux()

// Sintaxe: "MÉTODO /caminho" ou "/caminho" (qualquer método)
mux.HandleFunc("GET /usuarios", listarUsuarios)
mux.HandleFunc("POST /usuarios", criarUsuario)
mux.HandleFunc("GET /usuarios/{id}", buscarUsuario)       // path parameter
mux.HandleFunc("PUT /usuarios/{id}", atualizarUsuario)
mux.HandleFunc("DELETE /usuarios/{id}", deletarUsuario)

// Wildcard: {id...} captura tudo após o /
mux.HandleFunc("GET /arquivos/{caminho...}", servircArquivos)

// Path matching:
// "/foo" — corresponde apenas a /foo
// "/foo/" — corresponde a /foo/ e qualquer coisa abaixo
mux.HandleFunc("/estaticos/", http.StripPrefix("/estaticos/",
	http.FileServer(http.Dir("./public"))).ServeHTTP)
```

```go
// Extrair path parameters
func buscarUsuario(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")   // Go 1.22+

	n, err := strconv.Atoi(id)
	if err != nil {
		http.Error(w, "id inválido", http.StatusBadRequest)
		return
	}
	// usar n...
}
```

---

## 4. `http.Handler` — A Interface Central

```go
// Qualquer tipo que implementa ServeHTTP é um handler
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// Implementação como struct — permite injeção de dependências
type UsuarioHandler struct {
	repo   RepositorioUsuario
	logger *slog.Logger
}

func (h *UsuarioHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		h.listar(w, r)
	case http.MethodPost:
		h.criar(w, r)
	default:
		http.Error(w, "método não permitido", http.StatusMethodNotAllowed)
	}
}

func (h *UsuarioHandler) listar(w http.ResponseWriter, r *http.Request) {
	usuarios, err := h.repo.ListarTodos()
	if err != nil {
		h.logger.Error("erro ao listar", "err", err)
		http.Error(w, "erro interno", http.StatusInternalServerError)
		return
	}
	jsonResponse(w, http.StatusOK, usuarios)
}
```

---

## 5. Lendo a Request

```go
func meuHandler(w http.ResponseWriter, r *http.Request) {
	// Método e URL
	r.Method            // "GET", "POST", etc.
	r.URL.Path          // "/api/usuarios"
	r.URL.RawQuery      // "page=1&limit=10"
	r.URL.Query()       // url.Values — map[string][]string

	// Query params
	page := r.URL.Query().Get("page")       // "" se ausente
	limit := r.URL.Query().Get("limit")
	tags := r.URL.Query()["tags"]           // []string — multi-value

	// Path params (Go 1.22+)
	id := r.PathValue("id")

	// Headers
	auth := r.Header.Get("Authorization")
	ct := r.Header.Get("Content-Type")

	// Context — para dados associados à request
	ctx := r.Context()   // propaga cancellation e valores

	// Body
	defer r.Body.Close()   // sempre feche — evita goroutine leak

	// Para JSON
	var payload struct {
		Nome  string `json:"nome"`
		Email string `json:"email"`
	}
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()   // rejeita campos desconhecidos
	if err := dec.Decode(&payload); err != nil {
		http.Error(w, "JSON inválido: "+err.Error(), http.StatusBadRequest)
		return
	}

	// Form data
	if err := r.ParseForm(); err != nil {
		http.Error(w, "form inválido", http.StatusBadRequest)
		return
	}
	nome := r.FormValue("nome")

	// Upload de arquivo (multipart)
	r.ParseMultipartForm(10 << 20)   // 10MB em memória
	file, header, err := r.FormFile("arquivo")
	if err == nil {
		defer file.Close()
		fmt.Println(header.Filename, header.Size)
	}
}
```

---

## 6. Escrevendo a Response

```go
func meuHandler(w http.ResponseWriter, r *http.Request) {
	// IMPORTANTE: definir headers ANTES de WriteHeader ou Write
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("X-Request-ID", r.Header.Get("X-Request-ID"))
	w.Header().Add("Vary", "Accept-Encoding")

	// Status code (padrão: 200 se omitido)
	w.WriteHeader(http.StatusCreated)   // 201

	// Escrever corpo
	json.NewEncoder(w).Encode(struct {
		ID      int    `json:"id"`
		Mensagem string `json:"mensagem"`
	}{ID: 42, Mensagem: "criado"})
}

// Helpers comuns
http.Error(w, "não encontrado", http.StatusNotFound)
http.Redirect(w, r, "/nova-url", http.StatusSeeOther)
http.NotFound(w, r)

// Helper para JSON responses
func jsonResponse(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("erro ao serializar response", "err", err)
	}
}

// Helper para erros JSON
func jsonError(w http.ResponseWriter, status int, msg string) {
	jsonResponse(w, status, map[string]string{"erro": msg})
}
```

---

## 7. Middleware

Middlewares são funções que envolvem handlers, formando uma "cadeia":

```go
type Middleware func(http.Handler) http.Handler

// Logging estruturado
func comLogging(logger *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			inicio := time.Now()

			// Wrapper para capturar o status code
			sw := &statusWriter{ResponseWriter: w, status: 200}
			next.ServeHTTP(sw, r)

			logger.Info("request",
				"method", r.Method,
				"path", r.URL.Path,
				"status", sw.status,
				"duration", time.Since(inicio),
			)
		})
	}
}

type statusWriter struct {
	http.ResponseWriter
	status int
}
func (sw *statusWriter) WriteHeader(status int) {
	sw.status = status
	sw.ResponseWriter.WriteHeader(status)
}

// Recuperação de panic
func comRecovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				slog.Error("panic em handler", "err", err, "stack", debug.Stack())
				http.Error(w, "erro interno", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

// CORS
func comCORS(origens []string) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origem := r.Header.Get("Origin")
			for _, o := range origens {
				if o == "*" || o == origem {
					w.Header().Set("Access-Control-Allow-Origin", origem)
					break
				}
			}
			w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

			if r.Method == http.MethodOptions {
				w.WriteHeader(http.StatusNoContent)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

// Encadear middlewares (aplicados de dentro para fora)
func Chain(h http.Handler, ms ...Middleware) http.Handler {
	for i := len(ms) - 1; i >= 0; i-- {
		h = ms[i](h)
	}
	return h
}

// Uso
handler := Chain(mux,
	comRecovery,
	comLogging(logger),
	comCORS([]string{"https://meusite.com"}),
)
http.ListenAndServe(":8080", handler)
```

---

## 8. Servidor com Graceful Shutdown

```go
func main() {
	mux := configurarRotas()

	servidor := &http.Server{
		Addr:    ":8080",
		Handler: mux,

		// Timeouts — ESSENCIAIS em produção
		ReadTimeout:       15 * time.Second,   // tempo para ler toda a request
		ReadHeaderTimeout: 5 * time.Second,    // tempo para ler os headers
		WriteTimeout:      15 * time.Second,   // tempo para escrever a response
		IdleTimeout:       120 * time.Second,  // tempo de idle em Keep-Alive

		MaxHeaderBytes: 1 << 20,   // 1MB
	}

	// Iniciar em goroutine separada
	go func() {
		slog.Info("servidor iniciado", "addr", servidor.Addr)
		if err := servidor.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("erro fatal", "err", err)
			os.Exit(1)
		}
	}()

	// Aguardar sinal de encerramento (Ctrl+C, SIGTERM do Docker/k8s)
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
	<-quit

	slog.Info("encerrando servidor...")

	// Graceful shutdown: aguarda requests em andamento terminarem
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := servidor.Shutdown(ctx); err != nil {
		slog.Error("erro no shutdown", "err", err)
	}
	slog.Info("servidor encerrado")
}
```

---

## 9. Cliente HTTP

```go
// ✅ Reuse o mesmo cliente — connection pooling interno
var httpCliente = &http.Client{
	Timeout: 30 * time.Second,
	Transport: &http.Transport{
		MaxIdleConns:        100,
		MaxIdleConnsPerHost: 10,
		IdleConnTimeout:     90 * time.Second,
		TLSHandshakeTimeout: 10 * time.Second,
	},
}

// GET com context
func buscarUsuario(ctx context.Context, id int) (*Usuario, error) {
	url := fmt.Sprintf("https://api.exemplo.com/usuarios/%d", id)

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("criar request: %w", err)
	}
	req.Header.Set("Authorization", "Bearer "+token)

	resp, err := httpCliente.Do(req)
	if err != nil {
		return nil, fmt.Errorf("executar request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(io.LimitReader(resp.Body, 1024))
		return nil, fmt.Errorf("status %d: %s", resp.StatusCode, body)
	}

	var usuario Usuario
	if err := json.NewDecoder(resp.Body).Decode(&usuario); err != nil {
		return nil, fmt.Errorf("decodificar response: %w", err)
	}
	return &usuario, nil
}

// POST com JSON
func criarUsuario(ctx context.Context, u *Usuario) (*Usuario, error) {
	corpo, err := json.Marshal(u)
	if err != nil {
		return nil, err
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost,
		"https://api.exemplo.com/usuarios",
		bytes.NewReader(corpo),
	)
	req.Header.Set("Content-Type", "application/json")

	resp, err := httpCliente.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var novo Usuario
	json.NewDecoder(resp.Body).Decode(&novo)
	return &novo, nil
}
```

---

## 10. Context em Handlers

```go
// Propagar valores via context (request-scoped)
type contextKey string
const requestIDKey contextKey = "request_id"

func middlewareRequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := uuid.New().String()
		ctx := context.WithValue(r.Context(), requestIDKey, id)
		w.Header().Set("X-Request-ID", id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// Ler o valor no handler
func meuHandler(w http.ResponseWriter, r *http.Request) {
	reqID, _ := r.Context().Value(requestIDKey).(string)
	slog.Info("processando", "request_id", reqID)

	// O context cancela quando o cliente desconecta
	select {
	case resultado := <-operacaoLonga():
		jsonResponse(w, http.StatusOK, resultado)
	case <-r.Context().Done():
		// Cliente cancelou — não precisa escrever response
		slog.Info("cliente cancelou", "request_id", reqID)
	}
}

---

## 11. Por Dentro: HTTP, Syscalls e o Sistema Operacional

### Do `ListenAndServe` até o kernel

Quando você chama `http.ListenAndServe(":8080", handler)`, uma cadeia inteira de syscalls acontece antes de qualquer request ser processada:

```
http.ListenAndServe(":8080", handler)
          │
          ▼
net.Listen("tcp", ":8080")
          │
          ├─► syscall.Socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
          │       └─ retorna fd = 3  (file descriptor — inteiro no kernel)
          │
          ├─► syscall.Bind(fd, addr{port: 8080})
          │       └─ associa o socket à porta 8080
          │
          ├─► syscall.Listen(fd, backlog=128)
          │       └─ coloca o socket em modo de escuta
          │
          └─► loop: syscall.Accept(fd)
                  └─ bloqueia até nova conexão TCP chegar
                     retorna connFd (novo fd por conexão)
```

Cada `Accept()` retorna um novo file descriptor — o kernel cria uma entrada na **tabela de file descriptors do processo** para cada conexão TCP ativa. Em Go, esses fds são wrappados por `net.Conn`.

### Socket como arquivo: "tudo é arquivo" no POSIX

```
Processo Go
┌─────────────────────────────────┐
│  fd table (por processo)        │
│  fd 0 → stdin                   │
│  fd 1 → stdout                  │
│  fd 2 → stderr                  │
│  fd 3 → socket (listen)         │
│  fd 4 → conn de cliente A       │  ← net.Conn wrappa este fd
│  fd 5 → conn de cliente B       │
│  ...                            │
└─────────────────────────────────┘

Ler de net.Conn → read(fd, buf, len) → syscall SYS_READ no kernel
                                      kernel copia dados do buffer TCP para buf
```

Isso conecta diretamente com [[Arquivos]] (POSIX — tudo é fd) e [[System Calls]] (read/write/accept são syscalls).

### Goroutine por conexão: M:N threading, não 1:1

```
Modelo Apache (1 thread OS por conexão):          Modelo Go (goroutines):
─────────────────────────────────────────         ──────────────────────────────────
1000 conexões → 1000 threads OS                   1000 conexões → 1000 goroutines
1000 × ~8MB stack = ~8GB RAM                       1000 × ~4KB stack = ~4MB RAM
context switch: ~1-10μs (kernel)                   context switch: ~100ns (runtime)
```

Go usa um scheduler M:N (N goroutines sobre M threads OS). Cada goroutine começa com ~4KB de stack e cresce conforme necessário. O runtime multiplexia goroutines sobre um pool de threads OS (GOMAXPROCS threads por padrão = número de CPUs).

```
Goroutines (N)          OS Threads (M)        CPU cores
  g1 ─────────────┐
  g2 ──────────┐  │
  g3 ─────┐    │  ├───► P1 ──► thread1 ──► core1
  g4 ──┐  │    │  │
  g5 ─►│  │    │  ├───► P2 ──► thread2 ──► core2
  ...  └──┴────┘  │
                  └───► P3 ──► thread3 ──► core3
N goroutines (baratas) → M threads (= CPUs)
```

Isso conecta com [[Implementando Threads em User Space]] — o scheduler de goroutines é essencialmente um threading em user space com M:N mapping, similar ao que Tanenbaum descreve para threads de usuário que evitam syscalls de thread creation.

### I/O não-bloqueante + netpoller: como o Go não trava uma thread

```
Goroutine tenta ler do socket:
          │
          ▼
     fd não tem dados ainda
          │
          ▼
runtime park goroutine (estado: waiting)
          │
          ▼
netpoller registra fd no epoll/kqueue:
  Linux:  epoll_ctl(epfd, EPOLL_CTL_ADD, connFd, EPOLLIN)
  macOS:  kevent(kq, &change, ...)
          │
          ▼
thread OS fica livre para rodar outra goroutine
          │
          ▼
kernel sinaliza: "fd N tem dados" (epoll_wait retorna)
          │
          ▼
netpoller acorda a goroutine (estado: runnable)
          │
          ▼
goroutine executa Read() — dados já estão no buffer
```

O Go usa I/O não-bloqueante internamente (`O_NONBLOCK` no fd) + `epoll` no Linux / `kqueue` no macOS para multiplexar milhares de conexões sem bloquear threads. Isso conecta com [[Dispositivos de IO]] (I/O bloqueante vs não-bloqueante, polling vs interrupt-driven) e [[Estados de Processos]] (a goroutine vai para o estado "bloqueado/esperando" sem bloquear a thread OS).

### Comparação de modelos de servidores

```
Servidor    | Modelo              | Concorrência com 10k conexões
────────────┼─────────────────────┼──────────────────────────────
Apache httpd| 1 thread OS/conexão | 10k threads OS (~80GB RAM)
Nginx       | event loop (epoll)  | 1 thread + epoll, callbacks
Node.js     | event loop (libuv)  | 1 thread + epoll, callbacks
Go net/http | goroutine/conexão   | 10k goroutines (~40MB RAM)
            | + netpoller (epoll) | M threads OS (= num CPUs)
```

Go combina o modelo de programação simples (goroutines parecem threads — código linear, sem callbacks) com a eficiência de epoll/kqueue internamente.

---

## 12. Conexão com Sistemas Operacionais

- **[[System Calls]]**: `http.ListenAndServe` dispara uma cadeia de syscalls — `socket()`, `bind()`, `listen()`, `accept()`. Cada `Read`/`Write` em `net.Conn` vira `read(fd)`/`write(fd)` no kernel.

- **[[Arquivos]]**: No POSIX, sockets são file descriptors — inteiros indexando a tabela de fds do processo. `net.Conn` wrappa um fd gerenciado pelo kernel. "Tudo é arquivo" se aplica diretamente aqui.

- **[[Implementando Threads em User Space]]**: Goroutines são threads de usuário leves (M:N). O scheduler do runtime Go multiplexia N goroutines sobre M threads OS — exatamente o modelo de user-space threading descrito por Tanenbaum, com a vantagem de stacks crescentes e preempção cooperativa/assíncrona.

- **[[Dispositivos de IO]]**: O netpoller do Go usa `epoll` (Linux) ou `kqueue` (macOS) internamente. Goroutines bloqueadas em I/O de rede não bloqueiam threads OS — o fd é registrado no epoll e a goroutine é acordada quando há dados. I/O não-bloqueante (`O_NONBLOCK`) + event notification = concorrência escalável.

- **[[Estados de Processos]]**: Uma goroutine bloqueada em `conn.Read()` passa para o estado "waiting" (parked). O scheduler move a thread OS para outra goroutine runnable. Quando o netpoller detecta dados no fd, a goroutine volta para "runnable" — análogo a como o kernel move processos de "bloqueado" para "pronto".

- **[[Processos]]**: Cada conexão HTTP tem seu goroutine, mas todos compartilham o mesmo processo Go. A tabela de file descriptors é por processo — todas as goroutines acessam os fds via o mesmo PCB do processo.
```