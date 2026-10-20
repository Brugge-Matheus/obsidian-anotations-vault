---
tags:
  - go
  - go/pacotes
aliases:
  - Biblioteca padrão
  - Stdlib
  - Standard library
  - Pacote time
  - log/slog
  - flag
ordem: 32
modulo: "07 - Pacotes e Módulos"
---
# Pacotes Importantes da Stdlib

> **Nota 32 da trilha de Go.** A **biblioteca padrão** de Go é uma das mais completas entre as linguagens compiladas: servidor HTTP de produção, JSON, criptografia, testes, templates, compressão, logging estruturado... tudo sem dependências externas. Esta nota é um **mapa**: os pacotes que você vai usar com mais frequência, organizados por área, com exemplos curtos e as armadilhas mais comuns (como o famoso formato de data `2006-01-02`). Os pacotes que merecem estudo aprofundado têm notas próprias, indicadas em cada seção.
>
> **Pré-requisitos:** [[Criando Pacotes]], [[Sistema de Módulos (go mod)]].

---

## 🧭 De onde viemos

Em [[Criando Pacotes]] e [[Sistema de Módulos (go mod)]] vimos como o código é organizado e como dependências externas entram no projeto. Antes de adicionar uma dependência, vale perguntar: **a biblioteca padrão já resolve?** Muitas vezes, sim. E o provérbio *"a little copying is better than a little dependency"* ([[Convenções da linguagem]]) vale em dobro quando a stdlib tem o que você precisa.

> 💡 Toda a biblioteca padrão é coberta pela **promessa de compatibilidade do Go 1**: código escrito contra ela continua funcionando nas próximas versões. Além disso, existem os pacotes **`golang.org/x/...`** ("x" de experimental/estendido), mantidos pelo time do Go, mas fora da garantia: `x/sync/errgroup`, `x/text`, `x/crypto`, `x/net`, `x/tools`.

---

## 🗺️ 1. O mapa

| Área | Pacotes principais | Nota aprofundada |
| --- | --- | --- |
| Texto | `fmt`, `strings`, `strconv`, `unicode/utf8`, `bytes`, `regexp`, `text/template` | [[Strings em Profundidade]] |
| Coleções | `slices`, `maps`, `cmp`, `sort`, `container/heap`, `container/list` | [[Arrays e Slices]], [[Maps]] |
| Iteração | `iter` | [[Iteradores]] |
| Erros | `errors` | [[O Padrão error em Go]] |
| Tempo | `time` | **esta nota** |
| Entrada/saída | `io`, `bufio`, `os`, `io/fs`, `path/filepath`, `embed` | [[io.Reader e io.Writer]], [[Leitura e Escrita de Arquivos]] |
| Codificação | `encoding/json` (+ `json/v2`), `encoding/csv`, `encoding/base64`, `encoding/hex`, `encoding/binary`, `encoding/xml` | [[JSON (marshal e unmarshal)]] |
| Rede | `net`, `net/http`, `net/url`, `net/netip` | [[Sockets e o Pacote net]], [[HTTP (net-http)]] |
| Concorrência | `sync`, `sync/atomic`, `context` | [[sync.WaitGroup e sync.Mutex]], [[context.Context]] |
| Logs | `log`, `log/slog` | **esta nota** |
| Linha de comando e processos | `flag`, `os/exec`, `os/signal` | **esta nota** |
| Criptografia | `crypto/rand`, `crypto/sha256`, `crypto/hmac`, `crypto/tls`, `crypto/mlkem`, `crypto/mldsa` | **esta nota** |
| Matemática | `math`, `math/bits`, `math/big`, `math/rand/v2` | [[Operações Aritméticas]] |
| Runtime e diagnóstico | `runtime`, `runtime/debug`, `runtime/pprof`, `runtime/trace` | [[Profiling e Diagnóstico]] |
| Testes | `testing`, `testing/synctest`, `net/http/httptest` | [[Testes com go test]] |
| Reflexão | `reflect`, `unsafe` | [[Reflection]], [[unsafe e cgo]] |

---

