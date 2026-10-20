---
tags:
  - go
  - go/fundamentos
aliases:
  - Tipos em Go
  - Sistema de tipos do Go
  - Conversão de tipos
  - Type alias
ordem: 7
modulo: "02 - Fundamentos"
---
# Tipos de Dados

> **Nota 7 da trilha de Go.** O tipo de um valor diz **quanta memória ele ocupa**, **como os bits devem ser interpretados** e **quais operações são permitidas**. Nesta nota vamos conhecer o mapa completo do sistema de tipos de Go, estudar a fundo os **tipos básicos** (booleanos, inteiros, ponto flutuante, complexos), entender a diferença entre **tipo definido** e **alias**, e dominar as **conversões** (que em Go são sempre explícitas). Os tipos compostos (strings, slices, maps, structs...) ganham notas próprias no módulo 03.
>
> **Pré-requisitos:** [[Variáveis e constantes]].

---

## 🧭 De onde viemos

Em [[Variáveis e constantes]] vimos que toda variável tem um tipo fixo e que constantes sem tipo se adaptam ao contexto. Usamos `int`, `float64`, `string` e `bool` de forma intuitiva. Agora vamos responder: **quais tipos existem, como são representados na memória e como converter entre eles**.

> 💡 **Tipo:** um conjunto de valores possíveis junto com as operações válidas sobre eles. `uint8` é o conjunto {0, 1, ..., 255} com as operações aritméticas; `bool` é {false, true} com as operações lógicas. Para o compilador, o tipo também define o **tamanho** em bytes e o **alinhamento** na memória.

---

## 🗺️ 1. O mapa do sistema de tipos

```
                               TIPOS EM GO
                                    │
        ┌───────────────────────────┼──────────────────────────────┐
        ▼                           ▼                              ▼
     BÁSICOS                    COMPOSTOS                    COMPORTAMENTO
  (valores simples)       (construídos de outros)          (definem métodos)
        │                           │                              │
  bool                        array     [N]T                   interface
  string                      slice     []T                    (error, any,
  inteiros:                   map       map[K]V                 io.Reader...)
    int  int8 int16           struct    struct{...}
    int32 int64               ponteiro  *T
    uint uint8 uint16         função    func(...) ...
    uint32 uint64 uintptr     channel   chan T
  float32 float64
  complex64 complex128
  aliases: byte = uint8, rune = int32
```

| Categoria | Nota onde é estudado |
| --- | --- |
| Básicos (numéricos e `bool`) | **esta nota** |
| `string` | [[Strings em Profundidade]] |
| array e slice | [[Arrays e Slices]] |
| map | [[Maps]] |
| struct | [[Structs]] |
| ponteiro | [[Ponteiros]] |
| função | [[Funções — Fundamentos]] |
| interface | [[Interfaces]] |
| channel | [[Channels]] |
| tipos genéricos | [[Generics]] |

---

## ✅ 2. Booleano: `bool`

Só dois valores: `true` e `false`. Zero value: `false`.

```go
ativo := true
maior := idade >= 18      // comparações produzem bool
```

Go **não tem "truthy" e "falsy"** como PHP, Ruby, Python ou JavaScript:

```go
if 1 { }          // ❌ non-boolean condition in if statement
if "" { }         // ❌
if usuario { }    // ❌ (mesmo sendo ponteiro)

if contador != 0 { }    // ✅ sempre compare explicitamente
if nome != "" { }       // ✅
if usuario != nil { }   // ✅
```

> 💡 Um `bool` ocupa **1 byte**, não 1 bit, porque a menor unidade endereçável da memória é o byte (ver [[Bits e Bytes]]). Para guardar muitos booleanos de forma compacta, usa-se um inteiro como conjunto de bits (flags), como vimos com `iota` em [[Variáveis e constantes]].

---

## 🔢 3. Inteiros

### 3.1 Com sinal e sem sinal

