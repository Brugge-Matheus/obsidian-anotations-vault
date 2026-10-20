---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Type assertion
  - Type switch
  - Asserção de tipo
  - Interfaces opcionais
ordem: 24
modulo: "05 - Métodos, Interfaces e Generics"
---
# Type Assertion e Type Switch

> **Nota 24 da trilha de Go.** Uma interface esconde o tipo concreto do valor que guarda. Às vezes precisamos **recuperar** esse tipo: para acessar campos, para tratar um caso especial, ou para descobrir se o valor sabe fazer algo **a mais** do que a interface exige. A **type assertion** (`x.(T)`) faz isso para um tipo; o **type switch** (`switch v := x.(type)`) escolhe entre vários. Nesta nota veremos as duas formas (com panic e com "vírgula ok"), assertions para **interfaces** (o padrão das **interfaces opcionais**), type switch em detalhe, o uso com erros e JSON dinâmico, e como o runtime implementa a verificação.
>
> **Pré-requisitos:** [[Interfaces]].

---

## 🧭 De onde viemos

Em [[Interfaces]] vimos que uma variável de interface guarda **duas palavras**: um ponteiro para o tipo (ou para a itab) e um ponteiro para os dados. Vimos também que `any` aceita qualquer valor, mas "esconde" o tipo. Em [[Controle de Fluxo]] demos uma prévia do *type switch*. Agora vamos aprender a "abrir a caixa".

> 💡 **Tipo estático vs tipo dinâmico:** o tipo **estático** de uma variável é o declarado (`var x any`), conhecido pelo compilador. O tipo **dinâmico** é o tipo concreto do valor guardado naquele momento (`string`, `int`...), conhecido só em tempo de execução. Uma type assertion pergunta pelo tipo dinâmico.

---

## 🔍 1. Type assertion: `x.(T)`

### 1.1 Forma de um resultado: confiante (dá panic se errar)

Saída real:

```go
var x any = "olá"
s := x.(string)       // afirma: "x guarda uma string"
fmt.Println(s)        // olá

_ = x.(int)
// panic: interface conversion: interface {} is string, not int
```

### 1.2 Forma de dois resultados: "vírgula ok" (nunca dá panic)

```go
n, ok := x.(int)
fmt.Println(n, ok)    // 0 false   ← ok = false e n recebe o zero value de int

if s, ok := x.(string); ok {
	fmt.Println("é string:", s)
}
```

> ✅ Use a forma de **um** resultado só quando um erro de tipo for um **bug** (algo que "não pode acontecer"). Para dados vindos de fora ou casos realmente variáveis, use sempre a **vírgula ok**.

### 1.3 Regras

- `x` precisa ser de **tipo interface** (`any`, `error`, `io.Reader`...). Não se faz assertion em `int` ou `string`
- Se `T` é um **tipo concreto**, a assertion verifica se o tipo dinâmico de `x` é **exatamente** `T`
- Se `T` é uma **interface**, verifica se o tipo dinâmico de `x` **implementa** `T` (seção 2)
- Se `x` é nil, toda assertion falha (`ok = false` ou panic)

```go
var e error = &os.PathError{...}
pe := e.(*os.PathError)         // ✅ tipo exato: ponteiro
pe := e.(os.PathError)          // ❌ falha: o tipo dinâmico é *os.PathError, não os.PathError
```

> ⚠️ Type assertion **não é conversão**. Não transforma um valor de um tipo em outro: só "desembrulha" o valor que já é daquele tipo. `var x any = 42; x.(int64)` **falha**, porque o tipo dinâmico é `int`. Para converter, primeiro desembrulhe, depois converta: `int64(x.(int))`.

---

## 🧩 2. Assertion para interface: o padrão das interfaces opcionais

Quando `T` é uma interface, a pergunta é: **"o valor guardado também sabe fazer isto?"**. Isso permite que funções aproveitem capacidades **extras** quando disponíveis, sem exigi-las na assinatura.

### 2.1 Exemplo real: como `io.Copy` fica rápido

`io.Copy(dst, src)` só exige um `Reader` e um `Writer`. Mas, por dentro, ele faz:

```go
// simplificado de io.Copy
if wt, ok := src.(io.WriterTo); ok {    // a origem sabe se escrever sozinha?
	return wt.WriteTo(dst)              // → caminho otimizado
}
if rf, ok := dst.(io.ReaderFrom); ok {  // o destino sabe ler sozinho?
	return rf.ReadFrom(src)             // → caminho otimizado
}
// senão: laço genérico com um buffer de 32 KB
```

