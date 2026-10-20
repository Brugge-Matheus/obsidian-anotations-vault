---
tags:
  - go
  - go/io
aliases:
  - io.Reader
  - io.Writer
  - Pacote io
  - Streams em Go
  - io.Copy
ordem: 33
modulo: "08 - IO e Serialização"
---
# io.Reader e io.Writer

> **Nota 33 da trilha de Go.** Duas interfaces de **um método cada** sustentam praticamente todo o I/O de Go: **`io.Reader`** (algo de onde se lê bytes) e **`io.Writer`** (algo para onde se escreve bytes). Arquivos, conexões de rede, buffers em memória, compressores, hashes, a resposta HTTP, o terminal... todos falam essas duas interfaces. Isso permite **encaixar** peças como num encanamento: ler de um arquivo, descomprimir, calcular o hash e enviar pela rede, sem carregar tudo na memória. Nesta nota veremos o **contrato** exato de `Read` e `Write`, as interfaces relacionadas, como escrever as suas, as ferramentas de composição do pacote `io` e como isso tudo corresponde às syscalls `read`/`write` do kernel.
>
> **Pré-requisitos:** [[Interfaces]], [[Arrays e Slices]], [[O Padrão error em Go]].

---

## 🧭 De onde viemos

Em [[Interfaces]] vimos que interfaces pequenas são as mais poderosas e usamos `io.Writer` num exemplo de contador. Em [[Type Assertion e Type Switch]] vimos que `io.Copy` procura interfaces opcionais para ir mais rápido. Em [[Pacotes Importantes da Stdlib]] vimos que a stdlib fala direto com o kernel. Agora vamos estudar a abstração central do I/O de Go.

> 💡 **Stream (fluxo):** uma sequência de bytes lida ou escrita **aos poucos**, do início ao fim, sem precisar estar inteira na memória. Um arquivo de 10 GB, uma conexão de rede aberta por horas, a saída de um comando: todos são streams. `io.Reader` e `io.Writer` são a representação de streams em Go.

---

## 📖 1. `io.Reader`: o contrato do `Read`

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

**Quem chama** fornece um buffer `p`. **O `Read`** coloca até `len(p)` bytes nele e devolve quantos colocou (`n`) e um possível erro. Saída real lendo `"Olá, Go!"` com um buffer de 4 bytes:

```go
r := strings.NewReader("Olá, Go!")
buf := make([]byte, 4)
for {
	n, err := r.Read(buf)
	fmt.Printf("n=%d %q err=%v\n", n, buf[:n], err)
	if errors.Is(err, io.EOF) {
		break
	}
}
// n=4 "Olá" err=<nil>       ← "Olá" ocupa 4 bytes (á = 2 bytes)
// n=4 ", Go" err=<nil>
// n=1 "!" err=<nil>          ← menos que len(buf): normal!
// n=0 "" err=EOF             ← acabou
```

As regras do contrato (da documentação de `io.Reader`):

| Regra | Consequência para quem chama |
| --- | --- |
| `Read` pode devolver **menos** que `len(p)` mesmo sem ter acabado | Nunca assuma que o buffer foi preenchido; use sempre `buf[:n]` |
| `Read` pode devolver `n > 0` **e** `err != nil` na mesma chamada | **Processe os `n` bytes ANTES de olhar o erro** |
| Fim dos dados = `io.EOF` | `io.EOF` não é falha, é o sinal normal de "acabou" |
| `Read` não deve guardar `p` depois de retornar | O buffer pertence a quem chamou |
| Devolver `0, nil` é desencorajado | Significa "nada aconteceu"; tente de novo |

O laço correto, portanto:

```go
for {
	n, err := r.Read(buf)
	if n > 0 {
		processar(buf[:n])          // 1º: use os dados recebidos
	}
	if err == io.EOF {
		break                       // 2º: fim normal
	}
	if err != nil {
		return err                  // 3º: falha de verdade
	}
}
```

