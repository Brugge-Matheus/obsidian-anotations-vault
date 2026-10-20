---
tags:
  - go
  - go/io
aliases:
  - Arquivos em Go
  - os.ReadFile
  - os.OpenFile
  - bufio
  - os.Root
  - io/fs
ordem: 34
modulo: "08 - IO e Serialização"
---
# Leitura e Escrita de Arquivos

> **Nota 34 da trilha de Go.** Agora aplicamos `io.Reader`/`io.Writer` ao caso mais concreto: **arquivos**. Nesta nota veremos as formas rápidas (`os.ReadFile`/`os.WriteFile`), o controle fino com `os.Open`, `os.Create` e `os.OpenFile` (flags e permissões), leitura linha a linha com `bufio`, informações de arquivos, diretórios e travessia de árvores, arquivos temporários, **escrita atômica**, o sandbox **`os.Root`** contra *path traversal*, a abstração **`io/fs`** e, por dentro, as syscalls reais que cada operação dispara (com `strace`).
>
> **Pré-requisitos:** [[io.Reader e io.Writer]], [[Defer, Panic e Recover]], [[O Padrão error em Go]].

---

## 🧭 De onde viemos

Em [[io.Reader e io.Writer]] vimos que `*os.File` implementa `Reader`, `Writer`, `Closer` e `Seeker`, e que cada `Read`/`Write` vira uma syscall. Em [[Defer, Panic e Recover]] vimos o padrão `defer f.Close()` e o cuidado com o erro do `Close` em arquivos escritos. Em [[O Padrão error em Go]] conhecemos `fs.ErrNotExist` e `*fs.PathError`. Agora juntamos tudo no trabalho com arquivos reais.

> 💡 **Descritor de arquivo (file descriptor):** número inteiro que o kernel devolve ao abrir um arquivo e que identifica esse arquivo aberto nas syscalls seguintes. 0, 1 e 2 são stdin, stdout e stderr. Um `*os.File` de Go é, essencialmente, um descritor de arquivo com métodos. Você estudou descritores em [[Syscalls para Gerenciamento de Arquivos]].

---

## ⚡ 1. O jeito rápido: arquivo inteiro de uma vez

```go
dados, err := os.ReadFile("config.yaml")             // lê tudo para um []byte
if err != nil {
	return fmt.Errorf("ler config: %w", err)
}

err = os.WriteFile("saida.txt", []byte("olá\n"), 0o644)   // cria/trunca e escreve tudo
```

Saída real:

```go
os.WriteFile("work/a.txt", []byte("linha 1\nlinha 2\nlinha 3\n"), 0o644)
d, _ := os.ReadFile("work/a.txt")
fmt.Printf("%q\n", d)        // "linha 1\nlinha 2\nlinha 3\n"
```

| Use quando... | Evite quando... |
| --- | --- |
| O arquivo é pequeno (configuração, template, JSON pequeno) | O arquivo pode ser grande (logs, dumps, uploads): carrega tudo na memória |
| Você precisa de todo o conteúdo de uma vez | Dá para processar em fluxo |

---

## 🔧 2. Controle fino: `Open`, `Create`, `OpenFile`

```go
f, err := os.Open("dados.csv")      // somente leitura (O_RDONLY)
f, err := os.Create("saida.txt")    // leitura e escrita; cria ou TRUNCA (O_RDWR|O_CREATE|O_TRUNC, 0666)
f, err := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0o644)   // controle total
```

Sempre feche, de preferência logo depois de abrir:

```go
f, err := os.Open(caminho)
if err != nil {
	return err
}
defer f.Close()
```

### 2.1 Flags de abertura

| Flag | Significado |
| --- | --- |
| `os.O_RDONLY` / `O_WRONLY` / `O_RDWR` | Só leitura / só escrita / ambos (escolha exatamente um) |
| `os.O_CREATE` | Cria se não existir |
| `os.O_EXCL` | Com `O_CREATE`: **falha** se já existir (criação exclusiva) |
| `os.O_TRUNC` | Zera o arquivo ao abrir |
| `os.O_APPEND` | Toda escrita vai para o **fim** (atômico em relação a outros processos que também usam append) |
| `os.O_SYNC` | Cada escrita espera chegar ao disco (lento, seguro) |

Saída real de um append:

