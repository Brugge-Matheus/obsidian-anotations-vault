---
tags:
  - go
  - go/rede
aliases:
  - net/http
  - Servidor HTTP em Go
  - Cliente HTTP em Go
  - http.Handler
  - ServeMux
  - Middleware em Go
ordem: 44
modulo: "10 - Rede e Web"
---
# HTTP (net/http)

> **Nota 44 da trilha de Go.** O pacote **`net/http`** traz cliente e servidor HTTP prontos para produção na biblioteca padrão: muita gente roda serviços inteiros sem nenhum framework. Nesta nota veremos o servidor do zero (`Handler`, `HandlerFunc`, o roteador `ServeMux` com métodos e curingas), como ler requisições com segurança e escrever respostas corretamente, **middlewares**, a configuração de um `http.Server` de produção (timeouts, limites, proteção CSRF, **desligamento gracioso**), o **cliente** (timeouts, contexto, reaproveitamento de conexões, inclusive a mudança do Go 1.27), testes com `httptest`, e por dentro: uma goroutine por conexão, o cancelamento quando o cliente desiste, HTTP/2 e a comparação com Apache e Nginx.
>
> **Pré-requisitos:** [[Sockets e o Pacote net]], [[Interfaces]], [[context.Context]], [[JSON (marshal e unmarshal)]].

---

## 🧭 De onde viemos

Em [[Sockets e o Pacote net]] escrevemos um servidor TCP com `Listen`, `Accept` e uma goroutine por conexão, e vimos que TCP é um fluxo que precisa de **enquadramento**: HTTP/1.1 é exatamente um protocolo de enquadramento sobre TCP (linhas de cabeçalho até uma linha em branco, depois um corpo de tamanho `Content-Length`). Várias notas anteriores prometeram este assunto: [[Interfaces]] (`http.Handler`), [[Métodos e Receptores]] (`HandlerFunc`, um método num tipo função), [[Closures e Funções Anônimas]] (handlers e middlewares como closures), [[Composição vs Herança]] (embutir o `ResponseWriter`), [[Defer, Panic e Recover]] (recuperação de panics), [[O Padrão error em Go]] (respostas de erro) e [[context.Context]] (`r.Context()` e `Shutdown`). Do lado de redes e Nginx, você já viu [[HTTP e HTTPS]], [[Camada de Aplicação - 7]], [[Proxy x Proxy Reverso]] e [[Conexões, Workers e Keep-Alive]].

> 💡 **HTTP (HyperText Transfer Protocol):** protocolo de aplicação de **requisição e resposta**. O cliente envia um **método** (`GET`, `POST`...), um **caminho**, **cabeçalhos** e opcionalmente um **corpo**; o servidor devolve um **código de status** (`200`, `404`...), cabeçalhos e um corpo.

---

## 🧩 1. A peça central: `http.Handler`

Todo o servidor gira em torno de **uma interface de um método** (ver [[Interfaces]]):

```go
type Handler interface {
	ServeHTTP(w http.ResponseWriter, r *http.Request)
}
```

- `r *http.Request`: o que chegou (método, URL, cabeçalhos, corpo, contexto)
- `w http.ResponseWriter`: onde escrever a resposta (é um `io.Writer` com `Header()` e `WriteHeader(status)`)

Qualquer tipo com esse método é um handler. Como escrever um tipo para cada rota seria cansativo, existe o **adaptador** `HandlerFunc`, um tipo **função** com um método (o truque visto em [[Métodos e Receptores]]):

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	f(w, r)                     // o método só chama a própria função
}

// assim, qualquer função com a assinatura certa vira um Handler:
var h http.Handler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "olá")
})
```

Um servidor mínimo completo:

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /saude", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintln(w, `{"status":"ok"}`)
	})
	log.Fatal(http.ListenAndServe(":8080", mux))   // bloqueia; só retorna com erro
}
```

> ⚠️ Evite `http.HandleFunc(...)` (sem o `mux`) e `ListenAndServe(":8080", nil)`: eles usam o **`DefaultServeMux`**, uma variável global em que **qualquer pacote importado** pode registrar rotas (o `net/http/pprof`, por exemplo, registra `/debug/pprof/` só por ser importado). Crie o seu `ServeMux`.

---

## 🧭 2. Roteamento com `ServeMux` (Go 1.22+)

Desde o Go 1.22 o roteador padrão entende **métodos** e **curingas**, o que antes exigia bibliotecas como `gorilla/mux` ou `chi`. Saída real de várias requisições contra um único `mux`:

