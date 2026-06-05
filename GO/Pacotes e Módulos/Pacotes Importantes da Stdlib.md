---
tags:
  - go
  - go/módulos
---
# Pacotes Importantes da Stdlib

> A biblioteca padrão do Go é rica e cobre a maioria dos casos sem dependências externas. Esta página documenta os pacotes que você usará com mais frequência no dia a dia, com detalhes das adições mais recentes.
> 

---

## 1. `fmt` — Formatação e I/O

```go
import "fmt"

// Imprimir
fmt.Println("hello", "world")              // args separados por espaço, \n no final
fmt.Print("sem nova linha")
fmt.Printf("nome: %s, idade: %d\n", "Alice", 30)
fmt.Fprintf(w, "valor: %d", n)            // para io.Writer
fmt.Fprintln(os.Stderr, "erro!")           // para io.Writer com \n

// Construir strings
s := fmt.Sprintf("valor: %.2f", 3.14159)  // "valor: 3.14"
s = fmt.Sprintf("%T", 42)                  // "int"
s = fmt.Sprintf("%#v", []int{1,2,3})      // "[]int{1, 2, 3}"

// Erros
err := fmt.Errorf("operação falhou: %w", causeErr)

// Ler entrada
var nome string
n, err := fmt.Scan(&nome)                  // lê até espaço
n, err  = fmt.Scanln(&nome)               // lê até newline
n, err  = fmt.Scanf("%s %d", &nome, &id)  // formatado
```

---

## 2. `os` — Sistema Operacional

```go
import "os"

// Argumentos e ambiente
os.Args                                    // []string — Args[0] é o executável
os.Getenv("DATABASE_URL")
os.Setenv("DEBUG", "true")
os.LookupEnv("PORT")                       // valor, bool — distingue ausente de vazio

// Arquivos
os.ReadFile("config.json")                 // lê tudo
os.WriteFile("out.txt", data, 0644)       // escreve tudo
f, _ := os.Open("r.txt")                  // somente leitura
f, _ = os.Create("w.txt")                 // cria/trunca
f, _ = os.OpenFile("log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

// Diretórios
os.Mkdir("pasta", 0755)
os.MkdirAll("a/b/c", 0755)
os.Remove("arquivo.txt")
os.RemoveAll("pasta/")
os.Rename("antigo.txt", "novo.txt")

// Arquivo/diretório temporário
f, _ := os.CreateTemp("", "prefix-*.json")
dir, _ := os.MkdirTemp("", "meuapp-*")
defer os.Remove(f.Name())
defer os.RemoveAll(dir)

// Informações
info, err := os.Stat("arquivo.txt")
os.IsNotExist(err)
os.IsPermission(err)
errors.Is(err, os.ErrNotExist)   // forma moderna (Go 1.13+)
errors.Is(err, os.ErrPermission)

// Encerrar (sem executar defers!)
os.Exit(0)   // sucesso
os.Exit(1)   // erro
```

---

## 3. `strings` — Manipulação de Strings

