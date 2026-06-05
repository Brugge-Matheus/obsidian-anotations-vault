---
tags:
  - go
  - go/tooling
---
# Testes com go test

> Go tem suporte a testes embutido na toolchain. O comando `go test` compila um binário de teste, executa e reporta os resultados. Sem frameworks externos necessários para o básico — mas `testify` é amplamente adotado pela comunidade.
> 

---

## 1. Estrutura Básica e Descoberta

O `go test` descobre testes automaticamente por convenção:

```go
Regras:
- Arquivo de teste: termina em _test.go
- Função de teste: começa com Test (maiúsculo)
- Recebe: *testing.T (não pode ter outros parâmetros)
- Mesmo pacote: package X (testa internals) OU package X_test (testa API pública)
```

```go
// arquivo: mathutil/soma_test.go
package mathutil   // pode usar mathutil_test para testar apenas a API pública

import "testing"

func TestSomar(t *testing.T) {
	resultado := Somar(3, 4)
	if resultado != 7 {
		t.Errorf("Somar(3, 4) = %d; esperado 7", resultado)
	}
}
```

```bash
go test ./...                    # todos os pacotes
go test ./mathutil/...           # pacote específico
go test -v ./...                 # verbose — mostra cada teste
go test -run TestSomar ./...     # por nome (regex)
go test -run TestSomar/positivo  # subteste específico
go test -count=1 ./...           # desabilita cache
go test -timeout 60s ./...       # timeout por pacote
```

---

## 2. API de `testing.T`

```go
func TestExemplo(t *testing.T) {
	// Reportar falha sem parar
	t.Error("mensagem")
	t.Errorf("esperado %d, got %d", want, got)

	// Reportar falha e parar o teste IMEDIATAMENTE
	// (os testes abaixo desta linha não executam)
	t.Fatal("falha crítica")
	t.Fatalf("fatal: %v", err)

	// Log — aparece apenas com -v ou quando o teste falha
	t.Log("info de debug")
	t.Logf("estado: %+v", obj)

	// Pular
	if testing.Short() {
		t.Skip("pulando teste longo — use go test sem -short")
	}
	t.Skipf("pulando: %s", motivo)

	// Marcar como helper — o relatório de erro aponta para quem chamou o helper
	// (não para o helper em si)
	t.Helper()

	// Subteste
	t.Run("subcase", func(t *testing.T) {
		// t aqui é um T diferente, para o subteste
	})

	// Cleanup — executado após o teste terminar (incluindo falha)
	t.Cleanup(func() {
		// limpeza
	})

	// TempDir — cria diretório temporário, removido após o teste
	dir := t.TempDir()
}
```

---

## 3. Table-Driven Tests — O Padrão Go

O padrão mais idiomático — evita duplicação e torna adição de casos trivial:

```go
func TestDividir(t *testing.T) {
	casos := []struct {
		nome     string
		a, b     float64
		esperado float64
		errEsp   bool   // espera erro?
	}{
		{"divisão simples", 10, 2, 5, false},
		{"divisão por zero", 10, 0, 0, true},
		{"negativo", -6, 2, -3, false},
		{"fracionado", 7, 2, 3.5, false},
	}

	for _, tc := range casos {
		// t.Run cria subtestes — permitem -run específico e relatório granular
		t.Run(tc.nome, func(t *testing.T) {
			t.Parallel()   // subtestes podem rodar em paralelo

			resultado, err := Dividir(tc.a, tc.b)

			if tc.errEsp {
				if err == nil {
					t.Error("esperava erro, não recebeu nenhum")
				}
				return   // erro esperado — não verifica resultado
			}

			if err != nil {
				t.Fatalf("erro inesperado: %v", err)
			}

			if resultado != tc.esperado {
				t.Errorf("Dividir(%.1f, %.1f) = %.1f; esperado %.1f",
					tc.a, tc.b, resultado, tc.esperado)
			}
		})
	}
}
```

```bash
# Executar subtestes
go test -run TestDividir/fracionado    # exato
go test -run TestDividir/div.*         # regex
go test -run TestDividir               # todos os subtestes
```

---

## 4. `testify` — Asserções e Mocks

```bash
go get github.com/stretchr/testify
```

