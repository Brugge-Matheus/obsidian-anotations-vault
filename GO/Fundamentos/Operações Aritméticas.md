---
tags:
  - go
  - go/fundamentos
---
# Operações Aritméticas

> Go tem operadores aritméticos similares a C, com diferenças importantes: sem conversão implícita entre tipos, `++`/`--` são statements (não expressões), overflow silencioso com wrap-around, e suporte nativo a operações de bits. O compilador SSA aplica otimizações como strength reduction e constant folding.
> 

---

## 1. Como o Compilador Otimiza Operações

O backend SSA (Static Single Assignment) de Go aplica várias otimizações automáticas:

```go
Constant folding — avaliado em compilação:
const x = 3 * 4   // o compilador substitui por 12 diretamente

Strength reduction — operações equivalentes mais baratas:
n * 8    → SHL n, 3 (shift left por 3 bits — mais rápido que multiply)
n / 4    → SAR n, 2 (shift aritmético direito — mais rápido que divide)
n % 8    → AND n, 7 (AND com máscara — para potências de 2)

Dead code elimination:
const debug = false
if debug { fmt.Println("debug") }  // bloco removido em compilação

Inlining:
func somar(a, b int) int { return a + b }
// O compilador pode substituir somar(x, y) por (x + y) diretamente
```

---

## 2. Operadores Aritméticos

| Operador | Funciona com | Observação |
| --- | --- | --- |
| `+` | int, float, complex, **string** | Concatenação para string |
| `-` | int, float, complex |  |
| `*` | int, float, complex |  |
| `/` | int (inteira), float, complex | Inteiro: trunca em direção a zero |
| `%` | int apenas | Sinal segue o dividendo |

```go
a, b := 17, 5

fmt.Println(a + b)   // 22
fmt.Println(a - b)   // 12
fmt.Println(a * b)   // 85
fmt.Println(a / b)   // 3  — divisão inteira (trunca, não arredonda)
fmt.Println(a % b)   // 2  — resto

// Divisão inteira trunca em direção a zero (não ao negativo infinito):
fmt.Println( 7 /  2)  //  3   (não 3.5)
fmt.Println(-7 /  2)  // -3   (não -4 — trunca em direção a zero!)
fmt.Println( 7 / -2)  // -3

// Módulo — sinal segue o DIVIDENDO (não o divisor):
fmt.Println( 7 %  3)  //  1
fmt.Println(-7 %  3)  // -1  (não 2!)
fmt.Println( 7 % -3)  //  1

// Floats
fmt.Println(17.0 / 5.0)   // 3.4 — divisão real

// Divisão por zero: comportamento DIFERENTE para int e float
// var zero int = 0
// _ = 1 / zero      // panic em runtime: integer divide by zero
_ = 1.0 / 0.0         // +Inf — IEEE 754, sem panic!
_ = 0.0 / 0.0         // NaN

// String — concatenação com +
s := "Hello" + ", " + "World"   // "Hello, World"
```

---

## 3. Incremento e Decremento — São Statements, Não Expressões

Em Go, `++` e `--` só existem como **statements independentes**:

```go
x := 10
x++   // x = 11
x--   // x = 10

// ❌ Não são expressões — diferente de C, Java, PHP
// y := x++         // erro de compilação
// if x++ > 5 {}   // erro
// arr[x++] = 1    // erro

// ✅ Apenas como statement sozinho na linha
for i := 0; i < 10; i++ {   // i++ no for é a única exceção de posição
	processar(i)
}
```

> 💡 Essa restrição elimina toda uma classe de bugs de precedência que existem em C/C++ com expressões como `a[i++] = i++`. Em Go, é impossível.
> 

---

## 4. Operadores de Atribuição Composta

```go
x := 10
x += 5    // 15
x -= 3    // 12
x *= 2    // 24
x /= 4    // 6
x %= 4    // 2

// Bitwise
x = 0b1100
x &= 0b1010   // 0b1000 = 8  (AND)
x |= 0b0011   // 0b1011 = 11 (OR)
x ^= 0b1111   // 0b0100 = 4  (XOR)
x >>= 1       // 0b0010 = 2  (shift right)
x <<= 2       // 0b1000 = 8  (shift left)
x &^= 0b0110  // 0b1000 = 8  (AND NOT — bit clear)
```

---

## 5. Operadores de Bit

