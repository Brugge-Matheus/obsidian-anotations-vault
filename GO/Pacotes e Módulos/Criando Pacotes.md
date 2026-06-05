---
tags:
  - go
  - go/módulos
---
# Criando Pacotes

> Pacotes são a unidade de modularização em Go. Todo arquivo `.go` pertence a um pacote. O sistema de pacotes é simples — sem namespaces aninhados, sem `import as` obrigatório — mas tem regras precisas de visibilidade e convenções fortes.
> 

---

## 1. O Que é um Pacote

```go
Regras fundamentais:
1. Todo arquivo .go começa com "package nome"
2. Todos os arquivos em um mesmo DIRETÓRIO devem ter o mesmo package name
3. O package name e o nome do diretório NÃO precisam ser iguais (mas por convenção são)
4. Exceção: arquivos *_test.go podem usar "package X_test" para teste caixa-preta

Diretório:          mathutil/
Arquivo:            mathutil/soma.go
Package declaration: package mathutil   ← nome do pacote
Import path:        github.com/usuario/projeto/mathutil   ← caminho de import
```

```go
// arquivo: mathutil/soma.go
package mathutil   // declaração obrigatória na primeira linha (após comentário de pacote)

// Somar retorna a soma de dois inteiros.
func Somar(a, b int) int {
	return a + b
}

// arquivo: mathutil/media.go
package mathutil   // mesmo pacote, arquivo diferente — podem acessar símbolos não-exportados

func Media(nums ...float64) float64 {
	if len(nums) == 0 {
		return 0
	}
	total := 0.0
	for _, n := range nums {
		total += n
	}
	return total / float64(len(nums))
}
```

---

## 2. Visibilidade — Exportado vs Não-exportado

Em Go, a visibilidade é controlada pela **primeira letra** do identificador. Não há palavras-chave `public`, `private`, `protected`:

```go
package banco

// ═══════════════════════════════════
// EXPORTADOS — visíveis fora do pacote
// ═══════════════════════════════════

type Conta struct {           // exportado — pode ser usado por outros pacotes
	ID       int64            // exportado
	Titular  string           // exportado
	saldo    float64          // não-exportado — apenas o pacote banco acessa
	historico []Transacao     // não-exportado
}

func NovaConta(titular string) (*Conta, error) { ... }  // exportado
func (c *Conta) Depositar(valor float64) error { ... }   // exportado
func (c *Conta) Saldo() float64 { ... }                  // exportado

const TaxaJurosAnual = 0.12   // exportado
var LimitePadrao = 5000.0     // exportado

// ═══════════════════════════════════
// NÃO-EXPORTADOS — apenas dentro do pacote banco
// ═══════════════════════════════════

func (c *Conta) validarSaldo() bool { ... }   // não-exportado
const limiteMaximo = 50000                    // não-exportado
var pool *connectionPool                      // não-exportado

// Tipos não-exportados — úteis para implementação interna
type transacaoInternal struct {
	valor float64
	data  time.Time
}
```

### Verificar Implementação de Interface em Compilação

```go
// Padrão para garantir que um tipo satisfaz uma interface
var _ io.Reader = (*MeuReader)(nil)   // se *MeuReader não implementar io.Reader → erro de compilação
var _ Forma = Retangulo{}             // verifica tipo valor
var _ fmt.Stringer = (*Usuario)(nil)  // verifica tipo ponteiro
```

---

## 3. Estrutura de Diretórios — Layout Recomendado

### Projeto Simples (comando único)

```
calculadora/
├── go.mod
├── go.sum
├── main.go        ← package main
└── README.md
```

### Projeto com Múltiplos Pacotes

```
meu-projeto/
├── go.mod
├── go.sum
├── main.go                    ← package main (ou cmd/)
├── handlers/
│   ├── usuario.go             ← package handlers
│   └── produto.go
├── models/
│   ├── usuario.go             ← package models
│   └── produto.go
├── repository/
│   ├── interface.go           ← package repository
│   └── postgres.go
└── services/
    └── email.go               ← package services
```

### Standard Layout (projetos maiores)

```
meu-projeto/
├── cmd/
│   ├── servidor/
│   │   └── main.go            ← package main — servidor HTTP
│   └── worker/
│       └── main.go            ← package main — worker background
├── internal/                  ← código privado do módulo
│   ├── auth/
│   │   ├── jwt.go             ← package auth
│   │   └── jwt_test.go
│   ├── handler/
│   └── repository/
├── pkg/                       ← código público reutilizável por outros módulos
│   └── mathutil/
│       ├── soma.go
│       └── soma_test.go
├── api/                       ← definições de API (OpenAPI, proto)
│   └── openapi.yaml
├── configs/                   ← arquivos de configuração de exemplo
├── scripts/                   ← scripts de build, migração
├── testdata/                  ← dados para testes
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## 4. O Diretório `internal/` — Proteção Embutida no Compilador

```go
// Localização: github.com/usuario/projeto/internal/auth/jwt.go
package auth

