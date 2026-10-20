---
tags:
  - go
  - go/fundamentos
aliases:
  - Operadores aritméticos
  - Operadores de bits
  - Overflow em Go
  - Bitwise em Go
ordem: 8
modulo: "02 - Fundamentos"
---
# Operações Aritméticas

> **Nota 8 da trilha de Go.** Agora que conhecemos os tipos numéricos, vamos operar sobre eles: os operadores aritméticos, as regras de **divisão inteira e resto**, o **overflow silencioso**, os **operadores de bits** (muito usados em sistemas, redes e permissões), a precedência, e os pacotes `math`, `math/bits` e `math/rand/v2`. No final, descemos ao nível do processador para ver como cada operação vira instrução de máquina, incluindo o que realmente acontece numa divisão por zero.
>
> **Pré-requisitos:** [[Tipos de Dados]].

---

## 🧭 De onde viemos

Em [[Tipos de Dados]] vimos que cada tipo numérico tem um tamanho fixo e uma faixa de valores, que inteiros com sinal usam complemento de dois e que floats seguem IEEE 754. A pergunta agora é: **o que acontece quando operamos sobre eles**, especialmente nas bordas (quando o resultado não cabe, quando dividimos por zero, quando os sinais são negativos)?

---

## ➕ 1. Os operadores aritméticos

| Operador | Operação | Tipos aceitos |
| --- | --- | --- |
| `+` | soma | inteiros, floats, complexos, **strings** (concatenação) |
| `-` | subtração | inteiros, floats, complexos |
| `*` | multiplicação | inteiros, floats, complexos |
| `/` | divisão | inteiros (divisão inteira), floats, complexos |
| `%` | resto | **só inteiros** |

```go
a, b := 17, 5
fmt.Println(a+b, a-b, a*b, a/b, a%b)   // 22 12 85 3 2

fmt.Println(17.0 / 5.0)                // 3.4
fmt.Println("Olá, " + "Go")            // Olá, Go
```

### 1.1 Os dois operandos precisam ter o mesmo tipo

Go não converte automaticamente, nem entre `int` e `int64`:

```go
var a int = 10
var b int64 = 3
fmt.Println(a + b)          // ❌ invalid operation: a + b (mismatched types int and int64)
fmt.Println(int64(a) + b)   // ✅

var preco float64 = 9.9
qtd := 3
total := preco * qtd             // ❌ mismatched types float64 and int
total := preco * float64(qtd)    // ✅
total := preco * 3               // ✅ 3 é uma constante sem tipo: vira float64
```

A exceção são as **constantes sem tipo**, que se adaptam ao outro operando (ver [[Variáveis e constantes]]).

---

## ➗ 2. Divisão inteira e resto

### 2.1 A divisão inteira trunca em direção a zero

Saída real:

```go
fmt.Println(7/2, -7/2, 7/-2)    // 3 -3 -3
```

`-7/2` é `-3`, e não `-4`. Go descarta a parte fracionária (trunca em direção a zero), como C e Java. Python, ao contrário, arredonda para baixo (`-7 // 2 == -4`).

### 2.2 O resto tem o sinal do dividendo

```go
fmt.Println(7%3, -7%3, 7%-3)    // 1 -1 1
```

A regra que amarra divisão e resto é sempre: **`a == (a/b)*b + a%b`**. Com `a = -7, b = 3`: `(-2)*3 + (-1) = -7` ✅.

> ⚠️ **Armadilha clássica:** testar se um número é ímpar com `n%2 == 1` falha para negativos (`-3 % 2 == -1`). Use `n%2 != 0`.

> 💡 Para um "módulo matemático" sempre positivo (útil em índices circulares, como um buffer em anel): `((a % n) + n) % n`.

### 2.3 Divisão por zero

O comportamento depende do tipo:

```go
// INTEIROS — divisor constante: erro de COMPILAÇÃO
x := 10 / 0                 // ❌ invalid operation: division by zero

// INTEIROS — divisor variável: PANIC em tempo de execução
zero := 0
x := 10 / zero              // panic: runtime error: integer divide by zero

// FLOATS — sem panic, segue IEEE 754
z := 0.0
fmt.Println(1/z, -1/z, z/z) // +Inf -Inf NaN
```

