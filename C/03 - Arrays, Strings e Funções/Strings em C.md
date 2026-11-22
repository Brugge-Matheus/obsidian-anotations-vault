---
tags:
  - c
  - c/fundamentos
  - c/memoria
source: "Padrão ISO C (C17/C23: strings, <string.h>, <ctype.h>, <stdlib.h>); experimentos com gcc 13.3 e AddressSanitizer em Linux x86-64"
---
# Strings em C

📚 **Referência:** Padrão ISO C (C17/C23), `<string.h>`, `<ctype.h>`, `<stdlib.h>` e experimentos reais com `gcc` 13.3 e `-fsanitize=address` | Anterior: [[Arrays em C]] | Próxima: [[Funções em C]] | Compare com [[Strings em Profundidade]] (Go)

---

# 🔤 O que é uma string em C

**C não tem um tipo "string".** Uma string é uma convenção:

> 💡 **String em C:** um **array de `char`** cujo fim é marcado por um byte de valor **zero**, o **terminador nulo** (`'\0'`). Os bytes anteriores a ele são o texto.

Ela **não guarda o próprio tamanho**: para saber onde termina, é preciso **percorrer até achar o `'\0'`**. Valores reais de `char s[] = "oi";`:

```
  índice:    s[0]   s[1]   s[2]
  caractere:  'o'    'i'   '\0'
  valor:      111    105     0        → 3 bytes (sizeof s = 3)
```

Tudo o que as funções de texto fazem depende dessa convenção. **Sem o `'\0'`**, elas continuam lendo a memória adiante até **tropeçar num zero qualquer** (UB). Por isso, a regra número 1 das strings é: **sempre reserve 1 byte a mais para o terminador**.

> ⚠️ **`'a'` não é `"a"`.** Aspas **simples** são **um caractere** (um valor do tipo `int`, `sizeof('a')` = **4**); aspas **duplas** são **uma string** (`"a"` tem 2 bytes: `'a'` e `'\0'`, `sizeof("a")` = **2**; e `""`, a string vazia, tem **1**).

---

# 📝 Literais, declaração e escapes

Um **literal de string** (`"texto"`) é um array de `char` com o terminador incluído, guardado numa região **somente leitura** (`.rodata`, [[Variáveis e Constantes em C]]). Há duas maneiras de usá-lo, com efeitos **muito diferentes**:

| Declaração | O que acontece | Pode alterar? |
|---|---|---|
| `char s[] = "oi";` | **copia** o texto para um array seu (3 bytes) | ✅ |
| `char s[10] = "oi";` | copia e **zera o resto** (`s[2]` a `s[9]` valem 0) | ✅ |
| `char s[2] = "oi";` | cabe só o texto, **sem `'\0'`**: legal em C, mas **não é uma string** (cuidado) | ✅ |
| `const char *p = "oi";` | `p` **aponta** para o literal em `.rodata` | ❌ **UB** (*segfault*) |

A diferença entre as duas primeiras formas e a quarta (e o *segfault* real ao alterar um literal) está em [[Arrays e Ponteiros em C]]. A regra prática é: **para texto que será alterado, use array; para texto fixo, `const char *`**.

## Sequências de escape

| Escape | Significa | | Escape | Significa |
|---|---|---|---|---|
| `\n` | nova linha | | `\\` | barra invertida |
| `\t` | tabulação | | `\"` | aspas duplas |
| `\0` | byte zero (o terminador) | | `\xHH` | byte em hexadecimal (`"\x41"` = `"A"`) |
| `\r` | retorno de carro | | `\ooo` | byte em octal (`"\101"` = `"A"`) |

**Literais adjacentes são concatenados** pelo compilador: `"ab" "cd" "ef"` vale `"abcdef"`. Útil para quebrar textos longos em várias linhas do código.

---

# 📏 `strlen` × `sizeof`

Dois números diferentes, que se confundem:

| | Mede | Para `char g[10] = "oi";` |
|---|---|---|
| **`sizeof g`** | o **tamanho do array** (bytes reservados), em tempo de compilação | **10** |
| **`strlen(g)`** | o **texto**: caracteres **até o `'\0'`**, sem contá-lo, em tempo de execução | **2** |

O `strlen` (de `<string.h>`) **percorre** a string byte a byte, então custa **O(n)**: chamá-lo repetidamente numa condição de laço é um desperdício ([[Controle de Fluxo em C]]). Calcule uma vez e guarde. Escrever o seu próprio é um bom exercício de como as strings funcionam:

```c
size_t tam(const char *s) {
    size_t n = 0;
    while (s[n] != '\0') n++;          // para no terminador
    return n;
}
// tam("linguagem") → 9
```

O parâmetro é `const char *` (a função só lê), como visto em [[Ponteiros em C]].

---

# 🧰 A biblioteca `<string.h>`

| Função | Faz | Cuidado |
|---|---|---|
| `strlen(s)` | tamanho do texto | O(n) |
| `strcmp(a, b)` | **compara** conteúdo: `0` se iguais, **negativo/positivo** conforme a ordem | use só o **sinal** (`< 0`, `== 0`, `> 0`) |
| `strcpy(dst, src)` | **copia** `src` (com `'\0'`) para `dst` | **não verifica** se cabe |
| `strcat(dst, src)` | **acrescenta** `src` ao fim de `dst` | **não verifica** se cabe |
| `strncpy`, `strncat` | versões "com limite" | `strncpy` tem **armadilha** (abaixo) |
| `strchr(s, c)` / `strrchr` | 1ª / última ocorrência do caractere (ou `NULL`) | devolve um **endereço dentro de `s`**, não uma cópia |
| `strstr(s, sub)` | 1ª ocorrência da substring (ou `NULL`) | idem |
| `memcpy`, `memmove`, `memset`, `memcmp` | operam em **bytes**, **sem** olhar o terminador | `memcpy` com áreas que **se sobrepõem** é UB; use `memmove` |

Resultados reais: `strcmp("banana", "banana")` → igual; `("banana", "bananas")` → **negativo** (mais curta vem antes); `("banana", "abacate")` → **positivo**. `strchr("um dois tres", 'd')` devolveu `"dois tres"` e `strchr(..., 'z')` devolveu `NULL`. `memmove(b + 2, b, 4)` em `"abcdef"` deu `"ababcd"` (correto mesmo com sobreposição).

> ⚠️ **`==` compara endereços, não texto.** Dois arrays com o mesmo conteúdo (`"igual"`) deram `x == y` → **0**, mas `strcmp(x, y) == 0` → verdadeiro. O `gcc` avisa `-Warray-compare` ([[Arrays em C]]). Para **conteúdo**, sempre `strcmp`.

## Concatenar e copiar sem estourar

`strcpy`/`strcat` copiam **até o `'\0'` da origem**, sem saber o tamanho do destino. Com origem maior que o destino, **escrevem fora do array** (*buffer overflow*, [[Arrays em C]]). Teste real: `char buf[4]; strcpy(buf, "abcdefgh");` → o ASan acusa `stack-buffer-overflow ... WRITE of size 9`.

A resposta moderna é **`snprintf`** (de `<stdio.h>`): **nunca** escreve além do tamanho dado e **sempre termina** a string.

```c
char buf[8];
int n = snprintf(buf, sizeof buf, "%s-%d", "abcdef", 12345);
// buf = "abcdef-"  (7 caracteres + '\0')     n = 12  (tamanho que SERIA necessário)
```

Se `n >= sizeof buf`, houve **truncamento**: o retorno permite detectá-lo. Para montar textos, **prefira `snprintf`**; para copiar, `snprintf(dst, sizeof dst, "%s", src)` também funciona.

> ⚠️ **A armadilha do `strncpy`:** apesar do nome, ele **não é um `strcpy` seguro**. Se a origem **não cabe**, copia `n` bytes e **não escreve o `'\0'`**: com `char t[4]` e `strncpy(t, "abcdef", 4)`, o conteúdo ficou `a b c d` **sem terminador**. Se a origem é **curta**, ele **preenche o resto com zeros** (custo desnecessário em buffers grandes). O `gcc` avisa: `-Wstringop-truncation`. Se usar, termine à mão (`t[sizeof t - 1] = '\0';`).

---

# 🔠 Caracteres: `<ctype.h>`

Como `char` é um número, as funções de `<ctype.h>` classificam e convertem um caractere por vez: `isalpha`, `isdigit`, `isspace`, `isupper`, `islower`, `toupper`, `tolower`. Elas **esperam um valor representável em `unsigned char`** (ou `EOF`), então **converta**, como explicado em [[Conversões de Tipo em C]]:

```c
char up[] = "Ola, C99!";
for (size_t i = 0; up[i]; i++)                    // para no '\0'
    up[i] = (char)toupper((unsigned char)up[i]);
// "OLA, C99!"   (4 letras, 2 dígitos contados com isalpha/isdigit)
```

