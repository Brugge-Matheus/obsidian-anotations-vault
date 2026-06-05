---
tags:
  - go
  - go/oop
---
# Type Assertion e Type Switch

> Type assertion e type switch são os mecanismos para inspecionar e recuperar o tipo concreto armazenado em uma interface. Essenciais ao trabalhar com `any`, `error`, e código genérico.
> 

---

## 1. Como Interfaces Armazenam Tipos — Revisão

```
Interface (iface):
┌──────────┬──────────┐
│ *itab    │ *data    │
└──────────┴──────────┘
    │            │
    ▼            ▼
┌──────────┐ ┌──────────────┐
│ tipo     │ │ valor        │
│ métodos  │ │ concreto     │
└──────────┘ └──────────────┘

Type assertion: inspeciona o campo *itab para verificar o tipo
Type switch: compara *itab contra múltiplos tipos

Para interface{}/any (eface — sem métodos):
┌──────────┬──────────┐
│ *_type   │ *data    │  ← mais simples, sem itab
└──────────┴──────────┘
```

---

## 2. Type Assertion — Forma Básica

```go
// Sintaxe: valor.(Tipo) — panic se o tipo estiver errado
var i any = "hello"

s := i.(string)       // ok — i contém string
fmt.Println(s)        // "hello"

// n := i.(int)   ❌ panic: interface conversion:
//                        interface {} is string, not int
```

### Forma Segura — Dois Retornos (use sempre em produção)

```go
var i any = "hello"

s, ok := i.(string)
if ok {
	fmt.Println("string:", s)   // "string: hello"
} else {
	fmt.Println("não é string")
}

n, ok := i.(int)
fmt.Println(n, ok)   // 0 false — zero value + false, sem panic

// Forma inline idiomática
if s, ok := i.(string); ok {
	fmt.Println("processando string:", s)
}
```

---

## 3. Quando Usar Type Assertion

### 1. Recuperar tipo concreto de `any`

```go
func processar(v any) string {
	if s, ok := v.(string); ok {
		return strings.ToUpper(s)
	}
	if n, ok := v.(int); ok {
		return strconv.Itoa(n * 2)
	}
	return fmt.Sprintf("%v", v)
}
```

### 2. Verificar se implementa interface adicional

```go
// Verificar se um io.Reader também é io.Seeker
func lerComRewinding(r io.Reader) error {
	// Processa a leitura
	dados, err := io.ReadAll(r)
	if err != nil {
		return err
	}
	processar(dados)

	// Se o reader suporta Seek, volta ao início
	if seeker, ok := r.(io.Seeker); ok {
		seeker.Seek(0, io.SeekStart)
	}
	return nil
}

// Verificar io.Closer para fechar apenas quando necessário
func fecharSeSuportado(v any) {
	if closer, ok := v.(io.Closer); ok {
		closer.Close()
	}
}
```

### 3. Downcasting em hierarquia de interfaces

```go
type Animal interface{ Nome() string }
type Cachorro interface {
	Animal
	Latir() string
}

var a Animal = &Labrador{nome: "Rex"}

if dog, ok := a.(Cachorro); ok {
	fmt.Println(dog.Latir())   // "Au au!"
}
```

---

## 4. Type Switch — Múltiplos Tipos

Mais eficiente e legível que múltiplos `if v, ok := i.(T); ok`:

```go
func descrever(i any) string {
	switch v := i.(type) {
	case nil:
		return "nil"
	case bool:
		return fmt.Sprintf("bool: %v", v)
	case int, int8, int16, int32, int64:
		// v tem tipo do case mais genérico — aqui é any
		// para múltiplos tipos, use reflect ou case separados
		return fmt.Sprintf("inteiro: %v", v)
	case int:
		return fmt.Sprintf("int: %d (dobro: %d)", v, v*2)
	case float64:
		return fmt.Sprintf("float64: %.4f", v)
	case string:
		return fmt.Sprintf("string(%d chars): %q", len(v), v)
	case []byte:
		return fmt.Sprintf("[]byte(%d bytes)", len(v))
	case []int:
		return fmt.Sprintf("[]int com %d elementos", len(v))
	case map[string]any:
		return fmt.Sprintf("objeto com %d campos", len(v))
	case error:
		return "erro: " + v.Error()
	default:
		return fmt.Sprintf("tipo desconhecido: %T", v)
	}
}
```