### 2.4 O caso estranho: `MinInt / -1`

Saída real:

```go
m := math.MinInt64
d := -1
fmt.Println(m/d, m%d)       // -9223372036854775808 0
```

O resultado matemático (`+9223372036854775808`) não cabe em `int64`, então ele "dá a volta" e continua `MinInt64`. A especificação de Go define esse comportamento (sem panic). Vamos ver na seção 9 que o compilador precisa de um cuidado especial para isso, porque em x86 essa divisão **trava o processador** se feita de forma ingênua.

---

## 🔼 3. `++` e `--` são comandos, não expressões

```go
x := 10
x++         // x = 11
x--         // x = 10

y := x++        // ❌ syntax error: unexpected ++ at end of statement
if x++ > 5 {}   // ❌
v[i++] = 0      // ❌
++x             // ❌ não existe forma prefixa
```

Em Go, `x++` é um **comando** (statement) que não produz valor. Isso elimina de uma vez a classe de bugs de C com expressões como `a[i++] = i++` (cujo resultado nem é definido em C).

---

## 🟰 4. Atribuição composta

Todo operador binário aritmético ou de bits tem uma versão "operar e atribuir":

```go
x := 10
x += 5     // 15     (x = x + 5)
x -= 3     // 12
x *= 2     // 24
x /= 5     // 4
x %= 3     // 1
x <<= 4    // 16
x |= 1     // 17
x &^= 16   // 1
s := "Go"
s += "!"   // "Go!"
```

---

## 💥 5. Overflow: silencioso e definido

Quando o resultado de uma operação entre **variáveis** inteiras não cabe no tipo, Go **não** gera erro nem panic: o valor "dá a volta" (*wrap-around*), como um hodômetro. Saída real:

```go
var x int8 = 127
x++
var y uint8 = 0
y--
fmt.Println(x, y)    // -128 255
```

```
int8: 127 + 1

  0111_1111   ( 127)
+ 0000_0001   (   1)
───────────
  1000_0000   (-128)   ← o "vai um" chegou no bit de sinal

É aritmética MÓDULO 2^N: o resultado é sempre (a + b) mod 256, reinterpretado.
```

> 💡 **Wrap-around (aritmética modular):** comportamento em que o resultado é calculado módulo 2ᴺ (N = número de bits). Para inteiros sem sinal é a aritmética modular pura; para inteiros com sinal é a mesma coisa, só que os bits resultantes são interpretados em complemento de dois.

Comparação com outras linguagens:

| Linguagem | Overflow de inteiro com sinal |
| --- | --- |
| **Go** | Wrap-around **definido pela especificação** |
| C / C++ | **Comportamento indefinido** (o compilador pode assumir que nunca acontece e "otimizar" seu código de formas surpreendentes) |
| Java | Wrap-around definido |
| Rust | Panic em modo debug; wrap-around em release |
| Python | Não há overflow (inteiros de tamanho arbitrário) |
| PHP | Vira `float` silenciosamente |

Lembre: overflow de **constantes** é pego em compilação ([[Variáveis e constantes]]). Só operações com variáveis dão a volta silenciosamente.

### 5.1 Detectando overflow

```go
import "math/bits"

// Soma com "vai um" (carry): útil para detectar overflow em uint64
soma, carry := bits.Add64(math.MaxUint64, 1, 0)
fmt.Println(soma, carry)     // 0 1   ← carry = 1 significa que estourou

// Multiplicação com resultado de 128 bits (hi:lo)
hi, lo := bits.Mul64(a, b)   // hi != 0 → o produto não coube em 64 bits

// Teste manual para int
func somaSegura(a, b int) (int, bool) {
	c := a + b
	if (c > a) == (b > 0) {
		return c, true
	}
	return 0, false          // estourou
}
```

> 💡 Para números maiores que 64 bits, use `math/big` (`big.Int`, `big.Float`, `big.Rat`), que tem precisão arbitrária. É o que bibliotecas de criptografia usam. Desde o Go 1.27, `big.Int` tem o método `Divide` com modos de arredondamento (`Trunc`, `Floor`, `Round`, `Ceil`).

---

## 🧮 6. Operadores de bits

