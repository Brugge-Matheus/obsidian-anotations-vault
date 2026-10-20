---
tags:
  - go
  - go/introducao
aliases:
  - Instalação do Go
  - Hello World em Go
  - Primeiro Programa em Go
ordem: 3
modulo: "01 - Introdução"
---
# Ambiente e Primeiro Programa

> **Nota 3 da trilha de Go.** Hora de colocar a mão no código: instalar a toolchain, entender as variáveis de ambiente, criar um módulo, escrever o primeiro programa e destrinchar **cada linha** dele. No final, veremos os erros de compilação que todo iniciante encontra e por que eles existem.
>
> **Pré-requisitos:** [[O que é Go]], [[Como Go Funciona]].

---

## 🧭 De onde viemos

Em [[Como Go Funciona]] vimos que o compilador transforma pacotes `.go` em um binário estático que carrega o runtime. Agora vamos ver isso na prática: qual ferramenta chama o compilador, onde ficam as dependências baixadas e qual é a estrutura mínima de um programa.

---

## 🛠️ 1. Instalando o Go

A distribuição oficial fica em **go.dev/dl**. Ela traz o compilador, o linker, a biblioteca padrão e o comando `go`, tudo junto.

```bash
# Linux — instalação manual (forma oficial)
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.27.2.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc

# macOS
brew install go

# Windows
winget install GoLang.Go

# Conferir
go version
# go version go1.27.2 linux/amd64
```

> 💡 **Toolchain:** o conjunto de ferramentas que transforma código em programa: compilador (`compile`), linker (`link`), assembler (`asm`), formatador (`gofmt`), analisador (`vet`) etc. No Go, tudo é acessado através de um único comando: `go`.

### O editor

O editor mais usado é o **VS Code** com a extensão oficial **Go**, mas GoLand (JetBrains), Neovim e Zed também são populares. Todos usam o mesmo motor por baixo:

> 💡 **gopls** ("go please"): o *language server* oficial do Go. É ele que oferece autocomplete, "ir para definição", renomear símbolos, mostrar erros enquanto você digita e formatar ao salvar. O editor só exibe o que o gopls calcula.

Configure o editor para **formatar ao salvar** (rodar `gofmt`/`goimports`). Veremos o porquê em [[Convenções da linguagem]].

---

## 🌱 2. Variáveis de ambiente do Go

O comando `go env` mostra a configuração. As mais importantes:

```bash
$ go env GOROOT GOPATH GOMODCACHE GOCACHE GOPROXY GOTOOLCHAIN
/usr/local/go            # GOROOT
/home/matheus/go         # GOPATH
/home/matheus/go/pkg/mod # GOMODCACHE
/home/matheus/.cache/go-build  # GOCACHE
https://proxy.golang.org,direct  # GOPROXY
auto                     # GOTOOLCHAIN
```

| Variável | O que é | Precisa mexer? |
| --- | --- | --- |
| `GOROOT` | Onde o Go está instalado (compilador + biblioteca padrão) | Não |
| `GOPATH` | Pasta de trabalho do Go; por padrão `~/go` | Raramente |
| `GOBIN` / `$GOPATH/bin` | Onde `go install` coloca os executáveis | Adicione ao `PATH` |
| `GOMODCACHE` | Cache das dependências baixadas (`$GOPATH/pkg/mod`) | Não |
| `GOCACHE` | Cache de compilação (pacotes já compilados) | Não |
| `GOPROXY` | De onde baixar módulos (proxy oficial do Google por padrão) | Só em empresas com proxy privado |
| `GOPRIVATE` | Módulos que não devem passar pelo proxy público (repositórios privados) | Em projetos privados |
| `GOOS` / `GOARCH` | Plataforma de destino do build | Para cross-compilation |
| `GOTOOLCHAIN` | Como escolher a versão do Go (veja abaixo) | Não |
| `CGO_ENABLED` | Permite ou não chamar código C | Para binários 100% estáticos: `0` |

```bash
go env -w GOPRIVATE=github.com/minha-empresa/*   # grava uma configuração permanente
go env -u GOPRIVATE                              # remove
```