> 💡 No case com múltiplos tipos (`case int, int8`), a variável `v` recebe o tipo da interface original (neste caso `any`). Para acessar o valor tipado, use cases separados.
> 

---

## 5. Type Switch com Interfaces

```go
// Verificar se o tipo implementa interfaces específicas
type Formatavel interface{ Formatar() string }
type Validavel interface{ Validar() error }
type Salvavel interface{ Salvar() error }

func processarObjeto(v any) error {
	// Verificar validação primeiro
	if val, ok := v.(Validavel); ok {
		if err := val.Validar(); err != nil {
			return fmt.Errorf("validação: %w", err)
		}
	}

	// Type switch para comportamento específico
	switch obj := v.(type) {
	case Formatavel:
		fmt.Println(obj.Formatar())
	case Salvavel:
		return obj.Salvar()
	case fmt.Stringer:
		fmt.Println(obj.String())
	default:
		fmt.Printf("processando: %T\n", v)
	}
	return nil
}
```

---

## 6. Type Assertion em Erros — O Uso Mais Comum

```go
// errors.As é a forma preferida (percorre chain de wrapping)
var dbErr *ErrDB
if errors.As(err, &dbErr) {
	log.Printf("DB error código %d na tabela %s", dbErr.Codigo, dbErr.Tabela)
}

// Type assertion direta — não percorre wrapping
if dbErr, ok := err.(*ErrDB); ok {
	// SÓ funciona se err é DIRETAMENTE *ErrDB
	// Não funciona se err é um wrapped *ErrDB
}

// Type switch em erros — para tratamento de múltiplos tipos
func tratarErro(err error) int {
	switch e := err.(type) {
	case *ErrHTTP:
		return e.Status
	case *ErrValidacao:
		return 422
	case *ErrDB:
		if e.Codigo == 1045 {
			return 503   // banco indisponível
		}
		return 500
	default:
		if errors.Is(err, context.DeadlineExceeded) {
			return 504
		}
		return 500
	}
}
```

---

## 7. JSON Dinâmico — Aplicação Real

```go
// Processar JSON onde o tipo do campo varia
func processarCampo(nome string, valor any) string {
	switch v := valor.(type) {
	case string:
		return fmt.Sprintf("%s = %q", nome, v)
	case float64:
		// JSON numérico sempre vira float64 sem UseNumber()
		if v == float64(int64(v)) {
			return fmt.Sprintf("%s = %d", nome, int64(v))
		}
		return fmt.Sprintf("%s = %g", nome, v)
	case json.Number:
		// Com decoder.UseNumber()
		if i, err := v.Int64(); err == nil {
			return fmt.Sprintf("%s = %d", nome, i)
		}
		return fmt.Sprintf("%s = %s", nome, v.String())
	case bool:
		return fmt.Sprintf("%s = %v", nome, v)
	case []any:
		return fmt.Sprintf("%s = [%d itens]", nome, len(v))
	case map[string]any:
		return fmt.Sprintf("%s = {%d campos}", nome, len(v))
	case nil:
		return fmt.Sprintf("%s = null", nome)
	default:
		return fmt.Sprintf("%s = (%T)", nome, v)
	}
}
```

---

## 8. Performance

```go
// Type assertion é O(1) — apenas uma comparação de ponteiro de tipo
// Type switch é O(n) — mas o compilador pode otimizar para jump table

// Para hot paths com muitos tipos, considere:
// 1. Reorganizar cases mais comuns primeiro no switch
// 2. Usar interfaces específicas em vez de any
// 3. Usar generics quando os tipos são conhecidos em compilação

// Generics (Go 1.18+) — alternativa a type assertion quando tipos são fixos
func converter[T any](v any) (T, bool) {
	resultado, ok := v.(T)
	return resultado, ok
}

s, ok := converter[string](valor)
n, ok := converter[int](valor)
```

---

## 9. Resumo

