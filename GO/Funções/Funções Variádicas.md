---
tags:
  - go
  - go/funções
---
# Funções Variádicas

> Funções variádicas aceitam um número arbitrário de argumentos do mesmo tipo. Internamente, o compilador converte os argumentos em um slice — o parâmetro variádico é simplesmente `[]T`. São a base de funções como `fmt.Println`, `append` e `errors.Join`.
> 

---

## 1. Como o Compilador Trata Argumentos Variádicos

```go
func somar(nums ...int) int

Chamada: somar(1, 2, 3, 4, 5)

O compilador cria um slice temporário:
┌─────┬─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  4  │  5  │  ← array na stack (se pequeno o suficiente)
└─────┴─────┴─────┴─────┴─────┘
                  ↑
           []int{1,2,3,4,5}   ← slice passado como nums

Se já é um slice: somar(s...) — passa o slice diretamente (sem cópia)

go build -gcflags="-m" mostra: "... argument does not escape" (fica na stack)
ou "... escapes to heap" (alocado na heap se necessário)
```

---

## 2. Declaração e Uso Básico

```go
// O parâmetro variádico deve ser o ÚLTIMO
func somar(nums ...int) int {
	total := 0
	for _, n := range nums {   // nums é []int — slice normal
		total += n
	}
	return total
}

// Chamadas válidas
fmt.Println(somar())              // 0     — slice vazio: []int{}
fmt.Println(somar(1))             // 1
fmt.Println(somar(1, 2, 3))       // 6
fmt.Println(somar(10, 20, 30))    // 60

// Expandir slice existente com ...
nums := []int{1, 2, 3, 4, 5}
fmt.Println(somar(nums...))   // 15 — passa o slice direto (sem cópia)

// Verificar tipo interno do parâmetro
func inspecionar(args ...string) {
	fmt.Printf("tipo: %T\n", args)   // tipo: []string
	fmt.Printf("len: %d\n", len(args))
	fmt.Printf("nil? %v\n", args == nil)   // false quando chamado com args
}
```

---

## 3. Parâmetros Fixos + Variádico

```go
// Parâmetros fixos ANTES do variádico
func log(nivel string, args ...any) {
	fmt.Printf("[%s] ", nivel)
	fmt.Println(args...)   // args... para expandir na chamada de Println
}

log("INFO", "servidor iniciado na porta", 8080)
log("ERRO", "falha:", errors.New("timeout"))
log("DEBUG")   // sem args variádicos — ok, args = []any{}

// Múltiplos fixos + variádico
func criarTag(tag, classe string, conteudo ...string) string {
	attrs := ""
	if classe != "" {
		attrs = fmt.Sprintf(` class="%s"`, classe)
	}
	return fmt.Sprintf("<%s%s>%s</%s>",
		tag, attrs, strings.Join(conteudo, ""), tag)
}

fmt.Println(criarTag("p", "destaque", "Hello ", "World"))
// <p class="destaque">Hello World</p>
```

---

## 4. Expandindo Slices com `...`

```go
func somar(nums ...int) int { /* ... */ }

a := []int{1, 2, 3}
b := []int{4, 5, 6}

// Expandir na chamada
fmt.Println(somar(a...))          // 6
fmt.Println(somar(b...))          // 15
fmt.Println(somar(append(a, b...)...))   // 21

// Uso clássico: append com outro slice
c := append(a, b...)              // [1, 2, 3, 4, 5, 6]

// ⚠️ Cuidado: modificar o slice dentro da função variádica
// pode afetar o original quando passado com ...
func modificar(nums ...int) {
	nums[0] = 99   // modifica o array subjacente!
}
original := []int{1, 2, 3}
modificar(original...)
fmt.Println(original)   // [99, 2, 3] — foi modificado!

// Para evitar: faça uma cópia dentro da função
func modificarSeguro(nums ...int) {
	copia := make([]int, len(nums))
	copy(copia, nums)
	copia[0] = 99   // não afeta o original
}
```

---

## 5. `...interface{}` / `...any` — Qualquer Tipo

```go
// fmt.Println é definido assim:
func Println(a ...any) (n int, err error)

// Implementação própria que aceita qualquer coisa
func imprimir(args ...any) {
	for i, arg := range args {
		if i > 0 {
			fmt.Print(" ")
		}
		fmt.Printf("%v", arg)
	}
	fmt.Println()
}

imprimir(42, "hello", true, 3.14, []int{1, 2, 3})
// 42 hello true 3.14 [1 2 3]

// Type switch dentro de variádica
func serializar(parts ...any) string {
	var sb strings.Builder
	for i, p := range parts {
		if i > 0 {
			sb.WriteString(", ")
		}
		switch v := p.(type) {
		case string:
			sb.WriteString(strconv.Quote(v))
		case int, int64, float64:
			fmt.Fprintf(&sb, "%v", v)
		case bool:
			if v {
				sb.WriteString("true")
			} else {
				sb.WriteString("false")
			}
		case nil:
			sb.WriteString("null")
		default:
			fmt.Fprintf(&sb, "%v", v)
		}
	}
	return sb.String()
}
```

