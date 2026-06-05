---
tags:
  - go
  - go/io
---
# Leitura e Escrita de Arquivos

> Go expõe operações de arquivo em diferentes níveis de abstração — desde `os.ReadFile` que faz tudo de uma vez, até `bufio.Scanner` para streaming linha a linha. Entender qual usar em cada situação evita problemas de memória e performance.
> 

---

## 1. Modelo de I/O do Go — Interfaces

O design de I/O de Go é baseado em interfaces pequenas que compõem:

```go
io.Reader: Read(p []byte) (n int, err error)
io.Writer: Write(p []byte) (n int, err error)
io.Closer: Close() error
io.Seeker: Seek(offset int64, whence int) (int64, error)

*os.File implementa: Reader, Writer, Closer, Seeker, ReadFrom, WriteTo
strings.Reader implementa: Reader, Seeker
bytes.Buffer implementa: Reader, Writer
bufio.Reader implementa: Reader (com buffer)
bufio.Writer implementa: Writer (com buffer)

Qualquer código que aceita io.Reader funciona com:
- arquivos, HTTP bodies, strings, bytes, pipes, TCP connections...
```

---

## 2. Leitura Rápida — `os.ReadFile`

Para arquivos que cabem confortavelmente na memória (< ~100MB):

```go
import "os"

// Lê o arquivo inteiro de uma vez
dados, err := os.ReadFile("config.json")
if err != nil {
	if os.IsNotExist(err) {
		return nil, fmt.Errorf("arquivo não encontrado: config.json")
	}
	return nil, fmt.Errorf("lerConfig: %w", err)
}

// Converter para string (cópia — a []byte original é liberada pelo GC)
conteudo := string(dados)

// Para JSON — use diretamente os bytes sem converter para string
var cfg Config
if err := json.Unmarshal(dados, &cfg); err != nil {
	return nil, fmt.Errorf("config inválida: %w", err)
}
```

---

## 3. Escrita Rápida — `os.WriteFile`

```go
// Escreve atomicamente (cria ou trunca)
conteudo := []byte("linha 1\nlinha 2\nlinha 3\n")

// Permissões Unix: 0644 = dono: rw-, grupo: r--, outros: r--
err := os.WriteFile("saida.txt", conteudo, 0644)
if err != nil {
	return fmt.Errorf("escrever: %w", err)
}

// Para JSON
dados, _ := json.MarshalIndent(objeto, "", "  ")
os.WriteFile("resultado.json", dados, 0644)
```

---

## 4. Controle Granular — `os.Open` e `os.OpenFile`

```go
import (
	"io"
	"os"
)

// os.Open — somente leitura
f, err := os.Open("dados.txt")
if err != nil {
	return fmt.Errorf("abrir: %w", err)
}
defer f.Close()   // SEMPRE com defer — mesmo em caso de erro

// Ler tudo (para arquivos menores)
dados, err := io.ReadAll(f)

// Ler em chunks — para arquivos grandes
buf := make([]byte, 32*1024)   // buffer de 32KB
for {
	n, err := f.Read(buf)
	if n > 0 {
		processar(buf[:n])   // processar apenas os n bytes lidos
	}
	if err == io.EOF {
		break   // fim do arquivo — não é um erro
	}
	if err != nil {
		return fmt.Errorf("ler: %w", err)
	}
}

// Posicionamento (Seek)
offset, err := f.Seek(0, io.SeekStart)    // volta ao início
offset, err  = f.Seek(0, io.SeekEnd)     // vai ao fim (para saber tamanho)
offset, err  = f.Seek(-10, io.SeekCurrent)  // 10 bytes antes da posição atual
```

### `os.OpenFile` — Flags Detalhadas

```go
// Criar arquivo para escrita com append
f, err := os.OpenFile("app.log",
	os.O_APPEND|os.O_CREATE|os.O_WRONLY,
	0644,
)
if err != nil {
	return err
}
defer f.Close()

fmt.Fprintln(f, "nova linha de log")
```