> 💡 Na prática, você raramente escreve esse laço: as funções `io.Copy`, `io.ReadAll`, `io.ReadFull`, `bufio.Scanner` e os decodificadores (`json.NewDecoder`) já fazem isso corretamente.

---

## ✍️ 2. `io.Writer`: o contrato do `Write`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

`Write` escreve os bytes de `p` e devolve quantos escreveu. O contrato é mais rígido que o do `Read`:

| Regra | Detalhe |
| --- | --- |
| Se `n < len(p)`, **tem** que devolver um erro | Não existe "escrita parcial silenciosa" |
| Não pode modificar `p` | Nem temporariamente |
| Não deve guardar `p` | O buffer é de quem chamou |

> 💡 Repare na diferença com a syscall `write` de UNIX, que pode escrever menos bytes do que o pedido sem erro (por exemplo, num socket com buffer cheio). A implementação de `os.File.Write` em Go **repete** a syscall até escrever tudo ou falhar, para cumprir o contrato de `io.Writer`.

---

## 🧩 3. A família de interfaces do pacote `io`

```go
type Closer      interface { Close() error }
type Seeker      interface { Seek(offset int64, whence int) (int64, error) }
type ReaderAt    interface { ReadAt(p []byte, off int64) (n int, err error) }
type WriterAt    interface { WriteAt(p []byte, off int64) (n int, err error) }
type ByteReader  interface { ReadByte() (byte, error) }
type RuneReader  interface { ReadRune() (r rune, size int, err error) }
type StringWriter interface { WriteString(s string) (n int, err error) }

// interfaces opcionais de desempenho (ver Type Assertion e Type Switch)
type WriterTo   interface { WriteTo(w Writer) (n int64, err error) }
type ReaderFrom interface { ReadFrom(r Reader) (n int64, err error) }

// combinações
type ReadCloser      interface { Reader; Closer }
type WriteCloser     interface { Writer; Closer }
type ReadWriter      interface { Reader; Writer }
type ReadWriteCloser interface { Reader; Writer; Closer }
type ReadSeeker      interface { Reader; Seeker }
```

Quem implementa o quê (alguns exemplos):

| Tipo | Reader | Writer | Closer | Seeker | Observação |
| --- | --- | --- | --- | --- | --- |
| `*os.File` | ✅ | ✅ | ✅ | ✅ | Arquivos, `os.Stdin/Stdout/Stderr` |
| `net.Conn` | ✅ | ✅ | ✅ | ❌ | Conexões TCP/UDP/Unix |
| `*bytes.Buffer` | ✅ | ✅ | ❌ | ❌ | Buffer em memória que cresce |
| `*strings.Reader` | ✅ | ❌ | ❌ | ✅ | Ler de uma string |
| `*bytes.Reader` | ✅ | ❌ | ❌ | ✅ | Ler de um `[]byte` |
| `*bufio.Reader` / `Writer` | ✅ / ❌ | ❌ / ✅ | ❌ | ❌ | Bufferização |
| `*gzip.Reader` / `Writer` | ✅ / ❌ | ❌ / ✅ | ✅ | ❌ | Compressão |
| `hash.Hash` (sha256...) | ❌ | ✅ | ❌ | ❌ | Escrever = alimentar o hash |
| `http.ResponseWriter` | ❌ | ✅ | ❌ | ❌ | Corpo da resposta HTTP |
| `*http.Request.Body` | ✅ | ❌ | ✅ | ❌ | Corpo da requisição |
| `*strings.Builder` | ❌ | ✅ | ❌ | ❌ | Montar strings |

---

## 🛠️ 4. Escrevendo os seus

### 4.1 Um `Reader` que transforma outro (ROT13)

Um "filtro": embrulha um `Reader` e transforma os bytes que passam (saída real):

