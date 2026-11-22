---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: ponteiros, qualificadores, void *); experimentos com gcc 13.3, AddressSanitizer e UBSan em Linux x86-64"
---
# Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `-fsanitize=undefined` e `-fsanitize=address` | Anterior: [[Funções Variádicas em C]] | Próxima: [[Aritmética de Ponteiros em C]] | Compare com [[Ponteiros]] (Go)

---

# 📍 O que é um ponteiro

Lembrando [[Arrays em C]]: **endereço** é o número que identifica a posição de um byte na memória.

> 💡 **Ponteiro:** uma **variável cujo valor é um endereço**. Ele "aponta" para outro objeto: em vez de guardar o dado, guarda **onde o dado está**.

```
        x                    p
   ┌──────────┐         ┌──────────────┐
   │    10    │ ◄────── │ 0x7ffe…1c0   │     p guarda o ENDEREÇO de x
   └──────────┘         └──────────────┘
   endereço 0x7ffe…1c0   endereço 0x7ffe…1d8  (o próprio p também tem endereço)
```

Para que serve? Para **acessar ou alterar** um objeto **sem copiá-lo**, para que uma função modifique variáveis do chamador, para percorrer arrays e para trabalhar com memória alocada dinamicamente. É a ferramenta que dá a C o controle direto da memória ([[O que é C]]).

---

# 🔧 Os dois operadores: `&` e `*`

| Operador | Nome | Significado | Exemplo |
|---|---|---|---|
| `&x` | **endereço de** | o endereço de `x` | `int *p = &x;` |
| `*p` | **desreferência** (*dereference*) | o objeto **para onde `p` aponta** | `*p = 20;` |

```c
int x = 10;
int *p = &x;            // p aponta para x
*p = 20;                // altera x através de p
```

Resultados reais: `x=10`, `p` igual a `&x`, `*p=10`; depois de `*p = 20`, **`x` vale 20**.

> ⚠️ **O `*` tem dois papéis:** na **declaração** (`int *p`) significa "`p` **é** um ponteiro para `int`"; numa **expressão** (`*p`) significa "o valor **apontado** por `p`". O `&` também tem dois (E bit a bit, endereço de), como visto em [[Operadores em C]].

## Lendo uma declaração

`int *p;` lê-se "`p` é um ponteiro para `int`". O tipo do ponteiro é **`int *`**, e a declaração `int *a, b;` cria **só `a` como ponteiro**; `b` é um `int` (o `*` pertence ao nome). Por isso muitos preferem **uma declaração por linha**, como em [[Convenções e Estilo]].

---

# 🧬 O tipo do ponteiro importa

Todo ponteiro de dados tem o **mesmo tamanho**: **8 bytes** em x86-64 (`sizeof(int *)`, `sizeof(char *)` e `sizeof(double *)` deram 8), pois guardam um endereço e a CPU usa endereços de 64 bits ([[Processadores]]). Então para que serve o tipo? Para dizer ao compilador **quantos bytes ler e como interpretá-los** ao desreferenciar:

| Expressão | Lê | Porque |
|---|---|---|
| `*pi` com `int *pi` | 4 bytes, como `int` | `sizeof *pi` = 4 |
| `*pc` com `char *pc` | 1 byte | `sizeof *pc` = 1 |
| `*pd` com `double *pd` | 8 bytes, como `double` | `sizeof *pd` = 8 |

O mesmo endereço, lido por tipos diferentes, dá dados diferentes. Exemplo real: um `uint32_t v = 0x04030201;` visto byte a byte por um `unsigned char *`:

```c
unsigned char *b = (unsigned char *)&v;
// b[0] b[1] b[2] b[3]  →  01 02 03 04
```

O **byte menos significativo (01) está no menor endereço**: é a ordem ***little-endian*** do x86 ([[Bits e Bytes]]). Ler a memória como `unsigned char` é a forma permitida de inspecionar **qualquer** objeto byte a byte (importante em protocolos e *firmware*, [[MOC - Protocolos de Hardware]]).