## ⏰ 2. `time`: datas, durações e o formato de referência

### 2.1 `time.Time` e `time.Duration`

```go
agora := time.Now()
t := time.Date(2026, time.October, 20, 14, 5, 9, 0, time.UTC)

d := 90*time.Minute + 30*time.Second      // time.Duration (um int64 em nanossegundos)
fmt.Println(d, d.Hours())                 // 1h30m30s 1.5083333333333333

amanha := t.Add(24 * time.Hour)           // somar duração
diff := t.Sub(outra)                      // diferença → Duration
t.Before(outra), t.After(outra), t.Equal(outra)
time.Since(inicio)                        // = time.Now().Sub(inicio): medir tempo decorrido
time.Until(prazo)
```

> ⚠️ `time.Duration` é um **tipo definido** sobre `int64` ([[Tipos de Dados]]). `time.Sleep(5)` dorme **5 nanossegundos**! Sempre multiplique por uma unidade: `time.Sleep(5 * time.Second)`.

### 2.2 Formatação: o "layout de referência"

Go não usa `%Y-%m-%d` (C, Python, PHP) nem `yyyy-MM-dd` (Java). Usa uma **data de exemplo**, que você reescreve no formato desejado:

```
        Mon Jan 2 15:04:05 MST 2006
        ou, em ordem:  01/02 03:04:05PM '06 -0700
                        │  │  │  │  │      │    │
                       mês dia h min s    ano  fuso
                        1   2  3  4  5     6    7     ← 1, 2, 3, 4, 5, 6, 7!
```

Saídas reais:

```go
t.Format("2006-01-02 15:04:05")   // 2026-10-20 14:05:09
t.Format("02/01/2006")            // 20/10/2026   (formato brasileiro)
t.Format(time.RFC3339)            // 2026-10-20T14:05:09Z
t.Format(time.DateOnly)           // 2026-10-20 (constante do Go 1.20)
t.Weekday()                       // Tuesday

p, err := time.Parse("02/01/2006", "25/12/2026")
p.Format("Mon Jan 2")             // Fri Dec 25
```

| Elemento | Layout |
| --- | --- |
| Ano | `2006` (4 dígitos), `06` |
| Mês | `01`, `1`, `Jan`, `January` |
| Dia | `02`, `2`, `_2` |
| Hora | `15` (24h), `03`/`3` (12h) + `PM` |
| Minuto / segundo | `04` / `05` |
| Frações | `.000` (milissegundos), `.000000000` |
| Fuso | `MST`, `-0700`, `-07:00`, `Z07:00` |
| Dia da semana | `Mon`, `Monday` |

> ⚠️ Armadilha clássica: `"2006-01-02"` com `"02"` e `"01"` trocados (`"2006-02-01"`) formata **ano-dia-mês** sem dar erro.

### 2.3 Fusos horários

Saída real:

```go
loc, err := time.LoadLocation("America/Sao_Paulo")   // banco de fusos IANA
fmt.Println(t.In(loc).Format("15:04 MST"))           // 11:05 -03
```

> 💡 `LoadLocation` lê o banco de fusos do sistema (`/usr/share/zoneinfo`). Em imagens Docker `FROM scratch`, ele **não existe**: importe `_ "time/tzdata"` para embutir o banco no binário (~450 KB). Ver [[Imagens]].

### 2.4 Relógio monotônico

`time.Now()` guarda **dois** relógios: o de parede (data e hora, que pode "pular" com ajustes de NTP ou horário de verão) e um **monotônico** (só anda para frente). `t.Sub` e `time.Since` usam o monotônico automaticamente, então medições de duração não são afetadas por ajustes do relógio do sistema.

> 💡 Isso corresponde aos relógios `CLOCK_REALTIME` e `CLOCK_MONOTONIC` do kernel, que você viu ao estudar a syscall `time` e o problema do ano 2038 em [[Syscalls Diversas]].

### 2.5 Timers e tickers

