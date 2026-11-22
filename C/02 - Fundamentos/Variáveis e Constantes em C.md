---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23: declarations, initialization, constants); experimentos com gcc 13.3 em Linux x86-64"
---
# Variáveis e Constantes em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 | Anterior: [[Sintaxe, Blocos e Escopo em C]] | Próxima: [[Tipos de Dados em C]] | Compare com [[Variáveis e constantes]] (Go)

---

# 📦 O que é uma variável?

> 💡 **Objeto:** região de memória capaz de guardar um valor de um certo **tipo**. **Variável:** um **nome** (identificador) que dá acesso a um objeto.

```
   nome          endereço            conteúdo (4 bytes, tipo int)
 ┌───────┐     ┌───────────┐       ┌────┬────┬────┬────┐
 │ idade │ ──► │ 0x7ffe... │ ───►  │ 1E │ 00 │ 00 │ 00 │   = 30
 └───────┘     └───────────┘       └────┴────┴────┴────┘
```

Uma variável tem sempre quatro "atributos": **nome**, **tipo** (define tamanho e interpretação — veja [[Tipos de Dados em C]]), **endereço** e **valor**. Onde o objeto vive (stack, `.data`, `.bss`) depende da sua **duração**, vista em [[Sintaxe, Blocos e Escopo em C]].

---

# ✍️ Declarar, definir e inicializar

```c
int idade;              // declara E define (reserva memória)
int ano = 2026;         // define e INICIALIZA (dá o primeiro valor)
int a = 1, b = 2;       // várias na mesma linha (prefira uma por linha)
```

Três ideias que se confundem:

| Termo | Significado |
|---|---|
| **Declaração** | Diz ao compilador **que o nome existe** e qual o seu **tipo** |
| **Definição** | É a declaração que **também reserva a memória** do objeto |
| **Inicialização** | Dar o **primeiro valor** no momento da definição (`= 2026`) |

Em variáveis **locais**, toda declaração é também definição. A diferença aparece nas **globais**, entre arquivos — o mesmo princípio de *declaração × definição* de [[Arquivos de Cabeçalho e o Modelo de Execução]] e de `undefined reference` em [[Fases da Compilação]]:

```c
extern int contador;    // SÓ declara: "existe, definido em outro .c" (sem memória)
int contador = 0;       // define: aqui o objeto realmente existe
```

> 💡 **Atribuição ≠ inicialização:** `int x = 5;` **inicializa**; `x = 5;` (depois) **atribui**. São operações distintas (importante para `const`, arrays e structs).

---

# 🎲 Que valor a variável tem se eu não inicializar?

É uma das **maiores diferenças** entre C e Go (que sempre zera):

| Tipo de variável | Valor inicial se não inicializada |
|---|---|
| **Local automática** (stack) | **Indeterminado** — **lixo** do que estava antes naquela memória |
| **Global** ou **`static`** | **Zero** (garantido pelo padrão) |

Ler uma variável local **não inicializada** é **comportamento indefinido** ([[Como C Funciona]]) e o `-Wall` avisa (`'n' is used uninitialized`, visto em [[Convenções e Estilo]]). A garantia de zero das globais é o que explica a seção `.bss`: o compilador **não grava** os zeros no arquivo, só anota o tamanho e o SO zera ao carregar. Conferido com `nm`:

```c
int g_sem_valor;            // global sem valor
int g_com_valor = 5;        // global com valor
const int k = 7;            // constante global
static int s_interno;       // global estática (visível só neste .c)
```

```
D g_com_valor       ← .data   (inicializada)
B g_sem_valor       ← .bss    (zerada pelo SO)
R k                 ← .rodata (somente leitura)
b s_interno         ← .bss    (minúscula = símbolo local ao arquivo)
```

(Letras do `nm` — **maiúscula = símbolo global**, **minúscula = local ao arquivo**.) Regra prática: **sempre inicialize** variáveis locais.

---

# 🔒 Constantes: quatro formas

Em C, "constante" pode significar quatro coisas diferentes:

| Forma | Exemplo | É... |
|---|---|---|
| **Literal** | `42`, `3.14`, `'A'`, `"oi"` | Valor escrito direto no código |
| **`const`** | `const int max = 10;` | Variável **somente leitura** |
| **`#define`** | `#define MAX 10` | **Substituição de texto** (pré-processador) |
| **`enum`** | `enum { MAX = 10 };` | Constante **inteira** nomeada |

## `const`: somente leitura, não "constante de compilação"

```c
const int k = 1;
k = 2;        // error: assignment of read-only variable 'k'
```

O compilador **recusa** a escrita. Mas, **diferente do C++**, um `const` em C **não é uma constante de compilação**:

```c
const int n = 5;
int v[n];     // em C vira um VLA (array de tamanho variável); -Wvla avisa
```

> 💡 **VLA (*Variable-Length Array*):** array cujo tamanho só é conhecido **na execução**. Existe desde o C99, mas é frágil (vive na stack, sem checagem de tamanho). Para tamanhos **fixos** de verdade, use `#define` ou `enum`. O **C23** adiciona `constexpr` para objetos realmente constantes em compilação.

`const` guarda o objeto na memória (aparece no depurador, tem tipo e escopo); modificá-lo por um *cast* é **UB**. Combinado com ponteiros (`const char *p` × `char *const p`) tem regras próprias, que serão vistas em *Ponteiros*.

## `#define`: texto, não variável