Os operadores de bits trabalham diretamente nos bits dos inteiros. São fundamentais em programação de sistemas: flags, permissões, máscaras de rede, protocolos binários, hashes.

| Operador | Nome | Regra por bit |
| --- | --- | --- |
| `&` | AND | 1 se **os dois** bits são 1 |
| `\|` | OR | 1 se **pelo menos um** é 1 |
| `^` (binário) | XOR | 1 se os bits são **diferentes** |
| `&^` | AND NOT (*bit clear*) | 1 se o bit da esquerda é 1 **e** o da direita é 0 |
| `^` (unário) | NOT | inverte todos os bits |
| `<<` | deslocamento à esquerda | empurra os bits para a esquerda, entrando zeros |
| `>>` | deslocamento à direita | empurra para a direita (com sinal: repete o bit de sinal) |

Saída real:

```go
a := 0b1100_1010   // 202
b := 0b1010_1100   // 172

fmt.Printf("%08b\n", a&b)       // 10001000   AND
fmt.Printf("%08b\n", a|b)       // 11101110   OR
fmt.Printf("%08b\n", a^b)       // 01100110   XOR
fmt.Printf("%08b\n", a&^b)      // 01000010   AND NOT: liga em a só o que NÃO está em b
fmt.Printf("%08b\n", ^uint8(a)) // 00110101   NOT
```

```
         1100_1010   (a)
   &^    1010_1100   (b)
         ─────────
         0100_0010   ← cada bit 1 de b "apaga" o bit correspondente de a
```

> 💡 `&^` é exclusivo de Go. Equivale a `a & (^b)` e se chama *bit clear* porque "limpa" em `a` os bits que estão ligados em `b`. Note que Go usa `^` tanto para XOR (binário) quanto para NOT (unário); C usa `~` para o NOT.

### 6.1 Deslocamentos (shifts)

Saída real:

```go
fmt.Println(1<<3, 64>>2, -16>>2, uint8(200)>>1)   // 8 16 -4 100
```

- `x << n` equivale a `x * 2ⁿ` (enquanto couber)
- `x >> n` equivale a `x / 2ⁿ` arredondado para baixo
- Para inteiros **com sinal**, `>>` é **aritmético**: preserva o sinal (`-16 >> 2 == -4`)
- Para **sem sinal**, `>>` é **lógico**: entram zeros pela esquerda

Regras do deslocamento:

```go
var n int = -1
x := 1 << n          // panic: runtime error: negative shift amount

var u uint8 = 1
y := u << 10         // 0: deslocar uma VARIÁVEL além do tamanho zera o valor (sem erro)
z := uint8(1) << 10  // ❌ com constante, é erro de compilação: 1024 não cabe em uint8
```

### 6.2 Receitas práticas com bits

```go
type Flag uint8

const (
	Leitura  Flag = 1 << iota   // 0000_0001
	Escrita                     // 0000_0010
	Execucao                    // 0000_0100
)

p := Leitura | Escrita         // ligar várias flags       → 0000_0011
p |= Execucao                  // ligar uma flag            → 0000_0111
p &^= Escrita                  // desligar uma flag         → 0000_0101
p ^= Leitura                   // inverter uma flag         → 0000_0100
temExec := p&Execucao != 0     // testar uma flag           → true

// É potência de 2? (um único bit ligado)
func ehPotenciaDe2(n uint) bool { return n != 0 && n&(n-1) == 0 }
//   8 = 1000,  7 = 0111 → 1000 & 0111 = 0 ✅

// Isolar o bit menos significativo ligado
lowbit := n & -n

// Extrair um campo de bits: bits 4..7 de um byte
campo := (b >> 4) & 0x0F
```

Exemplo de redes (você estudou máscaras em [[Máscaras de rede CIDR]] e [[Determinando Rede, Host e Broadcast]]):

```go
ip   := uint32(192)<<24 | uint32(168)<<16 | uint32(1)<<8 | 77   // 192.168.1.77
mask := ^uint32(0) << (32 - 24)                                  // /24 = 255.255.255.0

rede      := ip & mask      // 192.168.1.0    (AND com a máscara)
broadcast := ip | ^mask     // 192.168.1.255  (OR com a máscara invertida)
```

---

## 📐 7. Precedência

Go tem só **5 níveis** para operadores binários (C tem 15):