Saídas reais:

```go
var r io.Reader = strings.NewReader("abc")
_, ok := r.(io.WriterTo)
fmt.Println(ok)     // true   ← strings.Reader tem WriteTo: io.Copy usa o atalho

var r2 io.Reader = io.LimitReader(strings.NewReader("abc"), 2)
_, ok2 := r2.(io.WriterTo)
fmt.Println(ok2)    // false  ← o LimitReader não tem: io.Copy usa o laço genérico
```

É assim que copiar de um `*os.File` para um socket pode usar a syscall **`sendfile`** (o kernel copia direto, sem passar os dados pelo espaço de usuário): `*net.TCPConn` implementa `ReaderFrom` e, ao receber um arquivo, chama `sendfile`. Você estudou o custo de copiar dados entre kernel e usuário em [[System Calls]].

### 2.2 Outros exemplos da biblioteca padrão

```go
if sw, ok := w.(io.StringWriter); ok {     // escrever string sem converter para []byte
	sw.WriteString("rápido")
}

if f, ok := w.(http.Flusher); ok {         // o ResponseWriter suporta enviar dados parciais? (streaming/SSE)
	f.Flush()
}

if s, ok := v.(fmt.Stringer); ok {         // o valor sabe se imprimir?
	return s.String()
}
```

> ⚠️ Lembra do aviso em [[Composição vs Herança]]? Se você **embrulha** um valor (por exemplo, um `http.ResponseWriter` num middleware), o embrulho **esconde** as interfaces opcionais do original, e assertions como `w.(http.Flusher)` passam a falhar. Desde o Go 1.20 existe `http.NewResponseController(w)`, que encontra essas capacidades mesmo através de embrulhos que implementam `Unwrap()`.

---

## 🎛️ 3. Type switch

Quando há vários tipos possíveis, uma sequência de assertions fica feia. O **type switch** testa vários de uma vez (saídas reais):

```go
func descrever(x any) string {
	switch v := x.(type) {          // .(type) só existe dentro de um switch
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("int %d (dobro %d)", v, v*2)       // aqui v é int
	case string:
		return fmt.Sprintf("string %q de %d bytes", v, len(v)) // aqui v é string
	case int8, int16:
		return fmt.Sprintf("inteiro pequeno %v (%T)", v, v)    // vários tipos: v continua any
	case error:
		return "erro: " + v.Error()                            // case com INTERFACE
	case fmt.Stringer:
		return "stringer: " + v.String()
	default:
		return fmt.Sprintf("outro tipo: %T", v)                // v continua any
	}
}
```

```
nil                         → nil
42                          → int 42 (dobro 84)
"go"                        → string "go" de 2 bytes
int8(3)                     → inteiro pequeno 3 (int8)
fmt.Errorf("falhou")        → erro: falhou
P{7} (tem String())         → stringer: P(7)
3.14                        → outro tipo: float64
[]int{1}                    → outro tipo: []int
```

Regras do type switch:

| Regra | Detalhe |
| --- | --- |
| `v := x.(type)` | Em cada `case` com **um** tipo, `v` tem esse tipo |
| `case A, B:` | Com vários tipos, `v` mantém o tipo de `x` (interface) |
| `case nil:` | Casa quando a interface é nil |
| `default:` | Nenhum casou; `v` tem o tipo de `x` |
| Ordem | Os cases são testados de cima para baixo: o **primeiro** que casar vence |
| `fallthrough` | Não é permitido em type switch |
| Sem `v :=` | `switch x.(type)` também é válido, se você não precisa do valor |

> ⚠️ **A ordem importa com interfaces.** Um tipo pode satisfazer várias interfaces. No exemplo, um tipo com os métodos `Error()` **e** `String()` cairia em `case error`, porque ele vem antes. Coloque os cases mais específicos primeiro.

O `v := x.(type)` é um **shadowing intencional**, como vimos em [[Sintaxe, Blocos e Escopo]]: muitas vezes se escreve `switch x := x.(type)` para reaproveitar o nome.

---

## ❌ 4. Type assertions com erros