| Flag | Significado |
| --- | --- |
| `os.O_RDONLY` | Somente leitura |
| `os.O_WRONLY` | Somente escrita |
| `os.O_RDWR` | Leitura e escrita |
| `os.O_CREATE` | Cria se não existir |
| `os.O_TRUNC` | Trunca ao abrir |
| `os.O_APPEND` | Sempre escreve no final |
| `os.O_EXCL` | Falha se já existir (operação atômica) |
| `os.O_SYNC` | Sincroniza imediatamente ao disco (fsync) |

---

## 5. Leitura Linha a Linha — `bufio.Scanner`

Para arquivos de texto grandes, `bufio.Scanner` é a forma mais idiomática:

```go
import (
	"bufio"
	"os"
)

f, err := os.Open("dados.csv")
if err != nil {
	return err
}
defer f.Close()

// Scanner com buffer padrão de 64KB por linha
scanner := bufio.NewScanner(f)

// Para linhas > 64KB, aumentar o buffer
const maxLineSize = 10 * 1024 * 1024   // 10MB
buf := make([]byte, maxLineSize)
scanner.Buffer(buf, maxLineSize)

numLinha := 0
for scanner.Scan() {
	numLinha++
	linha := scanner.Text()   // string — cópia segura
	// ou scanner.Bytes()     // []byte — referência ao buffer interno (usar imediatamente!)

	campos := strings.Split(linha, ",")
	processar(campos)
}

// CRÍTICO: verificar erro APÓS o loop
if err := scanner.Err(); err != nil {
	return fmt.Errorf("ler linha %d: %w", numLinha, err)
}
```

### Scanner por Palavras ou Tokens Customizados

```go
scanner := bufio.NewScanner(strings.NewReader("palavra1 palavra2 palavra3"))
scanner.Split(bufio.ScanWords)   // divide por espaços

for scanner.Scan() {
	fmt.Println(scanner.Text())   // "palavra1", "palavra2", "palavra3"
}

// Token customizado — dividir por vírgula
scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
	if i := bytes.IndexByte(data, ','); i >= 0 {
		return i + 1, data[:i], nil
	}
	if atEOF && len(data) > 0 {
		return len(data), data, nil
	}
	return 0, nil, nil
})
```

---

## 6. Escrita com Buffer — `bufio.Writer`

Essencial quando há muitas escritas pequenas — agrupa I/O em chamadas maiores:

```go
import "bufio"

f, _ := os.Create("output.txt")
defer f.Close()

// Sem buffer: cada Fprintln é uma syscall write()
// Com buffer: múltiplos Fprintln são agrupados em poucas syscalls
w := bufio.NewWriterSize(f, 64*1024)   // buffer de 64KB

for i := range 100000 {
	fmt.Fprintf(w, "linha %d: dados importantes\n", i)
}

// OBRIGATÓRIO: flush do buffer antes de fechar
if err := w.Flush(); err != nil {
	return fmt.Errorf("flush: %w", err)
}

// Padrão mais seguro: defer + verificação de erro
defer func() {
	if err := w.Flush(); err != nil && retErr == nil {
		retErr = err
	}
}()
```

---

## 7. I/O de Alta Performance — `io.Copy` e Readers Compostos

```go
// io.Copy — copia de Reader para Writer eficientemente
// Internamente usa um buffer de 32KB e loop de Read/Write
bytesCopiados, err := io.Copy(destino, origem)

// Para arquivos grandes: os.File.ReadFrom e os.File.WriteTo
// usam sendfile() no Linux — zero-copy entre descritores de arquivo
f1, _ := os.Open("origem.dat")
f2, _ := os.Create("destino.dat")
f2.ReadFrom(f1)   // sendfile() se disponível

// LimitReader — limitar leitura (proteção contra DoS)
limitado := io.LimitReader(r.Body, 10*1024*1024)   // máximo 10MB
dados, err := io.ReadAll(limitado)

// TeeReader — ler e simultaneamente escrever em outro lugar
hasher := sha256.New()
tee := io.TeeReader(f, hasher)
dados, _ := io.ReadAll(tee)
checksum := hasher.Sum(nil)   // hash calculado durante a leitura

// MultiWriter — escrever em múltiplos Writers
arquivo, _ := os.Create("log.txt")
multi := io.MultiWriter(arquivo, os.Stdout)   // vai para arquivo E terminal
fmt.Fprintln(multi, "esta linha vai para os dois destinos")

// MultiReader — concatenar múltiplos Readers
r1 := strings.NewReader("primeira parte ")
r2 := strings.NewReader("segunda parte")
combined := io.MultiReader(r1, r2)
dados, _ = io.ReadAll(combined)   // "primeira parte segunda parte"
```