```go
a := 0b1100_1010   // 202
b := 0b1010_1100   // 172

// AND — 1 apenas onde ambos têm 1
fmt.Printf("%08b\n", a & b)   // 10001000 = 136

// OR — 1 onde ao menos um tem 1
fmt.Printf("%08b\n", a | b)   // 11101110 = 238

// XOR — 1 onde são diferentes
fmt.Printf("%08b\n", a ^ b)   // 01100110 = 102

// NOT (unário) — inverte todos os bits
fmt.Printf("%b\n", ^uint8(a)) // 00110101 = 53

// AND NOT (&^) — bit clear — exclusivo do Go
// a &^ b = a & (^b) — limpa em 'a' os bits que estão em 1 em 'b'
fmt.Printf("%08b\n", a &^ b)  // 00000010 = 2

// Shift — multiplica/divide por potência de 2
n := 1
fmt.Println(n << 3)    // 8  = 1 × 2³
fmt.Println(64 >> 2)   // 16 = 64 ÷ 2²
```

### Aplicações Práticas de Bits

```go
// Flags de permissão com bitmask
const (
	FlagLeitura   uint8 = 1 << iota   // 00000001
	FlagEscrita                        // 00000010
	FlagExecucao                       // 00000100
)

p := FlagLeitura | FlagEscrita   // 00000011

// Verificar flag
temLeitura  := p&FlagLeitura != 0    // true
temExecucao := p&FlagExecucao != 0   // false

// Ativar flag
p |= FlagExecucao              // 00000111

// Desativar flag
p &^= FlagEscrita              // 00000101

// Toggle flag
p ^= FlagLeitura               // 00000100

// Verificar potência de 2 — truque clássico
func isPow2(n int) bool {
	return n > 0 && (n&(n-1)) == 0
	// n=8: 1000 & 0111 = 0000 → true
	// n=7: 0111 & 0110 = 0110 → false
}

// Arredondar para próxima potência de 2 (útil para buffers)
func nextPow2(n int) int {
	n--
	n |= n >> 1
	n |= n >> 2
	n |= n >> 4
	n |= n >> 8
	n |= n >> 16
	n |= n >> 32
	return n + 1
}
```

---

## 6. Overflow — Silencioso e Intencional

```go
// Inteiros em Go usam aritmética modular (wrap-around)
// Não há exceção, não há saturação — é especificado pela linguagem

var x int8 = 127
x++
fmt.Println(x)   // -128 — 0111_1111 + 1 = 1000_0000 = -128 em complemento de dois

var y uint8 = 0
y--
fmt.Println(y)   // 255 — underflow para o valor máximo

// Detectar overflow com math/bits
import "math/bits"

// Adição com overflow detection
sum, carry := bits.Add64(uint64(a), uint64(b), 0)
if carry != 0 {
	// overflow ocorreu
}

// Multiplicação 128-bit (resultado de uint64 × uint64)
hi, lo := bits.Mul64(a, b)

// Contagem de bits
bits.OnesCount64(0xFF)   // 8
bits.LeadingZeros64(1)   // 63
bits.TrailingZeros64(8)  // 3 (8 = 1000₂)
bits.Len64(255)          // 8 (255 precisa de 8 bits)
```

---

## 7. Avaliação Paralela em Atribuições Múltiplas

```go
// O lado direito é completamente avaliado ANTES de qualquer atribuição
a, b := 1, 2
a, b = b, a   // swap elegante — sem variável temporária
// Internamente: temp_a=b, temp_b=a; a=temp_a; b=temp_b
fmt.Println(a, b)   // 2 1

// Exemplo mais complexo
x, y := 1, 2
x, y = x+y, x-y   // x = 1+2 = 3, y = 1-2 = -1 (usa valores originais)
fmt.Println(x, y)   // 3 -1

// Usado no algoritmo de Fibonacci
a, b := 0, 1
for range 10 {
	fmt.Print(a, " ")
	a, b = b, a+b   // ambos atualizados com valores anteriores
}
// 0 1 1 2 3 5 8 13 21 34
```

---

## 8. Precedência de Operadores

```
Nível 5 (maior):  *   /   %   <<  >>  &   &^
Nível 4:          +   -   |   ^
Nível 3:          ==  !=  <   <=  >   >=
Nível 2:          &&
Nível 1 (menor):  ||
Unários:          +  -  !  ^  *  &  <-  (maior que todos os binários)
```