```go
type rot13 struct{ r io.Reader }

func (r rot13) Read(p []byte) (int, error) {
	n, err := r.r.Read(p)              // lê do Reader de baixo
	for i := 0; i < n; i++ {           // transforma só os n bytes lidos
		c := p[i]
		switch {
		case c >= 'a' && c <= 'z':
			p[i] = 'a' + (c-'a'+13)%26
		case c >= 'A' && c <= 'Z':
			p[i] = 'A' + (c-'A'+13)%26
		}
	}
	return n, err                      // repassa n e err (inclusive o EOF)
}

io.Copy(os.Stdout, rot13{strings.NewReader("Uryyb, Tbcure!\n")})
// Hello, Gopher!
```

### 4.2 Um `Writer` que observa os dados

Saída real, combinado com `io.MultiWriter` (seção 5):

```go
type contador struct{ letras, total int }

func (c *contador) Write(p []byte) (int, error) {
	for _, b := range p {
		if unicode.IsLetter(rune(b)) {
			c.letras++
		}
	}
	c.total += len(p)
	return len(p), nil                 // ✅ escreveu tudo
}
```

---

## 🔗 5. Compondo: o encanamento do pacote `io`

A grande vantagem das interfaces pequenas: peças genéricas que se encaixam.

### 5.1 Copiar e ler tudo

```go
n, err := io.Copy(dst, src)           // lê de src e escreve em dst até EOF (buffer de 32 KB)
io.CopyN(dst, src, 1024)              // copia exatamente N bytes
dados, err := io.ReadAll(r)           // lê tudo para um []byte (cuidado com o tamanho!)
_, err = io.ReadFull(r, buf)          // preenche buf inteiro ou dá erro
n, _ := io.Copy(io.Discard, r)        // consome e descarta (saída real: 10)
```

Saída real de `io.ReadFull` quando os dados acabam antes:

```go
full := make([]byte, 5)
_, err := io.ReadFull(strings.NewReader("abc"), full)
fmt.Println(err)                      // unexpected EOF
```

### 5.2 Peças de composição

Saídas reais:

```go
// Escrever em vários destinos ao mesmo tempo
var c contador
h := sha256.New()
var copia bytes.Buffer
w := io.MultiWriter(&c, h, &copia)
n, _ := io.Copy(w, strings.NewReader("abc 123 def"))
// n=11, c.letras=6, hash calculado, copia="abc 123 def"   ← tudo numa única passada

// Ler de várias fontes em sequência
io.ReadAll(io.MultiReader(strings.NewReader("abc"), strings.NewReader("def")))   // "abcdef"

// Limitar quanto pode ser lido (proteção contra entradas gigantes)
io.ReadAll(io.LimitReader(strings.NewReader("0123456789"), 4))                   // "0123"

// Ler um pedaço específico (offset 3, 4 bytes)
io.ReadAll(io.NewSectionReader(strings.NewReader("0123456789"), 3, 4))           // "3456"

// "Espionar" o que está sendo lido
var log bytes.Buffer
tee := io.TeeReader(strings.NewReader("dados"), &log)
io.ReadAll(tee)                        // log também recebeu "dados"
```

| Função | O que faz |
| --- | --- |
| `io.MultiWriter(w1, w2, ...)` | Um Writer que escreve em todos (como `tee` do shell) |
| `io.MultiReader(r1, r2, ...)` | Um Reader que lê um depois do outro (como `cat a b`) |
| `io.TeeReader(r, w)` | Tudo que for lido de `r` também é escrito em `w` |
| `io.LimitReader(r, n)` | Para depois de `n` bytes |
| `io.NewSectionReader(r, off, n)` | Um trecho de um `ReaderAt` |
| `io.Pipe()` | Liga um Writer a um Reader em memória, entre goroutines |
| `io.NopCloser(r)` | Adiciona um `Close()` que não faz nada |
| `io.Discard` | Um Writer que joga tudo fora (como `/dev/null`) |

### 5.3 Um encanamento real: comprimir em streaming