```go
mux.HandleFunc("GET /usuarios/{id}", ...)            // curinga: um segmento
mux.HandleFunc("GET /usuarios/eu", ...)              // literal: mais específico que {id}
mux.HandleFunc("GET /arquivos/{caminho...}", ...)    // curinga "resto do caminho"
mux.HandleFunc("GET /docs/", ...)                    // barra no fim: a árvore inteira
mux.HandleFunc("GET /{$}", ...)                      // {$}: só a raiz exata
```

```
GET    /usuarios/42                 → 200 "usuário 42"
GET    /usuarios/eu                 → 200 "o usuário logado"
DELETE /usuarios/42                 → 405 "Method Not Allowed" Allow=GET, HEAD
GET    /arquivos/img/2026/foto.png  → 200 "arquivo \"img/2026/foto.png\""
GET    /docs                        → 307 "<a href=\"/docs/\">Temporary Redirect</a>." Location=/docs/
GET    /docs/guia/intro             → 200 "árvore /docs/"
GET    /                            → 200 "só a raiz exata"
GET    /nada                        → 404 "404 page not found"
```

| Sintaxe do padrão | Significado |
| --- | --- |
| `"GET /caminho"` | Só esse método (e `GET` também aceita `HEAD`) |
| `"/caminho"` | Qualquer método |
| `{nome}` | Um segmento; leia com `r.PathValue("nome")` |
| `{nome...}` | O resto do caminho (só no fim) |
| Barra final (`/docs/`) | O prefixo e tudo abaixo |
| `{$}` | Só o caminho exato com a barra final |
| `"exemplo.com/"` | Restringe ao host |

Comportamentos automáticos que a saída mostra:

- **405 com `Allow`**: o caminho existe, mas não para esse método
- **Redirecionamento da barra final**: `/docs` → `/docs/`; desde o **Go 1.26** com **307** (antes era 301, "permanente", que navegadores guardam em cache)
- **O mais específico vence**, não o registrado primeiro: `/usuarios/eu` ganha de `/usuarios/{id}`

E se dois padrões se sobrepõem sem que nenhum seja mais específico, o `mux` **recusa no registro**, com uma mensagem exemplar. Saída real:

```go
mux.HandleFunc("GET /posts/{id}", h)
mux.HandleFunc("GET /{recurso}/latest", h)   // nenhum dos dois é mais específico
```

```
panic: pattern "GET /{recurso}/latest" (registered at .../main.go:9) conflicts with pattern "GET /posts/{id}" (registered at .../main.go:8):
	GET /{recurso}/latest and GET /posts/{id} both match some paths, like "/posts/latest".
	But neither is more specific than the other.
	GET /{recurso}/latest matches "/recurso/latest", but GET /posts/{id} doesn't.
	GET /posts/{id} matches "/posts/id", but GET /{recurso}/latest doesn't.
```

Como o conflito é detectado na **inicialização**, ele nunca chega à produção.

---

## 📥 3. Lendo a requisição

```go
func handler(w http.ResponseWriter, r *http.Request) {
	r.Method                          // "GET", "POST"...
	r.URL.Path                        // "/usuarios/42"
	r.PathValue("id")                 // "42" (curinga do padrão)
	r.URL.Query().Get("pagina")       // "?pagina=2" → "2" ("" se ausente)
	r.URL.Query()["tag"]              // []string: parâmetros repetidos
	r.Header.Get("Authorization")     // cabeçalhos (nome sem diferenciar maiúsculas)
	r.RemoteAddr                      // "ip:porta" de quem conectou (atrás de proxy, é o proxy!)
	r.Context()                       // cancelado se o cliente desistir (seção 9)
	r.Body                            // io.ReadCloser com o corpo (o servidor fecha por você)
}
```

### 3.1 JSON com limite de tamanho

O corpo vem do cliente, então **nunca** confie no tamanho dele. `http.MaxBytesReader` limita quanto será lido. Saída real de um handler de criação com limite de 64 bytes e JSON estrito:

```go
func criar(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 64) // no máximo 64 bytes
	var p struct {
		Nome string `json:"nome"`
	}
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	if err := dec.Decode(&p); err != nil {
		var mbe *http.MaxBytesError
		if errors.As(err, &mbe) {
			http.Error(w, fmt.Sprintf("corpo maior que %d bytes", mbe.Limit), http.StatusRequestEntityTooLarge)
			return
		}
		http.Error(w, "JSON inválido: "+err.Error(), http.StatusBadRequest)
		return
	}
	w.WriteHeader(http.StatusCreated)
	fmt.Fprintf(w, "criado: %s", p.Nome)
}
```