```go
import "strings"

s := "  Hello, Go World!  "

// Verificações — O(n)
strings.Contains(s, "Go")
strings.ContainsAny(s, "aeiou")           // contém alguma das letras?
strings.ContainsRune(s, 'G')
strings.HasPrefix(s, "  Hello")
strings.HasSuffix(s, "!  ")
strings.Count(s, "o")                     // contagem de ocorrências

// Busca
strings.Index(s, "Go")                    // -1 se não encontrar
strings.LastIndex(s, "o")
strings.IndexByte(s, 'G')
strings.IndexRune(s, 'G')
strings.IndexAny(s, "ABCG")

// Transformações
strings.ToUpper(s)
strings.ToLower(s)
strings.TrimSpace(s)
strings.Trim(s, " !")
strings.TrimLeft(s, " ")
strings.TrimRight(s, " ")
strings.TrimPrefix(s, "  ")
strings.TrimSuffix(s, "  ")
strings.TrimFunc(s, unicode.IsSpace)

// Substituição
strings.Replace(s, "o", "0", 1)          // n=-1 para todos
strings.ReplaceAll(s, "o", "0")

// Divisão e junção
strings.Split("a,b,c", ",")              // ["a", "b", "c"]
strings.SplitN("a,b,c", ",", 2)          // ["a", "b,c"]
strings.SplitAfter("a,b", ",")           // ["a,", "b"]
strings.Fields("  foo bar  ")            // ["foo", "bar"]
strings.Join([]string{"a","b"}, "-")     // "a-b"

// Go 1.18+: Cut — divide no separador, retorna antes/depois/encontrou
before, after, found := strings.Cut("user:alice", ":")   // "user", "alice", true
// Go 1.20+:
strings.CutPrefix("Hello, World", "Hello, ")   // "World", true
strings.CutSuffix("Hello, World", " World")    // "Hello,", true

// Builder
var sb strings.Builder
sb.Grow(100)   // pré-alocar (evita realocações)
sb.WriteString("Hello")
sb.WriteByte(',')
sb.WriteRune(' ')
sb.WriteString("World")
result := sb.String()   // "Hello, World"

// Reader — strings como io.Reader
r := strings.NewReader("conteúdo")
```

---

## 4. `strconv` — Conversões

```go
import "strconv"

// int ↔ string
strconv.Itoa(42)                          // "42" — mais rápido que Sprintf
n, err := strconv.Atoi("42")             // 42, nil

// Tipos precisos
strconv.FormatInt(255, 16)               // "ff" (base 16)
strconv.FormatInt(8, 2)                  // "1000" (base 2)
strconv.ParseInt("ff", 16, 64)          // 255, nil

strconv.FormatFloat(3.14, 'f', 2, 64)  // "3.14"
strconv.FormatFloat(3.14, 'e', 4, 64)  // "3.1400e+00"
strconv.FormatFloat(3.14, 'g', -1, 64) // "3.14" (mais curto)
strconv.ParseFloat("3.14", 64)          // 3.14, nil

strconv.FormatBool(true)                 // "true"
strconv.ParseBool("1")                  // true (aceita: 1,t,T,TRUE,true,True)

// Append — sem alocação extra ao adicionar ao buffer
buf := make([]byte, 0, 64)
buf = strconv.AppendInt(buf, 42, 10)    // "42"
buf = strconv.AppendFloat(buf, 3.14, 'f', 2, 64)

// Verificação
strconv.IsPrint('A')    // true
strconv.IsGraphic('A')  // true
```

---

## 5. `sort` e `slices` (Go 1.21+)

```go
import (
	"sort"
	"slices"
	"cmp"
)

nums := []int{3, 1, 4, 1, 5, 9}

// Pacote slices (Go 1.21+) — genérico, preferido ao sort para slices
slices.Sort(nums)                         // in-place, crescente
slices.SortStableFunc(nums, func(a, b int) int { return b - a })  // decrescente, estável

slices.Contains(nums, 5)                  // true
slices.Index(nums, 5)                     // índice ou -1
slices.Max(nums)                          // 9
slices.Min(nums)                          // 1

i, found := slices.BinarySearch(nums, 5)  // O(log n) — slice deve estar ordenado
slices.Equal(nums, outros)
slices.Compact(nums)                      // remove duplicatas adjacentes
slices.Reverse(nums)                      // in-place
copia := slices.Clone(nums)
nums = slices.Delete(nums, 2, 4)         // deleta índices 2 e 3
nums = slices.Insert(nums, 0, 99, 100)   // insere no índice 0

// cmp.Compare — comparador padrão para tipos ordered
cmp.Compare(3, 5)     // -1
cmp.Compare(5, 5)     // 0
cmp.Compare(7, 5)     // 1

// Ordenar structs
type Pessoa struct{ Nome string; Idade int }
pessoas := []Pessoa{{"Bob", 30}, {"Alice", 25}}
slices.SortFunc(pessoas, func(a, b Pessoa) int {
	return cmp.Compare(a.Idade, b.Idade)   // por idade crescente
})

// Pacote sort (pre-1.21 ou para sort.Interface)
sort.Ints(nums)
sort.Strings(strs)
sort.Slice(pessoas, func(i, j int) bool {
	return pessoas[i].Idade < pessoas[j].Idade
})
sort.SliceStable(pessoas, func(i, j int) bool {
	return pessoas[i].Nome < pessoas[j].Nome
})
```