> 💡 **GOPATH antigo vs módulos:** até o Go 1.11, **todo** código Go precisava morar dentro de `$GOPATH/src`, numa estrutura rígida. Desde os **Go Modules** (1.11, padrão desde 1.16), um projeto pode ficar em qualquer pasta, desde que tenha um arquivo `go.mod`. Hoje o GOPATH serve apenas como local de cache e de binários instalados. Se você vir tutoriais antigos falando em "colocar o projeto no GOPATH", ignore.

### Gerenciamento de versão do Go (`GOTOOLCHAIN`)

Desde o Go 1.21, o próprio comando `go` gerencia versões. Se um projeto declara no `go.mod` que precisa de uma versão mais nova do que a instalada, o `go` **baixa automaticamente** a toolchain correta e usa ela:

```
go.mod do projeto:  go 1.27.0
Go instalado:       1.26.3
GOTOOLCHAIN=auto →  baixa go1.27.x e usa só para esse projeto
```

Isso substitui ferramentas como `gvm`/`asdf` na maioria dos casos.

---

## 📦 3. Criando o primeiro projeto

Todo projeto Go moderno é um **módulo**:

```bash
mkdir ola && cd ola
go mod init github.com/brugge-matheus/ola
```

> 💡 **Módulo:** uma coleção de pacotes Go versionados juntos, definida por um arquivo `go.mod` na raiz. O nome do módulo (o *module path*) normalmente é a URL do repositório, porque é assim que outros projetos vão importar seu código. Para projetos locais que nunca serão publicados, qualquer nome serve (`ola`, `exemplo/ola`).

O comando cria o `go.mod`:

```go
module github.com/brugge-matheus/ola

go 1.26.0
```

> 💡 A linha `go 1.26.0` indica a **versão mínima da linguagem** que o módulo usa, e ativa as regras de linguagem daquela versão. Desde o Go 1.26, `go mod init` escreve por padrão a versão **anterior** à toolchain instalada (com Go 1.27 instalado, ele escreve `go 1.26.0`) para que o módulo funcione também para quem ainda está uma versão atrás. Para usar recursos do 1.27: `go get go@1.27`.

Veremos módulos a fundo em [[Sistema de Módulos (go mod)]].

---

## 👋 4. O primeiro programa, linha por linha

Crie `main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("Olá, Go!")
}
```

Rode:

```bash
$ go run .
Olá, Go!
```

Agora, cada linha:

### `package main`

Todo arquivo Go começa declarando a qual **pacote** pertence.

> 💡 **Pacote (package):** a unidade de organização de código em Go. Todos os arquivos `.go` de uma mesma pasta pertencem ao mesmo pacote e enxergam os nomes uns dos outros. É também a unidade de compilação (o compilador gera um objeto por pacote).

O nome `main` é **especial**: ele diz ao compilador "este pacote gera um **executável**". Qualquer outro nome (`package banco`, `package utils`) gera uma **biblioteca**, que só pode ser importada por outros pacotes.

### `import "fmt"`

Importa o pacote `fmt` (de *format*) da biblioteca padrão, que faz entrada e saída formatada. Para vários imports, usa-se parênteses:

```go
import (
	"fmt"
	"os"
	"strings"
)
```

O caminho do import é uma string. Pacotes da biblioteca padrão têm caminhos curtos (`"fmt"`, `"net/http"`); pacotes externos usam o *module path* (`"github.com/google/uuid"`).

### `func main() { ... }`

Declara a função `main`, o **ponto de entrada** do programa. Regras:

- Precisa existir **exatamente uma** `main` no pacote `main`
- Não recebe parâmetros e não retorna nada (argumentos de linha de comando vêm de `os.Args`; o código de saída vem de `os.Exit`)
- Quando `main` termina, **o programa inteiro termina** (como vimos em [[Como Go Funciona]])

A chave `{` **precisa** estar na mesma linha do `func`. Isso não é só estilo: veremos o motivo técnico em [[Sintaxe, Blocos e Escopo]].

### `fmt.Println("Olá, Go!")`

Chama a função `Println` do pacote `fmt`. Repare:

- O nome começa com **letra maiúscula** (`Println`). Em Go, isso significa que a função é **exportada**, ou seja, visível fora do pacote `fmt`. Uma função `println` minúscula seria privada do pacote (ver [[Convenções da linguagem]])
- Não há ponto e vírgula no final: o compilador insere automaticamente
- `Println` = *print line*: imprime os argumentos separados por espaço e quebra a linha

