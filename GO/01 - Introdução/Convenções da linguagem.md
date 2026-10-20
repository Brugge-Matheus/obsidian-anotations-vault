---
tags:
  - go
  - go/introducao
aliases:
  - Convenções do Go
  - Estilo Go
  - Idiomático
ordem: 4
modulo: "01 - Introdução"
---
# Convenções da linguagem

> **Nota 4 da trilha de Go.** Go é famosa por ter **uma única cara**: qualquer código Go, de qualquer empresa, parece ter sido escrito pela mesma pessoa. Isso acontece porque parte das convenções é **imposta pelas ferramentas e pelo compilador** e o resto é seguido quase universalmente pela comunidade. Esta nota apresenta essas regras **antes** de mergulharmos na sintaxe, para que todo código da trilha já nasça idiomático.
>
> **Pré-requisitos:** [[Ambiente e Primeiro Programa]].

---

## 🧭 De onde viemos

No [[Ambiente e Primeiro Programa]] escrevemos `fmt.Println` com **P maiúsculo** e vimos que `fmt.println` não compila. Também vimos que a `{` precisa ficar na mesma linha. Essas não são manias: são convenções que, em Go, têm **efeito real**. Nesta nota vamos entender o conjunto completo.

> 💡 **Idiomático:** código escrito do jeito que a comunidade de uma linguagem considera natural, aproveitando os recursos dela em vez de imitar outra linguagem. "Go idiomático" é um elogio frequente em code review; "parece Java escrito em Go" é uma crítica frequente.

---

## 📏 1. Três níveis de regra

Nem toda convenção tem a mesma força. Vale separar:

```
┌──────────────────────────────────────────────────────────────────────┐
│ NÍVEL 1 — O COMPILADOR OBRIGA (não compila se violar)                │
│   • Maiúscula = exportado, minúscula = privado do pacote             │
│   • Imports e variáveis locais não usados                            │
│   • { na mesma linha (por causa da inserção de ;)                    │
├──────────────────────────────────────────────────────────────────────┤
│ NÍVEL 2 — AS FERRAMENTAS PADRONIZAM (gofmt, go vet)                  │
│   • Formatação: indentação com tab, espaçamento, alinhamento         │
│   • Bugs comuns: Printf com verbo errado, cópia de Mutex...          │
├──────────────────────────────────────────────────────────────────────┤
│ NÍVEL 3 — A COMUNIDADE SEGUE (Effective Go, Code Review Comments)    │
│   • MixedCaps, acrônimos em caixa uniforme, nomes curtos             │
│   • Comentários de documentação, organização de imports              │
│   • Mensagens de erro minúsculas e sem ponto final                   │
└──────────────────────────────────────────────────────────────────────┘
```

As referências oficiais da comunidade são o **Effective Go** (go.dev/doc/effective_go), o **Go Code Review Comments** (wiki oficial) e o **Google Go Style Guide**.

---

## 🎨 2. Formatação — `gofmt` é lei

Go acabou com a discussão "tabs ou espaços? chave na mesma linha ou na de baixo?" criando **um formatador oficial sem opções de configuração**.

```bash
gofmt -w .            # formata todos os .go do diretório (escreve no arquivo)
gofmt -d main.go      # mostra o diff sem alterar
gofmt -l .            # lista arquivos que NÃO estão formatados
go fmt ./...          # atalho: roda gofmt -l -w em todos os pacotes

# goimports = gofmt + adiciona/remove/organiza imports automaticamente
go install golang.org/x/tools/cmd/goimports@latest
goimports -w .
```

O que o `gofmt` define:

- **Tabs** para indentação, espaços para alinhamento
- **Chave de abertura na mesma linha** (estilo K&R)
- Espaçamento ao redor de operadores conforme a precedência
- Alinhamento de comentários, campos de struct e tags em colunas