---

## 6. `maps` (Go 1.21+)

```go
import "maps"

m := map[string]int{"a": 1, "b": 2, "c": 3}

maps.Clone(m)                              // cópia rasa
maps.Copy(dst, m)                          // copia pares para dst
maps.Equal(m1, m2)                         // true se mesmos pares
maps.EqualFunc(m1, m2, func(v1, v2 int) bool { return v1 == v2 })
maps.DeleteFunc(m, func(k string, v int) bool { return v > 1 })

// Iteração (Go 1.23+) — retorna iter.Seq2[K, V]
for k, v := range maps.All(m) {
	fmt.Println(k, v)
}
// Chaves e valores como slices
slices.Collect(maps.Keys(m))   // []string
slices.Collect(maps.Values(m)) // []int
slices.Sorted(maps.Keys(m))    // chaves ordenadas
```

---

## 7. `time` — Tempo e Durações

```go
import "time"

// Referência de formato Go: Mon Jan 2 15:04:05 MST 2006
// (02/01/2006 para dia/mês/ano — diferente de toda outra linguagem!)

agora := time.Now()
agora.Format("2006-01-02 15:04:05")        // "2024-03-15 10:30:00"
agora.Format("02/01/2006")                 // "15/03/2024"
agora.Format(time.RFC3339)                 // "2024-03-15T10:30:00-03:00"
agora.Format(time.DateTime)                // "2024-03-15 10:30:00" (Go 1.20+)
agora.Format(time.DateOnly)                // "2024-03-15"           (Go 1.20+)
agora.Format(time.TimeOnly)                // "10:30:00"             (Go 1.20+)

// Parse
t, err := time.Parse("2006-01-02", "2024-03-15")
t, err  = time.Parse(time.RFC3339, "2024-03-15T10:30:00Z")
t, err  = time.ParseInLocation("2006-01-02", "2024-03-15", time.Local)

// Operações
amanha := agora.Add(24 * time.Hour)
semana := agora.AddDate(0, 0, 7)
diff := amanha.Sub(agora)   // time.Duration

// Durações
d := 2*time.Hour + 30*time.Minute
d.Hours()    // 2.5
d.Minutes()  // 150
d.String()   // "2h30m0s"

// Truncar/arredondar
agora.Truncate(time.Hour)   // início da hora
agora.Round(time.Minute)    // minuto mais próximo

// Timers
time.Sleep(500 * time.Millisecond)
<-time.After(3 * time.Second)

ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()   // SEMPRE pare!

// Unix timestamp
agora.Unix()        // segundos
agora.UnixMilli()   // milissegundos
agora.UnixMicro()   // microsegundos
agora.UnixNano()    // nanossegundos
time.Unix(ts, 0)    // de volta para time.Time

// Comparação
agora.Before(amanha)   // true
agora.After(amanha)    // false
agora.Equal(agora)     // true
agora.IsZero()         // true se time.Time{}
```

---

## 8. `log/slog` — Logging Estruturado (Go 1.21+)