```go
// Exemplos de precedência
2 + 3*4           // 14  — * antes de +
1 + 2 << 3        // 17  — << (nível 5) antes de + (nível 4)?
                  // Não! << é nível 5, + é nível 4 → << primeiro
                  // 1 + (2 << 3) = 1 + 16 = 17

// Regra de ouro: use parênteses para clareza
resultado := (a + b) * (c - d)
flags := (mask1 & val1) | (mask2 & val2)
check := (x > 0) && (y > 0) || (z == 0)
```

---

## 9. Pacote `math` — Funções Matemáticas

```go
import "math"

// Constantes
math.Pi       // 3.141592653589793
math.E        // 2.718281828459045
math.Phi      // 1.618033988749895 (razão áurea)
math.Sqrt2    // 1.4142135623730951
math.MaxFloat64  // 1.7976931348623157e+308
math.MaxInt      // 9223372036854775807 (Go 1.17+)
math.MinInt      // -9223372036854775808

// Funções
math.Abs(-3.14)         // 3.14
math.Sqrt(16)           // 4.0
math.Cbrt(27)           // 3.0 (raiz cúbica)
math.Pow(2, 10)         // 1024.0
math.Log(math.E)        // 1.0
math.Log2(1024)         // 10.0
math.Log10(1000)        // 3.0

math.Floor(3.7)         // 3.0 — maior inteiro ≤ x
math.Ceil(3.2)          // 4.0 — menor inteiro ≥ x
math.Round(3.5)         // 4.0 — arredonda para o mais próximo
math.Trunc(3.7)         // 3.0 — remove parte fracionária

math.Sin(math.Pi / 2)   // 1.0
math.Cos(0)             // 1.0
math.Tan(math.Pi / 4)   // ~1.0

math.Min(3.0, 5.0)      // 3.0
math.Max(3.0, 5.0)      // 5.0

// Para int (Go 1.21+ tem min/max builtin)
a, b := 3, 5
menor := min(a, b)   // 3 — builtin Go 1.21+
maior := max(a, b)   // 5 — builtin Go 1.21+

// Valores especiais
math.IsNaN(math.NaN())      // true — NaN != NaN por IEEE 754!
math.IsInf(math.Inf(1), 1)  // true (positive infinity)
```

---

## 10. Geração de Números Aleatórios (Go 1.22+)

```go
// math/rand/v2 — API modernizada, sem necessidade de Seed() manual
import "math/rand/v2"

// Uso direto — thread-safe, fonte global com seed automático
n := rand.IntN(100)        // [0, 100) — inteiro
f := rand.Float64()        // [0.0, 1.0)
b := rand.N(uint8(256))    // qualquer tipo numérico

// Com fonte determinística (para reprodutibilidade em testes)
src := rand.NewPCG(42, 1024)
r := rand.New(src)
fmt.Println(r.IntN(100))   // sempre o mesmo para seed 42

// Embaralhar slice
s := []int{1, 2, 3, 4, 5}
rand.Shuffle(len(s), func(i, j int) { s[i], s[j] = s[j], s[i] })

// crypto/rand — criptograficamente seguro
import (
	"crypto/rand"
	"math/big"
)

// Gerar número aleatório criptograficamente seguro
n2, err := rand.Int(rand.Reader, big.NewInt(100))

// Gerar bytes aleatórios (tokens, IDs, chaves)
token := make([]byte, 32)
_, err = rand.Read(token)
// token agora contém 32 bytes aleatórios e seguros
```

---

## 11. Resumo

| Operação | Go | Observação |
| --- | --- | --- |
| Divisão inteira | `a / b` | Trunca em direção a zero |
| Módulo | `a % b` | Sinal segue o dividendo |
| Incremento | `i++` | Statement, não expressão |
| Overflow | Wrap-around silencioso | Use `math/bits` para detectar |
| AND NOT | `a &^ b` | Exclusivo do Go |
| Swap | `a, b = b, a` | Avaliação paralela |
| Potência | `math.Pow(a, b)` | Não há operador `**` |
| Raiz | `math.Sqrt(x)` | Não há operador `√` |
| Random | `rand.IntN(n)` | `math/rand/v2` (Go 1.22+) |

---

## 12. Por Dentro: Aritmética, CPU e Hardware

### Inteiros → instruções de ALU diretamente