```go
import (
	"testing"
	"github.com/stretchr/testify/assert"   // continua após falha
	"github.com/stretchr/testify/require"  // para o teste imediatamente
)

func TestUsuario(t *testing.T) {
	u, err := NovoUsuario("Alice", "alice@ex.com")

	// require — quando o teste não faz sentido sem essa condição
	require.NoError(t, err, "NovoUsuario não deve retornar erro")
	require.NotNil(t, u)

	// assert — verificações independentes
	assert.Equal(t, "Alice", u.Nome)
	assert.Equal(t, "alice@ex.com", u.Email)
	assert.True(t, u.Ativo)
	assert.WithinDuration(t, time.Now(), u.CreatedAt, time.Second)

	// Erros
	_, err2 := NovoUsuario("", "")
	assert.Error(t, err2)
	assert.ErrorIs(t, err2, ErrDadosInvalidos)

	// Coleções
	assert.Len(t, u.Tags, 0)
	assert.Contains(t, "Alice", "lic")
	assert.ElementsMatch(t, []int{3, 1, 2}, []int{1, 2, 3})   // ordem não importa

	// Zero values
	assert.Zero(t, u.Pontos)
	assert.Empty(t, u.Bio)
}
```

---

## 5. Mocks com `testify/mock`

```go
import "github.com/stretchr/testify/mock"

// Interface a ser mockada
type RepositorioUsuario interface {
	BuscarPorID(id int) (*Usuario, error)
	Salvar(*Usuario) error
}

// Mock gerado manualmente (ou via mockery: go install github.com/vektra/mockery/v2@latest)
type MockRepositorio struct {
	mock.Mock
}

func (m *MockRepositorio) BuscarPorID(id int) (*Usuario, error) {
	args := m.Called(id)
	// args.Get(0) retorna interface{} — precisa de type assertion
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*Usuario), args.Error(1)
}

func (m *MockRepositorio) Salvar(u *Usuario) error {
	args := m.Called(u)
	return args.Error(0)
}

// Teste usando o mock
func TestBuscarUsuario(t *testing.T) {
	repo := new(MockRepositorio)

	// Configurar comportamento esperado
	repo.On("BuscarPorID", 42).Return(&Usuario{ID: 42, Nome: "Alice"}, nil)
	repo.On("BuscarPorID", 99).Return(nil, ErrNaoEncontrado)

	svc := NovoServico(repo)

	// Caso de sucesso
	u, err := svc.Buscar(42)
	require.NoError(t, err)
	assert.Equal(t, "Alice", u.Nome)

	// Caso de erro
	_, err = svc.Buscar(99)
	assert.ErrorIs(t, err, ErrNaoEncontrado)

	// Verificar que as expectativas foram cumpridas
	repo.AssertExpectations(t)
}
```

---

## 6. Testes de HTTP com `httptest`

```go
import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestCriarUsuarioHandler(t *testing.T) {
	// Opção 1: testar o handler diretamente (sem servidor real)
	body := strings.NewReader(`{"nome":"Alice","email":"alice@ex.com"}`)
	req := httptest.NewRequest(http.MethodPost, "/usuarios", body)
	req.Header.Set("Content-Type", "application/json")

	w := httptest.NewRecorder()
	handler := NovoHandler(mockRepo)
	handler.ServeHTTP(w, req)

	resp := w.Result()
	assert.Equal(t, http.StatusCreated, resp.StatusCode)

	var u Usuario
	json.NewDecoder(resp.Body).Decode(&u)
	assert.Equal(t, "Alice", u.Nome)

	// Opção 2: servidor real de teste (para testes de integração)
	servidor := httptest.NewServer(handler)
	defer servidor.Close()

	resp2, err := http.Post(servidor.URL+"/usuarios", "application/json",
		strings.NewReader(`{"nome":"Bob"}`))
	require.NoError(t, err)
	defer resp2.Body.Close()
	assert.Equal(t, http.StatusCreated, resp2.StatusCode)

	// Para TLS:
	// servidor := httptest.NewTLSServer(handler)
	// client := servidor.Client()   // cliente pré-configurado para o TLS do teste
}
```

---

## 7. Setup e Teardown

### `TestMain` — Controle do Pacote Inteiro

```go
// Executa antes/depois de todos os testes do pacote
func TestMain(m *testing.M) {
	// Setup
	db = iniciarBancoDeTeste()
	carregarFixtures(db)

	// Executar testes — m.Run() retorna o código de saída
	codigo := m.Run()

	// Teardown
	db.Close()
	os.RemoveAll("./testdata/temp/")

	os.Exit(codigo)   // importante: preserva o código de saída
}
```

### `t.Cleanup` — Cleanup por Teste