---

## ▶️ 5. Rodando e compilando

```bash
go run .                 # compila num diretório temporário e executa (desenvolvimento)
go run main.go           # o mesmo, passando o arquivo explicitamente

go build                 # gera o executável "ola" (nome = último trecho do module path)
go build -o bin/app .    # escolhe o nome/caminho de saída
./ola                    # executa o binário gerado

go install .             # compila e coloca o binário em $GOPATH/bin
```

| Comando | Gera arquivo? | Quando usar |
| --- | --- | --- |
| `go run .` | Não (só temporário) | Testar durante o desenvolvimento |
| `go build` | Sim, na pasta atual | Gerar o executável para distribuir |
| `go install` | Sim, em `$GOPATH/bin` | Instalar ferramentas para usar no terminal |

### Outras ferramentas que você vai usar desde o primeiro dia

```bash
go fmt ./...      # formata todos os arquivos (gofmt)
go vet ./...      # análise estática: encontra bugs comuns
go doc fmt.Println   # documentação de qualquer símbolo, direto no terminal
go mod tidy       # sincroniza go.mod/go.sum com os imports do código
go test ./...     # roda os testes (ver Testes com go test)
go fix ./...      # atualiza o código para APIs e recursos modernos (1.26+)
```

> 💡 `./...` significa "este diretório e todos os subdiretórios". É o padrão de caminho mais usado nos comandos do Go.

> 💡 **Go Playground** (go.dev/play): roda código Go no navegador. Ótimo para testar um trecho rápido sem criar projeto.

---

## 🧪 6. Segundo programa: argumentos e código de saída

Um programa um pouco mais real, que lê um argumento da linha de comando:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "uso: saudacao <nome>")
		os.Exit(1)
	}

	nome := os.Args[1]
	fmt.Printf("Olá, %s! Seu nome tem %d letras.\n", nome, len(nome))
}
```

Saída real:

```bash
$ go run .
uso: saudacao <nome>
exit status 1

$ go run . Matheus
Olá, Matheus! Seu nome tem 7 letras.

$ go run . João
Olá, João! Seu nome tem 5 letras.     ← 5?! "João" tem 4 letras...
```

O que apareceu de novo:

| Elemento | Explicação |
| --- | --- |
| `os.Args` | Slice de strings com os argumentos. `os.Args[0]` é o nome do programa; `os.Args[1]` é o primeiro argumento. São o `argv` do C, que o runtime leu na inicialização |
| `len(...)` | Função embutida que retorna o tamanho |
| `nome := ...` | **Declaração curta**: cria a variável e infere o tipo (`string`). Ver [[Variáveis e constantes]] |
| `fmt.Fprintln(os.Stderr, ...)` | Escreve na **saída de erro** (descritor de arquivo 2), e não na saída padrão (1) |
| `os.Exit(1)` | Termina o processo imediatamente com código de saída 1 (convenção UNIX: 0 = sucesso, ≠0 = erro) |
| `fmt.Printf` | Impressão formatada: `%s` = string, `%d` = inteiro decimal, `\n` = quebra de linha |

> ⚠️ **O mistério do "João" com 5 letras:** `len` conta **bytes**, não caracteres. Em UTF-8, o `ã` ocupa **2 bytes**. Esse é um dos detalhes mais importantes de strings em Go, e vamos resolver em [[Strings em Profundidade]] (a resposta envolve o tipo `rune`).

> ⚠️ `os.Exit` encerra o processo **na hora**: funções adiadas com `defer` **não** rodam. Veremos em [[Defer, Panic e Recover]].

> 💡 **Código de saída (exit status):** número que um processo devolve ao terminar, lido pelo processo pai (o shell, com `echo $?`). É o mesmo mecanismo da syscall `exit` e de `waitpid` que você viu em [[Término de Processos]] e [[Syscalls para Gerenciamento de Processos]].

---

## 🚨 7. Os primeiros erros de compilação (e por que existem)

Go é rígido com algumas coisas que outras linguagens só avisam. Estes são os erros que todo iniciante vê nas primeiras horas (mensagens reais do compilador):

### Erro 1 — import ou variável não usados

```go
import (
	"fmt"
	"os"      // importado, mas nunca usado
)