```
Nível 5 (primeiro):   *   /   %   <<   >>   &   &^
Nível 4:              +   -   |   ^
Nível 3:              ==  !=  <   <=   >    >=
Nível 2:              &&
Nível 1 (último):     ||

Unários (+ - ! ^ * & <-) vêm antes de todos os binários.
No mesmo nível: da esquerda para a direita.
```

```go
2 + 3*4         // 14
1 + 2<<3        // 17  → 1 + (2<<3): shift é "multiplicativo" em Go
a&mask == 0     // (a & mask) == 0   ← em C seria a & (mask == 0)! Go corrigiu isso
```

> 💡 Repare no último exemplo: em C, `&` tem precedência **menor** que `==`, uma decisão histórica que causa bugs até hoje. Go colocou `&` junto com `*`, que é o que todo mundo espera.

> ✅ Na dúvida, use parênteses. O `gofmt` ajuda visualmente: ele remove os espaços ao redor dos operadores de maior precedência (`2 + 3*4`).

---

## 📚 8. Pacotes de apoio

### 8.1 Funções embutidas `min` e `max` (Go 1.21+)

Saída real:

```go
fmt.Println(min(3, 1, 2), max(2.5, 1.0), min("b", "a"))   // 1 2.5 a
```

Aceitam qualquer quantidade de argumentos de um tipo **ordenável** (números e strings).

### 8.2 `math`

```go
math.Abs(-3.2)          // 3.2
math.Sqrt(2)            // 1.4142135623730951
math.Pow(2, 10)         // 1024   (não existe operador ** em Go)
math.Mod(7.5, 2)        // 1.5    (o % só funciona com inteiros)
math.Log(x), math.Log2(x), math.Log10(x), math.Exp(x)
math.Sin(x), math.Cos(x), math.Atan2(y, x)

// Arredondamentos (saída real com 2.5 e -2.5)
math.Round(2.5)         // 3   (meio vai para longe do zero)
math.Round(-2.5)        // -3
math.RoundToEven(2.5)   // 2   (arredondamento bancário)
math.Floor(-2.5)        // -3  (para baixo)
math.Ceil(-2.5)         // -2  (para cima)
math.Trunc(-2.5)        // -2  (em direção a zero)

math.MaxInt, math.MinInt64, math.MaxUint32, math.MaxFloat64, math.SmallestNonzeroFloat64
math.Inf(1), math.NaN(), math.IsNaN(x), math.IsInf(x, 0)
```

### 8.3 `math/bits` — instruções de CPU com nome bonito

Saída real:

```go
bits.OnesCount64(0xFF)      // 8   quantos bits ligados (popcount)
bits.LeadingZeros64(1)      // 63  zeros à esquerda
bits.TrailingZeros64(8)     // 3   zeros à direita
bits.Len64(255)             // 8   bits necessários para representar
bits.RotateLeft32(x, 5)     // rotação (os bits que saem entram do outro lado)
bits.ReverseBytes32(x)      // troca a ordem dos bytes (endianness)
```

O compilador transforma essas funções diretamente em instruções do processador (`POPCNT`, `LZCNT`, `TZCNT`, `ROL`, `BSWAP` em x86).

### 8.4 Números aleatórios: `math/rand/v2` e `crypto/rand`

```go
import "math/rand/v2"

rand.IntN(100)          // [0, 100)
rand.Float64()          // [0.0, 1.0)
rand.N(10*time.Second)  // genérica: funciona com qualquer tipo inteiro (até Duration)
rand.Shuffle(len(s), func(i, j int) { s[i], s[j] = s[j], s[i] })
rand.Perm(5)            // permutação aleatória de 0..4

// Determinístico (testes, simulações reproduzíveis)
r := rand.New(rand.NewPCG(42, 1024))
r.IntN(100)             // sempre o mesmo resultado com a mesma semente
r.N(100)                // Go 1.27+: N também como método genérico de *Rand
```

> 💡 O pacote `math/rand/v2` (Go 1.22) substituiu o antigo `math/rand`: não precisa mais de `rand.Seed`, a fonte global é segura para concorrência e tem algoritmos melhores (PCG, ChaCha8).

