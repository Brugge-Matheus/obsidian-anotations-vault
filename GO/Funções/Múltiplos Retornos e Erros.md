---
tags:
  - go
  - go/funções
---
# Múltiplos Retornos e Erros

> Múltiplos retornos são o mecanismo central de tratamento de erros em Go. Em vez de exceções que propagam implicitamente, Go usa retornos explícitos — cada falha é visível no código e impossível de ignorar sem intenção deliberada.
> 

---

## 1. Por Que Go Escolheu Múltiplos Retornos

```go
Linguagens com exceções (PHP, Java, Python):
try {
    resultado = dividir(a, b);
} catch (DivisaoException e) {
    // esqueceu de tratar → propagação silenciosa
}
// Qualquer função pode lançar exceção → difícil rastrear fluxo de erros

Go com múltiplos retornos:
resultado, err := dividir(a, b)
if err != nil {
    // compilador força receber err (declared and not used → erro)
    // você não tem como "esquecer" o erro inadvertidamente
}
```

---

## 2. Como o Compilador Implementa Múltiplos Retornos

```go
Go 1.17+ usa register-based ABI no AMD64:
- Até 9 retornos em registradores: AX, BX, CX, DI, SI, R8, R9, R10, R11
- Tipos maiores (structs, slices) → stack
- Interface{} → 2 registradores (tipo + valor)

func dividir(a, b float64) (float64, error):
  - resultado float64 → registrador X0 (float)
  - error interface → registrador AX (ponteiro para tipo) + BX (ponteiro para valor)
  - nil error → AX=0, BX=0

Não há overhead de "tuple" — os valores são retornos independentes.
```

---

## 3. Padrão Fundamental `(valor, error)`

```go
// Declaração — error é SEMPRE o último retorno
func dividir(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("divisão por zero")
	}
	return a / b, nil
}

// Consumo — verificar imediatamente
resultado, err := dividir(10, 3)
if err != nil {
	// Trate o erro aqui — log, return, ou fallback
	log.Fatal(err)
}
fmt.Printf("%.4f\n", resultado)   // 3.3333

// Propagação com contexto adicional
func calcularMedia(valores []float64) (float64, error) {
	if len(valores) == 0 {
		return 0, errors.New("calcularMedia: lista vazia")
	}
	soma := 0.0
	for _, v := range valores {
		soma += v
	}
	return soma / float64(len(valores)), nil
}
```

---

## 4. Retornos Nomeados

```go
// Nomes nos retornos — documentam o significado de cada valor
func buscarUsuario(id int) (usuario *Usuario, encontrado bool, err error) {
	// As variáveis são declaradas automaticamente (zero values)
	// usuario = nil, encontrado = false, err = nil

	row := db.QueryRow("SELECT * FROM users WHERE id=$1", id)
	if scanErr := row.Scan(&usuario.ID, &usuario.Nome); scanErr != nil {
		if errors.Is(scanErr, sql.ErrNoRows) {
			return nil, false, nil   // não encontrado — sem erro de sistema
		}
		err = fmt.Errorf("buscarUsuario(%d): %w", id, scanErr)
		return   // naked return — retorna usuario=nil, encontrado=false, err=...
	}

	return usuario, true, nil
}

// Uso mais poderoso: defer que modifica o retorno de erro
func executarTransacao(db *sql.DB, fn func(*sql.Tx) error) (err error) {
	tx, txErr := db.Begin()
	if txErr != nil {
		return fmt.Errorf("iniciar transação: %w", txErr)
	}

	defer func() {
		if err != nil {
			// err é o retorno nomeado — o defer pode modificá-lo!
			if rbErr := tx.Rollback(); rbErr != nil {
				err = fmt.Errorf("rollback falhou (%v) após: %w", rbErr, err)
			}
		} else {
			err = tx.Commit()   // sobrescreve err com resultado do commit
		}
	}()

	return fn(tx)   // err = resultado de fn(tx)
}
```

> ⚠️ "Naked return" (return sem argumentos) só é recomendado em funções muito curtas (< 5 linhas). Em funções longas, dificulta a leitura — é impossível saber o que está sendo retornado sem ler toda a função.
> 

---

## 5. Ignorar Retornos com `_`

```go
// ✅ Apenas quando você TEM CERTEZA que o erro não pode ocorrer
n, _ := strconv.Atoi("42")   // "42" é sempre um int válido

// ✅ Quando apenas um dos retornos interessa
_, err := fmt.Fprintf(os.Stderr, "erro: %v\n", err)

// ❌ NUNCA ignore erros de operações que podem falhar
f, _ := os.Open("config.json")   // se falhar, f=nil → panic na próxima linha!
defer f.Close()                   // panic: nil pointer dereference

// ✅ Sempre verifique
f, err := os.Open("config.json")
if err != nil {
	return fmt.Errorf("abrir config: %w", err)
}
defer f.Close()
```

---

## 6. Propagação de Erros — Adicionar Contexto

Cada camada adiciona contexto sem perder a causa raiz usando `%w`:

```go
// Camada mais baixa — erro bruto
func lerBytesArquivo(path string) ([]byte, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("lerBytesArquivo(%q): %w", path, err)
	}
	return data, nil
}

// Camada intermediária
func carregarConfig(path string) (*Config, error) {
	data, err := lerBytesArquivo(path)
	if err != nil {
		return nil, fmt.Errorf("carregarConfig: %w", err)
	}
	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		return nil, fmt.Errorf("carregarConfig: JSON inválido: %w", err)
	}
	return &cfg, nil
}

// A mensagem de erro resultante é autoexplicativa:
// "carregarConfig: lerBytesArquivo("config.json"): open config.json: no such file or directory"
```

---

## 7. `errors.Is` e `errors.As` — Verificar na Chain

```go
var ErrNaoEncontrado = errors.New("não encontrado")

err := fmt.Errorf("buscarUsuario(42): %w", ErrNaoEncontrado)

// errors.Is percorre toda a chain de wrapping
if errors.Is(err, ErrNaoEncontrado) {
	http.Error(w, "usuário não existe", http.StatusNotFound)
	return
}

// errors.As extrai o tipo concreto
type ErrValidacao struct{ Campo, Msg string }
func (e *ErrValidacao) Error() string { return fmt.Sprintf("%s: %s", e.Campo, e.Msg) }

err2 := fmt.Errorf("cadastro: %w", &ErrValidacao{"email", "formato inválido"})

var valErr *ErrValidacao
if errors.As(err2, &valErr) {
	fmt.Printf("Campo %q: %s\n", valErr.Campo, valErr.Msg)
}
```

---

## 8. Múltiplos Retornos Além de Error

### Retorno com Flag Booleana

```go
// Use (valor, bool) quando "não encontrado" é estado normal (não erro de sistema)
func buscarNoCache(chave string) (string, bool) {
	v, ok := cache[chave]
	return v, ok
}

if valor, ok := buscarNoCache("user:42"); ok {
	usar(valor)
} else {
	// buscar na fonte primária
}
```

### Múltiplos Valores de Dados

```go
func calcularEstatisticas(nums []float64) (media, desvio float64, err error) {
	if len(nums) == 0 {
		return 0, 0, errors.New("slice vazio")
	}

	for _, n := range nums {
		media += n
	}
	media /= float64(len(nums))

	for _, n := range nums {
		d := n - media
		desvio += d * d
	}
	desvio = math.Sqrt(desvio / float64(len(nums)))

	return media, desvio, nil
}

m, d, err := calcularEstatisticas([]float64{2, 4, 4, 4, 5, 5, 7, 9})
// m = 5.0, d = 2.0
```

---

## 9. Erros Sentinel — Boas Práticas

```go
// Definir erros sentinel exportados — comparáveis com errors.Is
var (
	ErrNaoEncontrado   = errors.New("não encontrado")
	ErrJaExiste        = errors.New("já existe")
	ErrPermissaoNegada = errors.New("permissão negada")
	ErrEntradaInvalida = errors.New("entrada inválida")
)

// Handler HTTP que converte errors → HTTP status
func tratarErroHTTP(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, ErrNaoEncontrado):
		http.Error(w, err.Error(), http.StatusNotFound)         // 404
	case errors.Is(err, ErrPermissaoNegada):
		http.Error(w, "sem permissão", http.StatusForbidden)    // 403
	case errors.Is(err, ErrEntradaInvalida):
		http.Error(w, err.Error(), http.StatusBadRequest)       // 400
	default:
		slog.Error("erro interno", "err", err)
		http.Error(w, "erro interno", http.StatusInternalServerError)  // 500
	}
}
```

---

## 10. Padrões de Retorno — Resumo

```go
// 1. Operação sem valor útil — apenas pode falhar
func deletar(id int) error

// 2. Operação que produz valor — pode falhar
func calcular(x int) (int, error)
func buscarUsuario(id int) (*Usuario, error)

// 3. Busca que pode não encontrar — "ausência" não é erro de sistema
func buscarNoCache(chave string) (string, bool)

// 4. Múltiplos resultados
func minMax(nums []int) (min, max int, err error)

// 5. Retornos nomeados para documentação
func parsearData(s string) (data time.Time, fuso *time.Location, err error)

// Convenção de propagação:
return fmt.Errorf("nomeDaFuncao(%v): %w", param, err)
// Resulta em: "camada3: camada2: camada1: erro raiz"
```

---

## 11. Conexão com Sistemas Operacionais

### Retornos Múltiplos e a ABI — Onde os Valores Ficam

Assim como argumentos de entrada, os valores de retorno seguem a **calling convention** da plataforma. O Go usa sua própria ABI register-based desde a versão 1.17:

```
func dividir(a, b float64) (float64, error)

Go AMD64 ABI — o que acontece nos registradores:

  Retorno (float64):
    X0  ← valor float (registrador SSE)

  Retorno (error interface):
    AX  ← ponteiro para o tipo  (itab — tabela de métodos da interface)
    BX  ← ponteiro para o valor (ou nil)

  Quando error = nil:
    AX = 0   (nil pointer)
    BX = 0

Antes do Go 1.17 — tudo na stack:
  ┌──────────────────┐ ← SP
  │   arg b (float)  │
  │   arg a (float)  │
  │   return addr    │
  │ ret0 (float64)   │ ← escrito pela função chamada
  │ ret1.type (err)  │
  │ ret1.val  (err)  │
  └──────────────────┘
```