```go
f, _ := os.OpenFile("work/a.txt", os.O_APPEND|os.O_WRONLY, 0)
fmt.Fprintln(f, "linha 4")      // *os.File é um io.Writer: fmt escreve nele direto
f.Close()
```

> 💡 Essas flags são **as mesmas** da syscall `open` de UNIX (`O_RDONLY`, `O_CREAT`...): o pacote `os` só repassa para o kernel. O Go acrescenta sempre `O_CLOEXEC` (fecha o descritor automaticamente se o processo fizer `exec` de outro programa), para não vazar arquivos abertos para processos filhos.

### 2.2 Permissões

O terceiro argumento (`0o644`) é o **modo** do arquivo, usado só quando ele é **criado**:

```
0o644  =  rw- r-- r--
           │   │   └── outros: ler
           │   └────── grupo: ler
           └────────── dono: ler e escrever

0o600  =  rw- --- ---   (só o dono: segredos, chaves)
0o755  =  rwx r-x r-x   (executáveis e diretórios)
```

> 💡 O valor efetivo é `modo & ^umask` (a *umask* do processo, tipicamente `022`, remove permissões). Por isso `os.Create` usa `0666` e o arquivo nasce como `0644`. Você estudou os bits `rwx`, dono/grupo/outros em [[Proteção]].

---

## 📜 3. Lendo em fluxo: linha a linha

Para arquivos grandes, leia aos poucos com `bufio.Scanner` (saída real):

```go
f, err := os.Open("work/a.txt")
if err != nil {
	return err
}
defer f.Close()

sc := bufio.NewScanner(f)
n := 0
for sc.Scan() {
	linha := sc.Text()          // a linha, sem o \n
	_ = linha
	n++
}
if err := sc.Err(); err != nil {   // erro de leitura aparece DEPOIS do laço
	return err
}
fmt.Println("linhas:", n)       // linhas: 4
```

A memória usada é a de **uma linha**, não a do arquivo: dá para processar um log de 50 GB.

> ⚠️ Lembrete de [[io.Reader e io.Writer]]: o `Scanner` recusa linhas maiores que 64 KB (`token too long`), a menos que você aumente o buffer com `sc.Buffer(make([]byte, 0, 1<<20), 10<<20)`.

### 3.1 Escrevendo em fluxo com buffer

```go
f, err := os.Create("relatorio.csv")
if err != nil {
	return err
}
defer f.Close()

w := bufio.NewWriter(f)              // agrupa escritas pequenas em blocos de 4 KB
for _, r := range registros {
	fmt.Fprintf(w, "%s,%d\n", r.Nome, r.Valor)
}
if err := w.Flush(); err != nil {    // ⚠️ esvazia o buffer; sem isso, o fim do arquivo se perde
	return err
}
```

> ✅ Para arquivos **escritos**, verifique o erro do `Close` também: o sistema pode só relatar falhas de escrita (disco cheio, NFS) no fechamento. Padrão: `defer func() { err = errors.Join(err, f.Close()) }()` com resultado nomeado (ver [[Defer, Panic e Recover]]).

---

## ℹ️ 4. Informações sobre arquivos

Saídas reais:

```go
info, err := os.Stat("work/a.txt")
fmt.Println(info.Name(), info.Size(), info.Mode(), info.IsDir())
// a.txt 32 -rw-r--r-- false
info.ModTime()                 // data da última modificação

_, err = os.Stat("work/nao.txt")
fmt.Println(errors.Is(err, fs.ErrNotExist))    // true   ← o jeito de testar "existe?"
```

```go
// "o arquivo existe?"
if _, err := os.Stat(p); errors.Is(err, fs.ErrNotExist) {
	// não existe
} else if err != nil {
	// outro erro (sem permissão, por exemplo): NÃO conclua que "não existe"
}
```

> ⚠️ **Condição de corrida "verificar e depois usar" (TOCTOU):** testar se existe e depois abrir abre uma janela em que outro processo pode criar/apagar o arquivo. Quando possível, **tente a operação diretamente** e trate o erro: para criar só se não existir, use `os.OpenFile(p, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0o644)`, que é atômico no kernel. Você viu condições de corrida em [[Race Condition]].

> 💡 `os.Stat` segue links simbólicos; `os.Lstat` informa sobre o próprio link. Por baixo, são as syscalls `stat`/`lstat`, que leem o **i-node** do arquivo (tamanho, permissões, datas), como você estudou em [[Syscalls para Gerenciamento de Diretórios]].