```go
// ❌ Antes do gofmt
func calcular(a,b int)(int,error){
if b==0{return 0,errors.New("divisão por zero")}
return a/b,nil}

// ✅ Depois do gofmt
func calcular(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("divisão por zero")
	}
	return a / b, nil
}
```

**Por que isso importa?**

1. **Diffs limpos:** um `git diff` mostra só mudanças de lógica, nunca de espaçamento
2. **Zero discussão em code review** sobre estilo
3. **Ferramentas automáticas** podem reescrever código (`gofmt -r`, `go fix`) e o resultado continua com a cara padrão

> 💡 O `gofmt` funciona lendo o código como **AST** (a árvore de sintaxe que vimos em [[Como Go Funciona]]) e imprimindo a árvore de volta com regras fixas. Por isso ele nunca muda o significado do programa.

> ✅ Configure o editor para formatar ao salvar e o CI para rejeitar código não formatado:
> ```bash
> test -z "$(gofmt -l .)" || { echo "rode gofmt"; exit 1; }
> ```

---

## 🔓 3. Visibilidade — a regra da letra maiúscula

Go não tem `public`, `private` nem `protected`. Existe **uma única regra**, aplicada a tudo (funções, tipos, variáveis, constantes, campos de struct, métodos):

> **Se o nome começa com letra maiúscula, ele é exportado (visível fora do pacote). Caso contrário, só é visível dentro do próprio pacote.**

```go
package banco

// ── EXPORTADOS: visíveis para quem importar "banco" ──
type Conta struct {
	Titular string   // campo exportado
	saldo   float64  // campo NÃO exportado: só o pacote banco mexe
}

func NovaConta(titular string) *Conta { return &Conta{Titular: titular} }
func (c *Conta) Depositar(v float64)   { c.saldo += v }
func (c *Conta) Saldo() float64        { return c.saldo }

const TaxaAnual = 0.12

// ── NÃO EXPORTADOS: detalhes internos do pacote ──
func (c *Conta) validar() bool { return c.saldo >= 0 }

const limiteInterno = 50_000
```

Em outro pacote:

```go
c := banco.NovaConta("Matheus")
c.Depositar(100)
fmt.Println(c.Saldo())   // ✅ 100
fmt.Println(c.Titular)   // ✅ "Matheus"
fmt.Println(c.saldo)     // ❌ c.saldo undefined (cannot refer to unexported field saldo)
c.validar()              // ❌ c.validar undefined (cannot refer to unexported method validar)
```

Pontos importantes:

- A unidade de encapsulamento é o **pacote**, não o tipo. Dois tipos do mesmo pacote enxergam os campos privados um do outro
- O padrão para expor um campo privado com controle é um **método** (`Saldo()`), como no exemplo
- A regra vale para letras Unicode: `Ação` é exportado, `ação` não

> 💡 Essa verificação é feita pelo **verificador de tipos do compilador**, em tempo de compilação. Não existe custo nenhum em tempo de execução. É parecido com `static` em C (que limita um símbolo ao arquivo), só que a fronteira em Go é o pacote e a checagem é feita antes do linker.

Veremos o encapsulamento com mais detalhes em [[Criando Pacotes]] (incluindo a pasta especial `internal/`).

---

## 🏷️ 4. Nomenclatura

### 4.1 MixedCaps — nunca underscore

```go
// ✅ Go usa MixedCaps (exportado) e mixedCaps (não exportado)
var maxTentativas int
func buscarUsuarioPorEmail(email string) {}
type ConexaoBanco struct{}

// ❌ snake_case e SCREAMING_CASE não são usados
var max_tentativas int
const TAXA_MAXIMA = 10   // em Go, constantes também são MixedCaps: TaxaMaxima
```

> 💡 Underscores aparecem em Go apenas em: nomes de arquivo (`user_test.go`, `file_linux.go`), no identificador vazio `_` e em separadores de dígitos (`1_000_000`).

### 4.2 Acrônimos com caixa uniforme