```go
import "log/slog"

// Logger padrão — vai para stderr como texto
slog.Info("servidor iniciado", "porta", 8080, "env", "prod")
slog.Warn("cache miss", "chave", "user:42", "latencia_ms", 150)
slog.Error("falha ao conectar", "err", err, "host", "db:5432")
slog.Debug("request recebida", "method", "POST", "path", "/api")

// Habilitar Debug (padrão é Info)
slog.SetLogLoggerLevel(slog.LevelDebug)

// Handler JSON — para produção
handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
	Level:     slog.LevelInfo,
	AddSource: true,   // inclui arquivo e linha no log
	ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
		// Customizar chaves — ex: renomear "time" para "timestamp"
		if a.Key == slog.TimeKey {
			a.Key = "timestamp"
		}
		return a
	},
})
logger := slog.New(handler)
slog.SetDefault(logger)   // define como logger padrão do pacote

// Logger com campos persistentes (context)
reqLogger := logger.With(
	"request_id", "abc-123",
	"user_id", 42,
	"service", "api",
)
reqLogger.Info("processando pedido", "pedido_id", 999)
// Output JSON:
// {"timestamp":"...","level":"INFO","source":{"function":"...","file":"...","line":42},
//  "msg":"processando pedido","request_id":"abc-123","user_id":42,"service":"api","pedido_id":999}

// Groups — namespace para atributos
logger.Info("conexão", slog.Group("db",
	slog.String("host", "localhost"),
	slog.Int("porta", 5432),
	slog.Duration("latencia", 2*time.Millisecond),
))
// Output: {..., "db":{"host":"localhost","porta":5432,"latencia":"2ms"}}

// Context — propagar logger via context
func withLogger(ctx context.Context, l *slog.Logger) context.Context {
	return context.WithValue(ctx, loggerKey{}, l)
}
func loggerFrom(ctx context.Context) *slog.Logger {
	if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok {
		return l
	}
	return slog.Default()
}

// slog.LogAttrs — mais eficiente para muitos atributos (evita boxing)
logger.LogAttrs(ctx, slog.LevelInfo, "operação concluída",
	slog.String("id", "123"),
	slog.Int("itens", 50),
	slog.Duration("duração", elapsed),
)
```

---

## 9. `context` — Cancelamento e Propagação

```go
import "context"

// Criar contexts
ctx := context.Background()       // raiz — nunca cancela
ctx = context.TODO()              // placeholder — ainda não sei qual contexto

// Com cancelamento manual
ctx, cancel := context.WithCancel(context.Background())
defer cancel()   // SEMPRE chame cancel para liberar recursos

// Com timeout
ctx, cancel = context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Com deadline absoluto
deadline := time.Now().Add(10 * time.Second)
ctx, cancel = context.WithDeadline(context.Background(), deadline)
defer cancel()

// Com valor (use sparingly — apenas para dados transversais)
type ctxKey struct{}
ctx = context.WithValue(ctx, ctxKey{}, "valor")
val := ctx.Value(ctxKey{})

// Verificar estado
ctx.Done()       // <-chan struct{} — fecha quando cancelado
ctx.Err()        // nil, context.Canceled, ou context.DeadlineExceeded
ctx.Deadline()   // time.Time, bool

// Uso em funções
func buscar(ctx context.Context, id int) (*Item, error) {
	select {
	case <-ctx.Done():
		return nil, ctx.Err()
	default:
	}

	// Propagar para chamadas subsequentes
	return db.QueryContext(ctx, "SELECT ...", id)
}

// Go 1.21+: context.WithoutCancel — cria cópia sem cancelamento
// (útil para operações de cleanup que não devem ser canceladas com a request)
ctxCleanup := context.WithoutCancel(ctx)
go cleanup(ctxCleanup)
```

---

## 10. Resumo — Pacotes Essenciais

| Pacote | Quando usar |
| --- | --- |
| `fmt` | Formatação, print, Sprintf |
| `os` | Arquivos, env, exit, args |
| `strings` | Manipulação de string, builder |
| `strconv` | Conversões string↔número |
| `errors` | Criar, verificar, juntar erros |
| `sort`  • `slices` (1.21+) | Ordenação, busca em slices |
| `maps` (1.21+) | Operações em maps |
| `time` | Datas, durações, timers |
| `log/slog` (1.21+) | Logging estruturado |
| `context` | Cancelamento, timeouts, valores |
| `io` | Interfaces de I/O, Copy, LimitReader |
| `bufio` | I/O com buffer, Scanner |
| `path/filepath` | Caminhos de arquivo, Walk |
| `encoding/json` | Serialização JSON |
| `net/http` | Cliente e servidor HTTP |
| `sync` | Mutex, WaitGroup, Once |
| `sync/atomic` | Operações atômicas de CPU |
| `math/rand/v2` (1.22+) | Números aleatórios |
| `crypto/rand` | Números criptograficamente seguros |
| `unicode/utf8` | Contagem e validação de runes |
| `regexp` | Expressões regulares |
| `reflect` | Inspeção de tipos em runtime |
| `testing` | Testes, benchmarks |