---

## 📁 5. Diretórios

Saídas reais:

```go
os.Mkdir("dir", 0o755)                 // cria um nível (erro se o pai não existir)
os.MkdirAll("work/sub/dir", 0o755)     // cria toda a cadeia (como mkdir -p)

es, _ := os.ReadDir("work")            // entradas, ordenadas por nome
for _, e := range es {
	fmt.Print(e.Name(), " ")           // a.txt sub
}

os.Remove("arquivo-ou-dir-vazio")
os.RemoveAll("work")                   // ⚠️ apaga tudo, recursivamente (como rm -rf)
os.Rename("velho", "novo")             // renomear/mover (atômico no mesmo sistema de arquivos)
```

### 5.1 Percorrendo uma árvore: `filepath.WalkDir`

Saída real (pulando a pasta `dir`):

```go
filepath.WalkDir("work", func(p string, d fs.DirEntry, err error) error {
	if err != nil {
		return err                         // erro ao ler este item: parar (ou return nil para ignorar)
	}
	if d.IsDir() && d.Name() == "dir" {
		return filepath.SkipDir            // não entrar nesta pasta
	}
	fmt.Println(" walk:", p, d.IsDir())
	return nil
})
//  walk: work true
//  walk: work/a.txt false
//  walk: work/sub true
//  walk: work/sub/b.go false
```

```go
m, _ := filepath.Glob("work/*/*.go")      // padrões estilo shell
fmt.Println(m)                            // [work/sub/b.go]
```

> 💡 Prefira `WalkDir` ao antigo `Walk`: ele não chama `stat` em cada arquivo (usa o tipo que já vem na listagem do diretório), o que é muito mais rápido em árvores grandes.

---

## 🧪 6. Arquivos e diretórios temporários

```go
f, err := os.CreateTemp("", "upload-*.bin")   // "" = diretório temporário do sistema (/tmp)
defer os.Remove(f.Name())                     // limpe depois!
defer f.Close()

dir, err := os.MkdirTemp("", "build-")
defer os.RemoveAll(dir)

// em testes, melhor ainda: t.TempDir() é apagado automaticamente
```

O `*` no padrão é trocado por um número aleatório, e o arquivo é criado com `O_EXCL` e permissão `0600`, o que evita ataques em que outro usuário cria o arquivo antes.

---

## 🛡️ 7. Escrita atômica

Escrever direto por cima de um arquivo importante (configuração, banco de dados simples) tem um risco: se o programa morrer no meio, o arquivo fica **pela metade**. A técnica clássica de UNIX: escrever num arquivo temporário **no mesmo diretório** e **renomear** por cima (saída real):

```go
func escreverAtomico(caminho string, dados []byte) error {
	tmp, err := os.CreateTemp(filepath.Dir(caminho), ".tmp-*")   // mesmo diretório = mesmo sistema de arquivos
	if err != nil {
		return err
	}
	defer os.Remove(tmp.Name())       // se algo falhar, não deixa lixo (depois do rename, falha inofensiva)

	if _, err := tmp.Write(dados); err != nil {
		tmp.Close()
		return err
	}
	if err := tmp.Sync(); err != nil { // garante que os dados chegaram ao disco
		tmp.Close()
		return err
	}
	if err := tmp.Close(); err != nil {
		return err
	}
	return os.Rename(tmp.Name(), caminho)   // ATÔMICO: quem ler vê o arquivo antigo OU o novo, nunca metade
}
```

As syscalls reais disparadas (`strace`):

```
openat(AT_FDCWD, "work/.tmp-1863157939", O_RDWR|O_CREAT|O_EXCL|O_CLOEXEC, 0600) = 3
fsync(3)                          = 0
renameat(AT_FDCWD, "work/.tmp-1863157939", AT_FDCWD, "work/config.json") = 0
unlinkat(AT_FDCWD, "work/.tmp-1863157939", 0) = -1 ENOENT    ← o defer tentou remover; já foi renomeado
```

> 💡 **Por que `Sync` (fsync)?** Um `write` bem-sucedido só garante que os dados estão no **page cache** do kernel (memória), não no disco. Se faltar energia antes do kernel gravar, os dados se perdem. `fsync` força a gravação. Bancos de dados fazem isso a cada transação. Você estudou o cache de disco em [[Armazenamento não Volátil]] e [[Memória]].

---