|  | Type Assertion | Type Switch |
| --- | --- | --- |
| Uso | Um tipo específico | Múltiplos tipos |
| Sintaxe | `v, ok := i.(T)` | `switch v := i.(type)` |
| Panic sem ok | Sim | Nunca |
| Percorre wrapping | Não | Não |
| Para erros | Prefira `errors.As` | Útil para despacho |
| Performance | O(1) | O(n) casos |
| Case múltiplo | — | `case T1, T2:` (v = tipo original) |
| Verificar interface | `_, ok := v.(Iface)` | `case Iface:` |

---

## 10. Como Type Assertion Funciona Internamente

### O que o Runtime Faz em `x.(T)`

```go
var i Forma = Retangulo{10, 5}
r := i.(Retangulo)   // type assertion
```

```
Passos em runtime:

1. Carrega i.tab (*itab)
2. Lê itab.type (*_type) — o descritor do tipo concreto armazenado
3. Compara itab.type == typeDescriptor(Retangulo)
   - É uma comparação de PONTEIRO (não string, não hash) — O(1)
4. Se igual: retorna i.data convertido para *Retangulo (ou cópia do valor)
5. Se diferente: panic("interface conversion: ...")

Para interface{} / any (eface):
1. Carrega i._type (*_type)
2. Compara i._type == typeDescriptor(T)
3. Resultado ou panic
```

Os descritores de tipo (`*_type`) são alocados estaticamente pelo compilador no segmento de dados do binário. A comparação é simplesmente `if ptr_a == ptr_b` — nenhuma string é comparada em runtime.

### Type Switch — Como o Compilador Gera o Código

```go
switch v := i.(type) {
case int:    ...
case string: ...
case error:  ...
}
```

```
Código gerado (esquemático):

  t := i.tab.type    // carrega o tipo uma vez
  if t == typeOf(int)    { v := *(*int)(i.data);    goto case_int    }
  if t == typeOf(string) { v := *(*string)(i.data); goto case_string }
  if t == typeOf(error)  {                          goto case_error  }
  // default

Para poucos cases: série de comparações (if-chain) — O(n)
Para muitos cases: o compilador pode gerar uma jump table — O(1)
  → a otimização depende do compilador e do número de cases
```

### Panic em Assertion Falha — Comparação com Exceções de Hardware

```go
r := i.(Retangulo)   // sem ok — panic se falhar
```

```
Fluxo de panic:
  type mismatch detectado em runtime
    → runtime.panicdottypeI() ou runtime.panicdottypeE()
      → runtime.panic()
        → unwind da goroutine stack (defer run)
          → se não recuperado: goroutine termina + mensagem de erro

Analogia com C/hardware:
  Em C: cast incorreto de void* → comportamento indefinido (UB)
        pode resultar em SIGSEGV, SIGBUS, ou dado corrompido silencioso
  Em Go: panic explícito com mensagem clara — "is string, not int"
         sem UB, sem corrupção silenciosa

  A diferença fundamental: Go tem informação de tipo em runtime (RTTI),
  C não tem — o void* não sabe o que aponta.
```

### RTTI — Runtime Type Information

Go mantém um descritor `_type` para cada tipo do programa, gerado pelo compilador e embutido no binário:

```
Struct _type (runtime/type.go, simplificada):
┌──────────────────────────────────────┐
│ size       uintptr   ← tamanho em bytes
│ ptrdata    uintptr   ← bytes com ponteiros (para GC)
│ hash       uint32    ← hash do tipo (para map, comparação rápida)
│ tflag      uint8     ← flags (exportado, comparable, etc.)
│ align      uint8     ← alinhamento
│ fieldAlign uint8     ← alinhamento como campo de struct
│ kind       uint8     ← bool=1, int=2, string=24, struct=25, ...
│ equal      func      ← função de comparação ==
│ gcdata     *byte     ← mapa de ponteiros para o GC
│ str        nameOff   ← nome do tipo (offset na seção de strings)
│ ptrToThis  typeOff   ← descritor de *T
└──────────────────────────────────────┘
```

O pacote `reflect` expõe esses descritores como `reflect.Type`. A mesma estrutura é usada por type assertion, type switch, fmt.Printf com `%T`, JSON encoding/decoding, e o GC.

---

## 11. Conexão com Sistemas Operacionais

### Type Assertion = Comparação de Ponteiros de Tipo — [[Gerenciamento de Memória]]