| Tipo | Tamanho | Faixa |
| --- | --- | --- |
| `int8` | 1 byte | −128 a 127 |
| `int16` | 2 bytes | −32.768 a 32.767 |
| `int32` | 4 bytes | ≈ −2,1 bilhões a 2,1 bilhões |
| `int64` | 8 bytes | ≈ −9,2 × 10¹⁸ a 9,2 × 10¹⁸ |
| `uint8` (= `byte`) | 1 byte | 0 a 255 |
| `uint16` | 2 bytes | 0 a 65.535 |
| `uint32` | 4 bytes | 0 a ≈ 4,3 bilhões |
| `uint64` | 8 bytes | 0 a ≈ 1,8 × 10¹⁹ |
| `int` | 4 ou 8 bytes | depende da plataforma (8 em 64 bits) |
| `uint` | 4 ou 8 bytes | depende da plataforma |
| `uintptr` | tamanho de um ponteiro | guarda um endereço como número |

O pacote `math` tem constantes com os limites (saída real em 64 bits):

```go
fmt.Println(math.MaxInt8, math.MinInt8, math.MaxUint8, math.MaxInt64, uint64(math.MaxUint64))
// 127 -128 255 9223372036854775807 18446744073709551615
```

> 💡 **Com sinal (signed) vs sem sinal (unsigned):** um inteiro com sinal reserva a capacidade de representar negativos usando **complemento de dois**; um sem sinal usa todos os bits para a magnitude, por isso vai até o dobro. Você estudou complemento de dois em [[Bits e Bytes]]: em 8 bits, `-1` é `1111_1111`, e `-128` é `1000_0000`.

```
int8 = 8 bits em complemento de dois

  0111_1111 =  127   (maior positivo)
  0000_0001 =    1
  0000_0000 =    0
  1111_1111 =   -1
  1000_0000 = -128   (menor negativo)
  └ bit de sinal
```

### 3.2 `int`: o inteiro padrão

`int` tem o tamanho da **palavra** (word) da arquitetura: 64 bits em amd64/arm64, 32 bits em plataformas de 32 bits. É o tipo de:

- índices e tamanhos (`len()` e `cap()` retornam `int`)
- contadores e laços
- constantes inteiras usadas sem contexto (`x := 10`)

> 💡 **Palavra (word):** a quantidade de bits que a CPU processa naturalmente de uma vez, igual ao tamanho dos registradores de uso geral. Você viu isso em [[Processadores]] ao estudar registradores.

> ⚠️ `int` e `int64` são **tipos diferentes**, mesmo tendo o mesmo tamanho em 64 bits. `var a int = 1; var b int64 = a` não compila.

### 3.3 `byte` e `rune`

São **aliases** (nomes alternativos para o mesmo tipo), criados para deixar a intenção clara:

| Alias | Tipo real | Usado para |
| --- | --- | --- |
| `byte` | `uint8` | dados brutos: conteúdo de arquivos, pacotes de rede, `[]byte` |
| `rune` | `int32` | um **caractere Unicode** (o *code point*) |

```go
var b byte = 'A'     // 65
var r rune = 'ç'     // 231
```

Veremos por que `rune` existe em [[Strings em Profundidade]] (é a resposta ao mistério do "João" com 5 letras).

### 3.4 `uintptr`

Um inteiro grande o suficiente para guardar um endereço de memória. Existe para casos de baixo nível (`unsafe`, syscalls, interoperabilidade com C). Um `uintptr` **não é um ponteiro**: o GC não o reconhece como referência, então o objeto pode ser coletado ou movido. Veremos em [[unsafe e cgo]].

### 3.5 Quando usar cada inteiro

| Situação | Tipo |
| --- | --- |
| Contadores, índices, tamanhos, uso geral | `int` |
| IDs de banco, timestamps Unix, tamanhos de arquivo | `int64` |
| Dados binários | `byte` |
| Caracteres Unicode | `rune` |
| Flags com bits, hashes, operações de bits | `uint32` / `uint64` |
| Formatos binários fixos (protocolos de rede, arquivos) | o tamanho exato (`uint16`, `uint32`...) |
| Dinheiro | `int64` em centavos (nunca float!) |

> ⚠️ Não use `uint` só porque "o valor nunca é negativo". Subtrair dois `uint` pode dar a volta para um número gigante (`uint(0) - 1` = 18446744073709551615). A recomendação da comunidade é usar `int` para quantidades e reservar `uint` para bits.

---

## 🌊 4. Ponto flutuante

