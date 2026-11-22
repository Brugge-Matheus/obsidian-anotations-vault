---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23: conversões, promoções inteiras, conversões aritméticas usuais); experimentos com gcc 13.3, UBSan e -Wconversion em Linux x86-64"
---
# Conversões de Tipo em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `-fsanitize=undefined` e `-Wconversion` | Anterior: [[Controle de Fluxo em C]] | Próxima: [[Arrays em C]] (módulo 03)

---

# 🔄 O que é uma conversão de tipo

> 💡 **Conversão de tipo:** transformar o **valor** de um tipo em um valor equivalente de **outro** tipo (por exemplo, `int` → `double`). **Não** é reinterpretar os bits: `(int)3.99` vale `3`, e não "os bits do `double` vistos como `int`".

Ela é necessária porque a CPU opera sobre operandos do **mesmo tipo** e tamanho ([[Processadores]]). Em C (tipagem **fraca**, [[O que é C]]) há dois caminhos:

| Tipo | Quem decide | Exemplo |
|---|---|---|
| **Implícita** (automática) | o **compilador** | `double d = 5;` · `c + 1` com `char c` |
| **Explícita** (*cast*) | **você**, com `(tipo)expressão` | `(double)total / n` |

A conversão **implícita** acontece em quatro situações: **(1)** nos operandos de um operador aritmético, relacional ou bit a bit ([[Operadores em C]]); **(2)** em **atribuições** e inicializações (o valor vai para o tipo do destino); **(3)** na **passagem de argumentos** e no `return` (para o tipo do parâmetro ou do retorno); **(4)** no `?:`, que unifica os tipos dos dois ramos. O restante da nota mostra as regras de cada uma e **onde elas enganam**.

---

# 🎒 Promoções inteiras (*integer promotions*)

> 💡 **Regra:** em uma expressão, todo tipo inteiro **menor que `int`** (`char`, `short`, `bool`, e seus equivalentes `unsigned`) é **promovido** a `int` (ou a `unsigned int`, se `int` não comportar todos os valores do tipo original) **antes** de operar.

Consequência: **quase não existe conta em 8 ou 16 bits em C**; tudo é feito em `int`. Resultados reais (Linux x86-64):

```c
char c1 = 100, c2 = 100;
sizeof(c1)       // 1
sizeof(c1 + c2)  // 4     ← a soma é um int
c1 + c2          // 200   (não estoura: 200 cabe em int, mesmo não cabendo em char)
unsigned char uc = 200;
uc * uc          // 40000 (int)
```

## A armadilha do `~` e dos deslocamentos em `uint8_t`

```c
uint8_t a = 0xF0;
~a                    // 0xFFFFFF0F   ← promovido a int ANTES de inverter
(~a == 0x0F)          // 0  (falso!)
((uint8_t)~a == 0x0F) // 1  (o cast "corta" de volta para 8 bits)
```

Ao inverter um byte você obtém um **`int` com 24 bits extras em 1**. O `gcc` até avisa (`-Wsign-compare`: *comparison of promoted bitwise complement of an unsigned value*). A cura é o **cast de volta** para o tipo estreito ou uma **máscara** (`~a & 0xFFu`), como em [[Operadores em C]].

> ⚠️ **Overflow escondido em tipos pequenos sem sinal:** `uint16_t x = 65535; x * x` é promovido a **`int` com sinal** e estoura: o UBSan acusa `signed integer overflow: 65535 * 65535 cannot be represented in type 'int'`. É **UB**, apesar de os operandos serem `unsigned`. Quando o resultado pode passar de `INT_MAX`, force um tipo maior ou sem sinal: `(uint32_t)x * x`.

---

# ⚖️ Conversões aritméticas usuais (*usual arithmetic conversions*)

Quando os dois operandos de um operador binário têm **tipos diferentes**, o compilador os leva a um **tipo comum**. O algoritmo, em ordem:

1. Se algum é **ponto flutuante**: converte ambos para o **maior** deles (`long double` > `double` > `float`);
2. Senão, aplica as **promoções inteiras** em ambos; se agora têm o mesmo tipo, acabou;
3. Se ambos têm **sinal** (ou ambos **não**), vai para o de **maior *rank*** (a ordem é `char < short < int < long < long long`);
4. Se o **sem sinal** tem *rank* **≥** ao do com sinal, o com sinal é convertido para **sem sinal**;
5. Senão, se o tipo **com sinal** consegue representar **todos** os valores do sem sinal, o sem sinal vai para **com sinal**;
6. Senão, ambos vão para o **sem sinal correspondente** ao tipo com sinal.

O passo 4 é a **fonte do bug mais famoso**: misturar `signed` com `unsigned` faz o negativo virar um número enorme.