## 🔒 8. `os.Root`: acesso confinado a um diretório (Go 1.24+)

Uma vulnerabilidade comum é o **path traversal**: o usuário envia um nome de arquivo como `../../etc/passwd`, e o servidor, ao fazer `filepath.Join(base, nome)`, acaba lendo fora da pasta permitida. O `os.Root` resolve isso no nível do sistema operacional (saídas reais):

```go
root, err := os.OpenRoot("work")       // "prende" as operações dentro de work/
if err != nil {
	return err
}
defer root.Close()

_, err = root.Open("../go.mod")
fmt.Println("root:", err)              // root: openat ../go.mod: path escapes from parent

f, err := root.Open("sub/b.go")        // ✅ dentro da raiz
```

Ele recusa `..` que saia da raiz **e** links simbólicos que apontem para fora, usando syscalls relativas a um descritor de diretório (`openat`) para evitar condições de corrida. Use sempre que nomes de arquivos vierem de entrada externa (uploads, URLs, arquivos compactados).

---

## 🗂️ 9. A abstração `io/fs`

O pacote `io/fs` (Go 1.16) define **interfaces** para sistemas de arquivos somente-leitura, de modo que o mesmo código funcione com o disco, com arquivos embutidos no binário, com um `.zip` ou com um sistema falso em testes (saídas reais):

```go
fsys := os.DirFS("work")                       // um diretório do disco como fs.FS
data, _ := fs.ReadFile(fsys, "sub/b.go")       // caminhos sempre com "/" e relativos
fmt.Println(string(data))                      // package b
matches, _ := fs.Glob(fsys, "*.txt")           // [a.txt]
fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error { ... })
```

| Implementação de `fs.FS` | Origem |
| --- | --- |
| `os.DirFS(dir)` | Diretório do disco |
| `embed.FS` | Arquivos embutidos com `//go:embed` (ver [[Criando Pacotes]]) |
| `*zip.Reader` | Conteúdo de um `.zip` |
| `fstest.MapFS` | Sistema de arquivos em memória, para testes |
| `root.FS()` | Um `os.Root` como `fs.FS` |

```go
// uma função que aceita fs.FS funciona com qualquer um deles
func carregarTemplates(fsys fs.FS) (*template.Template, error) {
	return template.ParseFS(fsys, "templates/*.html")
}

// em produção: carregarTemplates(arquivosEmbutidos)
// em teste:    carregarTemplates(fstest.MapFS{"templates/a.html": {Data: []byte("oi")}})
```

---

## 🔬 10. Por dentro: do `os.Open` ao disco

```
os.Open("work/a.txt")
   │
   ▼
syscall openat(AT_FDCWD, "work/a.txt", O_RDONLY|O_CLOEXEC) = 3      ← trace real
   │   kernel: resolve o caminho diretório por diretório (dentry cache),
   │   checa permissões no i-node, cria uma entrada na tabela de arquivos
   │   abertos e devolve o menor descritor livre (3)
   ▼
*os.File{fd: 3}  (+ registro no poller do runtime, quando aplicável)
   │
f.Read(buf) → syscall read(3, buf, len)
   │   kernel: os dados estão no PAGE CACHE?
   │     sim → copia da memória do kernel para buf (rápido, sem tocar o disco)
   │     não → pede ao driver/disco, espera (a thread bloqueia), copia
   ▼
f.Close() → syscall close(3)
```

As tabelas envolvidas, que você estudou em [[Arquivos]] e [[Syscalls para Gerenciamento de Arquivos]]:

```
 processo Go                     kernel
┌─────────────────┐     ┌────────────────────────┐     ┌──────────────────┐
│ tabela de        │     │ tabela de arquivos     │     │ i-node           │
│ descritores      │     │ abertos                │     │ (metadados +     │
│  0 → stdin       │     │  posição atual (offset)│     │  blocos no disco)│
│  1 → stdout      │────▶│  flags (O_APPEND...)   │────▶│  tamanho, modo,  │
│  2 → stderr      │     │  contador de refs      │     │  datas           │
│  3 → a.txt       │     └────────────────────────┘     └──────────────────┘
└─────────────────┘
```

> 💡 **Arquivos bloqueiam threads; rede não.** Para sockets, o runtime usa o netpoller (epoll) e não prende threads. Para arquivos comuns em disco, o Linux não oferece "prontidão" via epoll, então um `read` lento **bloqueia a thread** do sistema. O escalonador do Go percebe a syscall bloqueada e entrega o processador lógico (P) a outra thread, para que as outras goroutines continuem rodando. Veremos em [[Scheduler do Go]].