```go
timer := time.NewTimer(2 * time.Second)   // dispara uma vez
<-timer.C

ticker := time.NewTicker(time.Second)     // dispara periodicamente
defer ticker.Stop()
for t := range ticker.C { ... }

time.AfterFunc(5*time.Second, func() { ... })   // executa uma função depois de um tempo
```

> 💡 Desde o Go 1.23, timers e tickers não referenciados são coletados pelo GC mesmo sem `Stop()`, e seus channels são **síncronos** (desde o 1.27, sem opção de voltar ao comportamento antigo). Veremos seu uso com `select` em [[Select]].

---

## 📝 3. Logs: `log` e `log/slog`

### 3.1 `log`: simples

```go
log.Println("servidor iniciado")          // 2026/10/20 14:05:09 servidor iniciado
log.Printf("porta %d", 8080)
log.Fatal("não deu")                      // ⚠️ imprime e chama os.Exit(1): defers NÃO rodam
```

### 3.2 `log/slog`: logs estruturados (Go 1.21+)

Em produção, logs são lidos por máquinas (Loki, Elasticsearch, Datadog). O `slog` produz registros **estruturados** com pares chave-valor. Saídas reais (sem o campo de hora, para caber):

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
logger.Info("requisição", "método", "GET", "status", 200, slog.Duration("dur", 12*time.Millisecond))
// level=INFO msg=requisição método=GET status=200 dur=12ms

jl := slog.New(slog.NewJSONHandler(os.Stdout, nil))
jl.With("serviço", "api").Warn("lento", "ms", 830)
// {"level":"WARN","msg":"lento","serviço":"api","ms":830}
```

```go
slog.SetDefault(logger)                   // passa a valer para slog.Info(...) e para o pacote log
slog.Debug("detalhe")                     // abaixo do nível padrão (Info): não aparece
reqLog := logger.With("req_id", id)       // logger "filho" com campos fixos
slog.ErrorContext(ctx, "falhou", "erro", err)   // integra com context (rastreamento)
```

| Nível | Uso |
| --- | --- |
| `Debug` | Detalhes para desenvolvimento |
| `Info` | Eventos normais (requisição atendida, serviço iniciado) |
| `Warn` | Algo estranho, mas o sistema segue (lentidão, retry) |
| `Error` | Falhas que precisam de atenção |

> 💡 Go 1.26 adicionou `slog.NewMultiHandler`, para enviar o mesmo log a vários destinos (por exemplo, texto no terminal e JSON num arquivo).

---

## 🖥️ 4. Linha de comando e processos

### 4.1 `flag`: argumentos de linha de comando

Saída real:

```go
porta := flag.Int("porta", 8080, "porta do servidor")
verbose := flag.Bool("v", false, "modo verboso")
flag.Parse()                              // lê os.Args[1:]

// ./app -porta 9090 -v
fmt.Println(*porta, *verbose)             // 9090 true
flag.Args()                               // argumentos que sobraram (não-flags)
```

`flag` gera automaticamente a ajuda (`-h`). Para CLIs complexas com subcomandos (como `git commit`), a comunidade usa `spf13/cobra`.

### 4.2 `os`: o processo e o ambiente

```go
os.Args                        // argumentos
os.Getenv("HOME")              // variável de ambiente ("" se não existir)
v, ok := os.LookupEnv("PORTA") // distingue "vazia" de "não existe"
os.Exit(1)                     // termina (sem rodar defers)
os.Getpid(), os.Getwd(), os.Hostname()
os.Stdin, os.Stdout, os.Stderr // descritores 0, 1 e 2
```

### 4.3 `os/exec`: rodar outros programas

Saída real:

```go
out, err := exec.Command("echo", "via exec").Output()
fmt.Printf("%q %v\n", out, err)      // "via exec\n" <nil>