| Expressão | Resultado | Por quê |
|---|---|---|
| `-1 < 1u` | **`0`** (falso!) | `int` vs `unsigned int`: `-1` vira `4294967295` |
| `-1 > 0u` | **`1`** (verdadeiro!) | idem |
| `-1L < 1u` | `1` | `long` (64 bits) representa todo `unsigned int` (passo 5) |
| `-1L < 1ul` | **`0`** | `unsigned long` tem o mesmo *rank*: passo 4 |
| `sizeof(int) > -1` | **`0`** (falso!) | `size_t` (sem sinal, 8 bytes) vs `int`: `-1` vira o maior `size_t` |
| `sizeof(1 + 1L)` | `8` | `int` → `long` |
| `sizeof(1 + 1.0f)` / `sizeof(1 + 1.0)` | `4` / `8` | `int` → `float` / `double` |

> ✅ **Regra de ouro:** **não misture `signed` e `unsigned`** em comparações e contas. O `gcc` avisa com `-Wsign-compare` (parte do `-Wextra`). É por isso que `for (int i = 0; i < strlen(s); i++)` gera aviso ([[Controle de Fluxo em C]]): `strlen` devolve `size_t`.

---

# 🔁 Entre inteiros: com sinal ↔ sem sinal e estreitamento

O que acontece ao atribuir um valor a um tipo que **não o comporta**?

| Conversão | Resultado | Teste real |
|---|---|---|
| Qualquer inteiro → **`unsigned`** | **Definido:** módulo 2ᴺ (o valor "dá a volta") | `unsigned u = -1;` → `4294967295`; `unsigned char b = 300;` → **`44`** (300 − 256) |
| Valor → **tipo com sinal** que não o comporta | **Definido pela implementação** (ou sinal gerado) | `int8_t sx = 200;` → `-56`; `(int)3000000000u` → `-1294967296` (o `gcc` "dá a volta") |

Assim, `unsigned` → `signed` e o **estreitamento** (`long` → `int`, `int` → `char`) **podem perder informação sem aviso**, a não ser que você ligue os alertas (seção final). Para constantes visíveis o `gcc` já avisa (`-Woverflow`: *changes value from '300' to '44'*).

---

# 🔢 Entre inteiros e ponto flutuante

| Conversão | O que acontece | Teste real |
|---|---|---|
| `int` → `float`/`double` | Pode **perder precisão** se o inteiro tiver mais bits que a mantissa | `float f = 16777217;` → **`16777216`** ([[Tipos de Dados em C]]); `double` guarda exatamente qualquer `int` de 32 bits |
| `float`/`double` → `int` | **Trunca em direção a zero** (descarta a fração, **não arredonda**) | `(int)3.99` = **3**, `(int)-3.99` = **-3** |
| *float* → `int` **fora da faixa** | **UB** | `(int)1e10`: UBSan diz `1e+10 is outside the range of representable values of type 'int'`; sem o *sanitizer* imprimiu `-2147483648` (acaso do x86) |
| `double` → `float` | Arredonda e pode virar `inf` se for grande demais | — |

Para **arredondar** de verdade use `round`, `lround`, `floor`, `ceil` de `<math.h>` em vez de contar com a conversão.

## A armadilha da divisão inteira

A conversão acontece **depois** da divisão, não antes:

```c
double d = 7 / 2;            // 3.0   ← 7/2 é divisão inteira (3); só depois vira double
(double)7 / 2                // 3.5   ← o cast converte o 7 ANTES: 7.0 / 2
(double)(7 / 2)              // 3.0   ← cast tarde demais
int num = 7, den = 2;
(double)(num / den)          // 3.00  (média errada)
(double)num / den            // 3.50  (média certa)
```

Isso funciona porque o **cast é um operador unário de precedência alta**: `(double)num / den` é `((double)num) / den` ([[Operadores em C]]).

---

# 🎭 O cast explícito

```c
(tipo) expressão
```

- **Converte o valor** (regras acima) e produz um **novo valor**; a variável original **não muda**.
- É útil para: forçar divisão real, **estreitar de propósito** (`(uint8_t)x`), cortar os bits extras das promoções e **descartar** um resultado com `(void)f();`.
- ⚠️ **O cast silencia o compilador**: ele deixa de avisar justamente onde havia risco. Cada `(tipo)` deve ter uma **razão**; se estiver "para o aviso sumir", o problema continua lá.
- Casts entre **ponteiros** (`(char *)p`) e para **remover `const`** têm riscos próprios e serão tratados em [[Ponteiros em C]] e *Tipos Compostos*.

---

# 📞 Argumentos, retorno e funções variádicas

- Com um **protótipo** ([[Convenções e Estilo]]), cada argumento é **convertido para o tipo do parâmetro**, como numa atribuição; o `return` converte para o tipo de retorno.
- Em funções **variádicas** (`printf`, `...`) não há tipo declarado para os extras, então valem as **promoções de argumento padrão**: **`float` → `double`** e **`char`/`short` → `int`**. Por isso `printf("%f", 1.5f)` imprime `1.500000` e `printf("%c", 'x')` recebe um `int`. Por isso `%f` serve para `float` e para `double`, mas usar o formato errado para o tipo é **UB** ([[Tipos de Dados em C]]).

