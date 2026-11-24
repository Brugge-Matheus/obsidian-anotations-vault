---
tags:
  - c
  - c/memoria
  - c/funcoes
source: "Padrão ISO C (C17/C23: ponteiros para função, qsort e bsearch de <stdlib.h>); experimentos com gcc 13.3 e UBSan em Linux x86-64"
---
# Funções e Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23), `qsort`/`bsearch` de `<stdlib.h>` e experimentos reais com `gcc` 13.3 e `-fsanitize=undefined` | Anterior: [[Arrays e Ponteiros em C]] | Próxima: [[Alocação Dinâmica em C]] | Compare com [[Funções — Fundamentos]] (Go)


> 🧭 **Como ler esta nota:** são **três** ideias diferentes que usam as mesmas palavras. (1) **Ponteiro como parâmetro**: para a função alterar ou ler dados do chamador. (2) **Ponteiro como retorno**: o que é seguro devolver. (3) **Ponteiro *para função***: o próprio código vira um valor (*callbacks*, tabelas). Se o item 3 parecer estranho, leia primeiro os dois primeiros: eles são a base.
---

# 🔗 Onde funções e ponteiros se encontram

Já vimos [[Funções em C]] (sempre por valor) e [[Ponteiros em C]] (endereços). Juntos, eles aparecem de **três formas**:

| Uso                         | Ideia                                                               | Exemplo                      |
| --------------------------- | ------------------------------------------------------------------- | ---------------------------- |
| **Ponteiro como parâmetro** | a função **altera ou lê** dados do chamador sem copiá-los           | `void troca(int *a, int *b)` |
| **Ponteiro como retorno**   | a função **devolve o endereço** de um dado                          | `const char *strchr(...)`    |
| **Ponteiro *para* função**  | o **próprio código** vira um valor que se guarda e se passa adiante | `int (*op)(int, int)`        |

Os dois primeiros usam ponteiros de **dados**. O terceiro é diferente e é o que dá a C o seu mecanismo de **funções como valores**, base de *callbacks* e de tabelas de decisão.

---

# 📥 Ponteiros como parâmetros: os padrões

A mecânica (por valor, passar o endereço para alterar o original) está em [[Ponteiros em C]]. Aqui os **padrões de uso**:

| Padrão | Assinatura típica | Convenção |
|---|---|---|
| **Entrada** (só lê) | `size_t tamanho(const char *s)` | **`const T *`**: promete não alterar |
| **Saída** | `int dividir(int a, int b, int *q)` | o **retorno** é o *status*; o **resultado** vai pelo ponteiro |
| **Entrada e saída** | `void ordena(int *v, size_t n)` | altera o dado do chamador |
| **Array + tamanho** | `double media(const double *v, size_t n)` | o tamanho vai **junto** ([[Arrays e Ponteiros em C]]) |

O padrão **status + saída** é muito comum em C, onde não há exceções: o valor de retorno diz **se deu certo** e o ponteiro recebe o resultado. Teste real de `dividir`:

```c
int dividir(int a, int b, int *q) {
    if (b == 0) return -1;      // erro: NÃO mexe em *q
    *q = a / b;
    return 0;
}
// dividir(17, 5, &q) → retorno 0, q = 3
// dividir(1, 0, &q)  → retorno -1, q continua 3 (intacto)
```

> ✅ **Valide** parâmetros ponteiros que podem ser `NULL` (`if (q == NULL) return -1;`): desreferenciar `NULL` é UB ([[Ponteiros em C]]).

## Alterar um ponteiro do chamador: ponteiro para ponteiro

Se a função precisa **mover** o ponteiro do chamador (e não só o dado apontado), receba o **endereço do ponteiro**:

```c
void pula_espacos(const char **p) {
    while (isspace((unsigned char)**p)) (*p)++;     // *p é o ponteiro do chamador
}
// const char *t = "   ola";  pula_espacos(&t);  →  t aponta para "ola"
```