A operação `x.(T)` se resume a comparar dois ponteiros: `itab.type == &typeDescriptor_T`. Os descritores de tipo ficam no segmento `.rodata` do binário (somente leitura, compartilhado entre goroutines). Não há alocação em heap, não há comparação de strings — é uma comparação de endereço de 8 bytes. Isso só funciona porque Go garante **unicidade de descritores**: para cada tipo, existe exatamente um `*_type` no binário — o linker deduplica. A mesma garantia que o kernel dá para símbolos de módulo: cada `struct file_operations` de um driver é única e identificada pelo endereço.

### Type Switch e Geração de Código — [[Processadores]]

O compilador pode transformar um type switch com muitos cases em uma jump table baseada no campo `kind` do descritor de tipo — similar a como um switch em inteiros é otimizado. Para casos de interface (não tipo concreto), a comparação é entre ponteiros de itab. A CPU executa uma sequência de comparações (CMP + JE) ou salta via tabela indexada. Em loops hot com type switch, a branch prediction da CPU pode ajudar se o mesmo tipo aparece repetidamente — o preditor aprende o padrão.

### RTTI em Go vs DWARF em Debuggers — [[Gerenciamento de Memória]]

O `_type` de Go e as entradas DWARF em um binário ELF/Mach-O servem propósitos similares: descrever tipos para inspeção em runtime. DWARF é usado por `gdb`, `lldb` e `dlv` (Delve, o debugger Go) para mostrar o tipo de variáveis em um core dump. Go embute informação de tipo diretamente no runtime (não apenas como debug info) porque a linguagem precisa dela para reflection, GC e type assertion — não só para debugging. Um `core dump` de processo Go pode ser inspecionado com Delve, que usa ambas as fontes.

### Panic em Assertion Falha vs Exceções de Hardware — [[Processadores]]

Quando `x.(T)` falha sem o idioma `v, ok`, o runtime Go dispara um panic. O fluxo é: função de runtime detecta mismatch → chama `runtime.panic()` → unwind da stack da goroutine executando `defer`s → se não houver `recover()`, a goroutine morre. Isso contrasta com C, onde um cast errado de `void*` pode gerar uma exceção de hardware (SIGSEGV se o ponteiro inválido for dereferenciado, SIGBUS se desalinhado) — o SO entrega o sinal ao processo, que termina a menos que tenha um signal handler. Go internaliza o tratamento: o runtime é o "signal handler" que transforma erros de tipo em panics gerenciáveis.

### Reflection e Introspecção de Processos — [[Processos]]

O pacote `reflect` usa os mesmos `_type` descritores que type assertion. Com reflection, um programa Go pode inspecionar seus próprios tipos em runtime — nome do tipo, campos de uma struct, tags, métodos. Isso é análogo a como ferramentas de introspecção de processos funcionam no SO: `ptrace(2)` no Linux permite que um processo (debugger) leia a memória e registros de outro processo; Go permite que o próprio programa leia sua estrutura de tipos via `reflect`. Quando um processo Go crasheia e gera um core dump, o Delve usa `reflect` + DWARF para mostrar o estado das goroutines — combinando introspecção em nível de linguagem com introspecção em nível de OS.

### Tagged Unions em C vs Type Switch em Go — [[Gerenciamento de Memória]]

O equivalente manual de type switch em C são tagged unions:

```c
/* C — type tag manual (sem segurança) */
typedef enum { TYPE_INT, TYPE_STRING, TYPE_FLOAT } Tag;
typedef struct {
    Tag tag;
    union {
        int    i;
        char*  s;
        double f;
    } data;
} Variant;

void processar(Variant v) {
    switch (v.tag) {
    case TYPE_INT:    printf("%d\n", v.data.i); break;
    case TYPE_STRING: printf("%s\n", v.data.s); break;
    /* esquecer um case → sem aviso do compilador, comportamento silencioso */
    }
}
```

```go
// Go — type switch com segurança de tipos em runtime
func processar(v any) {
    switch x := v.(type) {
    case int:    fmt.Println(x)
    case string: fmt.Println(x)
    // default: sem isso, casos não tratados são ignorados silenciosamente
    //          mas o compilador alerta se você usou o valor x e o case pode ser unreachable
    }
}
// Se o tipo não bate na assertion sem ok → panic (não UB)
// O runtime conhece o tipo real via _type — nenhuma tag manual necessária
```