Não existe uma "tupla" em Go — são apenas múltiplos valores independentes colocados em registradores (ou stack). Ver [[Processadores]].

---

### Retornos Nomeados — Stack Frame com Nomes

Quando você usa retornos nomeados, o compilador **reserva espaço nomeado no stack frame** para esses valores desde o início da função:

```
func calcularEstatisticas(nums []float64) (media, desvio float64, err error)

Stack frame de calcularEstatisticas:
┌─────────────────────────┐ ← frame base
│  return address         │
│  saved RBP              │
├─────────────────────────┤
│  media   float64 = 0.0  │ ← alocado automaticamente (zero value)
│  desvio  float64 = 0.0  │ ← alocado automaticamente (zero value)
│  err     error   = nil  │ ← alocado automaticamente (zero value)
├─────────────────────────┤
│  variáveis locais...    │
└─────────────────────────┘ ← SP

Quando defer modifica err:
  o defer tem um ponteiro para o slot "err" no frame
  → modifica o valor diretamente antes do RET
```

Essa estrutura é o **stack frame** descrito em [[Processos]] — cada chamada de função empilha um frame com seus dados locais.

---

### Go error vs Exceções — Sem Stack Unwinding

A escolha de Go por `error` como valor tem um impacto enorme de performance em comparação com exceções de C++/Java:

```
Mecanismo de exceções (C++, Java):

  throw SomeException("falhou"):
    1. Cria objeto de exceção na heap
    2. Inicia "stack unwinding": percorre todos os frames
       na call stack, executando destructors/finally
    3. Procura um catch handler compatível
    4. Desfaz todos os frames intermediários
    
    Custo: proporcional à profundidade da stack
    Overhead: dezenas de microsegundos para stacks profundas

Go com error como valor:

  return 0, errors.New("falhou"):
    1. Coloca o valor de erro nos registradores de retorno
    2. Executa RET — stack não precisa ser desfeita
    3. O caller verifica imediatamente: if err != nil
    
    Custo: uma instrução RET + comparação com nil
    Overhead: nanosegundos
```

No nível de hardware, exceções C++ usam tabelas de "unwind info" no binário (seção `.eh_frame`) que o runtime percorre. Go não tem esse mecanismo para o fluxo normal — apenas `panic/recover` usa algo similar. Ver [[Processadores]].

---

### `errno` em C vs `error` em Go — A Lição das System Calls

O padrão de `(valor, error)` em Go é diretamente inspirado no problema do `errno` em C ao fazer system calls:

```c
// C — problema com errno:
// errno é uma variável GLOBAL (ou thread-local em POSIX)

ssize_t n = read(fd, buf, size);
if (n == -1) {
    // errno foi setado pela syscall — mas pode ter sido sobrescrito
    // por outra chamada entre a syscall e este if!
    perror("read failed");   // usa errno
}

// Em ambiente multi-threaded:
// Thread 1 faz read() → errno = EAGAIN
// Thread 2 faz write() → errno = EPIPE   (sobrescreve!)
// Thread 1 lê errno → vê EPIPE (errado!)
// POSIX resolve com errno thread-local (__errno_location())
// mas ainda é uma variável implícita e global-por-thread
```

```go
// Go — erro como valor explícito:
// A syscall retorna (resultado, errno) — Go encapsula isso:

n, err := syscall.Read(fd, buf)
if err != nil {
    // err é um valor local, não global
    // impossível ser sobrescrito por outra goroutine
}

// No pacote syscall, isso é implementado como:
func Read(fd int, p []byte) (n int, err error) {
    n, errno := read(fd, p)          // chamada ao OS
    if errno != 0 {
        err = errno                  // errno → error interface
    }
    return
}
```

O padrão `(resultado, errno)` das syscalls do Unix é exatamente o modelo que Go adotou para todo o código. Ver [[System Calls]] para entender como o kernel retorna erros via errno para o espaço de usuário.

---

### O Fluxo Completo: Syscall → errno → Go error

```
Programa Go chama os.ReadFile("config.json"):

  os.ReadFile
    └─ os.Open
         └─ syscall.Open           ← Go runtime
              └─ SYSCALL openat    ← instrução de CPU (modo kernel)
                   └─ kernel VFS   ← verifica permissões, encontra inode
                        └─ retorna -1 (erro) + errno = ENOENT (2)
              ← volta ao user space: AX=-1, errno=ENOENT
         └─ if errno != 0: return &PathError{...}
    └─ propagação: fmt.Errorf("...: %w", err)
  └─ caller recebe: nil, error{"open config.json: no such file or directory"}

Resultado: uma chain de errors legível com contexto em cada camada
```

Cada camada adiciona contexto com `%w` — exatamente o que [[System Calls]] descreve sobre como o kernel comunica erros ao processo via o mecanismo de errno.