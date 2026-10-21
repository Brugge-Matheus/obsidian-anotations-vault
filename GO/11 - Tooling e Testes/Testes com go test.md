---
tags:
  - go
  - go/tooling
aliases:
  - go test
  - Testes em Go
  - testing
  - Table-driven tests
  - Fuzzing
  - testing/synctest
  - httptest
ordem: 45
modulo: "11 - Tooling e Testes"
---
# Testes com go test

> **Nota 45 da trilha de Go.** Abrimos o módulo de **ferramentas**. Go traz testes **na própria toolchain**: o pacote `testing` e o comando `go test`, sem framework externo. Nesta nota veremos as convenções (arquivos `_test.go`, funções `TestXxx`), a API de `*testing.T`, **testes em tabela** com subtestes, *helpers*, testes "de dentro" e "de fora" do pacote, **exemplos executáveis**, **fuzzing** (que vai achar um bug de UTF-8 sozinho), dublês de teste com interfaces, `httptest`, testes de código **concorrente e com tempo** usando `testing/synctest` (horas de tempo simulado em milissegundos), cobertura, o detector de corridas, o cache de testes e como o `go test` funciona por dentro.
>
> **Pré-requisitos:** [[Interfaces]], [[Criando Pacotes]], [[Goroutines]], [[HTTP (net-http)]].

---

## 🧭 De onde viemos

Várias notas apontaram para cá: [[Structs]] (structs anônimas em tabelas de teste), [[Interfaces]] e [[Composição vs Herança]] (interfaces pequenas facilitam substituir dependências em testes), [[Criando Pacotes]] (testes caixa-branca vs caixa-preta), [[Leitura e Escrita de Arquivos]] (`t.TempDir()` e `fstest.MapFS`), [[Fundamentos de Concorrência]] (`go test -race`), [[Channels]], [[Select]] e [[Padrões de Concorrência]] (testar código com tempo usando `synctest`) e [[HTTP (net-http)]] (`httptest`). Em [[Convenções da linguagem]] vimos `go vet` e em [[Strings em Profundidade]] a diferença entre bytes e runas, que vai causar dois bugs nesta nota.

> 💡 **Teste automatizado:** código que executa outro código com entradas conhecidas e verifica se a saída é a esperada. Roda em segundos, a cada mudança, e impede que um bug corrigido volte (*regressão*).

---

## 📏 1. As convenções

| Regra | Exemplo |
| --- | --- |
| Arquivo termina em `_test.go` | `texto_test.go` (não entra no binário normal) |
| Função começa com `Test` + letra **maiúscula** | `func TestPalindromo(t *testing.T)` |
| Recebe exatamente `*testing.T` | Sem retorno |
| Benchmarks: `BenchmarkXxx(b *testing.B)` | Ver [[Benchmarks]] |
| Fuzz: `FuzzXxx(f *testing.F)` | Seção 6 |
| Exemplos: `ExampleXxx()` | Seção 5 |
| Dados de teste numa pasta `testdata/` | Ignorada pelo `go build` e pelo `./...` |

```bash
go test                      # pacote do diretório atual
go test ./...                # todos os pacotes do módulo
go test -v ./...             # mostra cada teste (e os t.Log)
go test -run 'Palindromo' .  # só testes cujo nome casa com a regex
go test -run 'TestX/caso'    # um subteste específico
go test -count=1 ./...       # ignora o cache
go test -failfast ./...      # para no primeiro teste que falhar
```

O `go test` também roda um **subconjunto do `go vet`** antes dos testes. Desde o Go 1.24 isso inclui o analisador `tests`, que pega erros que fariam o teste **nem rodar** (e, desde o 1.27, o `stdversion`, que acusa uso de APIs mais novas que a versão do `go.mod`). Saída real:

```go
func Testexpira(t *testing.T) {}       // "e" minúsculo: não é reconhecido como teste
func ExampleCache_Limpar() {}          // documenta um método que não existe
```

