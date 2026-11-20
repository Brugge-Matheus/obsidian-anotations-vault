---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23: <limits.h>, <float.h>, <stdint.h>, IEEE 754); experimentos com gcc 13.3 em Linux x86-64"
---
# Tipos de Dados em C

📚 **Referência:** Padrão ISO C (C17/C23), `<limits.h>`, `<float.h>`, `<stdint.h>` e experimentos reais com `gcc` 13.3 | Anterior: [[Variáveis e Constantes em C]] | Próxima: [[Operadores em C]] | Compare com [[Tipos de Dados]] (Go)

---

# 🧬 O que um tipo define

Na memória só há **bytes**; o **tipo** diz como interpretá-los. Os mesmos 4 bytes podem ser um `int`, um `float` ou quatro caracteres. Um tipo define:

1. **Quanto espaço** o valor ocupa (`sizeof`);
2. **Como os bits são interpretados** (inteiro com sinal, ponto flutuante...);
3. **Quais operações** são válidas.

Lembrando [[O que é C]]: a tipagem é **estática** (fixada na compilação) e **fraca** (há conversões implícitas, tema da nota [[Conversões de Tipo em C]]).

```
Tipos em C
├── Aritméticos
│   ├── Inteiros:        bool, char, short, int, long, long long  (com ou sem sinal)
│   └── Ponto flutuante: float, double, long double
├── void                 (ausência de valor)
└── Derivados            arrays, ponteiros, struct, union, enum, funções  (próximas notas)
```

---

# 🔢 Tipos inteiros

## O padrão só garante **mínimos**

C **não fixa** o tamanho dos tipos: garante apenas que `char` tem **1 byte** (por definição) e **tamanhos mínimos** com a ordem `short ≤ int ≤ long ≤ long long`. Por isso o tamanho "real" depende da plataforma, como visto em [[Como C Funciona]].

| Tipo | Mínimo garantido | Neste ambiente (Linux x86-64) | Faixa aqui (com sinal) |
|---|---|---|---|
| `char` | 8 bits | 1 byte | −128 a 127 |
| `short` | 16 bits | 2 bytes | −32 768 a 32 767 |
| `int` | 16 bits | 4 bytes | −2 147 483 648 a 2 147 483 647 |
| `long` | 32 bits | **8 bytes** | ±9,2 × 10¹⁸ |
| `long long` | 64 bits | 8 bytes | ±9,2 × 10¹⁸ |

Os limites exatos de **cada plataforma** vêm de **`<limits.h>`** (`INT_MIN`, `INT_MAX`, `UINT_MAX`, `LONG_MAX`, `CHAR_BIT`...). Valores reais obtidos: `INT_MAX = 2147483647`, `UINT_MAX = 4294967295`, `CHAR_BIT = 8` (bits por byte).

> ⚠️ **Varia por plataforma:** `long` tem **8** bytes no Linux 64 bits e **4** no Windows 64 bits; `int` tem **2** bytes num Arduino Uno (AVR) e **4** no ESP32. Quando o tamanho **importa**, use os tipos de largura fixa (abaixo).

## Com sinal e sem sinal

| Qualificador | Valores | Exemplo |
|---|---|---|
| `signed` (padrão para `int`) | Negativos e positivos | `int x = -5;` |
| `unsigned` | **Só ≥ 0**, com o dobro do alcance positivo | `unsigned int u = 4000000000u;` |

Em N bits, a faixa com sinal é de **−2ᴺ⁻¹ a 2ᴺ⁻¹−1**, e sem sinal de **0 a 2ᴺ−1**. Os negativos usam **complemento de dois** (veja [[Bits e Bytes]]; o C23 o torna obrigatório).

## Overflow: sem sinal dá a volta, com sinal é UB

| | Resultado ao passar do limite | Teste real |
|---|---|---|
| **`unsigned`** | **Definido**: dá a volta (aritmética módulo 2ᴺ) | `UINT_MAX + 1` = **0**; `unsigned char 255 + 1` = **0** |
| **`signed`** | **Comportamento indefinido** (UB) | `INT_MAX + 1` |

Rodando `INT_MAX + 1` com o *sanitizer* (`-fsanitize=undefined`, citado em [[O Compilador]]):