```
{"nome":"Ana"}               → 201 criado: Ana
{"nome":"Ana","idade":30}    → 400 JSON inválido: json: unknown field "idade"
{"nome":"AAAA...(100)"}      → 413 corpo maior que 64 bytes
{"nome":                     → 400 JSON inválido: unexpected EOF
```

(Decodificação de JSON e as opções do `json/v2` estão em [[JSON (marshal e unmarshal)]].)

> ⚠️ `r.RemoteAddr` atrás de um proxy reverso (Nginx, um balanceador) é o endereço do **proxy**. O IP do cliente vem em cabeçalhos como `X-Forwarded-For`, que só são confiáveis se o proxy os definir (ver [[Proxy x Proxy Reverso]]).

---

## 📤 4. Escrevendo a resposta

### 4.1 A ordem importa

Uma resposta HTTP é **status → cabeçalhos → corpo**, nessa ordem no fio. O `ResponseWriter` segue a mesma ordem:

```go
w.Header().Set("Content-Type", "application/json")   // 1) cabeçalhos
w.WriteHeader(http.StatusCreated)                     // 2) status (envia os cabeçalhos!)
json.NewEncoder(w).Encode(novo)                       // 3) corpo
```

O primeiro `Write` sem `WriteHeader` antes envia **200** implicitamente. Depois que o status saiu, mudar cabeçalhos ou status **não tem efeito**. Saída real de um handler que faz tudo na ordem errada:

```go
fmt.Fprint(w, "dados")                     // 1º Write: envia 200 implicitamente
w.Header().Set("X-Tarde", "ignorado")      // tarde demais
w.WriteHeader(http.StatusCreated)          // tarde demais
```

```
log do servidor: http: superfluous response.WriteHeader call from main.main.func1 (main.go:16)
status: 200 | X-Tarde:  | corpo: dados
```

### 4.2 Detecção de `Content-Type`

Sem `Content-Type`, o servidor **adivinha** pelos primeiros 512 bytes (`http.DetectContentType`). Saída real:

```
"<!DOCTYPE html><h1>o"         → 200, Content-Type: text/html; charset=utf-8
"{\"ok\":true}"                → 200, Content-Type: text/plain; charset=utf-8   ← JSON NÃO é detectado
"texto simples"                → 200, Content-Type: text/plain; charset=utf-8
"\x89PNG\r\n\x1a\n...."        → 200, Content-Type: image/png
```

> ⚠️ Respostas JSON precisam de `Content-Type: application/json` **explícito**. E adivinhar tipos é um risco de segurança (um texto enviado por usuário interpretado como HTML): defina sempre.

### 4.3 Ajudantes

```go
http.Error(w, "não encontrado", http.StatusNotFound)   // texto + status (define Content-Type e nosniff)
http.NotFound(w, r)
http.Redirect(w, r, "/login", http.StatusSeeOther)
http.ServeFile(w, r, "relatorio.pdf")                  // com Range, If-Modified-Since...
http.FileServerFS(os.DirFS("public"))                  // servir uma pasta (Go 1.22+)

func escreverJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("codificando resposta", "erro", err)   // o status já foi: só dá para registrar
	}
}
```

> 💡 Não exponha detalhes internos em mensagens de erro para o cliente (`err.Error()` de uma query SQL, caminhos de arquivo). Registre o erro completo no log e devolva uma mensagem genérica (ver [[O Padrão error em Go]]).

---

## 🧅 5. Middlewares

Um **middleware** é uma função que **recebe um handler e devolve outro**, adicionando comportamento antes e/ou depois. É o padrão **decorador**, feito com closures (ver [[Closures e Funções Anônimas]]):

```go
func comLog(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inicio := time.Now()
		sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(sw, r)                                  // chama o próximo
		slog.Info("req", "metodo", r.Method, "caminho", r.URL.Path,
			"status", sw.status, "duracao", time.Since(inicio))
	})
}

// captura o status embutindo o ResponseWriter (ver Composição vs Herança)
type statusWriter struct {
	http.ResponseWriter
	status int
}

func (sw *statusWriter) WriteHeader(code int) {
	sw.status = code
	sw.ResponseWriter.WriteHeader(code)
}

// permite que http.NewResponseController alcance o writer original (Flush, deadlines...)
func (sw *statusWriter) Unwrap() http.ResponseWriter { return sw.ResponseWriter }
```