```go
// ✅ O acrônimo fica todo maiúsculo ou todo minúsculo
userID      // não userId
HTTPServer  // não HttpServer
parseURL    // não parseUrl
xmlDecoder  // não-exportado começando com acrônimo: tudo minúsculo
ServeHTTP   // método da biblioteca padrão segue a regra
```

### 4.3 O tamanho do nome acompanha o escopo

Quanto menor o trecho de código em que o nome vive, mais curto ele pode ser:

```go
for i := range itens { }            // i: vive 3 linhas
for k, v := range mapa { }          // k, v: chave e valor
if err := salvar(); err != nil { }  // err: universal para erros
func (c *Cliente) Nome() string     // receptor: 1-2 letras do tipo
r io.Reader; w io.Writer            // convenções da biblioteca padrão
ctx context.Context                 // sempre "ctx"

func calcularImpostoAnual(...)      // função exportada: descritiva
var timeoutPadraoConexao = 30 * time.Second  // global: descritiva
```

### 4.4 Nomes de pacote

```go
// ✅ curto, uma palavra, minúsculo, sem underscore, substantivo
package http
package strconv
package user

// ❌ evitar
package http_util  // underscore
package HttpUtil   // maiúscula
package utils      // genérico demais: "utils de quê?"
package common     // idem
package helpers    // idem
```

O nome do pacote **faz parte** do nome de tudo que ele exporta, porque quem usa sempre escreve `pacote.Nome`. Por isso:

```go
package user

type User struct{}     // ❌ quem usa lê user.User (gagueira)
type Service struct{}  // ✅ user.Service
func New() *Service    // ✅ user.New()  — padrão da stdlib: bytes.NewBuffer, errors.New

// stdlib segue isso: http.Client (não http.HTTPClient), json.Encoder (não json.JSONEncoder)
```

### 4.5 Getters sem "Get"

```go
func (c *Conta) Saldo() float64      // ✅ getter: só o nome do campo
func (c *Conta) SetSaldo(v float64)  // ✅ setter: prefixo Set
func (c *Conta) GetSaldo() float64   // ❌ não idiomático em Go
```

### 4.6 Interfaces de um método terminam em "-er"

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Stringer interface { String() string }
type Closer interface { Close() error }
```

Veremos interfaces em [[Interfaces]].

---

## 📝 5. Comentários de documentação

Go não tem um formato especial como Javadoc. A documentação é **o comentário imediatamente acima da declaração**, e a ferramenta `go doc` (e o site **pkg.go.dev**) extrai isso automaticamente.

```go
// Package calc fornece operações aritméticas seguras.
//
// Todas as funções são seguras para uso concorrente.
package calc

// ErrDivisaoPorZero é retornado por [Dividir] quando o divisor é zero.
var ErrDivisaoPorZero = errors.New("calc: divisão por zero")

// Somar retorna a soma de a e b.
func Somar(a, b int) int { return a + b }

// Dividir retorna a / b.
// Retorna [ErrDivisaoPorZero] se b for zero.
//
// Exemplo:
//
//	q, err := calc.Dividir(10, 2) // q == 5
func Dividir(a, b int) (int, error) { ... }

// DividirFloat divide sem checagem.
//
// Deprecated: use [Dividir], que trata divisão por zero.
func DividirFloat(a, b float64) float64 { ... }
```

Regras:

| Regra | Exemplo |
| --- | --- |
| Começa com o **nome** do que está sendo documentado | `// Somar retorna...` |
| Frase completa, terminando com ponto | `...a soma de a e b.` |
| Pacote: `// Package nome ...` (uma vez, em qualquer arquivo, geralmente `doc.go`) | `// Package calc fornece...` |
| Link para outro símbolo com colchetes (Go 1.19+) | `[Dividir]`, `[io.Reader]` |
| Bloco de código: linha indentada com tab | `//	q, err := ...` |
| Título de seção: `# Título` | `// # Exemplos` |
| Marcar obsoleto: parágrafo começando com `Deprecated:` | editores riscam o nome |