Aqui `**p` é o caractere atual e `(*p)++` avança o ponteiro do chamador (precedência em [[Operadores em C]]). Padrão típico de analisadores de texto (*parsers*); [[Alocação Dinâmica em C]] mostra outro uso: uma função que **aloca** e devolve o ponteiro por `T **`.

---

# 📤 Retornando ponteiros: o que é seguro devolver

Uma função pode devolver um endereço, mas ele precisa **continuar válido** depois que ela termina. Os quatro destinos possíveis:

| Devolver o endereço de... | Seguro? | Observação |
|---|---|---|
| uma **variável local** (ou array local) | ❌ **UB** | some no `return`: ponteiro pendurado ([[Sintaxe, Blocos e Escopo em C]]); o `gcc` avisa `-Wreturn-local-addr` |
| algo **dentro de um parâmetro** | ✅ | `strchr`, `strstr` fazem isso; o dado vive com o chamador |
| uma variável **`static`** ou literal | ✅ | **cuidado**: o mesmo endereço a cada chamada |
| memória **alocada dinamicamente** | ✅ | o **chamador** precisa liberar ([[Alocação Dinâmica em C]]) |

Exemplo de ponteiro **para dentro do parâmetro** (a lógica de `strchr`):

```c
const char *acha(const char *s, char c) {
    for (; *s; s++) if (*s == c) return s;
    return NULL;                                   // "não achei"
}
// acha("linguagem", 'g') → "guagem" (deslocamento 3) ; acha(..., 'z') → NULL
```

Devolver `NULL` é a forma de sinalizar "não achou"/falha; **o chamador deve testar** antes de usar.

## A armadilha do `static`

```c
const char *rotulo(int n) {
    static char buf[16];                            // vive até o fim do programa
    snprintf(buf, sizeof buf, "item-%d", n);
    return buf;
}
const char *a = rotulo(1), *b = rotulo(2);
// a e b são o MESMO endereço: ambos mostram "item-2"   (testado: a == b)
```

A segunda chamada **sobrescreve** o resultado da primeira. Funções assim não são **reentrantes** (nem seguras com *threads*), como a antiga `strtok`. Alternativa mais segura: o **chamador fornece o buffer** (`void rotulo(char *dst, size_t cap, int n)`), como em `snprintf` ([[Strings em C]]).

---

# 🎯 Ponteiros para função

> 💡 **Ponteiro para função:** uma variável que guarda o **endereço do código** de uma função (a região de texto do programa, [[Como C Funciona]]). Com ele, **escolhe-se em tempo de execução qual função chamar**.

## Declarar e usar

```c
int soma(int a, int b) { return a + b; }

int (*f)(int, int);        // f: ponteiro para função (int,int) → int
f = soma;                  // o nome da função "decai" para seu endereço
f(2, 3);                   // 5  (chamada)
(*f)(2, 3);                // 5  (forma equivalente)
```

Os parênteses em `(*f)` são **essenciais** (precedência de `()` sobre `*`, [[Operadores em C]]):

| Declaração | Significa |
|---|---|
| `int (*f)(int, int)` | `f` é **ponteiro para função** que devolve `int` |
| `int *f(int, int)` | `f` é uma **função** que devolve `int *` |
| `int (*t[3])(int, int)` | `t` é um **array de 3** ponteiros para função |

Leia **de dentro para fora**, a partir do nome. Como a sintaxe é pesada, use **`typedef`** ([[Variáveis e Constantes em C]] cobre os apelidos de tipo; *Enums e `typedef`* aprofunda):

```c
typedef int (*op_fn)(int, int);          // op_fn = "ponteiro para função (int,int)→int"
int aplica(op_fn f, int a, int b) { return f(a, b); }
// aplica(sub, 10, 4) → 6
```

## O que dá e o que não dá