> ✅ Escreva `sizeof *p` em vez de `sizeof(int)` ao alocar ou copiar: o tamanho acompanha o tipo se ele mudar ([[Tipos de Dados em C]]).

## Endereços são virtuais e mudam

O endereço impresso (`0x7ffe…`) **não é** a posição física na RAM, e sim um **endereço virtual** do processo ([[Memória Virtual]], [[Espaços de Endereçamento]]). Executando o mesmo programa duas vezes, os valores **mudaram** (`0x7ffccd8e4ac0` e `0x7ffed391acf0`): é a randomização do kernel (ASLR). Nunca dependa do valor numérico de um endereço.

---

# 🚫 O ponteiro nulo e o ponteiro "selvagem"

Um ponteiro só é útil se apontar para algo **válido**. Há quatro situações:

| Estado | Como surge | Efeito ao desreferenciar |
|---|---|---|
| **Válido** | `p = &x;` | funciona |
| **Nulo** | `p = NULL;` (ou `0`) | **UB**; na prática, *segmentation fault* |
| **Selvagem** (*wild*) | `int *p;` **sem inicializar** (valor = lixo) | **UB**: pode ler/escrever em qualquer lugar |
| **Pendurado** (*dangling*) | aponta para memória que **já acabou** ([[Sintaxe, Blocos e Escopo em C]]) | **UB** |

**`NULL`** é o valor reservado para "não aponta para nada" (de `<stddef.h>`; o inteiro `0` em contexto de ponteiro também vale, e `p == NULL` e `!p` dão o mesmo resultado, testado). Desreferenciar `NULL` é UB, mas o sistema operacional **não mapeia** o endereço 0, então o programa **morre com `Segmentation fault`** (código de saída 139). Com o UBSan a mensagem é clara: `load of null pointer of type 'int'`.

> ✅ **Regras:** (1) **inicialize** todo ponteiro (`= &x`, `= NULL` ou `= malloc(...)`); (2) **verifique** se é `NULL` antes de usar quando ele puder falhar (`malloc`, `fopen`); (3) depois de liberar a memória, **atribua `NULL`** ao ponteiro. A nota *Erros Clássicos de Memória* detalha os danos.

---

# 🪆 Ponteiro para ponteiro

Como um ponteiro é uma variável, **ele também tem endereço**, que pode ser guardado em outro ponteiro:

```c
int x = 10;
int  *p  = &x;        // p  aponta para x
int **pp = &p;        // pp aponta para p
**pp = 30;            // *pp é p; **pp é x  →  x agora vale 30
```

O teste deu `x=30`, `*p=30` e `**pp=30`. Cada `*` "dá um passo" na cadeia. Aparece em funções que precisam **alterar um ponteiro do chamador** e em arrays de strings (`char **argv`, o segundo parâmetro de `main`, [[Como C Funciona]]).

---

# 📞 Passagem por valor e simulação de "por referência"

Em C **todo argumento é passado por valor**: a função recebe uma **cópia**. Alterar a cópia não afeta o original:

```c
void falha_copia(int a) { a = 99; }       // altera só a cópia
// m = 1;  falha_copia(m);  →  m continua 1
```

Para a função **modificar a variável do chamador**, passe seu **endereço** (o ponteiro é copiado, mas **aponta para o mesmo objeto**):

```c
void troca(int *a, int *b) {
    int t = *a;  *a = *b;  *b = t;
}
// m = 1, n = 2;  troca(&m, &n);  →  m = 2, n = 1
```

Esse é o único jeito de uma função devolver **mais de um resultado** (parâmetros de saída):

```c
void divide(int a, int b, int *q, int *r) { *q = a / b;  *r = a % b; }
// divide(17, 5, &q, &r)  →  q = 3, r = 2
```

> 💡 Os arrays seguem a mesma ideia: ao passá-los, a função recebe um **endereço** e por isso altera o original (visto em [[Arrays em C]]; explicação em [[Arrays e Ponteiros em C]]). Em Go o conceito é o mesmo, porém sem aritmética ([[Ponteiros]]).