---

## 11. Como Funciona Internamente

### `os` — Wrapper Fino sobre Syscalls

```
os.Open("/etc/hosts")
    ↓
syscall.Open("/etc/hosts", syscall.O_RDONLY, 0)
    ↓
SYS_OPEN (Linux: syscall número 2, ou openat=257)
    ↓ instrução SYSCALL (x86-64)
    ↓ CPU muda de ring 3 → ring 0
    ↓ kernel aloca file descriptor (entrada na tabela de FDs do processo)
    ↓ retorna fd (inteiro >= 0) ou -errno
    ↑ CPU volta para ring 3
syscall.Open retorna (fd int, err error)
os.Open retorna (*os.File, error)

*os.File é apenas um wrapper em torno do file descriptor inteiro:
  type File struct {
      *file           // ponteiro para estrutura interna
  }
  type file struct {
      pfd  poll.FD    // poll.FD contém o fd int e epoll state
      name string     // path para mensagens de erro
      ...
  }
```

### `net` — Syscalls de Socket

```
net.Dial("tcp", "example.com:80")
    ↓
socket(AF_INET, SOCK_STREAM, 0)          → cria socket (fd)
    ↓
connect(fd, &sockaddr{...}, len)         → conecta ao servidor
    ↓
net.TCPConn{fd: *netFD{...}}             → retorna a conexão

Operações de I/O da conexão:
conn.Write(data) → sendto(fd, data, 0, nil, 0) → SYS_SENDMSG
conn.Read(buf)   → recvfrom(fd, buf, 0, nil)   → SYS_RECVMSG

No Linux, o runtime Go usa epoll internamente:
  epoll_create1() → cria instância epoll
  epoll_ctl()     → registra fds para monitoramento
  epoll_wait()    → bloqueia aguardando eventos de I/O

Isso permite que goroutines bloqueiem em I/O sem bloquear uma thread OS.
```

### `sync.Mutex` — Futex no Linux

```
Mutex em userspace (não-contendido — caminho rápido):
  mu.Lock():
    CAS atômico: se mutex == 0, seta para 1 → retorna imediatamente
    Nenhuma syscall! Tudo em userspace com instrução LOCK CMPXCHG

Mutex contendido (outra goroutine segura o lock):
  1. Spin por um tempo curto (tenta adquirir novamente)
  2. Se ainda bloqueado: futex(FUTEX_WAIT, &mutex, ...)
     → syscall que adormece a goroutine no kernel
     → kernel coloca a goroutine na wait queue do futex
  3. Quando o dono libera: futex(FUTEX_WAKE, &mutex, 1)
     → acorda uma goroutine da wait queue

Nome "futex" = Fast Userspace muTEX
Caminho rápido: zero syscalls (apenas instrução atômica de CPU)
Caminho lento: 1-2 syscalls (wait + wake)
```

### `runtime` — O Sistema Operacional Interno do Go