- `sizeof f` = **8** (um endereço, como qualquer ponteiro, [[Ponteiros em C]]); `f == soma` e `&soma == soma` deram **verdadeiro**: `&soma` e `soma` são o mesmo endereço.
- **Não há aritmética** de ponteiros para função (`f + 1` não faz sentido), e **não há `void *` portátil** para ele.
- Deve ser **verificado contra `NULL`** antes de chamar, se puder ser nulo; chamar `NULL` é UB.
- **Chamar com o tipo errado** (converter entre assinaturas diferentes) é **UB**.

No código de máquina é uma **chamada indireta**: em vez de `call soma`, o `gcc -S` de `aplica` gerou `call rcx`, saltando para o endereço que estava no **registrador** ([[Funções em C]] mostra a chamada direta).

## O primeiro *callback*: "aplique esta função a todos"

Antes de `qsort`, o exemplo mais simples: uma função que recebe **outra função** como parâmetro e a chama para cada elemento.

```c
int dobra(int x)    { return 2 * x; }
int quadrado(int x) { return x * x; }

void aplica_a_todos(int *v, size_t n, int (*f)(int)) {     // f: a função que você escolhe
    for (size_t i = 0; i < n; i++) v[i] = f(v[i]);
}

int v[] = {3, 1, 2};
aplica_a_todos(v, 3, dobra);        // 6 2 4
aplica_a_todos(v, 3, quadrado);     // 36 4 16   (aplicada sobre o resultado anterior)
```

A função `aplica_a_todos` **não sabe** o que `f` faz: só sabe chamá-la. Quem usa decide o comportamento **passando o nome da função**, sem parênteses: `dobra` é o **endereço** da função, e `dobra(...)` seria uma **chamada**. É a ideia de `map` em outras linguagens (em Go, funções também são valores, [[Funções — Fundamentos]]).

## Declarações difíceis: o `typedef` resolve

A declaração clássica `signal` da biblioteca padrão é famosa por ser ilegível:

```c
void (*signal(int sig, void (*h)(int)))(int);        // "signal é uma função que recebe (int, ponteiro p/ função) e devolve ponteiro p/ função"
typedef void (*handler_t)(int);                      // com um apelido, vira simples:
handler_t signal(int sig, handler_t h);
```

As duas formas compilaram e funcionaram (um *handler* devolvido foi chamado: `handler 7`). **Regra:** quando uma declaração tiver mais de um nível de ponteiro e parênteses, **crie um `typedef` para a parte de dentro** e escreva a de fora com ele.

---

# 📋 Tabelas de funções (*dispatch tables*)

Um **array de ponteiros para função** substitui uma cadeia de `if`/`switch` ([[Controle de Fluxo em C]]), escolhendo o comportamento por **índice** ou **chave**:

```c
struct { char simbolo; op_fn fn; } tab[] = { {'+', soma}, {'-', sub}, {'*', mul} };
for (size_t i = 0; i < sizeof tab / sizeof tab[0]; i++)
    printf("7 %c 3 = %d\n", tab[i].simbolo, tab[i].fn(7, 3));
// 7 + 3 = 10   7 - 3 = 4   7 * 3 = 21
```

(O `struct` anônimo é assunto da próxima unidade; aqui basta ver "um par símbolo + função".) Também com `op_fn tabela[] = { soma, sub, mul };`, `tabela[2](6, 7)` deu **42**. Esse é o padrão por trás de **menus**, **máquinas de estados** e das **tabelas de vetores de interrupção** de microcontroladores ([[MOC - Protocolos de Hardware]]).

---

# 📞 *Callbacks*

> 💡 ***Callback*:** uma função que você **entrega** a outra função para que **ela a chame** no momento certo. Quem escreve a biblioteca não conhece a sua lógica, e você não conhece o algoritmo dela.

O exemplo canônico da biblioteca padrão é **`qsort`** (de `<stdlib.h>`):

```c
void qsort(void *base, size_t n, size_t tamanho, int (*cmp)(const void *, const void *));
```

Como `qsort` ordena **qualquer tipo**, usa `void *` e delega a **comparação** ao seu *callback*, que recebe **ponteiros para dois elementos** e devolve **negativo, zero ou positivo**:

```c
int cmp_int(const void *pa, const void *pb) {
    int a = *(const int *)pa, b = *(const int *)pb;     // converte void * e desreferencia
    return (a > b) - (a < b);                            // -1, 0 ou 1
}
int v[] = {42, -7, 15, 0, 99, 3};
qsort(v, 6, sizeof v[0], cmp_int);                      // -7 0 3 15 42 99
```

| Teste real | Resultado |
|---|---|
| `qsort` de `int` com `cmp_int` | `-7 0 3 15 42 99` |
| `bsearch(&chave, v, 6, sizeof v[0], cmp_int)` com `chave = 15` (busca binária em array **ordenado**) | índice **3** |
| `qsort` de strings (`const char *`) com `strcmp` | `ana bruno carla ze` |

Para um **array de strings** o *callback* recebe um ponteiro **para** `const char *`, daí `strcmp(*(const char *const *)pa, *(const char *const *)pb)` ([[Strings em C]]).

> ⚠️ **Não compare por subtração.** `return *a - *b;` é tentador, mas **estoura**: com `INT_MAX` e `-1` deu `-2147483648` (deveria ser positivo) e o UBSan acusou `signed integer overflow: 2147483647 - -1`. Use `(a > b) - (a < b)` ([[Tipos de Dados em C]]).

## Ordem decrescente: o mesmo `qsort`, outro *callback*

Para ordenar do **maior para o menor**, não se muda o `qsort`: troca-se o **callback**, invertendo os argumentos:

```c
int cmp_decresc(const void *a, const void *b) { return cmp_int(b, a); }   // note: (b, a)
qsort(v, 6, sizeof v[0], cmp_decresc);                                     // 99 42 15 3 0 -7
```

## Callback com contexto (`void *ctx`)

C não tem *closures*: um *callback* não "lembra" variáveis do chamador. A solução tradicional é passar um ponteiro extra, **`void *contexto`**, que a biblioteca repassa sem olhar:

```c
typedef void (*visita_fn)(int valor, void *ctx);
void para_cada(const int *v, size_t n, visita_fn f, void *ctx) {
    for (size_t i = 0; i < n; i++) f(v[i], ctx);
}
void acumula(int valor, void *ctx) { *(long *)ctx += valor; }

long total = 0;
para_cada(w, 4, acumula, &total);        // total = 10  (para w = {1,2,3,4})
```

Quem **escreve** o *callback* converte `ctx` de volta ao tipo certo (`(long *)ctx`): é a **única** garantia, e o compilador **não verifica** ([[Conversões de Tipo em C]]). Esse padrão (`função + void *`) está em bibliotecas de eventos, *threads* e interrupções. Em Go as funções são valores com *closures* ([[Closures e Funções Anônimas]]).

---

# 🧯 Erros mais comuns

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Passar o valor quando a função espera ponteiro | `troca(a, b)` em vez de `troca(&a, &b)` | erro de tipo (aviso do `gcc`) ou *segfault* | `&` na chamada |
| Esquecer o `*` dentro da função | `a = b;` em vez de `*a = *b;` | só muda a **cópia do ponteiro** | `*a = *b;` |
| Devolver endereço de local | `return &x;` | ponteiro **pendurado** | devolver por valor, receber buffer ou `malloc` |
| Devolver `static` e guardar dois resultados | `a = rotulo(1), b = rotulo(2)` | `a == b` (mesmo buffer) | pedir um buffer ao chamador |
| Não testar o `NULL` devolvido | `*acha(s, 'z')` | `Segmentation fault` | `if (r != NULL)` |
| Chamar `f(x)` sem testar `f == NULL` | ponteiro de função nulo | UB | validar antes |
| Parênteses faltando | `int *f(int, int)` em vez de `int (*f)(int, int)` | declara uma **função**, não um ponteiro | `(*f)` |
| Comparar por subtração no `qsort` | `return *a - *b;` | **estouro** com `INT_MAX` e `-1` | `(a > b) - (a < b)` |
| Chamar função com assinatura errada | converter `int (*)(int)` em outra | UB | usar o tipo exato (ou `typedef`) |
| Passar a função **chamada** em vez do nome | `qsort(..., cmp_int(a, b))` | erro de tipo | `qsort(..., cmp_int)` |