---

# 🔤 `char`, `<ctype.h>` e valores negativos

Neste ambiente `char` tem **sinal**: `(char)0xE9` (o "é" em Latin-1) vale **`-23`**, e `(unsigned char)` dessa variável vale **`233`**. As funções de `<ctype.h>` (`isalpha`, `toupper`...) esperam um valor **representável como `unsigned char`** (ou `EOF`): passar um `char` negativo é **UB**. A forma segura é converter:

```c
if (isalpha((unsigned char)c)) { ... }     // cast necessário e justificado
```

---

# 🧷 Conversões em ponteiros e booleanos (visão rápida)

| Caso | Regra | Detalhes |
|---|---|---|
| `void *` → qualquer ponteiro | **Implícita em C** (não em C++): `int *p = malloc(n * sizeof *p);` sem cast | [[Alocação Dinâmica em C]] |
| O inteiro **`0`** → ponteiro nulo | `p = 0;` (e `NULL`) é válido | [[Ponteiros em C]] |
| Ponteiro ↔ inteiro | Só com cast, e use **`uintptr_t`** ([[Tipos de Dados em C]]) | [[Ponteiros em C]] |
| Valor → `bool` | Qualquer valor **≠ 0** vira `1` | [[Tipos de Dados em C]] |

---

# 🚨 Deixando o compilador achar os problemas

Com `-Wall -Wextra` **quase nenhuma** conversão perigosa é avisada. Teste real com funções de uma linha:

| Código | `-Wall -Wextra` | Com `-Wconversion` |
|---|---|---|
| `int f(long v) { return v; }` | silêncio | `-Wconversion`: *'long int' to 'int' may change value* |
| `char c = v;` (`v` é `int`) | silêncio | `-Wconversion` |
| `unsigned u = v;` (`v` é `int`) | silêncio | `-Wsign-conversion`: *may change the sign of the result* |
| `int f(double d) { return d; }` | silêncio | `-Wfloat-conversion` |
| `float f(double d) { return d; }` | silêncio | `-Wfloat-conversion` |
| `int i; i < n` (`n` é `size_t`) | **`-Wsign-compare`** | idem |

Em C, o `-Wconversion` do `gcc` **já inclui** o `-Wsign-conversion`. Foi ela (com `-Wall -Wextra`) que achou os problemas extras do exemplo de [[Convenções e Estilo]]; ligue-a no seu comando padrão e **zere os avisos**, fazendo `cast` **apenas** onde a conversão é intencional.

---

# ✅ Resumo do Conceito

- **Conversão** muda o **valor** (não reinterpreta bits); **implícita** (compilador) em operadores, atribuições, argumentos e `?:`; **explícita** com o *cast* `(tipo)`
- **Promoções inteiras:** `char`/`short`/`bool` viram **`int`** antes de qualquer conta (`sizeof(c1 + c2)` = 4); `~` em `uint8_t` devolve `int` com bits extras (`~0xF0` ≠ `0x0F`); `uint16_t * uint16_t` pode ser **overflow com sinal (UB)**
- **Conversões usuais:** `float` > inteiros; com **sinal × sem sinal** do mesmo *rank*, **vence o sem sinal**: `-1 < 1u` é **falso** e `sizeof(int) > -1` também
- **Para `unsigned`:** módulo 2ᴺ (`300` → `44`); **para `signed` que não cabe:** definido pela implementação (`200` → `-56`)
- **`float` → `int`** trunca em direção a zero e é **UB** se estiver fora da faixa; `int` grande → `float` perde precisão
- **`7 / 2` é inteiro** mesmo atribuído a `double`; use `(double)a / b` (o cast converte só o `a`)
- **Variádicas:** `float` → `double`, `char`/`short` → `int`; **`ctype.h`** exige `(unsigned char)`
- **`-Wall -Wextra` não bastam:** use **`-Wconversion`** (inclui sign e float), evite misturar `signed`/`unsigned` e **não use cast para calar avisos**

---

# 🔗 Conexões com outras anotações

- [[Operadores em C]] — o cast como operador, precedência, `~`/`<<` e comparações afetados pelas promoções
- [[Tipos de Dados em C]] — tamanhos, overflow de `signed`/`unsigned`, `size_t`, `<stdint.h>` e IEEE 754
- [[Variáveis e Constantes em C]] — sufixos de literais (`u`, `L`, `f`) que definem o tipo da constante
- [[Controle de Fluxo em C]] — `i < strlen(s)`, `i >= 0` com `unsigned` e laços
- [[Convenções e Estilo]] — `-Wconversion`, `-Wsign-compare` e protótipos de função
- [[Como C Funciona]] e [[O Compilador]] — comportamento indefinido e *sanitizers*
- [[Processadores]] — por que a CPU precisa de operandos do mesmo tamanho
- [[Bits e Bytes]] — complemento de dois (por que `-1` vira `0xFFFFFFFF`)