---

## 8. Operações de Sistema de Arquivos

```go
// Informações sobre arquivo
info, err := os.Stat("arquivo.txt")
if err != nil {
	if errors.Is(err, os.ErrNotExist) {
		fmt.Println("não existe")
	}
	return
}
fmt.Println(info.Name())        // "arquivo.txt"
fmt.Println(info.Size())        // bytes
fmt.Println(info.ModTime())     // time.Time
fmt.Println(info.IsDir())       // bool
fmt.Println(info.Mode())        // fs.FileMode (ex: -rw-r--r--)

// Lstat — não segue symlinks
info, _ = os.Lstat("symlink.txt")

// Operações
os.Rename("antigo.txt", "novo.txt")   // mv — atômico no mesmo filesystem
os.Remove("arquivo.txt")              // rm — falha se diretório não-vazio
os.RemoveAll("pasta/")                // rm -rf

os.Mkdir("nova-pasta", 0755)          // mkdir
os.MkdirAll("a/b/c", 0755)           // mkdir -p

// Listar diretório (Go 1.16+)
entradas, _ := os.ReadDir(".")
for _, e := range entradas {
	fmt.Printf("%s  dir:%v\n", e.Name(), e.IsDir())
	// Info() retorna fs.FileInfo — lazy, para evitar stat extra
}

// Arquivo temporário
f, _ := os.CreateTemp("", "prefix-*.json")   // "" = temp dir do OS
defer os.Remove(f.Name())

dir, _ := os.MkdirTemp("", "meuapp-*")
defer os.RemoveAll(dir)
```

---

## 9. Walk — Percorrer Árvore de Diretórios

```go
import "path/filepath"

// filepath.WalkDir — mais eficiente que filepath.Walk (Go 1.16+)
err := filepath.WalkDir(".", func(caminho string, d os.DirEntry, err error) error {
	if err != nil {
		// Erro ao acessar este caminho — continuar ou retornar
		slog.Warn("erro ao acessar", "path", caminho, "err", err)
		return nil   // pular este arquivo, continuar
	}

	// Pular diretórios específicos
	if d.IsDir() {
		switch d.Name() {
		case "node_modules", ".git", "vendor":
			return filepath.SkipDir   // pula o diretório e todo seu conteúdo
		}
	}

	// Processar apenas arquivos .go
	if !d.IsDir() && filepath.Ext(caminho) == ".go" {
		info, err := d.Info()   // stat preguiçoso
		if err != nil {
			return err
		}
		fmt.Printf("%s (%d bytes)\n", caminho, info.Size())
	}

	return nil
})

// fs.WalkDir — para filesystems abstratos (Go 1.16+)
import "io/fs"

fsys := os.DirFS(".")
fs.WalkDir(fsys, ".", func(path string, d fs.DirEntry, err error) error {
	fmt.Println(path)
	return nil
})
```

---

## 10. Copiar Arquivo com Verificação

```go
func copiarArquivo(src, dst string) (escrito int64, err error) {
	origem, err := os.Open(src)
	if err != nil {
		return 0, fmt.Errorf("abrir origem %q: %w", src, err)
	}
	defer origem.Close()

	// Criar com permissões do arquivo de origem
	infoOrigem, err := origem.Stat()
	if err != nil {
		return 0, fmt.Errorf("stat origem: %w", err)
	}

	destino, err := os.OpenFile(dst, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, infoOrigem.Mode())
	if err != nil {
		return 0, fmt.Errorf("criar destino %q: %w", dst, err)
	}
	defer func() {
		closeErr := destino.Close()
		if err == nil {
			err = closeErr   // reportar erro do Close se não havia outro
		}
	}()

	// Usar sendfile se possível (zero-copy)
	n, err := io.Copy(destino, origem)
	if err != nil {
		return n, fmt.Errorf("copiar: %w", err)
	}

	return n, nil
}
```