```
./nomes_test.go:5:6: Testexpira has malformed name: first letter after 'Test' must not be lowercase
./nomes_test.go:7:1: ExampleCache_Limpar refers to unknown field or method: Cache.Limpar
FAIL	exemplo.com/sync [build failed]
```

---

## 🧪 2. A API de `*testing.T`

| Método | Faz |
| --- | --- |
| `t.Error(...)`, `t.Errorf(...)` | Marca falha e **continua** o teste |
| `t.Fatal(...)`, `t.Fatalf(...)` | Marca falha e **para** o teste (chama `runtime.Goexit`) |
| `t.Log(...)`, `t.Logf(...)` | Registra; aparece com `-v` ou se o teste falhar |
| `t.Skip(...)` | Pula o teste (ex.: `if testing.Short() { t.Skip("lento") }`) |
| `t.Helper()` | Marca a função como auxiliar (seção 3.2) |
| `t.Run(nome, f)` | Cria um **subteste** |
| `t.Parallel()` | Permite rodar em paralelo com outros testes paralelos |
| `t.Cleanup(f)` | Registra limpeza; roda ao fim, em ordem **LIFO** |
| `t.TempDir()` | Diretório temporário, apagado ao fim |
| `t.Context()` (1.24+) | Contexto cancelado ao fim do teste, **antes** dos cleanups |
| `t.Chdir(dir)` (1.24+) | Muda o diretório de trabalho só durante o teste |
| `t.Setenv(k, v)` | Variável de ambiente só durante o teste |
| `t.Attr(k, v)` (1.25+) | Emite um atributo no log (`=== ATTR`) |
| `t.Output()` (1.25+) | Um `io.Writer` que escreve no log do teste |
| `t.ArtifactDir()` (1.26+) | Diretório para arquivos gerados pelo teste (mantidos com `-artifacts`) |

Saída real com alguns deles (`go test -v`):

```go
func TestUtilidades(t *testing.T) {
	t.Cleanup(func() { t.Log("cleanup 1 (registrado primeiro, roda por último)") })
	t.Cleanup(func() { t.Log("cleanup 2") })
	t.Attr("issue", "42")
	t.Log("diretório temporário:", t.TempDir() != "")
}
```

```
=== RUN   TestUtilidades
=== ATTR  TestUtilidades issue 42
    cache_test.go:57: diretório temporário: true
    cache_test.go:51: cleanup 2
    cache_test.go:50: cleanup 1 (registrado primeiro, roda por último)
--- PASS: TestUtilidades (0.00s)
```

> ⚠️ `t.Fatal` só pode ser chamado na **goroutine do teste**: ele encerra a goroutine que o chamou (com `Goexit`), não o teste. Numa goroutine disparada pelo teste, use `t.Error` e retorne.

---

## 📋 3. Testes em tabela

### 3.1 O padrão

O jeito idiomático de testar vários casos: uma **tabela** (slice de structs anônimas, ver [[Structs]]) e um laço com `t.Run`, que cria um **subteste** por caso. O pacote de exemplo desta nota tem `Palindromo`, `Reverter` e `Iniciais`:

```go
func TestPalindromo(t *testing.T) {
	casos := []struct {
		nome    string
		entrada string
		quer    bool
	}{
		{"vazia", "", true},
		{"uma letra", "a", true},
		{"simples", "arara", true},
		{"com espaços e maiúsculas", "Socorram me subi no onibus em Marrocos", true},
		{"não palíndromo", "golang", false},
		{"com acento", "ótimo omitó", true},
	}
	for _, c := range casos {
		t.Run(c.nome, func(t *testing.T) {
			if got := Palindromo(c.entrada); got != c.quer {
				t.Errorf("Palindromo(%q) = %v; quer %v", c.entrada, got, c.quer)
			}
		})
	}
}
```

Saída real (`go test -v -run TestPalindromo`) com a primeira versão do código:

```
=== RUN   TestPalindromo
=== RUN   TestPalindromo/vazia
...
=== RUN   TestPalindromo/com_acento
    texto_test.go:21: Palindromo("ótimo omitó") = false; quer true
--- FAIL: TestPalindromo (0.00s)
    --- PASS: TestPalindromo/vazia (0.00s)
    --- PASS: TestPalindromo/uma_letra (0.00s)
    --- PASS: TestPalindromo/simples (0.00s)
    --- PASS: TestPalindromo/com_espaços_e_maiúsculas (0.00s)
    --- PASS: TestPalindromo/não_palíndromo (0.00s)
    --- FAIL: TestPalindromo/com_acento (0.00s)
```

O caso com acento pegou um bug: `Reverter` invertia **bytes**, e o `ó` ocupa 2 bytes em UTF-8, que ficaram trocados (ver [[Strings em Profundidade]]). Vantagens da tabela:

- Adicionar um caso é adicionar **uma linha**
- Cada caso tem nome: `go test -run 'TestPalindromo/com_acento'` roda só ele (espaços viram `_`)
- Um caso que falha não impede os outros de rodar
- A mensagem segue a convenção **`Função(entrada) = obtido; quer esperado`**

### 3.2 Helpers: `t.Helper()`

```go
func TestIniciais(t *testing.T) {
	verificar(t, Iniciais("matheus brugge"), "MB")
	verificar(t, Iniciais("  go   é legal "), "GÉL")     // linha 29
}

func verificar(t *testing.T, got, quer string) {
	t.Helper() // o erro aponta para quem chamou verificar, não para esta linha
	if got != quer {
		t.Errorf("obtido %q; quer %q", got, quer)
	}
}
```

Saída real:

```
--- FAIL: TestIniciais (0.00s)
    texto_test.go:29: obtido "G�L"; quer "GÉL"
```

Graças ao `t.Helper()`, o erro aponta a **linha 29** (o caso que falhou), não a linha do `t.Errorf` dentro do helper. E o bug é o mesmo de antes: `p[:1]` pegava o primeiro **byte** de "é", meio caractere, que vira `�`. A correção usa `utf8.DecodeRuneInString(p)`.

### 3.3 Paralelismo

`t.Parallel()` num teste (ou subteste) o coloca para rodar junto com os outros testes paralelos do pacote, até o limite de `-parallel` (padrão: `GOMAXPROCS`). Desde o Go 1.22, a variável do laço já é nova a cada volta, então não é mais preciso o `c := c` antes do `t.Run` (ver [[Goroutines]]).

---

## 🧱 4. Dentro ou fora do pacote

Como vimos em [[Criando Pacotes]], um arquivo `_test.go` pode declarar:

| Declaração | Enxerga | Uso |
| --- | --- | --- |
| `package texto` (o mesmo) | Tudo, inclusive não exportados | Testes caixa-branca de detalhes internos |
| `package texto_test` | Só a API pública (importa `"exemplo.com/texto"`) | Testes caixa-preta; exemplos; evitar ciclos de importação |

Os dois podem coexistir na mesma pasta.

---

## 📖 5. Exemplos executáveis

Funções `ExampleXxx` são **documentação** (aparecem no `go doc` e no pkg.go.dev ao lado de `Xxx`) **e** testes: a saída é comparada com o comentário `// Output:`. Saída real, antes da correção:

```go
package texto_test

func ExampleIniciais() {
	fmt.Println(texto.Iniciais("go é legal"))
	// Output: GÉL
}
```

```
--- FAIL: ExampleIniciais (0.00s)
got:
G�L
want:
GÉL
```

| Nome | Documenta |
| --- | --- |
| `Example()` | O pacote |
| `ExampleIniciais()` | A função `Iniciais` |
| `ExampleCache_Get()` | O método `Get` do tipo `Cache` |
| `ExampleIniciais_vazia()` | Um segundo exemplo de `Iniciais` (sufixo minúsculo) |

Sem o comentário `// Output:`, o exemplo é compilado mas não executado. Com `// Unordered output:`, a ordem das linhas não importa. O analisador `tests` acusa exemplos de coisas que não existem (seção 1).

