---
tags:
  - go
  - go/fundamentos
---
# Convenções da linguagem

> Go tem um conjunto pequeno e rigoroso de convenções. Diferente de outras linguagens onde estilo é opcional, em Go o compilador e as ferramentas **forçam** muitas delas — e a comunidade segue o resto de forma quase universal. O resultado é código legível por qualquer desenvolvedor Go.
> 

---

## 1. Formatação — `gofmt` é Lei

Go resolve o debate de formatação com uma ferramenta única. `gofmt` formata o código de acordo com o estilo oficial — sem configuração, sem argumentos, sem debates:

```bash
gofmt -w .           # formata todos os arquivos .go no diretório atual (in-place)
gofmt -d arquivo.go  # mostra o diff sem modificar
gofmt -l .           # lista arquivos que precisam de formatação

# goimports = gofmt + organiza imports automaticamente
goimports -w .

# Verificação no CI/CD
test -z "$(gofmt -l .)" || (echo "código não formatado"; exit 1)
```

O que `gofmt` define:

- **Indentação com tabs** (não espaços)
- **Chaves de abertura na mesma linha** (não linha nova)
- Espaços ao redor de operadores binários
- Sem espaço entre nome de função e parênteses

```go
// ✅ Após gofmt
func calcular(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("divisão por zero")
	}
	return a / b, nil
}

// ❌ Antes do gofmt (nunca escreva assim)
func calcular(a,b int)(int,error){
if b==0{return 0,errors.New("divisão por zero")}
return a/b,nil}
```

> 💡 Configure seu editor para executar `gofmt` (ou `goimports`) ao salvar. Todos os projetos Go sérios fazem isso. O CI/CD deve rejeitar código não formatado.
> 

---

## 2. Exportado vs Não-exportado — A Única Regra de Visibilidade

```go
package banco

// ═══════════ EXPORTADO (primeira letra MAIÚSCULA) ═══════════
// Visível e acessível de qualquer pacote que importar "banco"

type Conta struct {
	ID      int64
	Titular string
}

func (c *Conta) Depositar(valor float64) error { ... }

const TaxaJurosAnual = 0.12
var LimitePadrao float64 = 5000

// ═══════════ NÃO-EXPORTADO (primeira letra minúscula) ═══════════
// Acessível apenas dentro do pacote banco

func (c *Conta) validarSaldo() bool { ... }
const limiteAbsoluto = 50000
var pool *connectionPool
```

---

## 3. Nomenclatura

### Nomes de Pacote

```go
// ✅ Uma palavra, tudo minúsculo
package httputil
package mathutil
package user
package auth
package config

// ❌ Evitar
package http_util    // C/Python style
package HttpUtil     // Java style
package utils        // muito genérico
package helpers      // muito genérico
```

### Nomes de Variáveis e Funções — MixedCaps

```go
// ✅ Go usa MixedCaps (camelCase) — nunca underscore
var maxRetryCount int
func getUserByEmail(email string) { }
type DatabaseConnection struct { }

// ❌ Nunca underscore em identificadores Go (exceto test files e blank identifier)
var max_retry_count int       // C/Python style
func get_user_by_email() { } // proibido

// Escopo determina verbosidade do nome:
for i := range items { }         // i — escopo mínimo (loop)
for k, v := range mapa { }       // k, v — padrão para map iteration
err := fazerAlgo()               // err — universal para erros
buf := make([]byte, 1024)        // buf — local

func processarPedido(pedidoID int) error { }  // descritivo — escopo médio
```

### Acrônimos — Todos Maiúsculos ou Todos Minúsculos

```go
// ✅ Acrônimos mantêm capitalização uniforme
var userID int64          // ID todo maiúsculo
type HTTPServer struct {} // HTTP todo maiúsculo — não HttpServer
type URLParser struct {}  // URL todo maiúsculo — não UrlParser
func parseJSON() {}       // JSON todo maiúsculo

// Em contexto não-exportado — todos minúsculos
var xmlData []byte        // não XmlData
type httpClient struct {} // não HttpClient
```