> 💡 O limite de arquivos abertos por processo (`ulimit -n`, frequentemente 1024) é um recurso real: esquecer `Close()` em laços é uma causa comum de `too many open files`.

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esquecer `Close()` | `too many open files` | `defer f.Close()` logo após abrir |
| `defer f.Close()` em laço | Descritores acumulados | Extraia uma função por arquivo |
| Ignorar erro de `Close`/`Flush` na escrita | Dados perdidos sem aviso | Verifique os erros |
| `os.Create` em arquivo importante | Trunca antes de escrever; falha = arquivo vazio | Escrita atômica (temp + rename) |
| `os.ReadFile` em arquivo enorme | Memória esgotada | `bufio.Scanner` / streaming |
| `Stat` + depois abrir (TOCTOU) | Corrida com outros processos | Tente a operação; use `O_EXCL` |
| `filepath.Join(base, entradaDoUsuario)` | Path traversal (`../..`) | `os.Root` |
| `os.RemoveAll` com caminho errado/vazio | Apaga o que não devia | Valide o caminho; use diretórios temporários |
| Achar que `write` = no disco | Perda em queda de energia | `f.Sync()` quando a durabilidade importa |
| Permissão `0o644` para segredos | Outros usuários leem | `0o600` |

---

## 📋 Resumo

| Tarefa | Ferramenta |
| --- | --- |
| Ler/escrever arquivo pequeno inteiro | `os.ReadFile` / `os.WriteFile` |
| Abrir para ler / criar | `os.Open` / `os.Create` |
| Controle fino (append, exclusivo) | `os.OpenFile` + flags `O_*` + permissão |
| Ler linha a linha | `bufio.Scanner` |
| Escrever muito | `bufio.Writer` + `Flush` |
| Informações | `os.Stat` + `errors.Is(err, fs.ErrNotExist)` |
| Diretórios | `os.MkdirAll`, `os.ReadDir`, `filepath.WalkDir`, `filepath.Glob` |
| Temporários | `os.CreateTemp`, `os.MkdirTemp`, `t.TempDir()` |
| Escrever com segurança | Temp no mesmo diretório + `Sync` + `Rename` |
| Entrada de usuário como caminho | `os.Root` (1.24) |
| Abstrair o sistema de arquivos | `io/fs` (`os.DirFS`, `embed.FS`, `fstest.MapFS`) |

---

## 🔗 Conexões

**Dentro de Go**
- Os contratos de `Read`/`Write` e o `bufio` → [[io.Reader e io.Writer]]
- `defer Close` e erros de fechamento → [[Defer, Panic e Recover]]
- `fs.ErrNotExist`, `*fs.PathError` → [[O Padrão error em Go]], [[Criando Erros Customizados]]
- `embed.FS` → [[Criando Pacotes]]
- Ler e gravar JSON em arquivos → [[JSON (marshal e unmarshal)]]
- Syscalls bloqueantes e o escalonador → [[Scheduler do Go]]
- `t.TempDir()` e `fstest.MapFS` em testes → [[Testes com go test]]

**Sistemas Operacionais**
- [[Arquivos]] — arquivos, diretórios, caminhos e descritores
- [[Syscalls para Gerenciamento de Arquivos]] — `open`, `read`, `write`, `lseek`, `close`, `stat`
- [[Syscalls para Gerenciamento de Diretórios]] — `mkdir`, `rmdir`, `link`, `unlink`, i-nodes
- [[Proteção]] — permissões `rwx`, dono/grupo/outros
- [[Armazenamento não Volátil]] — por que `fsync` existe (cache vs disco)
- [[Race Condition]] — TOCTOU entre processos

**Docker e Nginx**
- [[Volumes e Persistência]] — arquivos em volumes montados e a importância de escrita atômica e `fsync`
- [[Filesystems e OverlayFS]] — rename atômico só funciona dentro do mesmo sistema de arquivos (camadas do overlay podem surpreender)
- [[Cache HTTP no Nginx]] — o Nginx grava o cache em arquivos temporários e renomeia, a mesma técnica da escrita atômica

---

⬅️ **Anterior:** [[io.Reader e io.Writer]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[JSON (marshal e unmarshal)]] ➡️