---

## 11. Resumo — Qual Abordagem Usar

| Situação | Abordagem |
| --- | --- |
| Arquivo pequeno inteiro (< 100MB) | `os.ReadFile` / `os.WriteFile` |
| Arquivo de texto, linha a linha | `bufio.Scanner` |
| Muitas escritas pequenas | `bufio.NewWriterSize(f, 64KB)`  • `Flush()` |
| Controle de posição (Seek) | `os.OpenFile`  • `Seek` |
| Log com append | `os.OpenFile` com `O_APPEND |
| Copiar entre arquivos | `io.Copy` (usa `sendfile` no Linux) |
| Calcular hash durante leitura | `io.TeeReader` |
| Limitar leitura (anti-DoS) | `io.LimitReader(r, maxBytes)` |
| Percorrer diretórios | `filepath.WalkDir` |
| Filesystem abstrato | `io/fs.WalkDir` |
| Arquivo temporário | `os.CreateTemp`  • `defer os.Remove` |

---

## 12. Por Dentro: Arquivos, File Descriptors e Syscalls

### Da função Go até o kernel: `os.Open` destrinchado

```
os.Open("dados.txt")
       │
       ▼
syscall.Open("dados.txt", O_RDONLY, 0)
       │
       ▼  (trap — instrução SYSCALL no x86-64)
kernel: SYS_OPEN (número 2 no Linux x86-64)
       │
       ├─► percorre path no VFS (Virtual File System)
       ├─► localiza inode no filesystem (ext4, btrfs, etc.)
       ├─► verifica permissões (UID/GID do processo vs bits rwx do inode)
       ├─► cria entrada na file descriptor table do processo
       └─► retorna fd (int) — ex: fd = 3

os.File wrappa este fd como { fd: 3, name: "dados.txt" }
```

O file descriptor é um **índice** na tabela de fds do processo (estrutura no PCB — Process Control Block do kernel). Conecta diretamente com [[System Calls]] e [[Arquivos]].

### File descriptors: tabela no kernel por processo

```
Processo Go (PID 1234)
┌──────────────────────────────────────────┐
│  PCB (Process Control Block)             │
│  ┌────────────────────────────────────┐  │
│  │  fd table                          │  │
│  │  fd 0 → pipe (stdin)               │  │
│  │  fd 1 → pipe (stdout)              │  │
│  │  fd 2 → pipe (stderr)              │  │
│  │  fd 3 → inode: /home/u/dados.txt   │  │
│  │  fd 4 → inode: /tmp/output.txt     │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘

Múltiplas goroutines acessando o mesmo *os.File
→ compartilham o mesmo fd
→ kernel serializa acessos via locks internos do VFS
```

Conecta com [[Processos]] (PCB, file descriptor table por processo) e [[Arquivos]] (inodes, VFS).

### `bufio.Reader`: por que reduzir syscalls importa

```
Sem bufio (io.ReadAll direto no *os.File):
  cada Read(p) de tamanho pequeno → syscall read(fd, buf, n)
  100 leituras de 1 byte = 100 syscalls
  cada syscall: ~1μs (context switch user→kernel→user)
  100 syscalls × 1μs = ~100μs overhead só de context switches

Com bufio.Reader (buffer de 4KB por padrão):
  leitura física: read(fd, internalBuf, 4096)  ← 1 syscall lê 4KB do kernel
  leituras lógicas: servidas do buffer em userspace (sem syscall)
  100 leituras de 1 byte = 1 syscall + 99 cópias de memória
  ~1μs + 99 × ~1ns = ~1.1μs total

Redução: ~100x menos syscalls = ~100x menos context switches
```

Conecta com [[System Calls]] (overhead de syscall = context switch, modo user→kernel, salvar registradores) e [[Processos]] (context switch cost).

### Permissões Unix: como `os.OpenFile(path, flags, perm)` funciona

```go
// 0644 em octal:
//   6 = 110 → dono:  rw-  (read + write)
//   4 = 100 → grupo: r--  (read apenas)
//   4 = 100 → outros: r-- (read apenas)