cmd := exec.CommandContext(ctx, "git", "log", "--oneline", "-5")
cmd.Dir = "/caminho/do/repo"
cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
err := cmd.Run()
var exitErr *exec.ExitError
if errors.As(err, &exitErr) {
	fmt.Println("código de saída:", exitErr.ExitCode())
}
```

> ⚠️ `exec.Command` **não** usa um shell: os argumentos vão direto para o programa (sem `*`, `|`, `$VAR`). Isso é **mais seguro** (sem injeção de comandos). Se precisar de shell, é explícito: `exec.Command("sh", "-c", "...")`, e aí cuidado com entrada do usuário.

> 💡 Por baixo, isso é o par `fork`/`exec` + `waitpid` que você estudou em [[Syscalls para Gerenciamento de Processos]] (no Linux, o Go usa `clone` com `CLONE_VFORK` ou `posix_spawn`-like para ser eficiente).

### 4.4 `os/signal`: reagir a sinais

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()            // espera Ctrl+C ou SIGTERM (o docker stop envia SIGTERM)
fmt.Println("encerrando com elegância...")
```

Veremos o desligamento gracioso de servidores em [[HTTP (net-http)]] e [[context.Context]]. Sinais foram estudados em [[Syscalls Diversas]] e no contexto de containers em [[Processo raiz (PID 1)]].

---

## 🔐 5. Criptografia

Saídas reais:

```go
h := sha256.Sum256([]byte("olá"))                       // hash SHA-256 ([32]byte)
fmt.Println(hex.EncodeToString(h[:])[:16])              // 9b186e077c7c6d04...

mac := hmac.New(sha256.New, chave)                      // assinatura com chave (webhooks, tokens)
mac.Write(mensagem)
assinatura := mac.Sum(nil)
hmac.Equal(assinatura, recebida)                        // ✅ comparação em tempo constante

token := make([]byte, 32)
crypto_rand.Read(token)                                 // bytes aleatórios seguros (crypto/rand)
```

| Pacote | Para quê |
| --- | --- |
| `crypto/rand` | Números aleatórios seguros (tokens, chaves). `rand.Text()` (1.24) gera uma string segura |
| `crypto/sha256`, `sha512`, `sha3` | Hashes |
| `crypto/hmac` | Autenticação de mensagens com chave |
| `crypto/aes`, `crypto/cipher` | Criptografia simétrica (use modos autenticados como GCM) |
| `crypto/ed25519`, `ecdsa`, `rsa` | Assinaturas digitais |
| `crypto/tls`, `crypto/x509` | TLS e certificados |
| `crypto/mlkem` (1.24), `crypto/mldsa` (1.27) | Criptografia **pós-quântica** (troca de chaves e assinaturas resistentes a computadores quânticos) |
| `golang.org/x/crypto/bcrypt`, `argon2` | Hash de **senhas** (lento de propósito) |

> ⚠️ Nunca guarde senhas com SHA-256 puro (é rápido demais para ataques de força bruta). Use **bcrypt** ou **argon2**.

> 💡 Desde o Go 1.24, o TLS do Go usa por padrão uma troca de chaves **híbrida pós-quântica** (X25519 + ML-KEM); o Go 1.26 habilitou mais variantes híbridas por padrão, e o Go 1.27 adicionou assinaturas ML-DSA a TLS e certificados. Você estudou o handshake TLS em [[TLS e SSL]].

---

## 🧾 6. Codificação e formatos

Saídas reais:

```go
base64.StdEncoding.EncodeToString([]byte("go!"))   // Z28h
hex.EncodeToString([]byte{0xca, 0xfe})             // cafe

r := csv.NewReader(strings.NewReader("nome,idade\nAna,30\nBia,25\n"))
regs, _ := r.ReadAll()                             // [[nome idade] [Ana 30] [Bia 25]]

// encoding/binary: inteiros ↔ bytes numa ordem definida (protocolos de rede!)
buf := make([]byte, 4)
binary.BigEndian.PutUint32(buf, 0xC0A8014D)        // 192.168.1.77 em "ordem de rede"
```

> 💡 **Ordem de bytes (endianness):** protocolos de rede usam *big-endian* ("ordem de rede"); processadores x86 e ARM usam *little-endian*. `encoding/binary` faz a conversão explícita. Você viu os cabeçalhos binários dos protocolos em [[TCP-IP]] e [[PDU (Protocol Data Unit)]].