```
ov.c:3:37: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

Sem o sanitizer o programa imprimiu `-2147483648`, mas isso é **sorte**: sendo UB, o compilador pode assumir que o overflow **nunca acontece** e otimizar de forma inesperada ([[O Compilador]]).

## `char`: um inteiro de 1 byte

`char` guarda **caracteres** (códigos ASCII) e também **bytes** crus. É um **tipo inteiro**: `'A'` vale 65 e `'A' + 1` vale 66.

- ⚠️ Se `char` tem sinal **depende da plataforma**: aqui é **com sinal** (`CHAR_MIN = -128`), mas em ARM Linux costuma ser **sem sinal**.
- Para **bytes** e dados binários use `unsigned char` ou `uint8_t`; use `char` só para **texto**.

---

# 🧮 Ponto flutuante

C segue o padrão **IEEE 754**: o número é guardado como **sinal + expoente + mantissa** (notação científica em binário).

| Tipo | Tamanho aqui | Dígitos decimais confiáveis | Epsilon (menor diferença de 1.0) |
|---|---|---|---|
| `float` | 4 bytes | ~6 (`FLT_DIG`) | 1,19 × 10⁻⁷ |
| `double` | 8 bytes | ~15 (`DBL_DIG`) | 2,22 × 10⁻¹⁶ |
| `long double` | 16 bytes (80 bits de precisão no x86-64) | ~18 | menor |

Os limites vêm de **`<float.h>`**. Use **`double`** como padrão (literais como `3.14` já são `double`).

## A armadilha: números decimais **não** são exatos

Frações como 0,1 **não têm representação exata em binário**:

```c
printf("%d  (%.17g)\n", 0.1 + 0.2 == 0.3, 0.1 + 0.2);
// 0  (0.30000000000000004)
```

**Nunca compare floats com `==`.** Compare com uma **tolerância**: `fabs(a - b) < 1e-9`. Outros efeitos reais: um `float` **perde** inteiros acima de 2²⁴ (`16777217` virou `16777216`); `1.0/0` dá `inf` e `0.0/0` dá `nan` (valores especiais do IEEE 754, sem erro em tempo de execução).

> 💡 **Divisão inteira × real:** `7 / 2` = **3** (truncada) e `7.0 / 2` = **3.5**; se **ambos** os operandos são inteiros, a divisão é inteira (mais em [[Operadores em C]]).

---

# ✅ `bool` e a ausência de valor: `void`

## `bool`

- **C23:** `bool`, `true` e `false` são **palavras-chave**.
- **C99–C17:** inclua **`<stdbool.h>`** (macros sobre o tipo `_Bool`).
- Qualquer valor **diferente de zero** vira `1`: `bool b = 5;` guarda `1` (testado; `sizeof(bool)` = 1).
- ⚠️ **Resultados de comparações são `int`** (`1` ou `0`), não `bool`: `sizeof(1 == 1)` = **4**. Em C, **0 é falso e qualquer outro valor é verdadeiro**.

## `void`

`void` significa "nada" e tem três usos:

| Uso | Exemplo |
|---|---|
| Função **sem retorno** | `void limpar(void);` |
| Função **sem parâmetros** | `int main(void)` (veja [[Convenções e Estilo]]) |
| **Ponteiro genérico** | `void *p` (visto em *Ponteiros*) |

---

# 📏 Tipos de largura fixa: `<stdint.h>`

Como `int` e `long` variam, **`<stdint.h>`** dá tipos com **tamanho exato**:

| Tipo | Significado | Tamanho |
|---|---|---|
| `int8_t` … `int64_t` | Inteiros **com sinal** | 1, 2, 4, 8 bytes |
| `uint8_t` … `uint64_t` | Inteiros **sem sinal** | 1, 2, 4, 8 bytes |
| `intptr_t`, `uintptr_t` | Inteiro que **cabe um ponteiro** | Como `void *` |
| `size_t` | Tamanhos e índices (resultado de `sizeof`) | 8 bytes aqui |
| `ptrdiff_t` | Diferença entre dois ponteiros | 8 bytes aqui |

Também existem `int_least32_t` ("pelo menos 32 bits") e `int_fast32_t` ("o mais rápido com pelo menos 32 bits"). Os valores `int8_t`…`int64_t` testados deram 1, 2, 4 e 8 bytes.

## Quando usar cada um

| Preciso de... | Use |
|---|---|
| Um contador ou número geral | **`int`** |
| Tamanhos, índices, `sizeof`, `strlen` | **`size_t`** |
| **Bytes** crus, registradores de hardware, protocolos | **`uint8_t`**, `uint16_t`, `uint32_t` |
| Formato binário de arquivo/rede com tamanho exato | `int32_t`, `uint64_t`... |
| Números reais | **`double`** |
| **Texto** | `char` |

Em firmware ([[MOC - Protocolos de Hardware]]) os tipos fixos são a regra: um registrador de 8 bits é `uint8_t`, o dado de um sensor de 16 bits é `uint16_t`.

---

# 🖨️ Formatos do `printf` por tipo

Cada tipo tem seu formato; **trocar o formato é UB** (o `-Wformat` avisa, como visto em [[Convenções e Estilo]]):

| Tipo | Formato | Observação |
|---|---|---|
| `int` | `%d` | |
| `unsigned int` | `%u` | |
| `long` / `long long` | `%ld` / `%lld` | |
| `size_t` | `%zu` | `z` = tamanho de `size_t` |
| `float` / `double` | `%f` (ou `%g`) | `float` é promovido a `double` no `printf` |
| `char` | `%c` | |
| string | `%s` | |
| `unsigned` em hexadecimal | `%x` | |
| ponteiro | `%p` | |
| `int32_t` etc. | `PRId32` (de `<inttypes.h>`) | `printf("%" PRId32, n);` |

Exemplo real: `printf("%u %" PRId32 " %zu %c %5.2f %x", 200, -5, 4, 'A', 3.14159, 255)` imprime `200 -5 4 A  3.14 ff`.

---

# 📐 `sizeof`: o tamanho em tempo de compilação

```c
sizeof(int)        // do tipo (parênteses obrigatórios)
sizeof x           // da expressão/variável (parênteses opcionais)
```

- Resulta em **`size_t`** e é calculado em **tempo de compilação** (exceto em VLAs, vistos em [[Variáveis e Constantes em C]]);
- **Não avalia** a expressão: `sizeof(i++)` **não** incrementa `i`;
- Boa prática: `sizeof *p` em vez de `sizeof(tipo)` ([[Convenções e Estilo]]).

---

# ✅ Resumo do Conceito

- O **tipo** define **tamanho**, **interpretação dos bits** e **operações válidas**; em C a tipagem é **estática e fraca**
- O padrão garante só **mínimos**: `char` = 1 byte, `short` ≥ 16 bits, `int` ≥ 16, `long` ≥ 32, `long long` ≥ 64; os limites reais estão em **`<limits.h>`** e **`<float.h>`**
- **`unsigned`** dá a volta no overflow (definido); **`signed`** em overflow é **UB** (`INT_MAX + 1`)
- Se **`char`** tem sinal **depende da plataforma**; use `uint8_t` para bytes
- **Ponto flutuante (IEEE 754):** `float` ~6 dígitos, `double` ~15; **nunca compare com `==`** (`0.1 + 0.2 != 0.3`); existem `inf` e `nan`
- **`bool`**: palavra-chave no C23; `<stdbool.h>` antes; comparações **devolvem `int`**; 0 é falso, o resto é verdadeiro
- **`void`**: função sem retorno, sem parâmetros (`(void)`) e ponteiro genérico
- **`<stdint.h>`**: `int32_t`, `uint8_t`, `size_t`, `intptr_t` para **tamanho exato**; padrão prático: `int`, `size_t`, `uint8_t`, `double`
- **`sizeof`** é calculado na compilação, devolve `size_t` e não avalia a expressão

---

# 🔗 Conexões com outras anotações

- [[Variáveis e Constantes em C]] — literais e seus tipos (`'A'` é `int`, `3.14` é `double`)
- [[Sintaxe, Blocos e Escopo em C]] — escopo e tempo de vida dos objetos
- [[Como C Funciona]] — tamanho dos tipos dependente de plataforma e UB
- [[O Compilador]] — `-fsanitize=undefined` e a regra *as-if*
- [[Convenções e Estilo]] — `-Wformat`, `-Wconversion`, `size_t` e `<stdint.h>`
- [[Bits e Bytes]] — complemento de dois e representação binária
- [[Tipos de Dados]] — o equivalente em Go (tipos de tamanho fixo por padrão)
- [[MOC - Protocolos de Hardware]] — `uint8_t`/`uint16_t` em registradores e protocolos
- [[Processadores]] — o tamanho de palavra da CPU explica o tamanho de `int` e dos ponteiros