---

## 🎲 6. Fuzzing

Testes em tabela verificam os casos que **você** pensou. O **fuzzing** (Go 1.18+) gera entradas **automaticamente**, guiado por cobertura de código, procurando uma que quebre uma **propriedade**.

> 💡 **Fuzzing:** técnica de teste que executa o código com milhões de entradas geradas e mutadas automaticamente, mantendo as que exploram caminhos novos do código ("interessantes"), até encontrar uma que cause falha ou panic.

```go
func FuzzReverter(f *testing.F) {
	for _, semente := range []string{"golang", "arara", " ", "!12345"} {
		f.Add(semente) // corpus inicial
	}
	f.Fuzz(func(t *testing.T, s string) {
		r := Reverter(s)
		if utf8.ValidString(s) && !utf8.ValidString(r) {          // propriedade 1
			t.Errorf("Reverter(%q) = %q: UTF-8 inválido", s, r)
		}
		if Reverter(r) != s {                                     // propriedade 2
			t.Errorf("Reverter(Reverter(%q)) != original", s)
		}
	})
}
```

Saída real com a versão que invertia bytes:

```
$ go test -run '^$' -fuzz=FuzzReverter -fuzztime=30s .
fuzz: elapsed: 0s, gathering baseline coverage: 0/4 completed
fuzz: elapsed: 0s, gathering baseline coverage: 4/4 completed, now fuzzing with 2 workers
fuzz: elapsed: 0s, execs: 670 (30221/sec), new interesting: 5 (total: 9)
--- FAIL: FuzzReverter (0.02s)
    --- FAIL: FuzzReverter (0.00s)
        fuzz_test.go:15: Reverter("Ò") = "\x92\xc3": UTF-8 inválido

    Failing input written to testdata/fuzz/FuzzReverter/78662d16986e277d
    To re-run:
    go test -run=FuzzReverter/78662d16986e277d
```

Em **20 ms** ele achou o bug. A entrada que falhou foi salva em `testdata/fuzz/`, num formato de texto:

```
go test fuzz v1
string("Ò")
```

Corrigindo `Reverter` para inverter **runas** e rodando de novo, o fuzzer achou um **segundo** problema, mais sutil (saída real):

```
fuzz: minimizing 41-byte failing input file
        fuzz_test.go:18: Reverter(Reverter("\xf5")) != original
    Failing input written to testdata/fuzz/FuzzReverter/15bd0ea6a71e1138
```

`"\xf5"` não é UTF-8 válido; ao virar `[]rune`, o byte inválido é trocado por `U+FFFD` (`�`) e a ida e volta não devolve o original. Ele também **minimizou** a entrada (de 41 bytes para 1) antes de reportar. A decisão aqui é de **contrato**: `Reverter` só é definida para UTF-8 válido, então o teste passa a pular essas entradas com `t.Skip`. Depois disso, 325 mil execuções sem falha.

| Detalhe | Comportamento |
| --- | --- |
| Sem `-fuzz` | O `go test` comum roda só as sementes e o corpus de `testdata/fuzz/` como testes normais (regressão garantida) |
| `-fuzz=Nome` | Fuzzing de verdade, num único alvo, em paralelo (um worker por CPU) |
| `-fuzztime=30s` ou `=10000x` | Por quanto tempo / quantas execuções |
| Tipos aceitos | `string`, `[]byte`, inteiros, `float`, `bool`, `rune` |
| Corpus gerado | Em `$GOCACHE/fuzz` (as entradas "interessantes") |

> ✅ Bons alvos de fuzzing: **parsers**, decodificadores, funções de ida e volta (`Decode(Encode(x)) == x`), comparação com uma implementação de referência simples. É assim que a equipe de Go encontra bugs em `encoding/*`, `net/url`, `regexp`...

---

## 🎭 7. Dublês de teste com interfaces