---

## 6. Padrão de Opções Funcionais com Variádica

O padrão mais elegante de Go para APIs configuráveis:

```go
// Tipo opção — closure que configura T
type Opcao[T any] func(*T)

// Configuração de servidor
type ConfigServidor struct {
	host      string
	porta     int
	timeout   time.Duration
	maxConns  int
	debug     bool
	tlsConfig *tls.Config
}

// Funções que retornam opções
func ComHost(host string) Opcao[ConfigServidor] {
	return func(c *ConfigServidor) { c.host = host }
}

func ComPorta(porta int) Opcao[ConfigServidor] {
	return func(c *ConfigServidor) {
		if porta < 1 || porta > 65535 {
			panic(fmt.Sprintf("porta inválida: %d", porta))
		}
		c.porta = porta
	}
}

func ComTimeout(d time.Duration) Opcao[ConfigServidor] {
	return func(c *ConfigServidor) { c.timeout = d }
}

func ComMaxConexoes(n int) Opcao[ConfigServidor] {
	return func(c *ConfigServidor) { c.maxConns = n }
}

func ComDebug() Opcao[ConfigServidor] {
	return func(c *ConfigServidor) { c.debug = true }
}

// Constructor com opções variádicas
func NovoServidor(opcoes ...Opcao[ConfigServidor]) (*Servidor, error) {
	cfg := &ConfigServidor{   // valores padrão
		host:     "localhost",
		porta:    8080,
		timeout:  30 * time.Second,
		maxConns: 100,
	}

	for _, opt := range opcoes {
		opt(cfg)
	}

	return &Servidor{cfg: cfg}, nil
}

// Uso limpo — a API nunca quebra ao adicionar novas opções
s, _ := NovoServidor(
	ComHost("0.0.0.0"),
	ComPorta(9090),
	ComTimeout(60 * time.Second),
	ComDebug(),
)
```

---

## 7. Variádicas da Stdlib — Exemplos Importantes

```go
// fmt.Errorf — múltiplos verbos
err := fmt.Errorf("operação %s falhou na linha %d: %w", nome, linha, causeErr)

// append — variádico para slices
s = append(s, 1, 2, 3)
s = append(s, outro...)

// errors.Join (Go 1.20+) — combinar múltiplos erros
err := errors.Join(err1, err2, err3)

// strings.Join (não variádica, mas similar)
strings.Join([]string{"a", "b", "c"}, ", ")

// http.Header.Add e Set
w.Header().Set("Content-Type", "application/json")
```

---

## 8. Performance — Quando Variádicas Alocam

```go
// Chamada com argumentos literais — pode ficar na stack (escape analysis)
somar(1, 2, 3)   // slice temporário pode ser alocado na stack

// Chamada com slice... — não cria novo slice
s := []int{1, 2, 3}
somar(s...)   // passa o slice existente, sem cópia

// Pré-alocar quando acumular muitos argumentos em um loop
resultado := make([]int, 0, 100)
for _, v := range dados {
	resultado = append(resultado, processar(v))
}
somar(resultado...)   // um único slice bem alocado

// Evitar: criar slices grandes e descartar
for _, item := range muitos_itens {
	somar(preparar(item)...)   // pode alocar a cada iteração
}
```

---

## 9. Resumo

| Aspecto | Detalhe |
| --- | --- |
| Tipo interno | `[]T` — slice normal |
| Posição | Sempre o ÚLTIMO parâmetro |
| Zero argumentos | `len(args) == 0`, `args != nil` não é garantido |
| Expandir slice | `f(s...)` — passa o slice direto, sem cópia |
| Modificar slice | Afeta o original se passado com `...` |
| `any` variádico | `func f(args ...any)` — qualquer tipo |
| Performance | Pode alocar na stack ou heap — escape analysis decide |
| Padrão de opções | `func NewT(opts ...OpçãoT)` — API extensível sem breaking changes |

---

## 10. Conexão com Sistemas Operacionais

### Como Go Compila `...T` — Slice Header na Stack

Quando o compilador Go vê `func somar(nums ...int)`, ele cria automaticamente um **slice header** na stack do chamador e o passa como argumento:

```
Chamada: somar(1, 2, 3, 4, 5)

O compilador transforma isso em:
  1. Aloca array temporário [5]int na stack (se não escapa para heap)
  2. Inicializa: [1, 2, 3, 4, 5]
  3. Cria slice header e passa como argumento:

Stack frame do caller:
┌──────────────────────────┐
│  array: [1][2][3][4][5]  │  ← 5 × 8 bytes = 40 bytes
├──────────────────────────┤
│  slice header (nums):    │
│    ptr → array acima     │  8 bytes
│    len = 5               │  8 bytes
│    cap = 5               │  8 bytes
└──────────────────────────┘

Dentro de somar(), nums é exatamente um []int normal:
  nums.ptr → aponta para o array acima
  nums.len = 5
  nums.cap = 5
```

Se o array "escapa" (escape analysis detecta que sobrevive ao frame), vai para a heap. Veja com `go build -gcflags="-m"`. Ver [[Processos]] para entender stack frame e layout de dados.

---

### Variádicas em C — `va_list` e Aritmética de Ponteiro

Em C, funções variádicas como `printf` usam uma técnica muito mais manual e perigosa, baseada em aritmética de ponteiro na stack:

```c
// C — como printf("valor: %d %s\n", 42, "hello") funciona internamente

#include <stdarg.h>

// Chamada: printf("valor: %d %s\n", 42, "hello")
// Na stack (x86-64 System V ABI):
//   args em registradores: RDI=fmt, RSI=42, RDX=&"hello"
//   se overflow: na stack em ordem reversa

void meu_printf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);   // ← macro: inicializa ponteiro para o próximo arg

    // Para cada %d, %s, etc:
    int n = va_arg(args, int);      // ← lê próximo int da stack/registrador
    char *s = va_arg(args, char*);  // ← lê próximo ponteiro

    // va_arg é uma MACRO que faz aritmética de ponteiro:
    // args = (char*)args + sizeof(int)  (aproximadamente)

    va_end(args);   // limpeza
}
```

O problema: **o compilador C não sabe os tipos dos argumentos variádicos**. Se você escrever `printf("%d", "uma string")`, o compilador em C clássico aceita silenciosamente — e você lê bytes do tipo errado na memória → **comportamento indefinido / crash**. Ver [[Processadores]] para entender como a stack pointer é usada para navegar pelos argumentos.

---

### Go vs C — Segurança de Tipos em Variádicas

A comparação direta mostra por que o design de Go elimina uma classe inteira de bugs de memória:

```
C — printf family (stack walking manual):

  printf("%d %s", 42, "hello"):
    ┌─────────────┐
    │ fmt ptr     │ → "%d %s"
    │ arg1 = 42   │ → int (RDI no x86-64 SysV)
    │ arg2 = ptr  │ → char* para "hello"
    └─────────────┘
    
  Risco: printf("%d", ptr)  ← trata ponteiro como int
         printf("%s", 42)   ← trata int como ponteiro → segfault
         printf(user_input)  ← format string attack!
  
  O compilador GCC/Clang adiciona warnings (-Wformat)
  mas são opcionais e podem ser ignorados.

Go — variádica type-safe:

  func Println(a ...any)
  
  somar(1, "dois", 3.0)  ← erro de compilação: cannot use "dois"
                              (type string) as type int
  
  Com ...any: qualquer tipo é aceito, mas você vê o tipo real
  via type switch — sem aritmética de ponteiro manual
```

Isso é **segurança de memória** — o Go garante que você nunca lê bytes de tipo errado através de uma variádica. Ver [[Gerenciamento de Memória]] para entender como type safety se relaciona com segurança de acesso à memória.

---

### Escape Analysis — Stack ou Heap para o Array Variádico

O compilador decide onde alocar o array temporário criado para argumentos variádicos:

```bash
# Verificar onde aloca
go build -gcflags="-m=2" ./...
```

```go
// Caso 1 — fica na stack (não escapa):
func processar() {
    somar(1, 2, 3)
    // "./main.go: ... argument does not escape"
    // Array [1,2,3] criado na stack de processar()
}

// Caso 2 — vai para a heap (escapa):
var global []int
func processar() {
    global = append(global, 1, 2, 3)
    // append é variádico: append(global, []int{1,2,3}...)
    // o slice pode realocar na heap
}

// Caso 3 — slice passado com ... (sem alocação nova):
nums := []int{1, 2, 3}
somar(nums...)   // passa o slice existente — nenhuma cópia, nenhuma alocação
```

A decisão de stack vs heap pelo escape analysis é exatamente o tema de [[Gerenciamento de Memória]] — o compilador analisa o tempo de vida das variáveis para colocá-las no lugar mais eficiente.