Erros são o uso mais comum de assertions em código real. Mas, desde o Go 1.13, a forma recomendada é **`errors.As`**, porque erros podem estar **embrulhados** uns nos outros (com `%w`), e uma assertion simples só olha a camada de fora:

```go
_, err := os.Open("/nao/existe")

// ⚠️ assertion direta: falha se alguém embrulhou o erro com fmt.Errorf("...: %w", err)
if pe, ok := err.(*os.PathError); ok {
	fmt.Println("caminho:", pe.Path)
}

// ✅ errors.As: procura em toda a cadeia de erros embrulhados
var pe *os.PathError
if errors.As(err, &pe) {
	fmt.Println("caminho:", pe.Path)
}

// ✅ Go 1.26+: errors.AsType, genérico e sem precisar declarar a variável antes
if pe, ok := errors.AsType[*os.PathError](err); ok {
	fmt.Println("caminho:", pe.Path)
}
```

Veremos a cadeia de erros em [[Wrapping e Unwrapping de Erros]].

---

## 🗂️ 5. JSON dinâmico: assertions na prática

Quando o formato do JSON não é conhecido, ele é decodificado em `map[string]any`, e cada valor precisa de assertion. Saídas reais:

```go
var dados map[string]any
json.Unmarshal([]byte(`{"nome":"Ana","idade":30,"tags":["go","rust"],"ativo":true}`), &dados)

// nome: string
// idade: float64          ← ⚠️ TODO número JSON vira float64!
// tags: []interface {}    ← arrays viram []any (não []string!)
// ativo: bool

idade := dados["idade"].(float64)
fmt.Println(int(idade))    // 30

for _, t := range dados["tags"].([]any) {
	nome, _ := t.(string)
	fmt.Println(nome)
}
```

| Tipo JSON | Tipo Go em `any` |
| --- | --- |
| objeto | `map[string]any` |
| array | `[]any` |
| string | `string` |
| número | `float64` (ou `json.Number` com `UseNumber`) |
| `true`/`false` | `bool` |
| `null` | `nil` |

> ✅ Sempre que o formato for conhecido, decodifique numa **struct** com tags. Fica tipado, mais rápido e sem assertions. Ver [[JSON (marshal e unmarshal)]].

---

## 🤔 6. Quando usar (e quando é cheiro de problema)

| ✅ Bons usos | ⚠️ Sinais de design ruim |
| --- | --- |
| Interfaces opcionais (`io.WriterTo`, `http.Flusher`) | Type switch gigante sobre os seus próprios tipos (devia ser um método da interface) |
| Erros específicos (via `errors.As`) | Assertion para "voltar" ao tipo concreto logo depois de passá-lo como interface |
| Dados genuinamente dinâmicos (JSON sem esquema, plugins) | `any` em parâmetros só para depois fazer switch (devia ser generics ou interface) |
| Implementar `fmt`-like: tratar alguns tipos de forma especial | Assertions espalhadas que quebram quando aparece um tipo novo |

```go
// ⚠️ cheiro: switch sobre tipos que você controla
func area(f any) float64 {
	switch s := f.(type) {
	case Retangulo: return s.L * s.A
	case Circulo:   return math.Pi * s.R * s.R
	}
	return 0          // um Triangulo novo cai aqui silenciosamente
}

// ✅ melhor: comportamento na interface (ver Interfaces)
type Forma interface{ Area() float64 }
```

---

## 🔬 7. Por dentro: como o runtime verifica

### 7.1 Assertion para tipo concreto: uma comparação de ponteiros

Cada tipo do programa tem um **descritor** único (`*_type`) no binário. Verificar `x.(string)` é comparar o ponteiro de tipo guardado em `x` com o endereço do descritor de `string`:

```
x.(string)  com x do tipo any (eface: _type + data)

    CMPQ  x._type, $type:string     ; mesmo descritor?
    JNE   falhou                    ; não → panic ou ok=false
    MOVQ  x.data, AX                ; sim → o valor é o que está em data
```

É praticamente gratuito: uma comparação e um desvio. Para interfaces com métodos (iface), compara-se o tipo dentro da itab.

### 7.2 Assertion para interface: busca e cache

`x.(io.WriterTo)` precisa descobrir se o tipo dinâmico tem o método `WriteTo`. O runtime:

1. Procura num **cache** global de itabs pelo par (tipo dinâmico, `io.WriterTo`)
2. Se não achar, compara a lista de métodos do tipo com a da interface (ambas ordenadas por nome, então é uma intercalação linear)
3. Cria a itab (ou registra que "não implementa") e guarda no cache