Para testar código que fala com banco, rede ou relógio, substitua a dependência por um **dublê** (*fake*, *stub*, *mock*). Em Go, isso não exige framework: basta o código depender de uma **interface pequena** (ver [[Interfaces]]), definida **por quem usa**:

```go
// no pacote do serviço: a interface descreve só o que o serviço precisa
type RepositorioUsuarios interface {
	Buscar(ctx context.Context, id int) (Usuario, error)
}

type Servico struct{ repo RepositorioUsuarios }

func (s *Servico) Saudacao(ctx context.Context, id int) (string, error) {
	u, err := s.repo.Buscar(ctx, id)
	if err != nil {
		return "", fmt.Errorf("saudação: %w", err)
	}
	return "Olá, " + u.Nome, nil
}

// no teste: um fake em memória
type repoFake map[int]Usuario

func (r repoFake) Buscar(_ context.Context, id int) (Usuario, error) {
	u, ok := r[id]
	if !ok {
		return Usuario{}, ErrNaoEncontrado
	}
	return u, nil
}

func TestSaudacao(t *testing.T) {
	s := &Servico{repo: repoFake{7: {Nome: "Ana"}}}

	got, err := s.Saudacao(t.Context(), 7)
	if err != nil || got != "Olá, Ana" {
		t.Fatalf("Saudacao(7) = %q, %v", got, err)
	}
	if _, err := s.Saudacao(t.Context(), 99); !errors.Is(err, ErrNaoEncontrado) {
		t.Fatalf("Saudacao(99): err = %v; quer ErrNaoEncontrado", err)
	}
}
```

| Tipo de dublê | O que é | Quando |
| --- | --- | --- |
| **Fake** | Implementação simplificada que funciona (map em memória) | O mais útil no dia a dia |
| **Stub** | Devolve respostas fixas | Forçar um caminho de erro |
| **Mock** | Verifica **quais chamadas** aconteceram | Quando a interação é o que importa |

> 💡 Bibliotecas como **`testify`** (`assert`, `require`, `mock`) e geradores como `mockery`/`gomock` são populares, mas opcionais. A biblioteca padrão, mais `t.Errorf` com mensagens claras, cobre a maioria dos casos. A cultura de Go tende a preferir **fakes** a mocks que verificam cada chamada, porque estes acoplam o teste à implementação e quebram a cada refatoração.

Para o sistema de arquivos, `testing/fstest.MapFS` é um fake pronto de `fs.FS` (ver [[Leitura e Escrita de Arquivos]]).

---

## 🌐 8. Testando HTTP

Como vimos em [[HTTP (net-http)]]:

```go
// handler isolado, sem rede
rec := httptest.NewRecorder()
handler.ServeHTTP(rec, httptest.NewRequest("GET", "/usuarios/42", nil))
if rec.Code != http.StatusOK { ... }

// cliente contra um servidor real numa porta local
srv := httptest.NewServer(handler)
defer srv.Close()
resp, err := srv.Client().Get(srv.URL + "/usuarios/42")
```

E o **Go 1.27** trouxe `httptest.NewTestServer(t, handler)`: um servidor numa **rede falsa em memória**, fechado automaticamente no fim do teste, feito para combinar com `synctest` (próxima seção).

---

## ⏱️ 9. Concorrência e tempo: `testing/synctest`

Código com `time.Sleep`, timeouts e goroutines é difícil de testar: ou o teste **espera de verdade** (lento), ou usa pausas curtas e fica **instável** (*flaky*). O pacote `testing/synctest` (estável desde o Go 1.25) resolve isso com uma **bolha**:

| Dentro de `synctest.Test(t, f)` | Efeito |
| --- | --- |
| O pacote `time` usa um **relógio falso** | Começa em 1/1/2000, 00:00 UTC |
| Quando **todas** as goroutines da bolha estão bloqueadas | O relógio **salta** para o próximo timer |
| `synctest.Wait()` | Espera todas as goroutines da bolha bloquearem |
| `synctest.Sleep(d)` (Go 1.27) | `time.Sleep(d)` + `Wait()` |