### Nomes de Tipos Não Repetem o Pacote

```go
// O pacote já provê o contexto — não repita
package user

type User struct { }      // ❌ redundante — fica user.User no código do caller
type Service struct { }   // ✅ fica user.Service
type Repository struct {}  // ✅ fica user.Repository
func New() *Service { }   // ✅ fica user.New() — claro e idiomático
```

---

## 4. Comentários de Documentação

Seguem um formato específico reconhecido por `godoc` e `pkg.go.dev`:

```go
// Package mathutil fornece funções matemáticas auxiliares.
// Todas as funções são seguras para uso concorrente.
//
// # Exemplos básicos
//
// Para somar dois inteiros:
//
//	resultado := mathutil.Somar(3, 4)   // 7
package mathutil

// ErrDivisaoPorZero é retornado por Dividir quando o divisor é zero.
// Use [errors.Is] para verificar:
//
//	if errors.Is(err, mathutil.ErrDivisaoPorZero) { ... }
var ErrDivisaoPorZero = errors.New("divisão por zero")

// Somar retorna a soma de a e b.
// Ambos os valores podem ser negativos.
func Somar(a, b int) int {
	return a + b
}

// Dividir retorna a divisão de a por b.
// Retorna [ErrDivisaoPorZero] se b for zero.
//
// Deprecated: Use [DividirSeguro] que retorna (float64, error).
func Dividir(a, b float64) float64 { ... }
```

Regras de documentação:

- Começa com o **nome do identificador**: `// NomeDaCoisa faz X`
- Termina com **ponto final**
- Para pacotes: `// Package nome faz X`
- Links: `[OutroTipo]` cria hyperlink no godoc (Go 1.19+)
- `Deprecated:` na primeira linha marca como obsoleto
- Exemplos de código: indentados com tab dentro do comentário

---

## 5. Organização de Imports

Três grupos separados por linha em branco — `goimports` faz isso automaticamente:

```go
import (
	// 1. Stdlib — ordem alfabética
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"time"

	// 2. Dependências externas — ordem alfabética
	"github.com/gin-gonic/gin"
	"go.uber.org/zap"
	"golang.org/x/sync/errgroup"

	// 3. Pacotes internos do módulo — ordem alfabética
	"github.com/empresa/projeto/internal/auth"
	"github.com/empresa/projeto/internal/config"
	"github.com/empresa/projeto/pkg/logger"
)
```

---

## 6. Tratamento de Erros — O Estilo Go

```go
// ✅ Verificar imediatamente após a chamada
resultado, err := operacao()
if err != nil {
	return fmt.Errorf("contexto adicional: %w", err)
}

// ✅ Mensagens de erro: minúsculas, sem ponto final
errors.New("arquivo não encontrado")         // ✅
errors.New("Arquivo não encontrado.")        // ❌ maiúscula + ponto

// ✅ Early return — guard clauses
func processar(req *Request) error {
	if req == nil {
		return errors.New("request nula")
	}
	if req.ID == 0 {
		return errors.New("ID obrigatório")
	}
	// lógica principal sem aninhamento
}

// ❌ Nesting desnecessário
func processar(req *Request) error {
	if req != nil {
		if req.ID != 0 {
			// lógica aqui — muito aninhado
		}
	}
}
```

---

## 7. Constantes com `iota`

```go
type DiaSemana int

const (
	Segunda DiaSemana = iota + 1   // 1
	Terca                          // 2
	Quarta                         // 3
	Quinta                         // 4
	Sexta                          // 5
	Sabado                         // 6
	Domingo                        // 7
)

func (d DiaSemana) String() string {
	nomes := [...]string{"", "Segunda", "Terça", "Quarta",
		"Quinta", "Sexta", "Sábado", "Domingo"}
	if d < Segunda || d > Domingo {
		return fmt.Sprintf("DiaSemana(%d)", int(d))
	}
	return nomes[d]
}

// Bitmask flags com iota
type Permissao uint

const (
	Leitura   Permissao = 1 << iota   // 001 = 1
	Escrita                           // 010 = 2
	Execucao                          // 100 = 4
	Admin     Permissao = Leitura | Escrita | Execucao  // 111 = 7
)
```