```
requisição ──► comRecuperacao ──► comLog ──► comAutenticacao ──► handler
resposta   ◄──      (defer)    ◄── (depois) ◄──                ◄──
```

Encadeando (o primeiro da lista é o **mais externo**):

```go
func encadear(h http.Handler, ms ...func(http.Handler) http.Handler) http.Handler {
	for i := len(ms) - 1; i >= 0; i-- {
		h = ms[i](h)
	}
	return h
}

handler := encadear(mux, comRecuperacao, comLog, comRequestID)
```

### 5.1 Panics em handlers

O `net/http` já **recupera** panics de handlers: registra no log e **fecha a conexão**, sem derrubar o servidor. Saída real (com o stack trace omitido):

```
log do servidor: http: panic serving 127.0.0.1:39476: assignment to entry in nil map
cliente: Get "http://127.0.0.1:32931": EOF
log do servidor: http: panic serving 127.0.0.1:39486: assignment to entry in nil map
cliente (2ª): Get "http://127.0.0.1:32931": EOF
```

O servidor continua no ar e atende a segunda requisição (que também entra em panic). Mas o cliente recebe só um **EOF**, sem status. Um middleware de recuperação próprio (como o de [[Defer, Panic e Recover]]) devolve um `500` decente e registra no seu formato de log.

> ⚠️ Essa proteção vale só para a goroutine do handler. Uma goroutine **disparada** dentro do handler que entre em panic derruba o processo inteiro (ver [[Goroutines]]).

### 5.2 Proteção contra CSRF (Go 1.25+)

`http.CrossOriginProtection` rejeita requisições **não seguras** (POST, PUT, DELETE...) vindas de **outro site**, usando os cabeçalhos `Sec-Fetch-Site`/`Origin` que navegadores modernos enviam, sem tokens nem cookies. Saída real:

```go
mux.HandleFunc("POST /transferir", ...)
protegido := http.NewCrossOriginProtection().Handler(mux)
```

```
POST Sec-Fetch-Site: same-origin → 200 transferência feita
POST Sec-Fetch-Site: cross-site  → 403 cross-origin request detected from Sec-Fetch-Site header
GET  Sec-Fetch-Site: cross-site  → 405 Method Not Allowed     ← GET é "seguro": passa (e cai no 405 do mux)
```

Há como liberar origens confiáveis (`AddTrustedOrigin`) e padrões de rota (`AddInsecureBypassPattern`).

---

## 🛡️ 6. O `http.Server` de produção

`http.ListenAndServe` usa um servidor **sem nenhum timeout**: um cliente pode abrir uma conexão e enviar um byte por minuto para sempre, prendendo uma goroutine e um descritor (o ataque **Slowloris**). Em produção, configure um `http.Server`:

```go
srv := &http.Server{
	Addr:    ":8080",
	Handler: handler,

	ReadHeaderTimeout: 5 * time.Second,   // ✅ o mais importante: tempo para receber os cabeçalhos
	ReadTimeout:       30 * time.Second,  // cabeçalhos + corpo inteiro
	WriteTimeout:      30 * time.Second,  // do fim da leitura dos cabeçalhos até o fim da resposta
	IdleTimeout:       120 * time.Second, // conexão keep-alive ociosa

	MaxHeaderBytes: 1 << 20,              // 1 MB de cabeçalhos
	ErrorLog:       slog.NewLogLogger(slog.Default().Handler(), slog.LevelError),
}
```

| Campo | Protege contra | Cuidado |
| --- | --- | --- |
| `ReadHeaderTimeout` | Clientes lentos de propósito (Slowloris) | Sempre defina |
| `ReadTimeout` | Uploads infinitamente lentos | Atrapalha uploads grandes legítimos; ajuste por rota com `http.NewResponseController(w).SetReadDeadline` |
| `WriteTimeout` | Respostas que nunca terminam | Mata **streaming** e downloads longos (mesma solução por rota) |
| `IdleTimeout` | Conexões keep-alive esquecidas | Se zero, usa o `ReadTimeout` |
| `MaxHeaderBytes` | Cabeçalhos gigantes | Padrão 1 MB |
| `MaxHeaderValueCount` (Go 1.27) | Milhares de valores de cabeçalho | Padrão: `DefaultMaxHeaderValueCount` = 500 |