### 9.1 Uma hora em zero segundos

Um cache com TTL de 1 hora. Saída real:

```go
func TestExpiracao(t *testing.T) {
	inicioReal := time.Now()
	synctest.Test(t, func(t *testing.T) {
		c := Novo(time.Hour)
		c.Set("chave", "valor")

		time.Sleep(59 * time.Minute) // relógio FALSO: avança na hora
		if _, ok := c.Get("chave"); !ok {
			t.Fatal("expirou cedo demais")
		}
		time.Sleep(2 * time.Minute)
		if _, ok := c.Get("chave"); ok {
			t.Fatal("deveria ter expirado")
		}
		t.Log("dentro da bolha, time.Now() =", time.Now().UTC().Format(time.DateTime))
	})
	t.Logf("1h01m de tempo falso em %v de tempo real", time.Since(inicioReal).Round(time.Millisecond))
}
```

```
=== RUN   TestExpiracao
    cache_test.go:28: dentro da bolha, time.Now() = 2000-01-01 01:01:00
    cache_test.go:30: 1h01m de tempo falso em 0s de tempo real
--- PASS: TestExpiracao (0.00s)
```

### 9.2 Timeout de HTTP sem esperar (Go 1.27)

Um backend que trava e um cliente com timeout de 30 s. Saída real:

```go
func TestTimeoutDoCliente(t *testing.T) {
	inicioReal := time.Now()
	synctest.Test(t, func(t *testing.T) {
		srv := httptest.NewTestServer(t, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			select { // um backend que trava (mas respeita o cancelamento)
			case <-time.After(time.Minute):
			case <-r.Context().Done():
			}
		}))
		cliente := srv.Client()
		cliente.Timeout = 30 * time.Second

		_, err := cliente.Get(srv.URL)
		if !errors.Is(err, context.DeadlineExceeded) {
			t.Fatalf("esperava timeout, veio %v", err)
		}
		t.Log("erro recebido:", err)
	})
	t.Logf("timeout de 30s verificado em %v de tempo real", time.Since(inicioReal).Round(time.Millisecond))
}
```

```
=== RUN   TestTimeoutDoCliente
    cache_test.go:49: erro recebido: Get "http://example.com": context deadline exceeded
    cache_test.go:51: timeout de 30s verificado em 1ms de tempo real
--- PASS: TestTimeoutDoCliente (0.00s)
```

Repare que, na rede falsa, o servidor responde em `http://example.com`. A primeira tentativa deste teste, com um handler que fazia só `time.Sleep(time.Minute)` sem olhar `r.Context()`, passou, mas imprimiu `httptest.Server blocked in Close after 5 seconds`: o servidor esperava o handler terminar. O teste revelou um handler que não respeitava cancelamento (ver [[context.Context]]).

> ⚠️ Medir o tempo real **dentro** da bolha não funciona: lá dentro, `time.Now()` também é falso. Por isso o `inicioReal` é lido fora.

---

## 📊 10. Cobertura, corridas, cache e ordem

### 10.1 Cobertura

Saída real, com o pacote corrigido:

```
$ go test -cover .
ok  	exemplo.com/texto	0.004s	coverage: 100.0% of statements

$ go test -coverprofile=c.out . && go tool cover -func=c.out
exemplo.com/texto/texto.go:11:	Reverter	100.0%
exemplo.com/texto/texto.go:20:	Palindromo	100.0%
exemplo.com/texto/texto.go:26:	Iniciais	100.0%
total:				(statements)	100.0%

$ go tool cover -html=c.out        # abre o código colorido (verde/vermelho) no navegador
```

> ⚠️ 100% de cobertura não significa "sem bugs": a primeira versão de `Iniciais` também teria 100% com um teste só de ASCII. Cobertura mostra o que **não** foi testado; não prova que o testado está certo. Desde o Go 1.20, até **binários** podem ser instrumentados (`go build -cover`, com `GOCOVERDIR`) para medir cobertura de testes de integração.