// JWT é acessível apenas de dentro de github.com/usuario/projeto
// e seus subpacotes.

// ❌ Se outro módulo tentar importar:
// import "github.com/usuario/projeto/internal/auth"
// → Erro de compilação:
//   use of internal package github.com/usuario/projeto/internal/auth not allowed

// ✅ Pacotes dentro do mesmo módulo podem importar:
// github.com/usuario/projeto/cmd/servidor
// github.com/usuario/projeto/internal/handler
```

---

## 5. Importar Pacotes

```go
import "fmt"                                        // stdlib
import "github.com/gin-gonic/gin"                  // externo

// Forma idiomática — múltiplos imports em bloco
import (
	// 1. Stdlib — em ordem alfabética
	"context"
	"errors"
	"fmt"
	"net/http"
	"strings"

	// 2. Externos — em ordem alfabética
	"github.com/gin-gonic/gin"
	"go.uber.org/zap"
	"golang.org/x/sync/errgroup"

	// 3. Internos do módulo — em ordem alfabética
	"github.com/empresa/projeto/internal/auth"
	"github.com/empresa/projeto/pkg/logger"
)
```

### Aliases de Import

```go
import (
	// Resolver conflito de nomes
	mrand "math/rand"
	crand "crypto/rand"

	// Nome mais curto para pacote com nome longo
	pg "github.com/lib/pq"

	// Import blank — executa init() sem usar o pacote diretamente
	// (registrar drivers, codecs, etc.)
	_ "github.com/lib/pq"            // registra driver PostgreSQL
	_ "image/png"                    // registra decoder PNG
	_ "time/tzdata"                  // embed timezone database (Go 1.15+)

	// Import ponto — importa todos os nomes para o namespace atual
	// Use com MUITA cautela — polui o namespace, dificulta leitura
	. "math"   // agora pode usar Sqrt em vez de math.Sqrt
)
```

---

## 6. A Função `init()` — Inicialização de Pacote

```go
package config

import (
	"log"
	"os"
)

var DatabaseURL string
var AppPorta string

// init() executa automaticamente quando o pacote é importado
// Antes do main(), depois de todos os var declarations
func init() {
	DatabaseURL = os.Getenv("DATABASE_URL")
	if DatabaseURL == "" {
		log.Fatal("DATABASE_URL não configurada")
	}

	AppPorta = os.Getenv("PORT")
	if AppPorta == "" {
		AppPorta = "8080"
	}
}
```

### Ordem de Inicialização

```
Para um pacote P que importa A e B, onde A importa C:

1. C é inicializado (vars + init())
2. A é inicializado (vars + init())
3. B é inicializado (vars + init())
4. P é inicializado (vars + init())
5. main() executa

Dentro de um pacote:
1. Variáveis de pacote, na ordem de dependência
2. Funções init(), na ordem de aparição nos arquivos
   (ordem de arquivos = ordem lexicográfica dos nomes)

Múltiplas funções init() em um arquivo: todas executam, na ordem de aparição
Múltiplos arquivos: init() de a.go antes de b.go (lexicográfico)
```

> ⚠️ Use `init()` com moderação — dificulta o rastreamento do fluxo de inicialização e o teste unitário. Prefira inicialização explícita via construtores ou configuração injetada.
> 

---

## 7. Documentação de Pacotes com `godoc`

```go
// Package mathutil fornece funções matemáticas auxiliares.
// É seguro para uso concorrente.
//
// # Exemplos
//
// Somar dois números:
//
//	resultado := mathutil.Somar(3, 4)   // 7
//
// Calcular média:
//
//	media := mathutil.Media(1.0, 2.0, 3.0)   // 2.0
package mathutil

// ErrDivisaoPorZero é retornado quando o divisor é zero.
// Use errors.Is para verificar:
//
//	if errors.Is(err, mathutil.ErrDivisaoPorZero) { ... }
var ErrDivisaoPorZero = errors.New("divisão por zero")

// Somar retorna a soma de a e b.
// Ambos os argumentos podem ser negativos.
func Somar(a, b int) int {
	return a + b
}

// Dividir retorna a divisão de a por b.
//
// Retorna [ErrDivisaoPorZero] se b for zero.
// Para divisão inteira segura, prefira [DividirInt].
//
// Deprecated: Use DividirSeguro que retorna (float64, error).
func Dividir(a, b float64) float64 {
	if b == 0 {
		panic(ErrDivisaoPorZero)
	}
	return a / b
}
```

```bash
# Gerar documentação local
go doc mathutil
go doc mathutil.Somar
go doc -all mathutil