| Tipo | Tamanho | Precisão aproximada | Faixa aproximada |
| --- | --- | --- | --- |
| `float32` | 4 bytes | ~7 dígitos decimais | ±3,4 × 10³⁸ |
| `float64` | 8 bytes | ~15–16 dígitos decimais | ±1,8 × 10³⁰⁸ |

`float64` é o padrão (`x := 3.14` é `float64`). Use `float32` só por economia de memória em grandes volumes (gráficos, ML) ou por exigência de uma API.

### 4.1 IEEE 754 por dentro

Os dois tipos seguem o padrão **IEEE 754**:

```
float64 (64 bits)
┌───┬─────────────┬──────────────────────────────────────────────────────┐
│ S │  expoente   │                   mantissa (fração)                  │
│ 1 │   11 bits   │                       52 bits                        │
└───┴─────────────┴──────────────────────────────────────────────────────┘
valor = (−1)^S × 1,mantissa × 2^(expoente − 1023)

Exemplo real: os bits de 1.0
math.Float64bits(1.0) = 0 01111111111 0000...0000
                        S  expoente=1023 mantissa=0  → 1,0 × 2^0 = 1.0
```

> 💡 **Ponto flutuante:** representação de números reais em notação científica binária. A vírgula "flutua" conforme o expoente. A consequência é que só uma quantidade finita de números é representável exatamente; os demais são aproximados para o mais próximo.

### 4.2 A armadilha do 0.1 + 0.2

`0.1` não tem representação exata em binário (assim como 1/3 não tem em decimal). Com **variáveis** (saída real):

```go
a, b := 0.1, 0.2
fmt.Println(a+b == 0.3, a+b)   // false 0.30000000000000004
```

Mas com **constantes** o resultado é diferente (saída real):

```go
fmt.Println(0.1+0.2 == 0.3)    // true   ← !
```

Por quê? Lembra que constantes sem tipo têm **precisão arbitrária** ([[Variáveis e constantes]])? O compilador calcula `0.1 + 0.2` com matemática exata e só depois converte para `float64`. Com variáveis, cada valor já foi arredondado para `float64` antes da soma.

Regras práticas:

```go
// ❌ nunca compare floats com ==
if a+b == 0.3 { }

// ✅ compare com tolerância
const eps = 1e-9
if math.Abs((a+b)-0.3) < eps { }

// ✅ dinheiro em inteiros (centavos)
precoCentavos := int64(1999)    // R$ 19,99
```

### 4.3 Valores especiais

```go
fmt.Println(math.Inf(1), -math.Inf(1), math.NaN() == math.NaN())
// +Inf -Inf false

x := 0.0
fmt.Println(1/x, -1/x, x/x)     // +Inf -Inf NaN   (sem panic!)
math.IsNaN(v)                   // a única forma correta de testar NaN
```

> 💡 **NaN (Not a Number):** resultado de operações indefinidas como `0/0` ou `√-1`. Pela norma IEEE 754, `NaN` é diferente de tudo, **inclusive de si mesmo**. Isso tem consequências até em maps (veremos em [[Maps]]).

### 4.4 Precisão do `float32`

```go
var x float32 = 16777216   // 2^24
fmt.Println(x+1 == x)      // true   ← o float32 não consegue representar 16777217
```

Com 24 bits de mantissa efetiva, `float32` perde a capacidade de representar todos os inteiros acima de 2²⁴. `float64` aguenta até 2⁵³.

---

## 🔵 5. Complexos

```go
c := complex(3, 4)          // complex128: 3+4i
d := 2 + 3i                 // literal imaginário
fmt.Println(real(c), imag(c))   // 3 4
fmt.Println(cmplx.Abs(c))       // 5 (pacote math/cmplx)
```

| Tipo | Partes |
| --- | --- |
| `complex64` | duas `float32` |
| `complex128` | duas `float64` (padrão) |

São raros fora de computação científica e processamento de sinais, mas fazem parte da linguagem.

---

## 🏷️ 6. Criando seus próprios tipos

### 6.1 Tipo definido: `type Nome TipoBase`

```go
type Celsius float64
type Fahrenheit float64
type UserID int64
```