```
Go source         SSA IR          x86-64 assembly
──────────        ──────          ───────────────
a + b        →    v3 = Add64      ADD  rax, rbx
a - b        →    v3 = Sub64      SUB  rax, rbx
a * b        →    v3 = Mul64      IMUL rax, rbx
a / b        →    v3 = Div64      IDIV rbx         ← divide RDX:RAX por RBX
a % b        →    v3 = Mod64      (resultado em RDX após IDIV)

ALU (Arithmetic Logic Unit) dentro da CPU:
  ┌─────────────────────────────────────┐
  │  Registradores (RAX, RBX, ...)      │
  │       │            │                │
  │       ▼            ▼                │
  │   ┌────────────────────────┐        │
  │   │  ALU                   │        │
  │   │  ADD / SUB / MUL / DIV │        │
  │   └────────────────────────┘        │
  │            │                        │
  │            ▼                        │
  │       EFLAGS register               │
  │       (CF, ZF, SF, OF, PF)         │
  └─────────────────────────────────────┘

CF = Carry Flag     (overflow em aritmética unsigned)
OF = Overflow Flag  (overflow em aritmética signed)
ZF = Zero Flag      (resultado == 0)
SF = Sign Flag      (resultado negativo)
```

Conecta com [[Processadores]] — a ALU é a unidade de execução de operações aritméticas. Flags EFLAGS são usadas por instruções de branch seguintes (JE, JNE, JO).

### Overflow silencioso: complemento de dois na prática

```
Go usa complemento de dois (two's complement) para inteiros com sinal:
Aritmética modular: resultado = resultado mod 2^N

int8 (8 bits, range -128 a 127):

  127 + 1:
  Binário: 0111_1111 + 0000_0001 = 1000_0000
                                   └── bit de sinal = 1 → interpretado como -128

  0111_1111  =  127
+ 0000_0001  =    1
────────────
  1000_0000  = -128  ← wrap-around (overflow silencioso em Go)

Diferença de outras linguagens:
  Rust (debug mode): panic em overflow (extra check emitido pelo compilador)
  Rust (release):    wrap-around silencioso (mesmo que Go)
  C:                 comportamento indefinido para signed overflow!
  Go:                SEMPRE wrap-around (definido pela spec)
```

Conecta com [[Bits e Bytes]] — complemento de dois, representação de inteiros negativos, wrap-around como aritmética modular.

### Divisão por zero: exceção de hardware

```
var a, b int = 10, 0
_ = a / b   // panic em Go

Por quê o panic?

Nível hardware (x86-64):
  IDIV rbx     ← se rbx == 0 → CPU gera #DE (Divide Error Exception)
                 Interrupt vector 0 — "Division by Zero"
                 CPU salva contexto, chama interrupt handler do kernel

Nível kernel (Linux):
  kernel recebe #DE
  → converte para sinal SIGFPE (floating point exception — nome histórico)
  → envia SIGFPE para o processo

Nível Go runtime:
  handler de SIGFPE no runtime Go
  → converte para panic("runtime error: integer divide by zero")
  → stack trace é coletado
  → panic se propaga (ou recover() o intercepta)

Comportamento diferente para float64:
  DIVSD xmm0, xmm1  (float divide)
  Se xmm1 == 0.0 → NÃO gera exceção! IEEE 754 define:
    1.0 / 0.0 = +Inf   (resultado válido)
    0.0 / 0.0 = NaN    (resultado válido)
```

Conecta com [[Processadores]] — CPU exceptions (trap/fault), interrupt vector table. O runtime Go "intercepta" exceções de hardware via signal handlers do OS.

### Ponto flutuante: IEEE 754 e FPU

```
float64 em Go → instrução SSE2 no x86-64 (não x87 FPU):
  a + b  →  ADDSD xmm0, xmm1   (double-precision scalar add)
  a * b  →  MULSD xmm0, xmm1
  a / b  →  DIVSD xmm0, xmm1

IEEE 754 double (64 bits):
  ┌───┬───────────┬────────────────────────────────────────┐
  │ S │ Exponent  │ Mantissa (fraction)                    │
  │ 1 │  11 bits  │  52 bits                               │
  └───┴───────────┴────────────────────────────────────────┘
  
  Valores especiais:
    Exp=11111111111, Mantissa=0        → ±Inf
    Exp=11111111111, Mantissa≠0        → NaN
    Exp=00000000000, Mantissa=0        → ±0.0
    Exp=00000000000, Mantissa≠0        → subnormal (≈ 0)

  Precisão: ~15-16 dígitos decimais significativos
  
  Armadilha clássica:
    0.1 + 0.2 ≠ 0.3   (0.30000000000000004 em IEEE 754)
    Razão: 0.1 e 0.2 não têm representação exata em binário
```