Esses timeouts são **deadlines** na conexão, exatamente os de [[Sockets e o Pacote net]].

### 6.1 Desligamento gracioso

Ao receber `SIGTERM` (o `docker stop`, ver [[Processo raiz (PID 1)]]), o servidor deve **parar de aceitar** conexões novas, **terminar** as requisições em andamento e só então sair. É o `srv.Shutdown(ctx)`:

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{Addr: ":8080", Handler: rotas(), ReadHeaderTimeout: 5 * time.Second}

	erros := make(chan error, 1)
	go func() { erros <- srv.ListenAndServe() }()

	select {
	case err := <-erros:                       // falhou ao subir (porta em uso...)
		log.Fatal(err)
	case <-ctx.Done():                         // chegou o sinal
	}

	slog.Info("desligando", "motivo", context.Cause(ctx))
	ctxDesligar, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctxDesligar); err != nil {   // espera as requisições em andamento
		slog.Error("desligamento forçado", "erro", err)
	}
}
```

Saída real de um experimento: uma requisição de 300 ms em andamento quando o `Shutdown` é chamado, e uma nova tentativa durante o desligamento:

```
[ 52ms] SIGTERM recebido: chamando Shutdown
[ 73ms] nova requisição durante o shutdown → Get "http://127.0.0.1:36189/lento": dial tcp 127.0.0.1:36189: connect: connection refused
[301ms] requisição antiga recebeu: resposta completa
[301ms] Serve retornou: http: Server closed | é ErrServerClosed? true
[328ms] Shutdown retornou: <nil>
```

| Etapa do `Shutdown` | Efeito |
| --- | --- |
| Fecha os listeners | Conexões novas são **recusadas** (o `connection refused` acima) |
| `Serve`/`ListenAndServe` retorna `http.ErrServerClosed` | Não é erro: é o sinal de que o desligamento começou |
| Fecha conexões ociosas e espera as ativas ficarem ociosas | A requisição de 300 ms termina normalmente |
| Se o `ctx` expirar antes | Devolve o erro do contexto (aí você pode chamar `srv.Close()` para forçar) |

> ⚠️ `Shutdown` **não** cancela os contextos das requisições em andamento nem espera goroutines que os handlers dispararam, e não fecha conexões "sequestradas" (WebSockets via `Hijack`). Para avisar trabalhos longos, use `srv.RegisterOnShutdown`, ou um `BaseContext` derivado do contexto do sinal.

---

## 📞 7. O cliente

### 7.1 Regras básicas

```go
var cliente = &http.Client{Timeout: 10 * time.Second}   // ✅ UM cliente, reaproveitado, COM timeout

func buscarUsuario(ctx context.Context, id int) (*Usuario, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet,
		fmt.Sprintf("https://api.exemplo.com/usuarios/%d", id), nil)
	if err != nil {
		return nil, err
	}
	req.Header.Set("Accept", "application/json")

	resp, err := cliente.Do(req)
	if err != nil {
		return nil, fmt.Errorf("buscar usuário %d: %w", id, err)   // rede, DNS, timeout, contexto
	}
	defer resp.Body.Close()                                      // ✅ SEMPRE

	if resp.StatusCode != http.StatusOK {                        // ✅ 404 e 500 NÃO são err!
		trecho, _ := io.ReadAll(io.LimitReader(resp.Body, 512))
		return nil, fmt.Errorf("buscar usuário %d: status %d: %s", id, resp.StatusCode, trecho)
	}

	var u Usuario
	if err := json.NewDecoder(resp.Body).Decode(&u); err != nil {
		return nil, fmt.Errorf("buscar usuário %d: decodificando: %w", id, err)
	}
	return &u, nil
}
```

| Regra | Por quê |
| --- | --- |
| Nunca use `http.Get`/`DefaultClient` em produção | O cliente padrão **não tem timeout**: uma requisição pode esperar para sempre |
| Reaproveite **um** `http.Client` | Ele carrega o pool de conexões; criar um por requisição desperdiça conexões |
| `NewRequestWithContext` | O cancelamento de quem chamou chega até a rede |
| `defer resp.Body.Close()` | Sem fechar, a conexão nunca volta ao pool (vazamento de conexões e goroutines) |
| Verifique `StatusCode` | `err` só cobre falhas de **transporte**; um `500` chega como `err == nil` |

### 7.2 Reaproveitamento de conexões (e a mudança do Go 1.27)

O `http.Transport` mantém conexões abertas (keep-alive) para reaproveitar entre requisições, evitando um novo handshake TCP (e TLS) a cada vez. Saída real, contando quantas de 3 requisições sequenciais reaproveitaram uma conexão (com `httptrace`), no Go 1.24 e no Go 1.27:

```
                        Go 1.24    Go 1.27