---

# 🧪 Para praticar

1. Escreva `void troca(int *a, int *b)` e `void ordena2(int *a, int *b)`, que coloca o menor em `*a` e o maior em `*b` usando `troca`.
2. Escreva `int divide(int a, int b, int *q, int *r)` que devolve `0` em sucesso ou `-1` se `b == 0`, e devolve o quociente e o resto pelos ponteiros.
3. Escreva `const char *ultimo_char(const char *s, char c)` que devolve o ponteiro para a **última** ocorrência de `c` (ou `NULL`), sem usar `strrchr`.
4. Escreva `void mapeia(int *v, size_t n, int (*f)(int))` e use-a com `dobra`, `quadrado` e uma função sua (por exemplo, `negativo`).
5. Use `qsort` para ordenar um array de `double` em ordem **crescente** e depois **decrescente**; lembre de **não** usar subtração (compare com `(x > y) - (x < y)`).
6. Crie uma tabela de funções com `{'+', soma}`, `{'-', sub}`, `{'*', mul}`, `{'/', divide}` e uma calculadora que leia `a op b` e escolha a função pela tabela.

---

# ✅ Resumo do Conceito

- **Parâmetro por ponteiro:** `const T *` para entrada, `T *` para saída/alteração; padrão **status no retorno + resultado por ponteiro** (`dividir`); `T **` para alterar o ponteiro do chamador (`pula_espacos`)
- **Retorno de ponteiro:** nunca de **local**; seguro se aponta **dentro de um parâmetro**, para **`static`** (mesmo endereço sempre: `a == b`), literal ou memória **alocada**; `NULL` sinaliza falha
- **Ponteiro para função** = endereço do código: `int (*f)(int, int)`; `f = soma`; `f(2, 3)` ≡ `(*f)(2, 3)`; `sizeof` = 8; sem aritmética; chamada **indireta** (`call rcx`); use **`typedef`**
- **`int (*f)(...)`** (ponteiro) ≠ **`int *f(...)`** (função que devolve ponteiro)
- **Tabelas de funções:** array de ponteiros para função substitui `if`/`switch` (menus, máquinas de estados, vetores de interrupção)
- ***Callback*** = função entregue a outra; `qsort`/`bsearch` usam `void *` e um comparador `int (*)(const void *, const void *)`; **não compare por subtração** (overflow)
- **`void *ctx`** passa o contexto aos *callbacks*; sem verificação de tipos
- **Callback simples:** `aplica_a_todos(v, n, dobra)` passa o **nome** da função; a ordem do `qsort` (crescente × decrescente) muda só trocando o *callback*; `typedef` torna legíveis as declarações com mais de um nível (`signal`)

---

# 🔗 Conexões com outras anotações

- [[Funções em C]] — chamada direta, `static` e ABI
- [[Ponteiros em C]] — `const`, `void *`, ponteiro para ponteiro e `NULL`
- [[Arrays e Ponteiros em C]] — array + tamanho como parâmetro e `*(v + i)`
- [[Strings em C]] — `strcmp`, `snprintf` e buffers fornecidos pelo chamador
- [[Operadores em C]] — precedência de `()` e `*` nas declarações
- [[Controle de Fluxo em C]] — `switch` × tabela de funções
- [[Tipos de Dados em C]] — overflow de `int` no comparador
- [[Conversões de Tipo em C]] — conversões de `void *` e de assinaturas
- [[Como C Funciona]] — a região de texto onde vive o código
- [[MOC - Protocolos de Hardware]] — tabelas de interrupção e *callbacks* em firmware
- [[Funções — Fundamentos]] e [[Closures e Funções Anônimas]] — o equivalente em Go (funções como valores e *closures*)