> ⚠️ `math/rand` **não** serve para segurança (tokens, senhas, chaves, IDs de sessão). Para isso use `crypto/rand`, que lê a fonte de entropia do kernel (`getrandom` no Linux):
> ```go
> import "crypto/rand"
> token := make([]byte, 32)
> rand.Read(token)              // 32 bytes imprevisíveis
> s := rand.Text()              // Go 1.24+: string aleatória segura (base32)
> ```

---

## 🔬 9. Por dentro: do operador à instrução de máquina

### 9.1 Inteiros viram instruções da ALU

```
Go            x86-64             o que a CPU faz
──────        ────────           ───────────────────────────────────────
a + b    →    ADDQ  BX, AX       soma na ALU e atualiza as flags
a - b    →    SUBQ  BX, AX
a * b    →    IMULQ BX, AX
a / b    →    IDIVQ BX           divide RDX:RAX por BX → quociente em RAX, resto em RDX
a & b    →    ANDQ  BX, AX
a << 3   →    SHLQ  $3, AX
```

> 💡 **ALU (Arithmetic Logic Unit):** o circuito da CPU que executa operações aritméticas e lógicas sobre os registradores. Depois de cada operação, ela atualiza o registrador de **flags** (zero, sinal, overflow, carry), que as instruções de desvio usam em seguida. Você viu a CPU por dentro em [[Processadores]].

### 9.2 O que realmente acontece na divisão por zero

Uma explicação comum (e errada para amd64) é que "a CPU gera uma exceção, o kernel manda `SIGFPE` e o runtime converte em panic". Compilando uma função de divisão e olhando o assembly gerado (`go build -gcflags=-S`, saída real resumida):

```go
//go:noinline
func div(a, b int) int { return a / b }
```

```
main.div:
    TESTQ  BX, BX              ; b == 0 ?
    JEQ    panico              ;   → pula para o panic
    CMPQ   BX, $-1             ; b == -1 ?
    JEQ    caso_especial       ;   → trata MinInt / -1 sem usar IDIV
    ...
    IDIVQ  BX                  ; divisão de verdade
    RET
panico:
    CALL   runtime.panicdivide ; "integer divide by zero"
```

Ou seja, o compilador de Go **insere verificações explícitas antes do `IDIV`**:

1. **`b == 0`** → chama `runtime.panicdivide` diretamente (não depende de sinal do SO)
2. **`b == -1`** → calcula `-a` sem usar `IDIV`, porque em x86 a instrução `IDIV` com `MinInt64 / -1` gera a exceção `#DE` (o resultado não cabe), e a especificação de Go diz que esse caso deve dar a volta sem erro

> 💡 Essas checagens custam quase nada: o *branch predictor* da CPU aprende rapidamente que o divisor quase nunca é zero. E quando o divisor é uma **constante** diferente de 0 e −1, nenhuma checagem é gerada.

### 9.3 Otimizações: o compilador evita instruções caras

`IDIV` é uma das instruções mais lentas (dezenas de ciclos). Quando o divisor é constante, o compilador troca a divisão por operações mais baratas (*strength reduction*). Saída real:

```
a / 8   (a é int, com sinal)          a / 7
─────────────────────────────         ─────────────────────────────────────
SARQ  $63, AX   ; AX = sinal(a)       SARQ  $63, AX
SHRQ  $61, AX   ; ajuste: +7 se a<0   ...
ADDQ  ...                             IMULQ CX       ; multiplica por um
SARQ  $3, AX    ; desloca 3 = /8                     ; "número mágico"
                                      SARQ  $1, DX   ; ≈ a * (2^65/7) >> 65
```

- **Divisão por potência de 2** vira deslocamento. Para inteiros com sinal, é preciso um ajuste para negativos (porque `>>` arredonda para baixo e a divisão de Go trunca em direção a zero). Para `uint`, seria só `SHRQ $3`
- **Divisão por outras constantes** vira **multiplicação por um inverso** pré-calculado seguida de deslocamento: muito mais rápido que `IDIV`
- `a % 8` com `uint` vira `ANDQ $7`
- `a * 8` vira `SHLQ $3` (ou `LEAQ`)

> ✅ Lição prática: escreva `x / 8` e `x % 2`, nunca `x >> 3` "para otimizar". O compilador já faz isso, e o código fica legível. Use `>>` quando a intenção é **manipular bits**.