Desde o Go 1.22, o compilador também coloca um **cache por ponto de chamada** nas assertions e type switches, acelerando os casos repetidos.

### 7.3 Type switch

O compilador gera, conforme o caso:

- comparações sequenciais de ponteiros de tipo, para poucos cases
- uma **busca binária** pelo `hash` do tipo (guardado no descritor e na itab), para muitos cases concretos
- consultas ao cache de itabs, para cases com interfaces

### 7.4 RTTI: o que permite tudo isso

Assertions, type switches, `%T` do `fmt`, `errors.As` e o pacote `reflect` dependem da **informação de tipos em tempo de execução** (RTTI) que o compilador grava no binário: nome do tipo, tamanho, alinhamento, onde há ponteiros (para o GC), métodos. É parte do motivo de um binário Go ser maior que um de C. Veremos o uso explícito dessa informação em [[Reflection]].

> 💡 Comparação com C: lá não existe RTTI. O padrão equivalente é a **união etiquetada** (*tagged union*): um campo `tipo` mais uma `union` com as variantes, e um `switch (v.tipo)` manual. Se alguém errar a etiqueta, o programa lê lixo. Em Go, a "etiqueta" é o ponteiro de tipo, mantido pelo próprio runtime, e uma assertion errada é detectada.

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Assertion de um resultado em dado incerto | `panic: interface conversion` | Use `v, ok := x.(T)` |
| Confundir assertion com conversão | `x.(int64)` falha quando o valor é `int` | Desembrulhe e depois converta |
| `T` vs `*T` na assertion | Falha por detalhe de ponteiro | Use o tipo dinâmico exato (`*os.PathError`) |
| Assertion direta em erro embrulhado | Não encontra o erro | `errors.As` / `errors.AsType` |
| Números de JSON como `int` | `x.(int)` falha | São `float64` (ou use structs) |
| Ordem dos cases com interfaces | Caso mais geral "rouba" o específico | Específicos primeiro |
| Embrulho esconde interfaces opcionais | `w.(http.Flusher)` falha | `http.NewResponseController` ou repasse os métodos |
| Type switch no lugar de um método | Novos tipos caem no `default` silenciosamente | Coloque o comportamento na interface |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `x.(T)` | Afirma o tipo dinâmico; panic se errado |
| `v, ok := x.(T)` | Forma segura; `ok = false` e zero value se errado |
| `T` concreto | Tipo dinâmico deve ser exatamente `T` |
| `T` interface | Tipo dinâmico deve implementar `T` (interfaces opcionais) |
| Não é conversão | Só desembrulha |
| Type switch | `switch v := x.(type)`; `v` tipado em cases de um tipo |
| Erros | Prefira `errors.As` / `errors.AsType` (1.26+) |
| JSON dinâmico | Números `float64`, arrays `[]any`, objetos `map[string]any` |
| Por dentro | Comparação de ponteiros de tipo; cache de itabs; RTTI no binário |

---

## 🔗 Conexões

**Dentro de Go**
- O par tipo + dados das interfaces → [[Interfaces]]
- Embrulhos que escondem interfaces → [[Composição vs Herança]]
- `errors.As`, `errors.AsType` e a cadeia de erros → [[Wrapping e Unwrapping de Erros]]
- JSON tipado vs dinâmico → [[JSON (marshal e unmarshal)]]
- `io.WriterTo`, `io.ReaderFrom` e `io.Copy` → [[io.Reader e io.Writer]]
- Alternativa com segurança de tipos → [[Generics]]
- Informação de tipos em execução → [[Reflection]]
- Prévia do type switch → [[Controle de Fluxo]]

**Sistemas Operacionais**
- [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]] — `sendfile` evita copiar dados para o espaço de usuário; `io.Copy` descobre se pode usá-la via interfaces opcionais
- [[A Linguagem C]] — uniões etiquetadas como o equivalente manual (e inseguro) do type switch

**Nginx**
- [[Conexões, Workers e Keep-Alive]] — servidores de alto desempenho (Nginx com `sendfile on;`, Go com `io.Copy` sobre `*net.TCPConn`) usam a mesma syscall para servir arquivos estáticos

---

⬅️ **Anterior:** [[Composição vs Herança]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Generics]] ➡️
