---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: ponteiros, qualificadores, void *); experimentos com gcc 13.3, AddressSanitizer e UBSan em Linux x86-64"
---
# Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `-fsanitize=undefined` e `-fsanitize=address` | Anterior: [[Funções Variádicas em C]] | Próxima: [[Aritmética de Ponteiros em C]] | Compare com [[Ponteiros]] (Go)

> 🧭 **Como ler esta nota:** ponteiros são o assunto que mais assusta quem aprende C, mas a ideia é uma só: **uma variável que guarda um endereço**. Vá em ordem: o que é e os dois operadores (`&` e `*`), um programa passo a passo, passagem para funções (onde ponteiros se tornam indispensáveis), `const`, `void *` e, no fim, os erros comuns e exercícios.

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

## Uma analogia

Imagine um **prédio com apartamentos numerados** (a memória). Uma variável `int x` é **um apartamento** com um morador (o valor `10`). O **endereço** é o **número do apartamento**. Um **ponteiro** é **um papel com o número de um apartamento anotado**: o papel não é o apartamento, nem o morador; ele só diz **onde** encontrá-los. Com o papel você pode ir até lá e **ler** ou **trocar** o morador. E o papel também é um objeto: ele fica guardado em **algum lugar** (tem o seu próprio endereço).

## As três coisas que se pode perguntar sobre um ponteiro

Dado `int x = 10; int *p = &x;`, há **três valores** diferentes envolvidos, e a maior parte da confusão vem de misturá-los. Saída real de um programa que imprime cada um:

| Expressão | O que é | Valor no teste |
|---|---|---|
| `x` | o **dado** (o morador) | `10` |
| `&x` | o **endereço** de `x` (o número do apartamento) | `0x7fff80e239b8` |
| `p` | o **conteúdo** do ponteiro: um endereço | `0x7fff80e239b8` (**igual a `&x`**, por isso `p == &x` deu `1`) |
| `*p` | o dado **para onde `p` aponta** | `10` (o mesmo que `x`) |
| `&p` | o endereço **do próprio ponteiro** | `0x7fff80e239c8` (outro lugar da memória) |
| `sizeof x` / `sizeof p` | tamanho do `int` / do ponteiro | `4` / `8` |

Resumo para guardar: **`p` é um endereço; `*p` é o dado nesse endereço; `&p` é onde o próprio `p` mora.**

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

# 🔬 Seguindo um programa passo a passo

Acompanhe o estado das variáveis a cada linha (saídas reais):

```c
int x = 10;
int *p = &x;          // (1) p aponta para x
*p = 20;              // (2) escreve em x através de p
int y = 5;
p = &y;               // (3) p muda de alvo: agora aponta para y
*p += 1;              // (4) altera y, não x
int *q = p;           // (5) q é uma cópia do ENDEREÇO em p
*q = 100;             // (6) altera y através de q
```

| Depois de | `x` | `y` | `p` aponta para | `q` aponta para |
|---|---|---|---|---|
| (1) | 10 | — | `x` | — |
| (2) `*p = 20` | **20** | — | `x` | — |
| (3) `p = &y` | 20 | 5 | **`y`** | — |
| (4) `*p += 1` | 20 | **6** | `y` | — |
| (5) `q = p` | 20 | 6 | `y` | `y` |
| (6) `*q = 100` | 20 | **100** | `y` | `y` |

Três lições:

- **Trocar o alvo** (`p = &y`) e **alterar o dado** (`*p = 20`) são coisas diferentes: sem o `*`, mexe-se no **ponteiro**; com o `*`, no **dado apontado**.
- **Dois ponteiros para o mesmo objeto** (`p` e `q`) são **apelidos** (*alias*): após `*q = 100`, `*p` também vale 100 (e `p == q` deu `1`).
- **Comparar ponteiros compara endereços, não dados.** Com `a = 7` e `b = 7` e `pa = &a`, `pb = &b`: `*pa == *pb` deu **1** (mesmo valor) e `pa == pb` deu **0** (objetos diferentes).

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

> ✅ **Regras:** (1) **inicialize** todo ponteiro (`= &x`, `= NULL` ou `= malloc(...)`); (2) **verifique** se é `NULL` antes de usar quando ele puder falhar (`malloc`, `fopen`); (3) depois de liberar a memória, **atribua `NULL`** ao ponteiro. A nota [[Erros Clássicos de Memória em C]] detalha os danos.

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

## Você já usa ponteiros: o `scanf`

Por que `scanf("%d", &idade)` leva um **`&`**? Porque o `scanf` é uma função: se recebesse `idade`, receberia uma **cópia** e não conseguiria alterar o original. Recebendo `&idade` (o **endereço**), ele **grava o valor digitado** direto na variável. Teste real: `scanf("%d %lf %c", &idade, &nota, &c)` com a entrada `30 8.5 x` deu `idade=30 nota=8.5 c=x`.