O `encoding/json` tem uma nota própria: [[JSON (marshal e unmarshal)]] (incluindo o `encoding/json/v2`, que desde o Go 1.27 é o motor por trás do pacote `encoding/json`).

---

## 🔤 7. Expressões regulares e templates

### 7.1 `regexp`

Saída real:

```go
re := regexp.MustCompile(`(\d{3})\.(\d{3})\.(\d{3})-(\d{2})`)
re.MatchString("cpf: 123.456.789-09")           // true
re.FindStringSubmatch("123.456.789-09")[1:]     // [123 456 789 09]
re.ReplaceAllString(s, "***")
```

> 💡 O `regexp` de Go usa o algoritmo **RE2**: tempo de execução **linear** garantido, sem o "backtracking catastrófico" de outras linguagens (que já derrubou serviços com uma regex maliciosa, ataque ReDoS). O preço: não suporta *backreferences* (`\1`) nem *lookahead*.

### 7.2 `text/template` e `html/template`

Saída real:

```go
tmpl := template.Must(template.New("x").Parse("Olá, {{.Nome}}! Você tem {{len .Itens}} itens.\n"))
tmpl.Execute(os.Stdout, map[string]any{"Nome": "Matheus", "Itens": []int{1, 2, 3}})
// Olá, Matheus! Você tem 3 itens.
```

> ⚠️ Para gerar **HTML**, use `html/template`: mesma API, mas com **escape automático** conforme o contexto (HTML, atributo, JavaScript, URL), o que previne XSS.

---

## 📂 8. Caminhos e arquivos (resumo)

Saída real:

```go
filepath.Join("home", "matheus", "..", "go", "main.go")   // home/go/main.go (limpa o "..")
filepath.Ext("foto.png")                                   // .png
filepath.Base("/a/b/c.txt")                                // c.txt
filepath.Dir("/a/b/c.txt")                                 // /a/b
filepath.WalkDir(raiz, func(p string, d fs.DirEntry, err error) error { ... })
```