lendo o corpo              2          2      ← o máximo: a 1ª sempre abre uma conexão
sem ler o corpo            0          2      ← NOVIDADE do 1.27
sem fechar o corpo         0          0      ← nunca reaproveita
```

Até o Go 1.26, uma conexão só voltava ao pool se o corpo fosse **lido até o fim** e fechado. Era comum ver `io.Copy(io.Discard, resp.Body)` antes do `Close` só por isso. Desde o **Go 1.27**, o `Close` **drena** o resto do corpo automaticamente, até um limite conservador. Já **não fechar** continua impedindo o reaproveitamento em qualquer versão.

| Campo do `Transport` | Padrão | Observação |
| --- | --- | --- |
| `MaxIdleConns` | 100 (no `DefaultTransport`) | Total de conexões ociosas guardadas |
| `MaxIdleConnsPerHost` | **2** | Baixo para um serviço que chama muito **um** host: aumente |
| `MaxConnsPerHost` | 0 (sem limite) | Limite total (ativas + ociosas) por host |
| `IdleConnTimeout` | 90 s | Quanto uma conexão ociosa espera |

> 💡 O `Client.Timeout` cobre a requisição **inteira**, inclusive ler o corpo. Para limites por fase (conectar, TLS, esperar cabeçalhos), configure `net.Dialer.Timeout`, `TLSHandshakeTimeout` e `ResponseHeaderTimeout` no `Transport`.

---

## 🧪 8. Testando com `httptest`

```go
// testar um handler sem rede: um "gravador" de respostas
rec := httptest.NewRecorder()
req := httptest.NewRequest("GET", "/usuarios/42", nil)
mux.ServeHTTP(rec, req)
if rec.Code != http.StatusOK { ... }

// testar um cliente contra um servidor de verdade, numa porta local
srv := httptest.NewServer(handlerFalso)
defer srv.Close()
cliente.Get(srv.URL + "/usuarios/42")
```

Os experimentos de roteamento, Content-Type e CSRF desta nota foram feitos com `NewRecorder`; os de panic, cancelamento e reaproveitamento, com `NewServer`. O **Go 1.27** adicionou `httptest.NewTestServer`, que usa uma **rede falsa em memória** e combina com `testing/synctest` (tempo falso), permitindo testar timeouts de 30 s instantaneamente. Veremos em [[Testes com go test]].

---

## 🔬 9. Por dentro

### 9.1 Do `ListenAndServe` ao seu handler

```
srv.ListenAndServe()
  └─ net.Listen("tcp", ":8080")          socket, bind, listen (ver Sockets e o Pacote net)
  └─ srv.Serve(ln)
       loop:
         conn := ln.Accept()              accept4 (netpoller: a goroutine estaciona sem conexão)
         go c.serve(ctx)                  ◄── UMA GOROUTINE POR CONEXÃO
              loop (keep-alive):
                req := c.readRequest()    lê a linha inicial e os cabeçalhos (com deadlines)
                startBackgroundRead()     observa o socket: EOF → cancela r.Context()
                handler.ServeHTTP(w, req) ◄── o seu código
                w.finishRequest()         envia o que estiver no buffer
                se a conexão não é keep-alive: sai
```

Três consequências práticas:

- **Concorrência de graça e perigosa**: seus handlers rodam em **várias goroutines ao mesmo tempo**. Qualquer estado compartilhado (um map de cache, um contador) precisa de mutex ou atômico (ver [[sync.WaitGroup e sync.Mutex]])
- **Keep-alive**: a mesma goroutine atende várias requisições da mesma conexão, **em sequência**
- **HTTP/2** (automático com TLS): uma conexão carrega várias requisições **simultâneas** (*streams*), e cada stream ganha sua goroutine. Desde o Go 1.24, `Server.Protocols` permite até HTTP/2 sem TLS; no **Go 1.27**, o servidor HTTP/2 passou a respeitar as **prioridades** enviadas pelo cliente (RFC 9218), desligáveis com `Server.DisableClientPriority`

### 9.2 Quando o cliente desiste

Enquanto o handler roda, o servidor mantém uma **leitura em segundo plano** no socket. Se o cliente fechar a conexão, essa leitura recebe EOF e **cancela `r.Context()`**. Saída real, com um cliente que desiste após 100 ms de um handler que levaria 5 s:

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	select {
	case <-time.After(5 * time.Second): // "consulta lenta"
		fmt.Fprint(w, "pronto")
	case <-r.Context().Done():
		fmt.Println("handler: desisti após", ..., "→", context.Cause(r.Context()))
	}
}))

ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, srv.URL, nil)
_, err := http.DefaultClient.Do(req)
fmt.Println("cliente:", err)
```