```go
func criarUsuarioDeTeste(t *testing.T, db *sql.DB) *Usuario {
	t.Helper()

	u := &Usuario{Nome: "Teste", Email: fmt.Sprintf("teste%d@ex.com", rand.Int())}
	require.NoError(t, db.Salvar(u))

	// Registrar limpeza — executada quando o teste terminar
	t.Cleanup(func() {
		db.Deletar(u.ID)
	})

	return u
}
```

---

## 8. Testes em Paralelo

```go
func TestParalelo(t *testing.T) {
	t.Parallel()   // este teste pode rodar junto com outros t.Parallel()

	// Ação que pode ser compartilhada com segurança
}

func TestTableParalelo(t *testing.T) {
	casos := []struct{ n int }{{1}, {2}, {3}}

	for _, tc := range casos {
		tc := tc   // captura correta (necessário antes do Go 1.22)
		t.Run(fmt.Sprintf("n=%d", tc.n), func(t *testing.T) {
			t.Parallel()   // subtestes também podem ser paralelos
			// ...
		})
	}
}
```

---

## 9. Flags Importantes

```bash
# Cobertura de código
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out      # visualizar no browser
go tool cover -func=coverage.out      # por função no terminal

# Race detector
go test -race ./...

# Testes curtos (pular testes lentos)
go test -short ./...   # dentro do teste: if testing.Short() { t.Skip() }

# Paralelismo
go test -parallel 8 ./...   # máximo 8 testes paralelos por vez

# Sem cache
go test -count=1 ./...

# Verbose
go test -v ./...
```

---

## 10. Exemplos — Documentação Executável

```go
// Aparecem na documentação E são executados como testes
func ExampleSomar() {
	fmt.Println(Somar(3, 4))
	// Output:
	// 7
}

func ExampleSomar_decimais() {   // sufixo após _ para múltiplos exemplos
	fmt.Println(Somar(-1, 1))
	// Output:
	// 0
}

// Sem output — executa mas não verifica a saída
func ExampleHandler() {
	// mostra como usar o handler sem verificar saída
}
```

---

## 11. Integração com `go test` — Flags Customizadas

```go
// Flags próprias para seus testes
var (
	atualizar   = flag.Bool("update", false, "atualizar golden files")
	integração  = flag.Bool("integration", false, "executar testes de integração")
	dbURL       = flag.String("db", "", "URL do banco para testes de integração")
)

func TestMain(m *testing.M) {
	flag.Parse()   // necessário para que as flags sejam parseadas
	os.Exit(m.Run())
}

func TestGolden(t *testing.T) {
	saida := gerarRelatorio()
	golden := "testdata/relatorio.golden"

	if *atualizar {
		os.WriteFile(golden, []byte(saida), 0644)
		return
	}

	esperado, _ := os.ReadFile(golden)
	assert.Equal(t, string(esperado), saida)
}
```

```bash
go test -run TestGolden -update      # atualiza os arquivos golden
go test -run TestInteg -integration -db postgres://... ./...
```

---

## 12. Por Dentro: Como `go test` Funciona

### Pipeline de compilação do teste

```
go test ./mathutil/...
       │
       ▼
compilador Go (gc) compila:
  - todos os arquivos .go do pacote
  - todos os arquivos _test.go do pacote
       │
       ▼
linker produz binário de teste: mathutil.test
  (ELF no Linux, Mach-O no macOS, PE no Windows)
       │
       ▼
./mathutil.test -test.v -test.run TestSomar
       │
       ▼
main() gerado automaticamente:
  1. chama TestMain (se existir)
  2. descobre funções Test* via tabela de símbolos
  3. executa cada TestXxx em ordem
  4. reporta resultados para stdout
       │
       ▼
go test coleta stdout e formata a saída
```

O binário de teste usa o **mesmo pipeline de compilação** que `go build` — SSA, escape analysis, inlining. Testes rodados com `-race` passam por uma fase adicional de instrumentação antes de gerar o binário. Conecta com [[Processadores]] (compilação para código nativo, ELF binary).

### Modelo de isolamento: 1 processo por pacote, não por teste

```
Diferença de isolamento entre frameworks:

Java (JUnit):  1 JVM por suíte, instância nova por teste
Go (go test):  1 processo por PACOTE, todos os testes compartilham memória

Implicação:
  - Estado global (vars de pacote) é compartilhado entre TestA e TestB
  - Ordem de execução afeta resultados se houver estado mútável
  - t.Cleanup() e defer garantem limpeza por teste
  - TestMain permite setup/teardown por pacote inteiro
```