> 💡 **Tipo definido (defined type):** um tipo **novo e distinto**, criado a partir de outro. Ele tem a mesma representação na memória e as mesmas operações do tipo base (chamado **tipo subjacente**, *underlying type*), mas o compilador o trata como incompatível com qualquer outro tipo.

```go
var c Celsius = 100
var f Fahrenheit = c          // ❌ cannot use c (variable of type Celsius) as Fahrenheit value
var g float64 = c             // ❌ mesmo sendo float64 "por baixo"

f = Fahrenheit(c*9/5 + 32)    // ✅ conversão explícita
fmt.Println(f)                // 212
```

Por que isso é útil?

1. **Segurança:** impede misturar coisas diferentes com a mesma representação (temperaturas em escalas diferentes, IDs de usuário com IDs de pedido, metros com pés)
2. **Métodos:** só tipos definidos podem ganhar métodos (veremos em [[Métodos e Receptores]])

```go
func (c Celsius) ParaFahrenheit() Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}
```

A biblioteca padrão usa isso o tempo todo: `time.Duration` é `int64`, `http.HandlerFunc` é uma função, `os.FileMode` é `uint32`.

> 💡 **O caso da sonda Mars Climate Orbiter (1999):** a NASA perdeu uma sonda porque um módulo calculava em libras-força·segundo e outro esperava newtons·segundo. Tipos definidos distintos (`type NewtonSeg float64`, `type LibraSeg float64`) transformariam esse erro em erro de compilação.

### 6.2 Alias: `type Nome = Outro`

```go
type MeuInt = int     // repare no "="
```

> 💡 **Alias de tipo:** apenas **outro nome** para um tipo existente. Não cria tipo novo: `MeuInt` e `int` são exatamente o mesmo tipo, intercambiáveis sem conversão.

Saída real:

```go
var mi MeuInt = 5
var i int = mi             // ✅ sem conversão
fmt.Printf("%T\n", mi)     // int    ← o nome do alias some
```

Aliases existem principalmente para **migração gradual de código** (mover um tipo de um pacote para outro sem quebrar quem usa) e para os nomes da linguagem: `byte = uint8`, `rune = int32`, `any = interface{}`. Desde o Go 1.24 também podem ter parâmetros de tipo (aliases genéricos, ver [[Generics]]).

### 6.3 Comparando

| | Tipo definido `type A B` | Alias `type A = B` |
| --- | --- | --- |
| Cria tipo novo? | Sim | Não |
| Precisa de conversão para B? | Sim | Não |
| Pode ter métodos próprios? | Sim | Não (os métodos seriam do próprio B) |
| `%T` mostra | `main.A` | o nome de B |
| Uso típico | Modelar o domínio | Migração e compatibilidade |

---

## 🔄 7. Conversões — sempre explícitas

Go **nunca** converte tipos automaticamente (exceto as constantes sem tipo, que vimos se adaptarem). A sintaxe é `T(valor)`.

### 7.1 Entre numéricos

Saídas reais:

```go
// float → int: TRUNCA em direção a zero (não arredonda)
fmt.Println(int(3.99), int(-3.99))      // 3 -3
fmt.Println(int(math.Round(3.99)))      // 4   ← para arredondar, use math.Round

// int maior → int menor: mantém só os bits de baixo (sem erro!)
var grande int64 = 300                  // 300 = 1_0010_1100
fmt.Println(int8(grande), uint8(grande))   // 44 44   (fica só 0010_1100)

// signed ↔ unsigned: reinterpreta os bits
var u uint = 0
u--
fmt.Println(u)                          // 18446744073709551615

// int → float: pode perder precisão em valores enormes
var n int64 = 1<<53 + 1
fmt.Println(int64(float64(n)) == n)     // false
```

> ⚠️ **Conversão de float fora da faixa para inteiro** (ex.: `int64(1e20)`) **não** dá erro e o resultado depende da implementação (em amd64 o resultado é `-9223372036854775808`). Verifique os limites antes de converter valores não confiáveis.

### 7.2 Strings e números — a armadilha do `string(int)`

```go
n := 65
s := string(n)          // ⚠️ "A", e não "65"!
```

`string(inteiro)` interpreta o número como um **code point Unicode**. Esse erro é tão comum que o `go vet` avisa:

```
conversion from int to string yields a string of one rune, not a string of digits
```