Saída real:

```go
var comprimido bytes.Buffer
gz := gzip.NewWriter(&comprimido)                                   // Writer que comprime
io.Copy(gz, strings.NewReader(strings.Repeat("go é legal! ", 100)))
gz.Close()                                                          // ⚠️ Close finaliza o formato!
fmt.Println("original 1300 → gzip", comprimido.Len())               // original 1300 → gzip 47

gr, _ := gzip.NewReader(&comprimido)                                // Reader que descomprime
tudo, _ := io.ReadAll(gr)
fmt.Println(len(tudo))                                              // 1300
```

Um exemplo mais realista, que nunca carrega o arquivo inteiro na memória:

```go
// Baixar um arquivo .gz, descomprimir, calcular o SHA-256 e salvar em disco
resp, _ := http.Get(url)
defer resp.Body.Close()

gr, _ := gzip.NewReader(resp.Body)        // rede → descompressão
f, _ := os.Create("dados.csv")
defer f.Close()
h := sha256.New()

io.Copy(io.MultiWriter(f, h), gr)         // descompressão → (arquivo + hash)
fmt.Printf("%x\n", h.Sum(nil))
```

```
 rede ──▶ resp.Body ──▶ gzip.Reader ──▶ io.Copy ──▶ MultiWriter ──┬──▶ arquivo
 (Reader)              (Reader)        (32 KB      (Writer)       └──▶ sha256
                                        por vez)
```

> 💡 Isso é a **filosofia UNIX** de pipes (`curl url | gunzip | tee dados.csv | sha256sum`) traduzida para tipos. Você estudou pipes em [[Arquivos]] e no [[Interpretador de Comandos Shell]].

### 5.4 `io.Pipe`: ligar um Writer a um Reader

Saída real:

```go
pr, pw := io.Pipe()
go func() {
	fmt.Fprint(pw, "do outro lado do pipe")   // escreve numa goroutine...
	pw.Close()
}()
b, _ := io.ReadAll(pr)                          // ...e lê na outra
fmt.Println(string(b))                          // do outro lado do pipe
```

Útil quando uma API quer um **Writer** (ex.: um codificador JSON) e outra quer um **Reader** (ex.: o corpo de uma requisição HTTP), sem montar tudo na memória. É síncrono: cada `Write` espera um `Read` correspondente. Veremos goroutines em [[Goroutines]].

---

## 🧺 6. `bufio`: menos syscalls

Cada `Read`/`Write` num `*os.File` ou `net.Conn` é uma **syscall** (uma troca de modo usuário → kernel, que custa centenas de nanossegundos). Ler 1 byte por vez de um arquivo de 1 MB seria 1 milhão de syscalls. O `bufio` coloca um **buffer** no meio:

```
sem bufio:   seu código ──Read(1 byte)──▶ kernel    × 1.000.000 syscalls
com bufio:   seu código ──ReadByte()──▶ [buffer 4 KB] ──Read(4 KB)──▶ kernel   × ~250 syscalls
```

```go
br := bufio.NewReader(f)              // buffer de 4 KB por padrão
linha, err := br.ReadString('\n')
b, err := br.ReadByte()
r, _, err := br.ReadRune()

bw := bufio.NewWriter(f)
fmt.Fprintf(bw, "linha %d\n", i)      // vai para o buffer
bw.Flush()                            // ⚠️ obrigatório: sem Flush, o final pode nunca ser escrito
```

Para ler linhas ou palavras, o `bufio.Scanner` é o mais prático (saída real):

```go
sc := bufio.NewScanner(strings.NewReader("um dois três"))
sc.Split(bufio.ScanWords)             // ScanLines (padrão), ScanWords, ScanRunes, ScanBytes
for sc.Scan() {
	fmt.Println(sc.Text())
}
if err := sc.Err(); err != nil { ... } // o erro aparece DEPOIS do laço
```