```
O pacote runtime implementa um mini-OS:

  Scheduler (runtime/proc.go):
    Goroutines = threads leves gerenciadas pelo runtime
    M (Machine) = thread OS real (pthread)
    P (Processor) = contexto de execução (GOMAXPROCS instâncias)
    G (Goroutine) = a goroutine em si

    Analogia com o OS:
    G ↔ Process/Thread (unidade de trabalho)
    P ↔ CPU (recurso de execução)
    M ↔ CPU físico (worker thread)

  GC (runtime/mgc.go):
    Tri-color mark-and-sweep concorrente
    Análogo ao gerenciador de memória virtual do OS:
    o OS mapeia páginas físicas; o GC rastreia objetos vivos no heap

  runtime.ReadMemStats() expõe:
    HeapAlloc: bytes alocados no heap
    NumGC: número de coletas de lixo
    GoroutineNum: goroutines ativas
    → instrospção do "kernel" Go, similar a /proc/meminfo no Linux
```

### `io` — Interfaces que Espelham File Descriptors POSIX

```
POSIX:                          Go (io package):
read(fd, buf, count)            Reader.Read(p []byte) (n int, err error)
write(fd, buf, count)           Writer.Write(p []byte) (n int, err error)
lseek(fd, offset, whence)       Seeker.Seek(offset int64, whence int) (int64, error)
close(fd)                       Closer.Close() error

io.Copy(dst, src):
  while true:
    n, err = src.Read(buf)        ← equivalente a read(srcfd, buf, 4096)
    if n > 0: dst.Write(buf[:n]) ← equivalente a write(dstfd, buf, n)
    if err == io.EOF: break

  io.EOF é análogo ao retorno 0 de read() (fim de arquivo no POSIX)

Qualquer tipo que implemente Reader pode ser usado onde um fd seria usado:
  *os.File, *bytes.Buffer, net.Conn, *gzip.Reader, ...
  → Duck typing para file descriptors
```

---

## 12. Conexão com Sistemas Operacionais

**`os` é um wrapper fino sobre syscalls — `os.Open` → `SYS_OPEN` → [[System Calls]] e [[Arquivos]]**
O pacote `os` não implementa nada por conta própria: cada operação de arquivo delega para `syscall.Open`, `syscall.Read`, `syscall.Write` etc., que emitem as syscalls correspondentes do kernel. `*os.File` é basicamente um inteiro (file descriptor) envolto em açúcar sintático Go. A hierarquia `os → syscall → kernel` é direta e rastreável.

**`net` usa syscalls de socket e `epoll` para I/O assíncrono → [[System Calls]] e [[Dispositivos de IO]]**
`net.Dial` emite `socket(2)` + `connect(2)`. Internamente, o runtime Go registra os file descriptors de rede em `epoll` (Linux) ou `kqueue` (macOS) para multiplexar I/O de milhares de goroutines em poucos threads OS — o mesmo mecanismo que servidores C (nginx, libevent) usam.

**`sync.Mutex` usa futex — lock sem syscall no caminho rápido → [[Threads POSIX]] e [[System Calls]]**
Um `sync.Mutex` não-contendido nunca emite syscall: usa `LOCK CMPXCHG` (instrução atômica de CPU) para adquirir o lock em userspace. Apenas quando contendido cai no `futex(FUTEX_WAIT)` — a syscall do Linux para mutexes eficientes. É exatamente como `pthread_mutex_lock` funciona internamente no glibc.

**`runtime` implementa um scheduler e GC — um mini-OS para goroutines → [[Processos]] e [[Gerenciamento de Memória]]**
O scheduler do runtime (modelo M:N) mapeia goroutines (G) em threads OS (M) através de processadores lógicos (P). É análogo ao scheduler do kernel que mapeia processos em CPUs físicas. O GC gerencia o heap como o kernel gerencia páginas de memória virtual. `runtime.ReadMemStats()` é o `/proc/meminfo` do mundo Go.

**`io.Reader`/`io.Writer` modelam diretamente `read(2)`/`write(2)` do POSIX → [[Arquivos]] e [[System Calls]]**
A assinatura `Read(p []byte) (n int, err error)` é `read(fd, buf, count)` em Go idiomático: mesmo contrato (retorna bytes lidos, `io.EOF` quando esgotado), mesma composabilidade (qualquer coisa que implemente `Reader` pode ser passada onde um fd seria usado). `io.Copy` é `sendfile(2)` em Go puro.