### 10.2 Corridas

`go test -race ./...` compila os testes com o detector de corridas de [[Fundamentos de Concorrência]]: qualquer data race observada durante os testes faz o pacote **falhar**. Junto com `-count=N` (repetir N vezes), aumenta a chance de pegar corridas que dependem de *timing*.

### 10.3 Cache

Saída real de duas execuções seguidas:

```
$ go test .
ok  	exemplo.com/texto	0.003s
$ go test .
ok  	exemplo.com/texto	(cached)
```

O resultado é reaproveitado se o **binário de teste** é o mesmo (nem o código nem as dependências mudaram), se as flags vêm só de um conjunto "cacheável" (`-run`, `-v`, `-short`, `-timeout`, `-cpu`...) e se os **arquivos do módulo** abertos e as **variáveis de ambiente** consultadas pelo teste não mudaram. Só resultados de sucesso são guardados. Para forçar a execução, o idiomático é `-count=1`.

### 10.4 Ordem aleatória

Testes não devem depender da ordem uns dos outros. `-shuffle=on` embaralha e imprime a semente, para reproduzir (saída real):

```
$ go test -shuffle=on -v .
-test.shuffle 1792555239239734698
...
$ go test -shuffle=1792555239239734698 .     # mesma ordem de novo
```

### 10.5 Outras flags úteis

| Flag | Uso |
| --- | --- |
| `-short` | Testes lentos se pulam com `if testing.Short() { t.Skip(...) }` |
| `-timeout 2m` | Mata o binário de teste que travar (padrão: 10 min), imprimindo todas as goroutines |
| `-json` | Saída estruturada para CI |
| `-artifacts` (1.26) | Mantém os arquivos de `t.ArtifactDir()` |
| `-tags integration` | Inclui arquivos com `//go:build integration` (ver [[Build, Flags e Cross-compilation]]) |
| `-cpu 1,2,4` | Roda com vários `GOMAXPROCS` |

---

## 🔬 11. Por dentro: como o `go test` funciona

```
go test ./texto
   │
   ├─ 1. go vet (subconjunto: atomic, bools, buildtag, directive, errorsas,
   │        ifaceassert, nilfunc, printf, slog, stdversion, stringintconv, tests)
   │
   ├─ 2. compila o pacote + os _test.go (package texto)
   │     compila o pacote externo de teste (package texto_test), se houver
   │
   ├─ 3. GERA um main.go:  func main() { m := testing.MainStart(..., tests, benchmarks,
   │                                                         fuzzTargets, examples)
   │                                    os.Exit(m.Run()) }   (ou chama seu TestMain)
   │
   ├─ 4. linka o binário  texto.test   (no cache, ou em -o)
   │
   └─ 5. executa  ./texto.test -test.v -test.run ...   e formata a saída
```

| Consequência | Detalhe |
| --- | --- |
| Um **processo por pacote** | `go test ./...` roda vários binários (em paralelo, até `-p`); dentro de um pacote, todos os testes compartilham variáveis globais |
| Cada teste é uma **goroutine** | `t.Fatal` usa `runtime.Goexit` para encerrar só ela; `t.Parallel` só libera goroutines que esperam |
| `TestMain(m *testing.M)` | Se existir, é chamado no lugar do laço padrão: setup e teardown do pacote, depois `os.Exit(m.Run())` |
| Binário de teste reaproveitável | `go test -c -o texto.test` gera e guarda o binário; útil para rodar em outra máquina ou com `perf`, `strace` |
| Testes são código nativo | O mesmo pipeline de compilação do `go build` (ver [[Como Go Funciona]]) |

> 💡 O fuzzing usa instrumentação de **cobertura** inserida pelo compilador (contadores em cada bloco básico, no estilo do libFuzzer) e roda vários processos *worker* coordenados pelo processo principal, que mantém o corpus.

---