Como visto em [[Fases da Compilação]], o pré-processador **troca o texto** antes de o compilador enxergar:

```c
#define PI 3.14159
#define DOBRO(x) ((x) * 2)      // parênteses: macros são só texto
```

| | `#define` | `const` |
|---|---|---|
| **Tem tipo** | ❌ | ✅ |
| **Respeita escopo** | ❌ (vale até o fim do arquivo) | ✅ |
| **Aparece no depurador** | ❌ | ✅ |
| **Usável em tamanho de array fixo** | ✅ | ❌ (vira VLA) |

## `enum`: constantes inteiras agrupadas

```c
enum cor { VERMELHO, VERDE = 5, AZUL };    // 0, 5, 6
```

Valores começam em **0** e crescem de 1 em 1, a não ser que você fixe. Resultado real: `VERMELHO=0 VERDE=5 AZUL=6`, e `sizeof(VERDE)` é **4**: **as constantes de `enum` têm tipo `int`**. Um `enum` em C **não é um tipo "seguro"**: é só `int` com nomes. Mais em *Enums e typedef*.

## `volatile`: o "primo" do `const`

```c
volatile uint8_t *reg = ...;
```

`volatile` avisa o compilador de que o valor **pode mudar fora do programa** (registrador de hardware, rotina de interrupção), então ele **não pode** otimizar leituras repetidas ([[O Compilador]] — a regra *as-if*). Aparece em código de [[Dispositivos de IO]] e firmware.

---

# 🔤 Literais: o valor escrito no código

| Tipo | Forma | Exemplos e pegadinhas |
|---|---|---|
| **Inteiro decimal** | `42` | tipo `int` |
| **Hexadecimal** | `0x2A` | `0x2A` = 42 |
| **Octal** | `052` | ⚠️ `052` = **42**: um **zero à esquerda** vira octal! |
| **Binário** | `0b101010` | Padrão só no **C23** (extensão do GCC/Clang antes) |
| **Sufixos** | `10U`, `10L`, `10UL`, `10LL` | `U` = unsigned, `L` = long |
| **Ponto flutuante** | `3.14`, `1e3` | tipo **`double`** por padrão; `3.14f` = `float` |
| **Caractere** | `'A'` | ⚠️ em C tem tipo **`int`** (valor 65), não `char` |
| **String** | `"abc"` | Array de `char` terminado em `\0` |

Conferido em `sizeof`: `sizeof('A')` = **4** (em C++ seria 1), `sizeof(3.14)` = **8**, `sizeof(3.14f)` = **4**.

## Literais de string são **somente leitura**

Vivem em `.rodata` ([[Como C Funciona]]). Tentar alterá-los é **UB**:

```c
char *s = "abc";
char t[] = "abc";
t[0] = 'x';       // ✅ OK: t é uma CÓPIA em memória gravável
s[0] = 'x';       // ❌ Segmentation fault (exit code 139)
```

`char t[] = "abc"` **copia** o texto para um array seu; `char *s = "abc"` apenas **aponta** para o literal. Detalhes em [[Strings em C]]. Boa prática: `const char *s = "abc";`, que faz o compilador impedir o erro.

---

# 🧭 Quando usar cada uma

| Preciso de... | Use |
|---|---|
| Um valor numérico fixo, usado em tamanho de array ou `switch` | `enum { N = 10 };` ou `#define N 10` |
| Um valor fixo **com tipo e escopo** | `static const double pi = 3.14159;` |
| Conjunto de valores nomeados relacionados | `enum` |
| Texto/bloco de código gerado (condicional, macro) | `#define` |
| Algo que **muda por fora** do programa | `volatile` |

Nomes e inicialização já seguem [[Convenções e Estilo]] (`MAIUSCULAS` para constantes, inicializar sempre).

---

# ✅ Resumo do Conceito

- **Objeto** = região de memória com tipo; **variável** = nome para um objeto (nome, tipo, endereço, valor)
- **Declaração** apresenta o nome; **definição** reserva memória; **inicialização** dá o primeiro valor; `extern` **só declara**
- **Locais não inicializadas = lixo** (UB se lidas); **globais e `static` = zero** (por isso existe a `.bss`)
- Quatro "constantes": **literal**, **`const`**, **`#define`**, **`enum`**
- `const` é **somente leitura**, mas **não constante de compilação** em C (`int v[n]` vira VLA); `#define` é **texto** (sem tipo nem escopo); `enum` dá constantes **`int`**
- Literais: cuidado com **octal** (`052` = 42), **`'A'` é `int`** em C, `3.14` é `double`
- **Strings literais são somente leitura** (`.rodata`): `char *s = "abc"; s[0]='x';` → *segfault*; `char t[] = "abc";` é uma cópia editável
- `volatile` indica valores que mudam por fora (hardware, interrupções)

---

# 🔗 Conexões com outras anotações

- [[Sintaxe, Blocos e Escopo em C]] — escopo e tempo de vida das variáveis
- [[Tipos de Dados em C]] — o tipo de cada variável e literal
- [[Como C Funciona]] — regiões `.data`, `.bss`, `.rodata`, stack e UB
- [[Fases da Compilação]] — pré-processador (`#define`) e declaração × definição no linker
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — `extern` e arquivos de cabeçalho
- [[Convenções e Estilo]] — nomes, inicialização e avisos
- [[Variáveis e constantes]] — o equivalente em Go (que zera sempre)
- [[Dispositivos de IO]] — onde `volatile` é essencial