> ⚠️ O `Scanner` tem um limite de **64 KB por linha** por padrão; linhas maiores dão erro `bufio.Scanner: token too long`. Para linhas enormes, use `sc.Buffer(buf, max)` ou `bufio.Reader.ReadString`.

Veremos `bufio` com arquivos em [[Leitura e Escrita de Arquivos]].

---

## 🧠 7. Boas práticas de design

### 7.1 Aceite `io.Reader`/`io.Writer`, não tipos concretos

```go
// ❌ só funciona com arquivos
func ContarLinhas(caminho string) (int, error)

// ✅ funciona com arquivo, rede, string, buffer, gzip, HTTP...
func ContarLinhas(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	n := 0
	for sc.Scan() {
		n++
	}
	return n, sc.Err()
}

// e é trivial de testar:
ContarLinhas(strings.NewReader("a\nb\nc"))   // 3, sem criar arquivos
```

### 7.2 Prefira streaming a `io.ReadAll`

`io.ReadAll` carrega tudo na memória. Com dados de tamanho desconhecido (upload de usuário, resposta de API externa), isso pode esgotar a memória. Processe em fluxo (`json.NewDecoder(r)`, `bufio.Scanner`, `io.Copy`) e **limite** o tamanho: `io.LimitReader(r, 10<<20)` ou, em servidores HTTP, `http.MaxBytesReader`.

### 7.3 Quem abre, fecha

Se uma função recebe um `io.Reader`, ela **não** fecha (não sabe se pode). Quem abriu o arquivo/conexão é quem chama `Close`, normalmente com `defer`.

---

## 🔬 8. Por dentro: das interfaces às syscalls

### 8.1 `Read`/`Write` são as syscalls `read`/`write`

Para um `*os.File` no Linux:

```
f.Read(buf)
  └─ os.(*File).Read
      └─ internal/poll.(*FD).Read            ← trava o FD para leitura, integra com o netpoller
          └─ syscall.Read(fd, buf)
              └─ SYSCALL  read(fd, ptr, len)  ← o kernel copia bytes do page cache
                                                 (ou do buffer do socket) para buf
```

A interface `io.Reader` tem **exatamente a forma** da syscall `read(fd, buffer, tamanho) → bytes lidos`: quem chama fornece o buffer, e o resultado é a quantidade lida. Não é coincidência: Go herdou o modelo de I/O do UNIX ("tudo é arquivo"), que você estudou em [[Syscalls para Gerenciamento de Arquivos]] e [[Arquivos]].

> 💡 Por que o **chamador** fornece o buffer? Para que o mesmo buffer seja **reutilizado** a cada chamada, sem alocar memória nova. É a mesma razão pela qual a syscall `read` recebe um ponteiro em vez de devolver um.

### 8.2 Rede: Readers que não bloqueiam threads

Num `net.Conn`, um `Read` sem dados disponíveis **não** bloqueia a thread do sistema operacional: a goroutine é estacionada e o **netpoller** do runtime (epoll no Linux) a acorda quando chegam dados. Por isso um servidor Go atende milhares de conexões com poucas threads. Veremos em [[Sockets e o Pacote net]] e [[Scheduler do Go]].

### 8.3 Cópia sem passar pelo espaço de usuário

```
io.Copy(conexãoTCP, arquivo)
   │
   ├─ o destino implementa io.ReaderFrom?  *net.TCPConn: sim
   │      └─ ReadFrom detecta que a origem é um *os.File
   │            └─ syscall sendfile(socket, arquivo, ...)   ← o KERNEL copia direto:
   │                                                          disco/page cache → socket
   │                                                          sem passar pela memória do processo
   └─ (Linux) entre sockets ou pipes, splice() faz o mesmo
```