---

# 🔒 `const` e ponteiros

O `const` pode proteger **o dado** apontado, **o ponteiro** ou **ambos**. Regra de leitura: **da direita para a esquerda**.

| Declaração | Lê-se | Pode mudar `*p`? | Pode mudar `p`? |
|---|---|---|---|
| `const int *p` | `p` é ponteiro para `int` **constante** | ❌ | ✅ |
| `int *const p` | `p` é ponteiro **constante** para `int` | ✅ | ❌ |
| `const int *const p` | ponteiro **constante** para `int` **constante** | ❌ | ❌ |

Erros reais do `gcc`: `*p1 = 5` → *assignment of read-only location '\*p1'*; `p2 = &b` → *assignment of read-only variable 'p2'*.

A forma mais usada é **`const T *`** em **parâmetros**: promete que a função **só lê** o dado (`size_t tamanho(const char *s)`), documentando e protegendo ([[Convenções e Estilo]]; `const` em variáveis está em [[Variáveis e Constantes em C]]).

---

# 🧩 `void *`: o ponteiro genérico

`void *` aponta para **qualquer tipo**, mas **sem saber qual**: por isso **não pode ser desreferenciado** diretamente (não se sabe quantos bytes ler). Em C ele converte **implicitamente** de e para qualquer ponteiro de dados:

```c
double d = 2.5;
void *gen = &d;
double *pd = gen;          // conversão implícita em C (em C++ exigiria cast)
printf("%.1f", *pd);       // 2.5
```

É a base de `malloc` (devolve `void *`), de `memcpy` e de funções genéricas como `qsort` ([[Conversões de Tipo em C]] mostra o papel do `void *` e do `uintptr_t`). Para imprimir um endereço com `%p`, converta para `void *`: `printf("%p", (void *)p)`.

---

# ✅ Resumo do Conceito

- **Ponteiro** = variável que guarda um **endereço**; `&x` obtém o endereço, `*p` acessa o objeto apontado; o `*` na declaração significa "ponteiro para"
- Todo ponteiro de dados tem **8 bytes** (x86-64); o **tipo** define **quantos bytes ler** e como interpretá-los (por isso `unsigned char *` inspeciona bytes: `0x04030201` aparece como `01 02 03 04`, *little-endian*)
- Endereços são **virtuais** e **mudam a cada execução** (ASLR)
- **Inicialize sempre**: `NULL` indica "nada"; desreferenciar `NULL`, ponteiro **selvagem** (não inicializado) ou **pendurado** é **UB** (na prática, *segfault* no caso nulo)
- **`**pp`** percorre ponteiro para ponteiro
- Em C tudo é **passado por valor**; para alterar uma variável do chamador (ou devolver vários resultados) passe seu **endereço**
- **`const int *p`** protege o dado, **`int *const p`** protege o ponteiro; use `const T *` em parâmetros somente leitura
- **`void *`** é genérico: converte implicitamente em C e **não pode ser desreferenciado**

---

# 🔗 Conexões com outras anotações

- [[Arrays em C]] — endereços de elementos contíguos e arrays passados a funções
- [[Aritmética de Ponteiros em C]] — somar e subtrair ponteiros (próxima nota)
- [[Arrays e Ponteiros em C]] — por que um array "vira" um ponteiro
- [[Tipos de Dados em C]] — `sizeof`, `size_t`, `uintptr_t` e `void`
- [[Sintaxe, Blocos e Escopo em C]] — ponteiro pendurado e tempo de vida
- [[Operadores em C]] — `&` e `*` unários e sua precedência
- [[Conversões de Tipo em C]] — conversões entre ponteiros e `void *`
- [[Como C Funciona]] — `main`, `argv` e regiões de memória
- [[Memória Virtual]] e [[Espaços de Endereçamento]] — o que um endereço realmente é
- [[Bits e Bytes]] — *endianness* e representação binária
- [[MOC - Protocolos de Hardware]] — leitura de bytes e registradores
- [[Ponteiros]] — o equivalente em Go