Esquecer o `&` é um erro clássico: `scanf("%d", idade)` entrega o **valor** (lixo) como se fosse um endereço. O `gcc` avisa (`format '%d' expects argument of type 'int *', but argument 2 has type 'int'`) e, ao executar, o programa terminou com **`Segmentation fault`**. Observe também: as variáveis com `%s` (strings, [[Strings em C]]) **não levam `&`**, pois o próprio array já é, na prática, um endereço ([[Arrays e Ponteiros em C]]).

## Mais exemplos de parâmetros de saída

```c
void incrementa(int *c) { (*c)++; }                     // os parênteses importam: (*c)++
// int cont = 0; incrementa(&cont); incrementa(&cont);  →  cont = 2

void media(const int *v, int n, double *saida) {        // devolve o resultado POR PARÂMETRO
    int s = 0;
    for (int i = 0; i < n; i++) s += v[i];
    *saida = (double)s / n;
}
// int v[] = {4, 6, 8}; double m; media(v, 3, &m);      →  m = 6.0
```

Repare que, nas duas funções, o **`*`** aparece **dentro** da função (para acessar o dado) e o **`&`** aparece **na chamada** (para entregar o endereço). Leia sempre assim: **`&` para dar o endereço, `*` para usar o dado**. E `*c++` não é `(*c)++`: sem parênteses, o `++` anda o **ponteiro** (precedência em [[Operadores em C]], percurso em [[Aritmética de Ponteiros em C]]).

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

# 🧯 Erros mais comuns

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Não inicializar | `int *p; *p = 5;` | ponteiro **selvagem**: UB | `int *p = &x;` ou `= NULL` |
| Desreferenciar `NULL` | `int *p = NULL; *p = 1;` | `Segmentation fault` | testar `if (p != NULL)` antes |
| Confundir `p` com `*p` | `p = 20;` (queria `*p = 20;`) | o **endereço** vira 20 (aviso do `gcc`) | `*p = 20;` |
| `*` na declaração mal lido | `int *a, b;` | só `a` é ponteiro (`sizeof a` = 8, `sizeof b` = 4) | uma declaração por linha |
| `scanf` sem `&` | `scanf("%d", n);` | `Segmentation fault` | `scanf("%d", &n);` |
| Comparar dados com `==` entre ponteiros | `if (pa == pb)` | compara **endereços** | `if (*pa == *pb)` |
| Guardar o endereço de uma local e usar depois | `return &x;` (x local) | ponteiro **pendurado** | devolver por valor ou usar `malloc` ([[Funções e Ponteiros em C]]) |
| `(*c)++` escrito como `*c++` | `*c++` | anda o ponteiro, não o dado | `(*c)++` |

Para detectar: `-Wall -Wextra` e `-fsanitize=address,undefined` ([[Erros Clássicos de Memória em C]]).

---

# 🧪 Para praticar

1. Declare `int a = 3, b = 4;` e `int *pa = &a, *pb = &b;`. Sem executar, diga o valor de `*pa + *pb`, de `pa == pb` e de `*pa == a`. Depois confira.
2. Escreva `void zera(int *x)` que coloca 0 na variável do chamador, e `void troca(int *a, int *b)` sem olhar a nota.
3. Escreva `void min_max(const int *v, int n, int *min, int *max)` que devolve o menor e o maior valor por **parâmetros de saída**.
4. Escreva um programa que lê dois inteiros com `scanf` e imprime a soma usando uma função `void soma(int a, int b, int *resultado)`.
5. Imprima `&x`, `p` e `&p` de um programa seu. Quais dois coincidem? Rode duas vezes: o que muda?

---

# ✅ Resumo do Conceito

- **Ponteiro** = variável que guarda um **endereço**; `&x` obtém o endereço, `*p` acessa o objeto apontado; o `*` na declaração significa "ponteiro para"
- Todo ponteiro de dados tem **8 bytes** (x86-64); o **tipo** define **quantos bytes ler** e como interpretá-los (por isso `unsigned char *` inspeciona bytes: `0x04030201` aparece como `01 02 03 04`, *little-endian*)
- Endereços são **virtuais** e **mudam a cada execução** (ASLR)
- **Inicialize sempre**: `NULL` indica "nada"; desreferenciar `NULL`, ponteiro **selvagem** (não inicializado) ou **pendurado** é **UB** (na prática, *segfault* no caso nulo)
- **`**pp`** percorre ponteiro para ponteiro
- Em C tudo é **passado por valor**; para alterar uma variável do chamador (ou devolver vários resultados) passe seu **endereço**
- **`const int *p`** protege o dado, **`int *const p`** protege o ponteiro; use `const T *` em parâmetros somente leitura
- **`p`** é um endereço, **`*p`** é o dado nesse endereço, **`&p`** é onde o próprio ponteiro mora; **trocar o alvo** (`p = &y`) ≠ **alterar o dado** (`*p = 20`); dois ponteiros para o mesmo objeto são **apelidos**; `pa == pb` compara **endereços**
- **`scanf` precisa do `&`** (recebe o endereço para gravar o valor); sem ele, *segfault*; parâmetros de saída: `&` na chamada, `*` dentro da função
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