```
cliente: Get "http://127.0.0.1:42651": context deadline exceeded
handler: desisti após 100ms → context canceled
```

É a cadeia completa de [[context.Context]]: o prazo do **cliente** fecha a conexão TCP → o **servidor** percebe pelo netpoller → `r.Context()` é cancelado → se o handler repassou `r.Context()` para o banco (`QueryContext`), a **consulta também é cancelada**.

### 9.3 Comparando modelos de servidor

| Servidor | Modelo | 10 mil conexões ociosas |
| --- | --- | --- |
| Apache (MPM prefork) | Um **processo** por conexão ([[Processos e Forks no CGI-FastCGI]]) | 10 mil processos |
| Apache (MPM worker) | Uma **thread** do SO por conexão | 10 mil threads (~8 MB de stack virtual cada) |
| Nginx | Poucos processos, cada um com um **laço de eventos** sobre `epoll` ([[Conexões, Workers e Keep-Alive]]) | Alguns processos; código em callbacks/máquinas de estado |
| Node.js | Uma thread com laço de eventos (libuv/epoll) | Uma thread; código assíncrono |
| **Go `net/http`** | Uma **goroutine** por conexão + netpoller (`epoll`) | 10 mil goroutines (~2–8 KB cada) sobre `GOMAXPROCS` threads |

Go junta o **modelo de programação** do servidor multithread (código sequencial, bloqueante, fácil de ler) com a **execução** do servidor orientado a eventos (poucas threads, `epoll`): é a síntese das três arquiteturas de [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]].

> 💡 Na prática, é comum colocar um **Nginx na frente** de um serviço Go: o Nginx cuida de TLS, arquivos estáticos, compressão, cache e balanceamento ([[Proxy x Proxy Reverso]], [[Load Balance]]), e o Go cuida da aplicação. Mas, diferente de PHP com FastCGI, o Go **não precisa** disso para funcionar: ele já é um servidor HTTP completo (compare com [[Servidores de aplicação Embutidos]]).

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `ListenAndServe` sem timeouts | Conexões presas por clientes lentos (Slowloris) | `http.Server` com `ReadHeaderTimeout` etc. |
| Usar o `DefaultServeMux` | Rotas registradas por pacotes importados | `http.NewServeMux()` |
| Cabeçalho depois do `Write` | Ignorado; log "superfluous WriteHeader" | Cabeçalhos → status → corpo |
| JSON sem `Content-Type` | Cliente recebe `text/plain` | Defina explicitamente |
| Ler o corpo sem limite | Memória esgotada | `http.MaxBytesReader` |
| Estado global em handlers sem trava | Data race (handlers são concorrentes) | Mutex/atômico; `go test -race` |
| `WriteTimeout` global com streaming | Downloads cortados | `ResponseController` por rota |
| `http.Get`/`DefaultClient` sem timeout | Requisições penduradas para sempre | `&http.Client{Timeout: ...}` |
| Um `http.Client` por requisição | Sem reaproveitamento de conexões | Um cliente compartilhado |
| Esquecer `resp.Body.Close()` | Vazamento de conexões e goroutines | `defer resp.Body.Close()` |
| Tratar só `err` | 404/500 passam como sucesso | Verifique `StatusCode` |
| `MaxIdleConnsPerHost` padrão (2) | Muitas conexões novas para um mesmo host | Aumente no `Transport` |
| Ignorar `r.Context()` | Trabalho continua depois que o cliente saiu | Repasse o contexto para tudo |
| Sair sem `Shutdown` | Requisições cortadas no deploy | `signal.NotifyContext` + `Shutdown` |
| Confiar em `r.RemoteAddr` atrás de proxy | IP errado em logs e limites | `X-Forwarded-For` de um proxy confiável |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `http.Handler` | Interface `ServeHTTP(w, r)`; `HandlerFunc` adapta funções |
| `ServeMux` (1.22+) | Métodos, `{curingas}`, `{resto...}`, `{$}`; o mais específico vence; conflitos são panic no registro; 405 e 307 automáticos |
| Requisição | `PathValue`, `URL.Query()`, `Header`, `Body` com `MaxBytesReader`, `Context()` |
| Resposta | Cabeçalhos → `WriteHeader` → corpo; 200 implícito; defina o `Content-Type` |
| Middleware | `func(http.Handler) http.Handler`; embutir o `ResponseWriter` + `Unwrap` |
| Panics | O servidor recupera e fecha a conexão; middleware próprio devolve 500 |
| CSRF (1.25+) | `http.CrossOriginProtection` com `Sec-Fetch-Site` |
| `http.Server` | `ReadHeaderTimeout` sempre; timeouts são deadlines no socket |
| Desligamento | `signal.NotifyContext` + `Shutdown(ctx)`; `ErrServerClosed` é normal |
| Cliente | Um `Client` com timeout; `NewRequestWithContext`; feche o corpo; verifique o status |
| Pool (1.27) | `Close` drena o corpo e reaproveita a conexão; `MaxIdleConnsPerHost` = 2 por padrão |
| Testes | `httptest.NewRecorder`, `NewServer`, `NewTestServer` (1.27) |
| Por dentro | Goroutine por conexão (HTTP/1) ou por stream (HTTP/2); leitura em segundo plano cancela `r.Context()` |