---

## 8. Layout de Projeto — Standard Layout

```bash
meu-projeto/
├── cmd/
│   └── servidor/
│       └── main.go         ← package main — entry point
├── internal/               ← código privado (não importável externamente)
│   ├── auth/
│   ├── handler/
│   └── repository/
├── pkg/                    ← código público reutilizável por outros módulos
│   └── mathutil/
├── api/                    ← definições de API (OpenAPI 3.x, proto)
├── configs/                ← templates e exemplos de configuração
├── scripts/                ← scripts de build, migração, CI
├── testdata/               ← fixtures e dados para testes
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## 9. `go vet` e Ferramentas de Qualidade

```bash
# Análise estática oficial — detecta bugs comuns
go vet ./...

# O que go vet detecta:
# - Printf com formato errado: fmt.Printf("%d", "string")
# - Mutex copiado: mu2 := mu  (cópia de Mutex — bug!)
# - Código inacessível após return/panic
# - Testes sem assinatura correta
# - Loop variable capture (antes do Go 1.22)
# - Composites literals com campos faltando em tipos exportados

# staticcheck — linter mais completo (padrão da comunidade)
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...

# golangci-lint — agregador de múltiplos linters
golangci-lint run

# Detecção de race conditions — ESSENCIAL em CI
go test -race ./...
go run  -race main.go

# Checagem de vulnerabilidades
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

---

## 10. Resumo das Principais Convenções

| Aspecto | Regra |
| --- | --- |
| Formatação | `gofmt` — obrigatório, sem configuração |
| Indentação | Tabs (não espaços) |
| Exported | Primeira letra MAIÚSCULA |
| Unexported | Primeira letra minúscula |
| Nomes | MixedCaps — nunca underscore |
| Acrônimos | `HTTP`, `URL`, `ID` — todos maiúsculos ou todos minúsculos |
| Pacotes | Uma palavra, minúsculas |
| Tipos no pacote | Não repetem o nome do pacote |
| Comentários | Começam com o nome: `// Tipo faz X`, terminam com ponto |
| Erros | Minúsculas, sem ponto final, sem prefixo "Error:" |
| Imports | Três grupos: stdlib / externos / internos |
| Erro ao retornar | Sempre verificar; propagar com `%w`; early return |

---

## 11. Por Dentro: Como Convenções Conectam com o Sistema

### Exportado/não-exportado: visibilidade via tabela de símbolos

```
package banco

func Depositar(...) { }   // exportado → símbolo público no objeto .a
func validar(...) { }     // não-exportado → símbolo privado (local)

No arquivo objeto gerado (ELF/Mach-O):
  .symtab:
    banco.Depositar   GLOBAL   0x4001a0   (visível pelo linker)
    banco.validar     LOCAL    0x4001f0   (invisível externamente)

Analogia com C:
  Go exported   ↔  int funcao()       (extern — linkagem externa)
  Go unexported ↔  static int funcao() (linkagem interna ao arquivo .o)

O compilador Go enforça esta regra em COMPILE TIME:
  // pacote externo tentando acessar:
  banco.validar()  →  erro: "cannot refer to unexported name banco.validar"
  // verificado antes de gerar código — sem runtime overhead
```

Conecta com [[Processadores]] — symbol table no linker, linkagem estática (GLOBAL vs LOCAL symbols, similar a `extern` vs `static` em C).

### `gofmt`: formatação como padrão de toolchain

```
gofmt -w arquivo.go:
  1. Parse do arquivo → AST (Abstract Syntax Tree)
  2. Reformata o AST usando regras fixas do printer
  3. Escreve o resultado de volta ao arquivo

O que é fixo (sem configuração):
  - Tabs para indentação (não espaços — decisão deliberada do Go team)
  - Chaves de abertura na mesma linha (K&R style, não Allman)
  - Alinhamento de struct tags e imports
  - Espaços em torno de operadores binários

Por que isso importa para tooling?
  - diff entre versões mostra apenas mudanças reais (sem "ruído" de formatação)
  - grep e sed em código Go sempre funciona com layout previsível
  - o compilador pode assumir AST canônico — otimizações mais simples

Comparação:
  Kernel Linux: tem coding standards (checkpatch.pl), mas não enforça 100%
  Go:           gofmt é parte do CI — código não-formatado não passa no review
```