Note a condição `up[i]`: o **terminador vale 0 (falso)**, então o laço para sozinho ([[Controle de Fluxo em C]]).

---

# 🌍 Bytes × caracteres: o problema do UTF-8

Em C, `char` é **um byte**, e o `strlen` conta **bytes**, não caracteres visíveis. Textos em **UTF-8** (o padrão do Linux, da web e do Go) usam **de 1 a 4 bytes** por caractere:

| Texto | `strlen` (bytes) | Caracteres visíveis |
|---|---|---|
| `"ola"` | 3 | 3 |
| `"olá"` | **4** | 3 (o `á` ocupa **2** bytes) |
| `"日本"` | **6** | 2 (3 bytes cada) |

Consequências: `strlen` **não** é a "quantidade de letras"; `toupper` **não** converte `á`; indexar `s[i]` pode cair **no meio** de um caractere. Funciona bem para ASCII; para Unicode de verdade é preciso uma biblioteca (ou `wchar_t`/`<uchar.h>`). Em Go a `string` é tratada como UTF-8 de forma nativa ([[Strings em Profundidade]]).

---

# 🔢 Texto ↔ número

| Direção | Ferramenta | Observação |
|---|---|---|
| texto → `int` | `atoi(s)` | **sem detecção de erro**: `atoi("abc")` e `atoi("0")` dão ambos **0** |
| texto → `long` | **`strtol(s, &fim, base)`** | informa **onde parou** (`fim`) e sinaliza estouro com `errno` |
| número → texto | `snprintf(buf, n, "%d", v)` | veja acima |

Teste real do `strtol`: `strtol("123abc", &fim, 10)` devolveu `123` e `fim` ficou apontando para `"abc"` (o que **sobrou**); com `"99999999999999999999"` devolveu `LONG_MAX` e `errno == ERANGE`. Por isso, **prefira `strtol`** (família `strtoul`, `strtod`) para entradas de usuário, verificando `fim` e `errno`. O assunto completo fica em *stdlib e Utilitários*.

---

# ✅ Resumo do Conceito

- **String em C** = array de `char` terminado por **`'\0'`** (valor 0); **não guarda o tamanho**; reserve **sempre 1 byte a mais**
- **`'a'`** é um caractere (`int`, 4 bytes); **`"a"`** é uma string (2 bytes); `""` tem 1
- **`char s[] = "..."`** copia (pode alterar); **`const char *p = "..."`** aponta para `.rodata` (alterar é UB); literais adjacentes se concatenam
- **`sizeof`** = tamanho do array; **`strlen`** = texto até o `'\0'`, custa **O(n)**
- **`==` compara endereços**: para conteúdo use **`strcmp`** (confie só no **sinal**)
- **`strcpy`/`strcat` não verificam limites** (*buffer overflow*); **`strncpy` pode não terminar** a string; a ferramenta segura é **`snprintf`** (sempre termina, retorna o tamanho necessário)
- **`memcpy`** não aceita sobreposição, **`memmove`** aceita; `strchr`/`strstr` devolvem endereços **dentro** da string ou `NULL`
- **`<ctype.h>`** exige `(unsigned char)`; o laço `while (s[i])` para sozinho no terminador
- **UTF-8:** `strlen` conta **bytes** (`"olá"` = 4); `toupper` só serve a ASCII
- **`atoi`** não detecta erro; use **`strtol`** com `fim` e `errno`

---

# 🔗 Conexões com outras anotações

- [[Arrays em C]] — strings são arrays: memória contígua, limites e `memcpy`
- [[Arrays e Ponteiros em C]] — `char s[]` × `char *s` e o *segfault* ao alterar literais
- [[Ponteiros em C]] — `const char *` em parâmetros e retorno de `strchr`/`strstr`
- [[Aritmética de Ponteiros em C]] — percorrer strings com ponteiros (`while (*e) e++`)
- [[Variáveis e Constantes em C]] — literais e a seção `.rodata`
- [[Tipos de Dados em C]] — `char`, `size_t` e `sizeof`
- [[Conversões de Tipo em C]] — `(unsigned char)` com `ctype` e o `char` negativo
- [[Controle de Fluxo em C]] — laços sobre strings e `strlen` na condição
- [[Convenções e Estilo]] — `-Wall -Wextra` avisa `-Wstringop-truncation` e `-Warray-compare`
- [[Strings em Profundidade]] — o equivalente em Go (UTF-8 nativo, imutável, com tamanho)