// O kernel verifica:
//   1. UID do processo == UID do arquivo? → aplica bits do dono
//   2. GID do processo == GID do arquivo? → aplica bits do grupo
//   3. Senão → aplica bits de outros

os.OpenFile("app.log", os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0644)
// Flags mapeiam direto para constantes POSIX:
//   O_WRONLY  = 0x001   (write only)
//   O_APPEND  = 0x400   (atomic append — lseek + write atômico no kernel)
//   O_CREATE  = 0x040   (cria se não existir)
```

Conecta com [[Arquivos]] (inodes, permission bits, UID/GID) e [[Processos]] (processo tem UID/GID no PCB, kernel verifica antes de cada syscall de arquivo).

### `os.File.Stat()` → inode lookup

```
f.Stat()
    │
    ▼
syscall.Fstat(fd)   ← versão com fd (mais rápido que stat com path)
    │
    ▼
kernel: SYS_FSTAT
    ├─► encontra inode associado ao fd
    ├─► copia metadados do inode para struct stat:
    │     st_size    (tamanho em bytes)
    │     st_mtime   (última modificação)
    │     st_mode    (tipo + permissões)
    │     st_ino     (número do inode)
    │     st_nlink   (contagem de hard links)
    └─► retorna para userspace

fs.FileInfo.Size(), .ModTime(), .Mode() leem estes campos
```

Conecta com [[Arquivos]] (inode, metadados de arquivo, estrutura do filesystem).

### Buffers do kernel: page cache

```
userspace              kernel              disco
─────────              ──────              ─────
os.ReadFile()
    │
    ▼
read(fd, buf, n)  →  page cache lookup
                         │
                    cache hit? ──yes──► copia página RAM → buf (sem I/O)
                         │
                        no
                         │
                         ▼
                  disk I/O: lê bloco do disco → page cache → buf

Resultado: mesmo sem bufio, leituras repetidas do mesmo arquivo
são servidas do page cache (RAM) — o kernel faz buffering automático.

bufio.Reader adiciona um buffer em USERSPACE para reduzir syscalls.
Page cache é o buffer no KERNEL para reduzir I/O de disco.
São duas camadas de buffering complementares.
```

Conecta com [[Gerenciamento de Memória]] (page cache, virtual memory — páginas de arquivo mapeadas na memória física pelo kernel).

---

## 13. Conexão com Sistemas Operacionais

- **[[System Calls]]**: `os.Open` → `syscall.Open` → `SYS_OPEN` no kernel. Cada `Read()` em `*os.File` emite `read(fd, buf, n)`. `bufio.Reader` reduz drasticamente o número de syscalls ao servir leituras do buffer em userspace — cada syscall tem custo de ~1μs de context switch (user→kernel→user).

- **[[Arquivos]]**: File descriptors são inteiros retornados pelo kernel que indexam a fd table do processo. `os.File` encapsula um fd. `os.Stat()` → `fstat(fd)` lê metadados do inode (tamanho, permissões, timestamps). Flags de permissão (`0644`) mapeiam para os bits rwxrwxrwx do inode Unix.

- **[[Processos]]**: A fd table é por processo — cada processo tem a sua no PCB. O kernel verifica UID/GID do processo contra os bits de permissão do arquivo a cada syscall de abertura. Goroutines do mesmo processo compartilham a fd table.

- **[[Gerenciamento de Memória]]**: O kernel mantém um **page cache** — páginas de arquivos em RAM para evitar I/O de disco. Mesmo `os.ReadFile` sem `bufio` aproveita o page cache. `bufio.Reader` acrescenta uma segunda camada de buffer em userspace para minimizar syscalls. São duas camadas complementares de buffering.

- **[[System Calls]]** (custo): cada `read()` ou `write()` syscall exige salvar registradores, trocar de modo (user→kernel), executar no contexto do kernel e retornar. Isso custa ~1μs. `bufio.Writer.Flush()` agrupa 100 escritas pequenas numa única `write()` syscall, reduzindo overhead ~100x.