O jeito certo:

```go
s := strconv.Itoa(65)               // "65"
s := fmt.Sprint(65)                 // "65" (mais lento, mais genérico)
n, err := strconv.Atoi("65")        // 65, nil  (pode falhar: retorna erro)
f, err := strconv.ParseFloat("3.14", 64)

s := string(rune(65))               // "A" — quando você QUER o caractere, seja explícito
```

### 7.3 Strings e bytes

```go
b := []byte("olá")      // [111 108 195 161]  (UTF-8: o "á" ocupa 2 bytes)
s := string(b)          // "olá"
r := []rune("olá")      // [111 108 225]      (3 caracteres)
```

Essas conversões **copiam** os dados (strings são imutáveis, slices não). Veremos o porquê e as otimizações do compilador em [[Strings em Profundidade]].

### 7.4 Tabela-resumo de conversões

| De → Para | Como | Cuidado |
| --- | --- | --- |
| `int` → `float64` | `float64(i)` | Perde precisão acima de 2⁵³ |
| `float64` → `int` | `int(f)` | Trunca; fora da faixa é indefinido |
| `int64` → `int8` | `int8(i)` | Descarta bits altos silenciosamente |
| `int` → `uint` | `uint(i)` | Negativos viram números enormes |
| `int` → `string` (dígitos) | `strconv.Itoa(i)` | Nunca `string(i)` |
| `string` → `int` | `strconv.Atoi(s)` | Retorna erro se inválido |
| `string` ↔ `[]byte` | `[]byte(s)`, `string(b)` | Copia |
| `string` → `[]rune` | `[]rune(s)` | Copia e decodifica UTF-8 |
| Tipo definido ↔ base | `Celsius(f)`, `float64(c)` | Sem custo (mesma representação) |
| Interface → tipo concreto | `v.(T)` (type assertion) | Não é conversão; ver [[Type Assertion e Type Switch]] |

---

## 🧩 8. Regras do sistema de tipos

Três conceitos que explicam quando algo compila:

> 💡 **Tipo subjacente (underlying type):** o tipo "por baixo" de um tipo definido. O subjacente de `Celsius` é `float64`; o de `float64` é ele mesmo. Conversões são permitidas entre tipos com o mesmo subjacente.

> 💡 **Identidade de tipos:** dois tipos são idênticos se têm o mesmo nome (tipos definidos), ou se são literais de tipo com a mesma estrutura (`[]int` é idêntico a `[]int`). `Celsius` e `Fahrenheit` têm o mesmo subjacente, mas **não** são idênticos.

> 💡 **Atribuibilidade:** um valor pode ser atribuído a uma variável de tipo T sem conversão se: tem o tipo idêntico a T; ou é uma constante sem tipo representável em T; ou é `nil` e T é ponteiro/slice/map/chan/func/interface; ou T é uma interface que o valor implementa; ou (caso especial) os tipos têm o mesmo subjacente e pelo menos um deles **não** é um tipo nomeado.

O último caso explica isto:

```go
type Lista []int

var l Lista = []int{1, 2, 3}   // ✅ []int não tem nome, e o subjacente de Lista é []int
var x []int = l                // ✅ pelo mesmo motivo
```

---

## 🌀 9. `any` — o tipo que aceita tudo

```go
var v any      // any é um alias de interface{}

v = 42
v = "texto"
v = []int{1, 2}
```

`any` é uma **interface vazia**: como não exige nenhum método, todo tipo a satisfaz. Mas para usar o valor de volta como o tipo concreto, é preciso uma *type assertion* (`v.(string)`). Use com moderação: você troca a segurança do compilador por verificações em tempo de execução. Veremos em [[Interfaces]] e [[Type Assertion e Type Switch]], e a alternativa segura, [[Generics]].

---

## 📏 10. Tamanho e alinhamento

Saída real em amd64 (`unsafe.Sizeof` retorna o tamanho em bytes):

```go
unsafe.Sizeof(true)              // 1
unsafe.Sizeof(int(0))            // 8
unsafe.Sizeof(uintptr(0))        // 8
unsafe.Sizeof(complex128(0))     // 16
unsafe.Sizeof("")                // 16  (ponteiro + tamanho)
unsafe.Sizeof([]int{})           // 24  (ponteiro + len + cap)
unsafe.Sizeof(map[int]int{})     // 8   (um ponteiro para a estrutura interna)
unsafe.Sizeof(any(nil))          // 16  (ponteiro de tipo + ponteiro de dados)
```