> 💡 Use `path/filepath` para caminhos do **sistema de arquivos** (usa `\` no Windows) e `path` para caminhos com `/` sempre (URLs, caminhos dentro de `embed.FS`).

Leitura e escrita de arquivos, `bufio` e o sistema de arquivos estão em [[Leitura e Escrita de Arquivos]].

---

## 📚 9. Estruturas de dados extras

Saída real com um heap (fila de prioridade):

```go
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any          { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

mh := &MinHeap{5, 2, 8}
heap.Init(mh)
heap.Push(mh, 1)
fmt.Println(heap.Pop(mh), heap.Pop(mh))    // 1 2   (sempre sai o menor)
```

| Pacote | Estrutura |
| --- | --- |
| `container/heap` | Heap binário (fila de prioridade) sobre qualquer `heap.Interface` |
| `container/list` | Lista duplamente ligada |
| `container/ring` | Lista circular |
| `unique` (1.23) | *Interning*: uma cópia por valor, comparação O(1) |
| `weak` (1.24) | Ponteiros fracos (não impedem o GC), para caches |
| `hash/maphash` | Hash rápido e seguro para suas próprias tabelas |

> 💡 Um heap de prioridades é exatamente a estrutura usada em escalonadores (escolher o processo de maior prioridade) e em timers (o próximo a disparar). Você viu escalonamento por prioridade em [[Escalonamento em Sistemas Interativos]] e a estrutura em [[Árvores Binárias]].

---

## 🔬 10. Por dentro: a stdlib fala direto com o kernel

Em Linux, a biblioteca padrão de Go **não usa a libc** (quando cgo está desligado): o pacote `syscall` (e o interno `internal/syscall/unix`) emite as instruções `SYSCALL` diretamente. Camadas:

```
seu código           os.ReadFile("a.txt")
                            │
pacote os            os.Open → (*File).Read      ← API portável, erros com contexto
                            │
internal/poll        poll.FD.Read                ← integra com o netpoller (não bloqueia threads)
                            │
pacote syscall       syscall.Read(fd, buf)       ← uma função por syscall
                            │
assembly do runtime  SYSCALL                     ← instrução da CPU: entra no kernel
                            │
kernel Linux         sys_read → VFS → ext4/pipe/socket
```

É por isso que um binário Go roda em qualquer Linux sem depender da versão da glibc instalada, e por isso o mesmo código serve para arquivos, pipes e sockets (a interface `read`/`write` do kernel vira as interfaces `io.Reader`/`io.Writer`). Você estudou esse caminho em [[System Calls]].

> 💡 Exceções em que a stdlib usa a libc via cgo: resolução de nomes DNS em alguns sistemas (`net`), busca de usuários (`os/user`). Com `CGO_ENABLED=0`, o Go usa implementações próprias em Go puro. No macOS, o Go sempre chama a libSystem (a Apple não garante estabilidade das syscalls diretas).

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `time.Sleep(5)` | Dorme 5 ns | `5 * time.Second` |
| Layout de data errado | Formatação estranha sem erro | Lembre do `01/02 03:04:05PM '06 -0700` |
| `LoadLocation` em imagem scratch | Erro "unknown time zone" | `import _ "time/tzdata"` |
| `log.Fatal` com defers pendentes | Limpeza não acontece | Devolva erro até o `main` |
| `exec.Command("ls *.go")` | Procura um programa chamado "ls *.go" | `exec.Command("ls")` + `filepath.Glob`, ou `sh -c` com cuidado |
| `text/template` para HTML | Vulnerável a XSS | `html/template` |
| SHA-256 para senhas | Força bruta fácil | bcrypt / argon2 |
| `math/rand` para tokens | Previsível | `crypto/rand` |
| Comparar MACs com `==` / `bytes.Equal` | Vazamento por tempo (timing attack) | `hmac.Equal` / `subtle.ConstantTimeCompare` |

---

## 📋 Resumo

| Área | Destaques |
| --- | --- |
| Tempo | `time.Time`, `Duration`, layout `2006-01-02 15:04:05`, fusos IANA, relógio monotônico |
| Logs | `slog` estruturado (texto/JSON, níveis, `With`) |
| CLI e processos | `flag`, `os`, `os/exec` (sem shell), `os/signal` |
| Cripto | `crypto/rand`, SHA-2/3, HMAC, TLS, pós-quântica (ML-KEM, ML-DSA) |
| Formatos | base64, hex, CSV, `encoding/binary` (endianness), JSON |
| Texto | `regexp` (RE2, linear), `text/template`, `html/template` (anti-XSS) |
| Estruturas | `container/heap`, `unique`, `weak` |
| Por dentro | Syscalls diretas no Linux, sem libc |

---

## 🔗 Conexões

**Dentro de Go**
- Texto e `fmt` → [[Strings em Profundidade]]
- I/O, arquivos e `bufio` → [[io.Reader e io.Writer]], [[Leitura e Escrita de Arquivos]]
- JSON → [[JSON (marshal e unmarshal)]]
- Rede e HTTP → [[Sockets e o Pacote net]], [[HTTP (net-http)]]
- Timers e tickers com `select` → [[Select]]
- Sinais e cancelamento → [[context.Context]]
- Testes → [[Testes com go test]]

**Sistemas Operacionais**
- [[System Calls]] — a stdlib emite syscalls diretamente, sem libc
- [[Syscalls para Gerenciamento de Processos]] — `os/exec` = fork/exec/waitpid
- [[Syscalls Diversas]] — sinais, relógios do sistema e o problema de 2038
- [[Escalonamento em Sistemas Interativos]] — filas de prioridade (heaps) em escalonadores

**Redes, Nginx e Docker**
- [[TLS e SSL]] e [[SSL-TLS no Nginx]] — `crypto/tls` implementa o mesmo protocolo
- [[TCP-IP]] — ordem de bytes de rede e `encoding/binary`
- [[Processo raiz (PID 1)]] — `SIGTERM` no `docker stop` e `signal.NotifyContext`
- [[Imagens]] — `time/tzdata` e certificados CA em imagens mínimas

---

⬅️ **Anterior:** [[Sistema de Módulos (go mod)]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[io.Reader e io.Writer]] ➡️