---

## 🔗 Conexões

**Dentro de Go**
- O servidor TCP, deadlines e o netpoller por baixo → [[Sockets e o Pacote net]]
- `http.Handler` como interface de um método → [[Interfaces]]
- `HandlerFunc`: método num tipo função → [[Métodos e Receptores]]
- Handlers e middlewares como closures → [[Closures e Funções Anônimas]]
- Embutir `http.ResponseWriter` → [[Composição vs Herança]]
- `Unwrap` e interfaces opcionais (`Flusher`) → [[Type Assertion e Type Switch]]
- Recuperação de panics → [[Defer, Panic e Recover]]
- Respostas de erro → [[O Padrão error em Go]], [[Wrapping e Unwrapping de Erros]]
- Corpos JSON → [[JSON (marshal e unmarshal)]]
- Corpos como streams → [[io.Reader e io.Writer]]
- `r.Context()`, `NotifyContext`, `Shutdown` → [[context.Context]]
- Handlers concorrentes e estado compartilhado → [[sync.WaitGroup e sync.Mutex]], [[Goroutines]]
- Limitar taxa e concorrência por cliente → [[Padrões de Concorrência]]
- Testes de handlers e `synctest` → [[Testes com go test]]
- `net/http/pprof` → [[Profiling e Diagnóstico]]

**Sistemas Operacionais**
- [[System Calls]] — `socket`, `bind`, `listen`, `accept`, `read`, `write`
- [[Arquivos]] — cada conexão é um descritor; o limite `ulimit -n`
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — Go como síntese dos modelos
- [[Implementações Híbridas]] — goroutines (N) sobre threads (M)
- [[Dispositivos de IO]] e [[Estados de Processos]] — E/S orientada a eventos; goroutines em `[IO wait]`

**Redes**
- [[HTTP e HTTPS]] — o protocolo, métodos, status, TLS
- [[Camada de Aplicação - 7]] e [[TCP-IP]] — onde o HTTP vive
- [[TLS e SSL]] — o que `ListenAndServeTLS` e o HTTP/2 usam
- [[Endereços, URLs e Domínios]] — `r.URL`, `r.Host`

**Nginx e Docker**
- [[Proxy x Proxy Reverso]] e [[API Gateway]] — Nginx na frente do serviço Go
- [[Conexões, Workers e Keep-Alive]] — o modelo de eventos do Nginx vs goroutines
- [[Cache HTTP]] e [[Compressão]] — o que muitas vezes fica a cargo do proxy
- [[Load Balance]] e [[Servidores de Backup e Health Check]] — rotas de saúde (`GET /saude`) e desligamento gracioso durante deploys
- [[Servidores de aplicação Embutidos]] e [[CGI e FastCGI]] — por que Go não precisa de FastCGI
- [[Processo raiz (PID 1)]] — o `SIGTERM` que dispara o `Shutdown`

---

⬅️ **Anterior:** [[Sockets e o Pacote net]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Testes com go test]] ➡️