Repare: `string`, slice, map e interface têm tamanho fixo, **independente do conteúdo**. Eles são pequenos "cabeçalhos" que apontam para os dados verdadeiros, que ficam em outro lugar da memória. Esse é um dos conceitos mais importantes do módulo 03.

> 💡 **Alinhamento:** a regra de que um valor de N bytes deve começar num endereço múltiplo de N (até o tamanho da palavra). Um `int64` fica em endereços múltiplos de 8. A CPU lê a memória em blocos alinhados; um acesso desalinhado pode custar dois acessos ou até causar exceção em algumas arquiteturas. O alinhamento cria "buracos" (padding) dentro de structs, como veremos em [[Structs]].

---

## ⚠️ 11. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| Comparar floats com `==` | `a+b == 0.3` | Tolerância (`math.Abs(x-y) < eps`) |
| Dinheiro em float | `preco := 19.99` | `int64` em centavos ou biblioteca decimal |
| `string(n)` para converter número | `string(65)` = `"A"` | `strconv.Itoa(n)` |
| `int(f)` esperando arredondamento | `int(2.9)` = 2 | `int(math.Round(f))` |
| Conversão para tipo menor | `int8(300)` = 44 | Verifique a faixa antes |
| `uint` para "nunca negativo" | `a - b` dá a volta | Use `int` |
| Misturar `int` e `int64` | `var x int64 = len(s)` | `int64(len(s))` |
| Testar NaN com `==` | `x == math.NaN()` sempre falso | `math.IsNaN(x)` |

---

## 📋 Resumo

| Tópico | Ponto-chave |
| --- | --- |
| Categorias | Básicos, compostos (array, slice, map, struct, ponteiro, func, chan) e interfaces |
| `bool` | Só `true`/`false`; sem truthy/falsy |
| Inteiros | `int8`..`int64`, `uint8`..`uint64`; `int` = palavra da CPU |
| `byte` / `rune` | Aliases de `uint8` / `int32` |
| Floats | IEEE 754; `float64` padrão; nunca compare com `==` |
| Constantes vs variáveis | Constantes são exatas; variáveis já estão arredondadas |
| Tipo definido | `type A B`: novo tipo, precisa de conversão, aceita métodos |
| Alias | `type A = B`: mesmo tipo, outro nome |
| Conversão | `T(v)`, sempre explícita; trunca, descarta bits, reinterpreta |
| String ↔ número | `strconv`, nunca `string(int)` |
| Tamanhos | Cabeçalhos de string/slice/map/interface têm tamanho fixo |

---

## 🔗 Conexões

**Dentro de Go**
- Operações sobre esses tipos → [[Operações Aritméticas]] e [[Operadores de Comparação e Lógicos]]
- Strings, bytes e runes → [[Strings em Profundidade]]
- Métodos em tipos definidos → [[Métodos e Receptores]]
- Interfaces e `any` → [[Interfaces]]
- Alinhamento e padding → [[Structs]]
- Restrições de tipo com `~T` → [[Generics]]
- `uintptr` e `unsafe` → [[unsafe e cgo]]

**Sistemas Operacionais**
- [[Bits e Bytes]] — complemento de dois, representação binária e por que o byte é a menor unidade endereçável
- [[Processadores]] — o tamanho da palavra define o tamanho de `int`, `uint` e `uintptr`; registradores de 64 bits
- [[Memória]] — o acesso alinhado à memória e o barramento explicam as regras de alinhamento
- [[Espaços de Endereçamento]] — `uintptr` guarda um endereço virtual do espaço de endereçamento do processo

**Redes**
- [[Converter binário em decimal]] e [[O que é IPv4]] — um endereço IPv4 é um `uint32`; cada octeto é um `byte`
- [[Endereço MAC]] — 6 bytes, naturalmente modelado como `[6]byte`

---

⬅️ **Anterior:** [[Variáveis e constantes]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Operações Aritméticas]] ➡️