# Servidor de documentação
go install golang.org/x/tools/cmd/godoc@latest
godoc -http=:6060
# Acesse http://localhost:6060/pkg/github.com/usuario/projeto/mathutil/
```

---

## 8. Testes — Package `X` vs `X_test`

Go permite dois estilos de teste no mesmo diretório:

```go
// arquivo: mathutil/soma_test.go

// Opção 1: package mathutil (caixa branca — acessa não-exportados)
package mathutil

func TestFuncaoInterna(t *testing.T) {
	// acessa funcaoPrivada() diretamente
}

// Opção 2: package mathutil_test (caixa preta — apenas API pública)
package mathutil_test

import "github.com/usuario/projeto/mathutil"

func TestSomar(t *testing.T) {
	resultado := mathutil.Somar(3, 4)
	if resultado != 7 {
		t.Errorf("esperado 7, obtido %d", resultado)
	}
}
```

> 💡 Prefira `package X_test` para a maioria dos testes — você testa a API pública da mesma forma que um usuário externo. Use `package X` apenas quando precisar acessar internals.
> 

---

## 9. Convenções de Nomenclatura de Pacotes

```go
// ✅ Uma palavra, minúsculas
package httputil
package mathutil
package user
package auth
package config

// ✅ Acrônimos mantidos em maiúsculas
package httpserver   // não httpServer
package xmlparser    // não xmlParser

// ❌ Evitar nomes genéricos
package util     // muito amplo — o que é util?
package common   // idem
package helpers  // idem
package misc     // idem

// ❌ Underscore ou camelCase
package http_util  // C/Python style
package HttpUtil   // Java style

// ✅ O nome do tipo NÃO repete o pacote
// (o pacote já provê o contexto)
package user

type User struct { }      // ❌ redundante — fica user.User
type Service struct { }   // ✅ — fica user.Service
type Repository struct {}  // ✅ — fica user.Repository

// Funções construtoras
func New(nome string) *Service { }          // ✅ — user.New()
func NewService(nome string) *Service { }   // ✅ — quando há múltiplos tipos
```

---

## 10. Pacote `main` — Entry Point

```go
// cmd/servidor/main.go
package main   // especial: o compilador sabe que este é o entry point

import (
	"log/slog"
	"os"
	"os/signal"
	"syscall"

	"github.com/empresa/projeto/internal/config"
	"github.com/empresa/projeto/internal/handler"
	"github.com/empresa/projeto/internal/repository"
)