```bash
go doc calc            # documentação do pacote
go doc calc.Dividir    # documentação de um símbolo
go doc -all calc       # tudo
go doc -ex bytes       # lista exemplos executáveis (Go 1.27+)
```

> ✅ Todo identificador **exportado** deve ter comentário de documentação. Linters como `revive` e `staticcheck` avisam quando falta.

---

## 📥 6. Organização de imports

Três grupos separados por linha em branco, cada um em ordem alfabética (o `goimports` faz isso sozinho):

```go
import (
	// 1. Biblioteca padrão
	"context"
	"fmt"
	"net/http"

	// 2. Dependências externas
	"github.com/google/uuid"
	"golang.org/x/sync/errgroup"

	// 3. Pacotes do próprio módulo
	"github.com/brugge-matheus/app/internal/auth"
)
```

Formas especiais de import (veremos em [[Criando Pacotes]]):

```go
import (
	str "strings"                      // alias: usar como str.ToUpper
	_ "github.com/lib/pq"              // import só pelo efeito colateral (init)
	. "math"                           // dot import: Sqrt em vez de math.Sqrt — evite
)
```

---

## 🧰 7. Ferramentas de qualidade

```bash
go vet ./...            # análise estática oficial (roda junto com go test)
staticcheck ./...       # linter mais completo, padrão de fato da comunidade
golangci-lint run       # agregador de dezenas de linters, muito usado em CI
govulncheck ./...       # procura vulnerabilidades conhecidas nas dependências usadas
go fix ./...            # "modernizadores": reescreve código para APIs/recursos novos
go test -race ./...     # detector de condições de corrida (ver Goroutines)
```

Exemplos do que o `go vet` pega:

```go
fmt.Printf("%d\n", "texto")      // verbo %d com string
var mu sync.Mutex; mu2 := mu      // cópia de Mutex (bug sutil)
return; fmt.Println("nunca")      // código inalcançável
fmt.Errorf("falhou: %w", ptrErr)  // %w com ponteiro para tipo de erro por valor (1.27+)
```

> 💡 **`go fix` modernizado (Go 1.26+):** em vez de fixers históricos, agora ele aplica *modernizers* que trocam padrões antigos por recursos atuais (por exemplo, um laço manual de busca por `slices.Contains`, ou `wg.Add(1); go func(){ defer wg.Done() ... }()` por `wg.Go(...)`). É uma ótima forma de aprender o Go atual: rode `go fix -diff ./...` em código antigo e veja o que muda.

---

## 🗂️ 8. Layout de projeto

**Não existe um layout oficial obrigatório.** O repositório `golang-standards/project-layout`, muito citado, **não é oficial** e é exagerado para a maioria dos projetos. A recomendação oficial (go.dev/doc/modules/layout) é **começar simples e crescer conforme a necessidade**:

```
# Pequeno: um executável
app/
├── go.mod
└── main.go

# Médio: executável + pacotes internos
app/
├── go.mod
├── main.go
└── internal/            ← só este módulo pode importar (o compilador garante)
    ├── auth/
    └── storage/

# Maior: vários executáveis
app/
├── go.mod
├── cmd/
│   ├── api/main.go      ← cada pasta em cmd/ gera um binário
│   └── worker/main.go
└── internal/
    ├── auth/
    ├── order/
    └── storage/
```

Regras práticas:

- Organize por **domínio** (`order/`, `payment/`), não por camada técnica (`models/`, `controllers/`, `services/`), que é um hábito de frameworks MVC de outras linguagens
- Use `internal/` para tudo que não deve ser API pública
- `pkg/` é opcional e controverso; muitos projetos não usam

---

## 💬 9. Convenções que veremos mais adiante

Algumas convenções dependem de assuntos que ainda vamos estudar. Fica aqui o mapa:

| Convenção | Resumo | Nota |
| --- | --- | --- |
| Erros como último retorno | `func f() (T, error)` | [[Múltiplos Retornos e Erros]] |
| Mensagens de erro | minúsculas, sem ponto final, com contexto: `"abrir config: %w"` | [[O Padrão error em Go]] |
| Retorno antecipado | tratar o erro e sair cedo, deixando o caminho feliz sem indentação | [[Controle de Fluxo]] |
| Enums com `iota` | constantes sequenciais tipadas | [[Variáveis e constantes]] |
| Construtores `NewX` | funções que criam valores prontos para uso | [[Structs]] |
| "Aceite interfaces, retorne structs" | parâmetros abstratos, retornos concretos | [[Interfaces]] |
| `ctx` como primeiro parâmetro | `func f(ctx context.Context, ...)` | [[context.Context]] |
| Zero value útil | tipos devem funcionar sem inicialização explícita | [[Variáveis e constantes]] |

---

## 🧘 10. Go Proverbs — a filosofia em frases

Rob Pike resumiu a cultura Go em provérbios curtos (go-proverbs.github.io). Alguns que vão aparecer pela trilha:

| Provérbio | Significado |
| --- | --- |
| *Clear is better than clever.* | Código claro vence código esperto |
| *Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.* | Ninguém ama o estilo, todo mundo ama não discutir estilo |
| *Make the zero value useful.* | Um tipo deve funcionar sem inicialização |
| *The bigger the interface, the weaker the abstraction.* | Prefira interfaces pequenas |
| *A little copying is better than a little dependency.* | Copiar 10 linhas é melhor que adicionar uma dependência |
| *Errors are values.* | Erros são dados comuns, programáveis |
| *Don't just check errors, handle them gracefully.* | Adicione contexto ao propagar |
| *Don't communicate by sharing memory, share memory by communicating.* | Prefira channels a variáveis compartilhadas |
| *Concurrency is not parallelism.* | Estrutura (concorrência) ≠ execução simultânea (paralelismo) |

---

## 📋 Resumo

| Aspecto | Regra |
| --- | --- |
| Formatação | `gofmt`/`goimports`, sem configuração; formatar ao salvar |
| Visibilidade | Maiúscula = exportado; minúscula = privado do **pacote** |
| Nomes | MixedCaps; nunca underscore; acrônimos uniformes (`userID`, `HTTPServer`) |
| Tamanho do nome | Proporcional ao escopo (`i`, `err`, `ctx` vs nomes descritivos) |
| Pacotes | Uma palavra minúscula; não repetir o nome do pacote nos tipos |
| Getters | `Saldo()`, não `GetSaldo()` |
| Interfaces de 1 método | Sufixo `-er` (`Reader`, `Stringer`) |
| Documentação | Comentário acima da declaração, começando pelo nome |
| Imports | stdlib / externos / internos |
| Layout | Comece simples; `cmd/` para binários, `internal/` para o privado |
| Ferramentas | `go vet`, `staticcheck`, `golangci-lint`, `govulncheck`, `go fix` |

---

## 🔗 Conexões

**Dentro de Go**
- Próximo passo, a sintaxe em si → [[Sintaxe, Blocos e Escopo]]
- Pacotes, `internal/` e imports especiais → [[Criando Pacotes]]
- Estilo de erros → [[O Padrão error em Go]]
- Interfaces pequenas → [[Interfaces]]

**Sistemas Operacionais**
- [[A Linguagem C]] — Go herda o estilo K&R de chaves e a sintaxe enxuta de C; a regra da maiúscula substitui o `static`/`extern` de C para controlar visibilidade
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — em C a interface pública de um módulo é o arquivo `.h`; em Go é simplesmente o conjunto de nomes com letra maiúscula
- [[Syscalls para Gerenciamento de Arquivos]] — a convenção de erro de C (retorno `-1` + `errno`) é a ancestral do `(valor, error)` que veremos em Go

---

⬅️ **Anterior:** [[Ambiente e Primeiro Programa]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Sintaxe, Blocos e Escopo]] ➡️