É a mesma otimização do `sendfile on;` do Nginx, que você viu em [[Conexões, Workers e Keep-Alive]]. E ela acontece **automaticamente**, porque `io.Copy` verifica as interfaces opcionais ([[Type Assertion e Type Switch]]).

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Usar `buf` inteiro em vez de `buf[:n]` | Lixo de leituras anteriores | Sempre `buf[:n]` |
| Checar o erro antes de processar `n` | Perde os últimos bytes | Processe `n > 0` primeiro |
| Tratar `io.EOF` como falha | Erro reportado no fim normal | `errors.Is(err, io.EOF)` → fim |
| Esquecer `Flush()` do `bufio.Writer` | Fim do arquivo faltando | `defer bw.Flush()` (e verifique o erro) |
| Esquecer `Close()` do `gzip.Writer` | Arquivo .gz corrompido | `Close()` finaliza o formato |
| `io.ReadAll` em entrada sem limite | Memória esgotada | Streaming + `io.LimitReader` |
| Linhas > 64 KB com `Scanner` | `token too long` | `sc.Buffer(...)` ou `ReadString` |
| Writer customizado que devolve `n < len(p)` sem erro | Viola o contrato; dados perdidos | Devolva erro (`io.ErrShortWrite`) |
| Função recebendo `io.Reader` e fechando-o | Fecha algo que não abriu | Quem abre, fecha |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `io.Reader` | `Read(p) (n, err)`; pode ler menos; `n > 0` com erro é possível; `io.EOF` = fim |
| `io.Writer` | `Write(p) (n, err)`; `n < len(p)` exige erro |
| Família | `Closer`, `Seeker`, `ReaderAt`, `StringWriter`, `WriterTo`, `ReaderFrom`... |
| Composição | `Copy`, `ReadAll`, `ReadFull`, `MultiWriter`, `MultiReader`, `TeeReader`, `LimitReader`, `Pipe`, `Discard` |
| `bufio` | Buffer para reduzir syscalls; `Scanner` para linhas/palavras; `Flush`! |
| Design | Aceite `io.Reader`/`io.Writer`; prefira streaming; quem abre, fecha |
| Por dentro | Mesma forma da syscall `read`/`write`; netpoller na rede; `sendfile`/`splice` automáticos |

---

## 🔗 Conexões

**Dentro de Go**
- Interfaces pequenas e satisfação implícita → [[Interfaces]]
- Interfaces opcionais (`WriterTo`, `ReaderFrom`) → [[Type Assertion e Type Switch]]
- Embrulhar um Reader (decorador) → [[Composição vs Herança]]
- Arquivos, `bufio` e o sistema de arquivos → [[Leitura e Escrita de Arquivos]]
- Codificar/decodificar em streaming → [[JSON (marshal e unmarshal)]]
- `net.Conn` e o netpoller → [[Sockets e o Pacote net]]
- Corpos de requisição e resposta → [[HTTP (net-http)]]
- `io.Pipe` entre goroutines → [[Goroutines]]
- `io.EOF` como erro sentinela → [[O Padrão error em Go]]

**Sistemas Operacionais**
- [[Syscalls para Gerenciamento de Arquivos]] — `read`, `write`, `lseek`, `close`: a forma exata de `Reader`, `Writer`, `Seeker`, `Closer`
- [[Arquivos]] — "tudo é arquivo", descritores e pipes
- [[Dispositivos de IO]] — buffers e por que reduzir o número de operações de I/O importa
- [[Interpretador de Comandos Shell]] — pipes `|`, `tee` e `cat` como as versões de shell de `io.Copy`, `MultiWriter` e `MultiReader`

**Nginx e Docker**
- [[Conexões, Workers e Keep-Alive]] — `sendfile`, a mesma otimização de `io.Copy` entre arquivo e socket
- [[Compressão]] — gzip em streaming, como o `gzip.Writer` encaixado num `http.ResponseWriter`
- [[Volumes e Persistência]] — streams de leitura/escrita em arquivos montados

---

⬅️ **Anterior:** [[Pacotes Importantes da Stdlib]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Leitura e Escrita de Arquivos]] ➡️