## ⚠️ 12. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `Testxxx` com minúscula após `Test` | O teste nunca roda | O vet (via `go test`) avisa; use maiúscula |
| `t.Fatal` numa goroutine disparada | Não para o teste; comportamento confuso | `t.Error` + `return` na goroutine |
| Helper sem `t.Helper()` | Erro aponta para o helper | `t.Helper()` na primeira linha |
| Testes dependentes de ordem ou de globais | Passam sozinhos, falham juntos | `t.Cleanup`, `-shuffle=on` |
| `time.Sleep` para "esperar" goroutines | Testes lentos e instáveis | `synctest`, channels, WaitGroup |
| Só testar ASCII | Bugs de UTF-8 em produção | Casos com acento, emoji; fuzzing |
| Achar que 100% de cobertura = correto | Falsa segurança | Bons casos e propriedades |
| Mocks verificando cada chamada | Testes quebram a cada refatoração | Fakes com comportamento |
| Rodar sem `-race` no CI | Corridas silenciosas | `go test -race ./...` |
| `os.Exit` sem `m.Run()` no `TestMain` | Nenhum teste roda | `os.Exit(m.Run())` |
| Esperar que `-count=1` seja padrão | Resultado "(cached)" de um teste que lê algo externo | `-count=1` para testes de integração |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Convenções | `_test.go`, `TestXxx(t *testing.T)`, pasta `testdata/` |
| `go test` | Roda vet + compila um binário de teste por pacote; cache de resultados |
| `Error` vs `Fatal` | Continua vs para (com `Goexit`, só na goroutine do teste) |
| Tabela | Slice de casos + `t.Run`: subtestes nomeados e filtráveis |
| `t.Helper` | Erros apontam para quem chamou |
| `t.Context`, `Chdir`, `Attr`, `Output`, `ArtifactDir` | Novidades de 1.24–1.26 |
| Exemplos | Documentação verificada por `// Output:` |
| Fuzzing | Propriedades + entradas geradas; falhas salvas em `testdata/fuzz` |
| Dublês | Interfaces pequenas + fakes; frameworks opcionais |
| `httptest` | `NewRecorder`, `NewServer`, `NewTestServer` (1.27, rede falsa) |
| `synctest` | Relógio falso numa bolha; horas em milissegundos |
| Cobertura | `-cover`, `-coverprofile`, `go tool cover`; não prova correção |
| Outras | `-race`, `-shuffle`, `-short`, `-timeout`, `-json` |

---

## 🔗 Conexões

**Dentro de Go**
- Medir desempenho com o mesmo pacote `testing` → [[Benchmarks]]
- Profiles a partir de testes e benchmarks → [[Profiling e Diagnóstico]]
- Build tags para testes de integração → [[Build, Flags e Cross-compilation]]
- Interfaces pequenas e definidas pelo consumidor → [[Interfaces]], [[Composição vs Herança]]
- Testes caixa-branca e caixa-preta → [[Criando Pacotes]]
- Structs anônimas nas tabelas → [[Structs]]
- O bug de bytes vs runas → [[Strings em Profundidade]]
- `errors.Is` nas verificações → [[Wrapping e Unwrapping de Erros]]
- `t.TempDir`, `fstest.MapFS` → [[Leitura e Escrita de Arquivos]]
- `-race` e data races → [[Fundamentos de Concorrência]]
- Goroutines, timers e cancelamento testados com `synctest` → [[Goroutines]], [[Select]], [[context.Context]], [[Padrões de Concorrência]]
- `httptest` → [[HTTP (net-http)]]
- `go vet` e convenções → [[Convenções da linguagem]]

**Sistemas Operacionais**
- [[Processos]] — um processo de teste por pacote; estado global compartilhado dentro dele
- [[Término de Processos]] — o código de saída do binário de teste decide se o CI passa
- [[Race Condition]] — o que `-race` e `-count` procuram
- [[Implementando Threads em User Space]] — cada teste roda numa goroutine

---

⬅️ **Anterior:** [[HTTP (net-http)]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Benchmarks]] ➡️