func main() {
	x := 10   // declarado, mas nunca usado
	fmt.Println("oi")
}
```

```
./main.go:5:2: "os" imported and not used
./main.go:9:2: declared and not used: x
```

**Por quê?** Imports não usados deixam o build mais lento (lembra que build rápido foi uma motivação do Go?) e variáveis não usadas quase sempre indicam bug (você calculou algo e esqueceu de usar). Se precisar ignorar um valor de propósito, use o identificador vazio `_` (ver [[Variáveis e constantes]]).

### Erro 2 — chave na linha seguinte

```go
func main()
{
	fmt.Println("oi")
}
```

```
./main.go:6:1: syntax error: unexpected semicolon or newline before {
```

**Por quê?** O compilador insere `;` automaticamente no fim de linhas como `func main()`. Fica `func main();` e a `{` da linha de baixo vira um erro. Veremos a regra exata em [[Sintaxe, Blocos e Escopo]].

### Erro 3 — tipo incompatível e nome não exportado

```go
var idade int = "trinta"
fmt.println(idade)
```

```
./main.go:6:18: cannot use "trinta" (untyped string constant) as int value in variable declaration
./main.go:7:6: undefined: fmt.println
```

**Por quê?** Tipagem estática (não existe conversão implícita de string para int) e a regra de visibilidade (`println` minúsculo não é exportado pelo pacote `fmt`).

> 💡 Repare que o compilador mostra `arquivo:linha:coluna`. Editores com gopls mostram esses erros antes mesmo de você salvar.

---

## 🗂️ 8. Estrutura mínima de um projeto

```
ola/
├── go.mod        ← define o módulo e a versão do Go
├── go.sum        ← hashes das dependências (aparece quando você adiciona uma)
└── main.go       ← package main + func main()
```

Conforme o projeto cresce, ele ganha pacotes em subpastas:

```
ola/
├── go.mod
├── main.go                  ← package main
└── saudacao/
    └── saudacao.go          ← package saudacao (importado como "github.com/brugge-matheus/ola/saudacao")
```

Veremos organização de pacotes em [[Criando Pacotes]].

---

## 📋 Resumo

| Tópico | Ponto-chave |
| --- | --- |
| Instalação | go.dev/dl; `go version` para conferir |
| Editor | Qualquer um com **gopls**; formatar ao salvar |
| GOPATH | Hoje é só cache (`pkg/mod`) e binários (`bin`); projetos ficam onde você quiser |
| Módulo | `go mod init <caminho>` cria o `go.mod` |
| Versão do Go | Linha `go` do `go.mod`; `GOTOOLCHAIN=auto` baixa a versão necessária |
| Programa executável | `package main` + `func main()` |
| Exportado | Nome com letra maiúscula (`fmt.Println`) |
| Rodar / compilar | `go run .` / `go build` / `go install` |
| Argumentos e saída | `os.Args`, `os.Exit(código)`, `os.Stderr` |
| Erros clássicos | Import/variável não usados, `{` na linha seguinte, tipos incompatíveis |

---

## 🔗 Conexões

**Dentro de Go**
- Regras de formatação e nomes → [[Convenções da linguagem]]
- Por que a `{` precisa ficar na mesma linha → [[Sintaxe, Blocos e Escopo]]
- `:=`, `var` e o identificador `_` → [[Variáveis e constantes]]
- Por que `len("João")` é 5 → [[Strings em Profundidade]]
- `go.mod`, `go.sum` e dependências → [[Sistema de Módulos (go mod)]]
- `os.Exit` vs `defer` → [[Defer, Panic e Recover]]

**Sistemas Operacionais**
- [[Interpretador de Comandos Shell]] — o shell faz `fork` + `exec` do seu binário e lê o código de saída com `$?`
- [[Término de Processos]] e [[Syscalls para Gerenciamento de Processos]] — `os.Exit(n)` chama a syscall `exit_group(n)`; o pai recebe o status via `waitpid`
- [[Arquivos]] e [[Syscalls para Gerenciamento de Arquivos]] — `os.Stdout` e `os.Stderr` são os descritores de arquivo 1 e 2
- [[Bits e Bytes]] — base para entender por que `ã` ocupa 2 bytes em UTF-8

---

⬅️ **Anterior:** [[Como Go Funciona]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Convenções da linguagem]] ➡️