func main() {
	// Logger estruturado desde o início
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))
	slog.SetDefault(logger)

	// Carregar configuração
	cfg, err := config.Carregar()
	if err != nil {
		slog.Error("falha ao carregar configuração", "err", err)
		os.Exit(1)
	}

	// Montar dependências (Dependency Injection manual)
	repo := repository.NovoPostgres(cfg.DatabaseURL)
	srv := handler.NovoServidor(cfg, repo, logger)

	// Iniciar e aguardar shutdown
	go func() {
		slog.Info("servidor iniciado", "porta", cfg.Porta)
		if err := srv.Iniciar(); err != nil {
			slog.Error("servidor encerrado", "err", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
	<-quit

	slog.Info("encerrando...")
	srv.Encerrar()
}
```

> 💡 `main()` deve ser curta e focada em montar dependências. A lógica de negócio fica nos pacotes internos. Isso facilita testes e manutenção.
>

---

## 11. Como Funciona Internamente

### Pacote = Unidade de Compilação

```
Todos os arquivos .go de um diretório são compilados JUNTOS:

  banco/
    conta.go        ─┐
    transacao.go     ├──► go tool compile → banco.a  (archive/object)
    validacao.go    ─┘

  banco.a contém:
  ┌────────────────────────────────────────────┐
  │  .text  → código das funções               │
  │  .data  → variáveis globais inicializadas   │
  │  .bss   → variáveis globais zero-valued     │
  │  .rodata→ strings e constantes             │
  │  tabela de símbolos exportados              │
  └────────────────────────────────────────────┘

  Semelhante a um .o (object file) em C, mas empacotado com
  informações de tipo e interface de Go.
```

### Ordem de Inicialização — Análogo ao `.init_array` do ELF

```
Em binários ELF (Linux), o loader chama construtores antes de main():
  .init_array → array de ponteiros de função → chamados antes de main()
  Bibliotecas C++ registram constructors estáticos aqui.

Em Go:
  1. Dependências são inicializadas primeiro (topological sort)
  2. Dentro de cada pacote: variáveis de pacote (por ordem de dependência)
  3. Depois: todas as funções init() na ordem de aparição
  4. Por último: main.main()

  Estrutura análoga no binário Go:
  runtime.init → init de deps → init do pacote → main.main

  Exemplo com dependências:
  main imports A e B; A imports C

  ordem garantida:
  C.init() → A.init() → B.init() → main.init() → main.main()
```

### Exportado vs Não-Exportado — Visibilidade de Símbolos

```
Em C (linker):
  static int contador = 0;       // símbolo LOCAL (não vaza para outros .o)
  int ContadorPublico = 0;       // símbolo GLOBAL (visível ao linker)

  Visibilidade controlada pelo LINKER no momento do link.

Em Go:
  var contador int               // não-exportado: só dentro do pacote
  var ContadorPublico int        // exportado: visível fora do pacote

  Diferença: Go verifica em TEMPO DE COMPILAÇÃO, não no link.
  O compilador recusa import de símbolo não-exportado com erro claro:
  "cannot refer to unexported name banco.validarSaldo"

  Mais seguro: erro em compile time, não em link time ou runtime.
```

### Importações Circulares — Prevenção de Deadlocks de Init

```
Por que Go PROÍBE importações circulares:

  Se A imports B e B imports A:

       A.init() precisa de B estar inicializado
       B.init() precisa de A estar inicializado
         ↓
       deadlock de inicialização — impossível resolver a ordem

  Analogia: deadlock clássico de processos (ciclo no grafo de dependências):
    Processo P1 segura recurso R1, espera R2
    Processo P2 segura recurso R2, espera R1
    → nenhum progride

  Go força a estrutura de dependências a ser um DAG (Directed Acyclic Graph):
  sem ciclos → sempre existe uma ordem topológica válida de inicialização.
```

### Restrição `internal/` — Fronteira Kernel/Userspace

```
No Linux:
  Código userspace NÃO pode chamar funções internas do kernel diretamente
  A fronteira é enforçada pelo hardware (ring 0 vs ring 3)
  Para usar serviços do kernel: APENAS via syscalls (interface pública)

Em Go:
  Pacotes fora do módulo NÃO podem importar internal/
  A fronteira é enforçada pelo go tool em TEMPO DE BUILD
  Para usar funcionalidades internas: apenas via API pública do módulo

  github.com/empresa/projeto/internal/auth
       ↑ acessível APENAS por github.com/empresa/projeto/...
       ↑ qualquer outro módulo → erro de compilação

  Proteção mais fraca que hardware (pode ser burlada com reflect),
  mas suficiente para prevenir uso acidental de APIs privadas.
```

---

## 12. Conexão com Sistemas Operacionais

**Pacote = unidade de compilação; todos os .go do diretório formam um object file → [[Processadores]]**
O `go tool compile` compila todos os arquivos `.go` de um diretório juntos num único arquivo `.a` (archive). Esse arquivo contém seções `.text` (código), `.data` (globals), `.rodata` (constantes) — exatamente as seções de um `.o` produzido por `gcc`. O `go tool link` depois combina esses archives com o runtime Go para produzir o binário final.

**`init()` executa antes de `main()` em ordem topológica — análogo ao `.init_array` do ELF → [[Processadores]]**
Binários ELF têm uma seção `.init_array` com ponteiros de função chamados pelo loader antes de `main()`. Go implementa o mesmo conceito: o runtime chama as funções `init()` de todos os pacotes na ordem correta (dependências primeiro), depois chama `main.main()`. É a mesma sequência de startup de um binário Linux.

**Exportado (maiúscula) vs não-exportado — controle de visibilidade de símbolos → [[Processadores]]**
Em C, a visibilidade de símbolos é controlada pelo linker (palavra-chave `static` torna o símbolo local ao arquivo objeto). Em Go, a fronteira é verificada pelo **compilador** em tempo de compilação, gerando erro explícito se código externo tentar acessar símbolos não-exportados. Mais seguro: falha em compile time, não em link time.

**Importações circulares proibidas — previne deadlocks de inicialização → [[Processos]]**
Se `A` importa `B` e `B` importa `A`, `init()` de `A` precisa de `B` já inicializado, e vice-versa: deadlock clássico, análogo a dois processos esperando recursos um do outro. Go proíbe ciclos no grafo de importações em tempo de compilação, garantindo que sempre exista uma ordenação topológica válida.

**Restrição `internal/` espelha a fronteira kernel/userspace → [[System Calls]]**
Assim como código userspace só acessa o kernel via syscalls (a interface pública), código de outros módulos só acessa um módulo Go via sua API pública. O diretório `internal/` cria uma fronteira enforçada pelo compilador: tentar importar `internal/` de fora do módulo gera erro de build — o equivalente a tentar chamar uma função `static` do kernel diretamente do userspace.