### 9.4 Floats usam as unidades SSE/AVX

```
a + b (float64)   →   ADDSD  X1, X0     (Add Scalar Double, registradores XMM)
a * b             →   MULSD  X1, X0
a / b             →   DIVSD  X1, X0     ← divisão por 0.0 NÃO gera exceção: dá ±Inf/NaN
```

Em amd64, Go usa as instruções SSE2 (e não a antiga FPU x87) para ponto flutuante. A unidade de ponto flutuante segue IEEE 754, que define `1/0 = +Inf` e `0/0 = NaN` com as exceções mascaradas, por isso não há panic.

### 9.5 Overflow é gratuito

Como a CPU já faz aritmética módulo 2⁶⁴ nos registradores, o wrap-around de Go **não custa nada**: é o comportamento natural do hardware. Linguagens que detectam overflow (Rust em debug, Swift) precisam emitir uma instrução de checagem de flag (`JO`, *jump if overflow*) depois de cada operação.

---

## ⚠️ 10. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| Misturar tipos numéricos | `preco * qtd` (float64 × int) | `preco * float64(qtd)` |
| Esperar arredondamento na divisão inteira | `-7/2 == -4`? Não, é `-3` | Saiba que trunca em direção a zero |
| Teste de ímpar com `== 1` | `-3%2 == 1` é falso | `n%2 != 0` |
| Média com inteiros | `(a+b)/2` pode estourar e trunca | `a + (b-a)/2`; ou converta para float |
| Overflow silencioso em contadores | `int32` que passa de 2,1 bilhões | Tipo maior ou `math/bits` |
| Divisão por variável zero | panic | Valide o divisor antes |
| `math/rand` para segredos | tokens previsíveis | `crypto/rand` |
| Precedência de `&` "à moda C" | — | Em Go `a&b == 0` já é `(a&b) == 0` |
| `%` com floats | `7.5 % 2` não compila | `math.Mod(7.5, 2)` |

---

## 📋 Resumo

| Tópico | Ponto-chave |
| --- | --- |
| Operandos | Mesmo tipo obrigatório (constantes sem tipo se adaptam) |
| Divisão inteira | Trunca em direção a zero; `a == (a/b)*b + a%b` |
| Resto | Sinal do dividendo |
| Divisão por zero | Constante: erro de compilação; variável inteira: panic; float: ±Inf/NaN |
| `++` / `--` | Comandos, sem forma prefixa, sem valor |
| Overflow | Wrap-around definido pela especificação, custo zero |
| Bits | `&`, `\|`, `^`, `&^` (exclusivo de Go), `<<`, `>>` |
| `>>` | Aritmético para signed, lógico para unsigned |
| Precedência | 5 níveis; `&` e `<<` são "multiplicativos" |
| Pacotes | `math`, `math/bits`, `math/big`, `math/rand/v2`, `crypto/rand` |
| Por dentro | Checagem explícita antes do `IDIV`; strength reduction; SSE para floats |

---

## 🔗 Conexões

**Dentro de Go**
- Os tipos sobre os quais operamos → [[Tipos de Dados]]
- Comparações e lógica booleana → [[Operadores de Comparação e Lógicos]]
- Constantes com precisão arbitrária e `iota` para flags → [[Variáveis e constantes]]
- Panics e como recuperá-los → [[Defer, Panic e Recover]]
- Medir se uma otimização vale a pena → [[Benchmarks]]

**Sistemas Operacionais**
- [[Bits e Bytes]] — complemento de dois, que explica o wrap-around e o deslocamento aritmético
- [[Processadores]] — ALU, registradores, flags e o pipeline que torna as checagens de divisão quase gratuitas
- [[Proteção]] — as permissões `rwx` como bits combinados com `|` e testados com `&`

**Redes**
- [[Máscaras de rede CIDR]], [[Determinando Rede, Host e Broadcast]] e [[Cálculo de Sub-rede]] — rede = IP AND máscara; broadcast = IP OR NOT máscara
- [[Converter binário em decimal]] — o mesmo raciocínio de bits por trás de `<<` e `|`

---

⬅️ **Anterior:** [[Tipos de Dados]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Operadores de Comparação e Lógicos]] ➡️