Conecta com [[Processos]] — toolchain padronizado garante builds reproduzíveis. O processo de build de Go é determinístico em parte porque formatação é canônica.

### Tratamento de erros: convenção vs mecanismo

```go
// Em C: erros via valor de retorno (int) ou errno global
// Convenção, não enforçado:
int fd = open("file.txt", O_RDONLY);
// sem verificação → ninguém impede, compilador não reclama
fd = open("noexist", O_RDONLY);   // retorna -1, errno=ENOENT — silencioso!

// Em Go: mesma filosofia (convenção) mas com pattern idiomático claro
f, err := os.Open("file.txt")
// if err != nil não é enforçado pelo compilador
// mas: go vet avisa sobre erros ignorados em alguns casos
// e: a convenção é tão forte que PRs sem checagem são rejeitados

// A diferença para Java (exceptions):
//   Java: compilador força try/catch para checked exceptions
//   Go:   convenção + code review + go vet
//   Vantagem Go: sem custo de exception overhead (stack unwinding)
//   Desvantagem Go: possível ignorar erros acidentalmente

// Mensagem de erro idiomática:
errors.New("arquivo não encontrado")  // minúscula, sem ponto
// Por quê minúscula?
//   erros são frequentemente concatenados: fmt.Errorf("abrir config: %w", err)
//   "abrir config: arquivo não encontrado" — coerente
//   "abrir config: Arquivo não encontrado." — estranha a maiúscula no meio
```

Conecta com [[System Calls]] — convenção de verificação de erros em C (retorno negativo + errno) é o ancestral direto do padrão `(value, error)` do Go. Ambos são convenções, não enforcement do compilador/hardware.

### Nomes de pacotes: uma palavra, impacto no linker

```
package httputil   // uma palavra, minúscula

Impacto no compilador/linker:
  - Todos os símbolos do pacote têm prefixo: httputil.FuncX
  - O linker usa esse prefix para resolução de símbolos entre pacotes
  - Colisão de nomes entre pacotes é impossível (namespace explícito)

Diferente de C:
  #include "httputil.h"   // flat namespace — colisão possível!
  void Connect() { }      // se duas libs definem Connect() → link error

Go:
  httputil.Connect()  vs  tcputil.Connect()
  → namespaces separados → sem colisão → linker resolve sem ambiguidade

Por que não underscores?
  - Identificadores com _ eram reservados no C para implementações
  - Go escolheu camelCase para uniformidade
  - gofmt enforça: nunca terá código Go com snake_case em produção
```

Conecta com [[Processadores]] — namespaces de símbolos no linker, resolução de símbolos em tempo de link, symbol table.

---

## 12. Conexão com Sistemas Operacionais

- **[[Processadores]]**: Exportado (PascalCase) vs não-exportado (camelCase) é enforçado pelo compilador em compile time via tabela de símbolos — símbolos exportados ficam como `GLOBAL` no `.symtab` do objeto, não-exportados como `LOCAL`. Análogo a `extern` vs `static` em C, mas verificado pelo compilador antes de chegar ao linker.

- **[[Processos]]**: `gofmt` garante formatação canônica do código — parte da cadeia de toolchain Go que torna builds determinísticos e diffs limpos. Similar ao papel de `checkpatch.pl` no kernel Linux, mas mais rígido: código não-formatado é rejeitado em CI.

- **[[System Calls]]**: O padrão `if err != nil { return err }` herda diretamente da convenção de erro de C (`if (fd < 0) { /* handle errno */ }`). Ambos são convenções, não enforcement da linguagem ou do hardware. A diferença: Go retorna `error` explicitamente no tipo; C usa variável global `errno` — menos thread-safe.