Isso conecta com [[Processos]] — cada invocação de `go test ./pacote` cria um novo processo OS. Testes do mesmo pacote correm no mesmo processo, compartilhando heap e globals.

### Race detector: shadow memory e instrumentação

```
go test -race:
                    código original        código instrumentado
                    ─────────────          ───────────────────
                    x = 42                 shadow[&x] = {goroutine, clock}
                                           x = 42
                    
                    y = x                  check(shadow[&x], current_goroutine)
                                           y = x

Shadow memory layout:
  Para cada 8 bytes de memória real → 4 bytes de shadow memory
  Overhead de memória: ~50-100% extra
  
  Shadow byte guarda:
    - qual goroutine fez o último acesso
    - se foi read ou write
    - logical clock (para detectar happens-before violations)

  Se goroutine A escreveu sem lock,
  e goroutine B lê sem lock e sem happens-before:
  → race detector dispara: "DATA RACE"
```

```bash
# Overhead do race detector:
# Tempo de execução: ~5-20x mais lento
# Memória:          ~50-100% mais uso
# Por isso: use em CI, não em produção

go test -race ./...   # instrumenta e detecta races
```

Conecta com [[Gerenciamento de Memória]] (shadow memory — byte extra por byte real de memória para rastrear acessos) e [[Threads POSIX]] (race conditions, data races, happens-before).

### `-count=N`: detectando flakiness por race conditions

```go
// Teste flaky (intermitente) — causa comum: race condition de timing

var contador int   // global compartilhado — BUG: sem proteção

func TestContador(t *testing.T) {
    go func() { contador++ }()
    time.Sleep(1 * time.Millisecond)   // heurística frágil
    if contador != 1 {
        t.Error("esperado 1")   // falha 10-30% das vezes
    }
}

// go test -count=10 executa o teste 10 vezes consecutivas
// Se falhar em algumas rodadas → teste é flaky
// go test -race -count=10 → detecta a race condition explicitamente
```

Conecta com [[Threads POSIX]] — race conditions são sensíveis ao timing; rodar N vezes aumenta probabilidade de manifestação.

### Testes paralelos: goroutines no mesmo processo

```
t.Parallel() marca o teste para execução concorrente:

Sequência de execução com t.Parallel():

1. go test inicia TestA → t.Parallel() → pausa TestA, coloca na fila paralela
2. go test inicia TestB → t.Parallel() → pausa TestB, coloca na fila paralela
3. go test inicia TestC → t.Parallel() → pausa TestC, coloca na fila paralela
4. Todos os testes sequenciais terminam
5. Testes paralelos são desbloqueados e executam concorrentemente
   └── goroutines do runtime Go, não threads OS separadas

go test -parallel 4  →  no máximo 4 testes paralelos simultaneamente
                         runtime scheduler → M:N sobre threads OS
```

Conecta com [[Processos]] (mesmo processo, goroutines compartilham heap) e [[Implementando Threads em User Space]] (scheduler de goroutines gerencia paralelismo dos testes, não o kernel).

---

## 13. Conexão com Sistemas Operacionais

- **[[Processadores]]**: `go test` compila `_test.go` através do mesmo pipeline que `go build` — parsing, type check, SSA IR, otimizações, geração de código nativo, linking. O binário resultante é ELF (Linux), Mach-O (macOS) ou PE (Windows) — código nativo puro, sem interpretação.

- **[[Processos]]**: Cada `go test ./pacote` cria um processo OS separado para executar os testes daquele pacote. Dentro do processo, todos os `TestXxx` compartilham o mesmo espaço de memória. Estado global mútável entre testes é um bug clássico — `t.Cleanup()` e `TestMain` existem para gerenciar este ciclo de vida.

- **[[Gerenciamento de Memória]]**: O race detector (`-race`) usa **shadow memory** — para cada byte de memória real, aloca bytes extras de shadow para rastrear qual goroutine acessou, quando e de que forma (read/write). Overhead de ~5-20x no tempo e ~50-100% na memória. Abordagem similar ao AddressSanitizer do LLVM.

- **[[Threads POSIX]]**: `-count=N` e `-race` são ferramentas para detectar race conditions — bugs que dependem de timing não determinístico entre goroutines concorrentes. Race conditions são o problema central de programação concorrente com memória compartilhada.

- **[[Implementando Threads em User Space]]**: `t.Parallel()` usa goroutines (threads de usuário) para rodar testes concorrentemente. O scheduler Go multiplexia sobre threads OS — o kernel não sabe que há "testes paralelos", apenas vê goroutines no pool de threads Go.