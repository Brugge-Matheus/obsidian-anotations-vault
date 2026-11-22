---
tags:
  - c
  - c/funcoes
source: "Padrão ISO C (C17/C23: <stdarg.h>, promoções de argumento padrão); experimentos com gcc 13.3 em Linux x86-64"
---
# Funções Variádicas em C

📚 **Referência:** Padrão ISO C (C17/C23), `<stdarg.h>` e experimentos reais com `gcc` 13.3 | Anterior: [[Recursão em C]] | Próxima: [[Ponteiros em C]] (módulo 04) | Compare com [[Funções Variádicas]] (Go)

---

# 🎛️ O que é uma função variádica

> 💡 **Função variádica:** uma função que aceita um **número variável de argumentos**. Na declaração, as reticências **`...`** representam "zero ou mais argumentos extras".

O exemplo mais conhecido é o próprio **`printf`**: `printf("oi")`, `printf("%d", x)` e `printf("%d %s %f", a, b, c)` são a **mesma função** com quantidades diferentes de argumentos. A declaração real é:

```c
int printf(const char *format, ...);
```

Esta nota mostra **como escrever** funções assim e, principalmente, **por que são perigosas**: o compilador **não sabe** quantos argumentos extras existem, nem de que tipo são ([[Funções em C]] mostra que, com protótipo, o compilador confere argumentos; aqui ele **para de conferir** depois do `...`).

---

# 🧰 A biblioteca `<stdarg.h>`

Para ler os argumentos extras, a função usa quatro peças de **`<stdarg.h>`**:

| Peça | Papel |
|---|---|
| **`va_list ap`** | variável que "percorre" a lista de argumentos extras |
| **`va_start(ap, ultimo)`** | **inicia** `ap`, a partir do **último parâmetro nomeado** (`ultimo`) |
| **`va_arg(ap, tipo)`** | **devolve o próximo** argumento, **interpretado como `tipo`**, e avança |
| **`va_end(ap)`** | **encerra** o uso (obrigatório antes de retornar) |

(Existe também `va_copy(dst, src)`, para duplicar a lista e percorrê-la duas vezes.)

## Exemplo: somar quantos números forem passados

```c
int soma(int n, ...) {                  // n = quantidade de números extras
    va_list ap;
    va_start(ap, n);                    // começa depois de 'n'
    int total = 0;
    for (int i = 0; i < n; i++)
        total += va_arg(ap, int);       // lê o próximo, como int
    va_end(ap);
    return total;
}
// soma(3, 10, 20, 30) → 60      soma(0) → 0
```

Outro com `double`: `media(3, 1.5, 2.5, 3.5)` devolveu **2.50** (cada argumento lido com `va_arg(ap, double)`).

> 💡 Em C **até o C17** é obrigatório ter **pelo menos um parâmetro nomeado** antes do `...` (é dele que `va_start` parte). O C23 passou a permitir `int f(...)`.

---

# 🔍 O problema: a função não sabe quantos argumentos recebeu

`va_arg` **não avisa** quando a lista acaba nem confere o tipo. O **chamador e a função precisam combinar** uma forma de saber a quantidade. As três mais usadas:

| Técnica | Como | Exemplo |
|---|---|---|
| **Contador** | o 1º parâmetro informa a quantidade | `soma(3, 10, 20, 30)` |
| **Sentinela** | um valor especial marca o fim | `junta(buf, cap, "ab", "cd", (char *)NULL)` |
| **String de formato** | o texto diz quais e quantos argumentos vêm | `printf("%d %s", n, nome)` |

Com a **sentinela** `NULL`, a função lê `const char *` até receber `NULL` (testado: `junta` formou `abcdef`). Note o *cast* `(char *)NULL`: sem ele, o tipo passado por `...` não é garantido ([[Conversões de Tipo em C]]).

## Se o combinado falha: lixo na saída (UB)

Chamar `soma(5, 1, 2, 3)` (diz que há **5** números, passa **3**) fez `va_arg` ler **além do que existe**, resultado: **`-1650588794`** (lixo da pilha/registradores). Igualmente, `printf("%d", 3.5)` imprimiu **`185264720`**. É **comportamento indefinido**: sem verificação de limites e sem verificação de tipos, como em [[Arrays em C]].

---

# 🔄 Promoções de argumento padrão

Os argumentos extras **não têm tipo declarado**, então o compilador aplica as **promoções de argumento padrão** (explicadas em [[Conversões de Tipo em C]]):

| Passado | Chega como |
|---|---|
| `char`, `short` (e `bool`) | **`int`** |
| `float` | **`double`** |
| demais (`int`, `long`, `double`, ponteiros) | sem mudança |

Consequência importante: **dentro** da função variádica, leia sempre o tipo **promovido**: `va_arg(ap, int)` (nunca `char`) e **`va_arg(ap, double)`** (nunca `float`). O `gcc` avisa: *'float' is promoted to 'double' when passed through '...'; so you should pass 'double' not 'float' to 'va_arg'*. Por isso o `printf("%f", x)` funciona tanto com `float` quanto com `double`, e `%c` recebe um `int` ([[Tipos de Dados em C]]).

---

# 🛠️ Como o `printf` funciona (uma versão mínima)

O **formato** diz ao `printf` quais tipos ler com `va_arg`. Uma versão simplificada:

```c
void meu_printf(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    for (const char *p = fmt; *p; p++) {
        if (*p != '%') { putchar(*p); continue; }        // texto comum
        p++;                                              // olha o código após o '%'
        switch (*p) {
        case 'd': printf("%d",   va_arg(ap, int));    break;
        case 'c': putchar(       va_arg(ap, int));    break;   // char chega como int
        case 's': fputs(         va_arg(ap, const char *), stdout); break;
        case 'f': printf("%.2f", va_arg(ap, double)); break;   // float chega como double
        case '%': putchar('%'); break;
        default:  putchar('?'); break;
        }
    }
    va_end(ap);
}
// meu_printf("%s tem %d anos e %f de altura, nota %c (100%%)\n", "Ana", 30, 1.7, 'A');
// → Ana tem 30 anos e 1.70 de altura, nota A (100%)
```

Isso explica por que o formato **errado** é **UB**: `%d` manda ler `int`, mas se você passou `double`, os bytes lidos estão no lugar errado. Usa-se o `switch` ([[Controle de Fluxo em C]]) e o percurso `*p` até o `'\0'` ([[Strings em C]]).

## Como os argumentos chegam

Os extras seguem a mesma convenção de chamada de [[Funções em C]] (registradores e pilha). Há **um detalhe exclusivo** das variádicas no x86-64: antes de `call printf`, o compilador coloca em **`eax` o número de registradores de ponto flutuante** usados. No `gcc -S` de `printf("%d %f\n", 1, 2.0)` apareceu `mov eax, 1` (um `double` passado em `xmm0`) logo antes do `call printf@PLT`.

---

# 🧩 Encaminhar a lista: `vprintf`, `vsnprintf` e `format`

Para escrever uma **função de log** que se comporta como `printf`, não dá para chamar `printf(fmt, ...)` diretamente: o `...` não pode ser repassado. Use as versões que recebem uma **`va_list`**: `vprintf`, `vsnprintf` (de `<stdio.h>`).

```c
__attribute__((format(printf, 2, 3)))                    // GNU: confere o formato
void log_msg(int nivel, const char *fmt, ...) {
    char buf[64];
    va_list ap;
    va_start(ap, fmt);
    vsnprintf(buf, sizeof buf, fmt, ap);                 // trunca com segurança
    va_end(ap);
    printf("[nivel %d] %s\n", nivel, buf);
}
// log_msg(2, "valor=%d nome=%s", 42, "ana") → [nivel 2] valor=42 nome=ana
```

O atributo **`format(printf, 2, 3)`** (extensão do `gcc`/`clang`) diz: "o parâmetro 2 é um formato e os argumentos começam no 3". Com ele o compilador **confere** as chamadas, como faz com o `printf`, e avisa (`-Wformat`): `format '%d' expects argument of type 'int', but argument 2 has type 'char *'` e `format '%s' expects a matching 'char *' argument`. É a **única** proteção disponível; use-o em toda função tipo `printf`.

---

# ⚠️ Quando evitar

| Problema | Alternativa |
|---|---|
| **Nenhuma checagem de tipos** | `format` (acima) ou uma função por tipo |
| **Quantidade ou tipos incertos** | passe um **array** e o **tamanho** (`soma(const int *v, size_t n)`, [[Arrays e Ponteiros em C]]) |
| Segurança: um **formato vindo do usuário** (`printf(texto)`) permite ler/escrever memória | **sempre** `printf("%s", texto)` |
| `va_list` é "de uso único" e sem retorno | `va_copy` se precisar de duas passagens |

Funções variádicas são úteis em **interfaces de formatação** (`printf`, `scanf`, logs, `open`, `exec*`). Para o código do dia a dia, **arrays com tamanho** são mais seguros. Um parente em tempo de compilação são as **macros variádicas** (`__VA_ARGS__`), vistas em *O Pré-processador*.

---

# ✅ Resumo do Conceito

- **Função variádica** usa `...`; o exemplo clássico é `printf`; em C17 precisa de **pelo menos um parâmetro nomeado** (C23 dispensa)
- **`<stdarg.h>`**: `va_list`, **`va_start(ap, ultimo)`**, **`va_arg(ap, tipo)`**, **`va_end(ap)`** (e `va_copy`)
- A função **não sabe** a quantidade nem os tipos: combine por **contador**, **sentinela** (`NULL`) ou **string de formato**; descombinar é **UB** (lixo, como `soma(5, 1, 2, 3)` e `printf("%d", 3.5)`)
- **Promoções padrão:** `char`/`short` → `int`, `float` → `double`; leia com `va_arg(ap, int)` e `va_arg(ap, double)`
- O **`printf`** lê o formato e usa `va_arg` no tipo correspondente; no x86-64 `eax` informa quantos registradores de ponto flutuante foram usados
- Para **encaminhar** argumentos use **`vsnprintf`/`vprintf`** e proteja a função com **`__attribute__((format(printf, ...)))`**
- Prefira **array + tamanho** quando possível; nunca passe um formato controlado pelo usuário

---

# 🔗 Conexões com outras anotações

- [[Funções em C]] — protótipos, convenção de chamada e `(void)`
- [[Conversões de Tipo em C]] — promoções de argumento padrão e `-Wformat`
- [[Tipos de Dados em C]] — tabela de formatos do `printf`
- [[Strings em C]] — `vsnprintf`/`snprintf` e o percurso até o `'\0'`
- [[Controle de Fluxo em C]] — o `switch` do `meu_printf`
- [[Arrays e Ponteiros em C]] — array + tamanho como alternativa segura
- [[Convenções e Estilo]] — `-Wall -Wextra` e avisos de formato
- [[Funções Variádicas]] — o equivalente em Go (`...T` vira um *slice*, com tipo e tamanho)