Conecta com [[Bits e Bytes]] — IEEE 754, representação de floats, bits de sinal/expoente/mantissa. [[Processadores]] — FPU/SSE2, instruções SIMD.

### Shift operators: instruções SHL/SHR

```go
n << 3   // shift left  3 posições = n × 2³ = n × 8
n >> 2   // shift right 2 posições = n ÷ 2² = n ÷ 4

x86-64:
  n << 3  →  SHL rax, 3     (Shift Logical Left)
  n >> 2  →  SAR rax, 2     (Shift Arithmetic Right — preserva sinal)
             SHR rax, 2     (Shift Logical Right — para unsigned)

Barrel shifter (hardware dentro da ALU):
  Realiza o shift em 1 ciclo de clock para qualquer quantidade
  Sem barrel shifter (CPUs antigas): shift de N bits = N ciclos

O compilador Go usa strength reduction automática:
  n * 8   →  SHL n, 3     (1 ciclo vs 3+ ciclos de MUL)
  n / 4   →  SAR n, 2     (1 ciclo vs ~20 ciclos de IDIV)
  n % 16  →  AND n, 15    (1 ciclo vs ~20 ciclos de IDIV)
```

Conecta com [[Processadores]] — barrel shifter, SHL/SHR/SAR instructions, strength reduction optimization.

### `math/bits`: operações de nível de CPU em Go

```go
import "math/bits"

// bits.OnesCount64 → instrução POPCNT (Population Count) no x86
bits.OnesCount64(0xFF)    // 8
// Assembly: POPCNT RAX, RBX  (conta bits 1 em 1 instrução)

// bits.LeadingZeros64 → instrução LZCNT ou BSR
bits.LeadingZeros64(1)    // 63
// Assembly: LZCNT RAX, RBX

// bits.TrailingZeros64 → instrução TZCNT ou BSF
bits.TrailingZeros64(8)   // 3  (8 = 1000₂ → 3 zeros à direita)

// bits.Mul64 → instrução MUL (resultado 128 bits em RDX:RAX)
hi, lo := bits.Mul64(a, b)
// Assembly: MUL rbx  → resultado em RDX:RAX (128 bits!)

// O compilador Go usa essas instruções diretamente quando disponíveis
// No hardware sem suporte: usa sequências de instruções equivalentes
```

Conecta com [[Processadores]] — POPCNT, LZCNT, TZCNT são instruções específicas de CPU. `math/bits` é a forma idiomática de acesso a estas instruções no Go.

---

## 13. Conexão com Sistemas Operacionais

- **[[Processadores]]**: Operações aritméticas em `int` e `float64` compilam para instruções diretamente na ALU (ADD, SUB, IMUL, IDIV) e FPU/SSE2 (ADDSD, MULSD). O compilador SSA aplica strength reduction (`n*8 → SHL n,3`) para substituir operações caras por equivalentes mais rápidas.

- **[[Bits e Bytes]]**: Inteiros em Go usam complemento de dois — overflow resulta em wrap-around silencioso (`int8(127)+1 == -128`). Diferente de C (undefined behavior em signed overflow) e Rust/debug (panic). Floats seguem IEEE 754: 64 bits com 1 bit de sinal, 11 de expoente, 52 de mantissa. Divisão float por zero produz `±Inf` ou `NaN` (definido pelo IEEE 754), não panic.

- **[[Processadores]]** (CPU exceptions): Divisão inteira por zero (`a/0`) faz a CPU gerar `#DE` (Divide Error — interrupt vector 0). O kernel converte para `SIGFPE` e o runtime Go o intercepta, convertendo para `panic`. É um exemplo de exceção de hardware viajando até o código Go.

- **[[Processadores]]** (SIMD e bits): `math/bits.OnesCount64` compila para `POPCNT`, `LeadingZeros` para `LZCNT`, `TrailingZeros` para `TZCNT` — instruções de CPU expostas diretamente pela stdlib Go para máxima eficiência em manipulação de bits.