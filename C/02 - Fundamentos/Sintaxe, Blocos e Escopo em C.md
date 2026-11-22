---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23, seções de lexical elements, statements e scopes); experimentos com gcc 13.3 em Linux x86-64"
---
# Sintaxe, Blocos e Escopo em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 | Anterior: [[Convenções e Estilo]] | Próxima: [[Variáveis e Constantes em C]] | Compare com [[Sintaxe, Blocos e Escopo]] (Go)

---

# 🔤 Como o compilador "lê" o código: tokens

Na fase de **análise léxica** do compilador ([[O Compilador]]), o texto é quebrado em **tokens**. Um programa C é uma sequência de tokens de seis categorias:

| Categoria | O que é | Exemplos |
|---|---|---|
| **Palavras-chave** | Palavras **reservadas** pela linguagem | `int`, `if`, `return`, `while`, `struct` |
| **Identificadores** | Nomes que **você** dá a variáveis, funções e tipos | `contador`, `soma`, `lista_t` |
| **Constantes (literais)** | Valores escritos direto no código | `42`, `3.14`, `'A'` |
| **Literais de string** | Texto entre aspas duplas | `"Ola"` |
| **Operadores** | Símbolos que operam sobre valores | `+`, `==`, `&&`, `->` |
| **Pontuadores** | Símbolos de estrutura | `;`, `,`, `{ }`, `( )`, `[ ]` |

**Regras dos identificadores:** formados por **letras, dígitos e `_`**, **não** podem começar com dígito e **diferenciam maiúsculas de minúsculas** (`Total` ≠ `total`). Não podem ser palavras-chave, e convém evitar os nomes **reservados** vistos em [[Convenções e Estilo]].

> 💡 **Formato livre:** espaços, tabs e quebras de linha **só separam tokens** — não têm significado próprio (ao contrário de Python). `int x=1;` e `int   x =\n 1 ;` são idênticos para o compilador.

## O lexer é "ganancioso" (*maximal munch*)

O analisador léxico sempre forma o **token mais longo possível**. Testado:

```c
int a = 1, b = 2;
int c = a+++b;          // lido como (a++) + b, não a + (++b)
printf("c=%d a=%d\n", c, a);      // c=3 a=2
```

`a+++b` vira `a`, `++`, `+`, `b`. Moral: **use espaços** para deixar a intenção explícita (`a++ + b`).

---

# 🏗️ A estrutura de um arquivo C

> 💡 **Unidade de tradução (*translation unit*):** um arquivo `.c` **depois** do pré-processamento (o `.i` visto em [[Fases da Compilação]]), isto é, o seu código mais tudo que os `#include` colaram. É a unidade que o compilador traduz de cada vez, gerando um `.o`.

Dentro de uma unidade de tradução, só podem existir **declarações e definições** no **nível de arquivo**: variáveis globais, funções, tipos. **Instruções** (`if`, atribuições, chamadas) **só podem existir dentro de funções**.

```c
#include <stdio.h>            // diretiva (tratada pelo pré-processador)

int total = 0;                // definição de variável global (nível de arquivo)

int soma(int a, int b) {      // definição de função
    return a + b;             // instrução, dentro da função
}
```

---

# 🧱 Declarações, expressões e instruções

| Conceito | O que é | Exemplo |
|---|---|---|
| **Expressão** | Algo que **produz um valor** | `a + b`, `soma(2, 3)`, `x = 5` |
| **Instrução** (*statement*) | Uma **ação** do programa | `return 0;`, `if (...) {...}`, `x = 5;` |
| **Declaração** | **Apresenta** um nome (e seu tipo) ao compilador | `int x;`, `double media = 0.0;` |

- **Instrução de expressão:** uma expressão seguida de `;` vira instrução (`x = 5;`, `puts("oi");`).
- **O `;` termina** a instrução (em Pascal ele *separa*; em C ele *termina*, e faltar um é o erro de sintaxe mais comum).
- **Instrução nula:** um `;` sozinho não faz nada. Cuidado: `if (x);` ou `while (c);` terminam a instrução **ali** — o bloco que vem depois **sempre** executa/roda fora do `if`.
- Em C, **atribuição é expressão** (`x = 5` vale 5), o que permite `a = b = 0;` e também o clássico bug `if (x = 5)` (atribui em vez de comparar). O `-Wall` avisa (veja [[Convenções e Estilo]]).

---

# 📦 Blocos

> 💡 **Bloco (instrução composta):** conjunto de declarações e instruções entre `{ }`, tratado como **uma instrução só**. É o que permite `if`, `for`, `while` e funções terem **várias** instruções no corpo.

```c
if (n > 0) {          // bloco: várias instruções "viram uma"
    total += n;
    contagem++;
}
```

- Blocos **podem aninhar** e cada um cria um **novo escopo** (veja abaixo).
- **Onde declarar variáveis?** No C89 só no **início** do bloco; a partir do **C99**, em **qualquer ponto**, inclusive no `for` (`for (int i = 0; ...)`). Prefira declarar **perto do uso**.
- **Sempre use chaves**, mesmo para uma instrução (regra do `goto fail` em [[Convenções e Estilo]]).

## O `else` "pendurado" (*dangling else*)

Sem chaves, o `else` **pertence ao `if` mais próximo**, não ao que o **recuo** sugere:

```c
if (a)
    if (b) puts("A");
else puts("B");          // recuado como se fosse do 'if (a)'
```

Com `a = 1, b = 0`, a saída real é **`B`**: o `else` pertence ao `if (b)` — o recuo **mente**. O compilador avisa (`-Wdangling-else`: *suggest explicit braces*). Com chaves, a ambiguidade desaparece.

---

# 🔭 Escopo

> 💡 **Escopo:** a **região do código** onde um nome é **visível** (pode ser usado). Fora dela, o nome simplesmente não existe para o compilador.

C tem **quatro** tipos de escopo:

| Escopo | Onde o nome vale | Exemplo |
|---|---|---|
| **Arquivo** | Do ponto da declaração até o **fim do arquivo** | Variável global, função |
| **Bloco** | Do ponto da declaração até o `}` do bloco | Variável local, parâmetro, `i` do `for` |
| **Função** | A função inteira (**só para *labels*** do `goto`) | `fim:` |
| **Protótipo** | Só dentro dos parênteses de um protótipo | `int f(int tamanho);` |

## Sombreamento (*shadowing*)

Um nome declarado em um bloco **esconde** o de mesmo nome em um bloco externo:

```c
int g = 10;                          // escopo de arquivo
int main(void) {
    int x = 1;
    {
        int x = 2;                   // esconde o x externo
        printf("interno x=%d g=%d\n", x, g);    // interno x=2 g=10
    }
    printf("externo x=%d\n", x);                // externo x=1
}
```

Legal, mas **confunde**. O `-Wshadow` avisa: `declaration of 'x' shadows a previous local`. Por isso ele está na lista de flags de [[Convenções e Estilo]].

---

# ⏳ Escopo × tempo de vida

São conceitos **diferentes**:

> 💡 **Escopo** = **onde** o nome é visível (conceito do **código-fonte**, resolvido na compilação).
> **Tempo de vida** (*storage duration*) = **por quanto tempo** o objeto existe na memória (conceito da **execução**).

| Duração | Quando existe | Onde vive ([[Como C Funciona]]) | Exemplo |
|---|---|---|---|
| **Automática** | Do bloco até seu `}` | **Stack** | Variável local comum |
| **Estática** | **Durante todo** o programa | `.data` / `.bss` | Globais e `static` |
| **Alocada** | De `malloc` até `free` | **Heap** | `malloc(100)` |
| **De thread** | Enquanto a thread existe | Área da thread | `_Thread_local` |

Uma variável local `static` mostra a diferença: mesmo escopo de bloco, mas vida de programa inteiro. Resultado real de chamar `conta()` três vezes:

```c
void conta(void) {
    int a = 0;            // automática: recriada a cada chamada
    static int s = 0;     // estática: criada UMA vez, mantém o valor
    a++; s++;
    printf("auto=%d static=%d\n", a, s);
}
// auto=1 static=1   |   auto=1 static=2   |   auto=1 static=3
```

## Quando escopo e vida se desencontram: o ponteiro pendurado

Ao sair do bloco, uma variável **automática deixa de existir** (o espaço da pilha é reaproveitado). Devolver seu endereço é **comportamento indefinido** ([[Como C Funciona]]):

```c
int *f(void) { int x = 42; return &x; }    // x morre quando f retorna
```

```
warning: function returns address of local variable [-Wreturn-local-addr]
```

Isso se chama **ponteiro pendurado** (*dangling pointer*) e será aprofundado em [[Stack vs Heap em C]] e [[Erros Clássicos de Memória em C]]. Em Go o compilador move `x` para o heap automaticamente (*escape analysis*); **em C, a responsabilidade é sua**.

> 📌 **Mais à frente:** `static` e `extern` em **escopo de arquivo** controlam a **linkagem** (visibilidade **entre** arquivos `.c`), assunto da nota *static, extern e Linkagem*.

---

# ✅ Resumo do Conceito

- O compilador divide o código em **tokens**: palavras-chave, identificadores, constantes, strings, operadores e pontuadores; o formato é **livre** e o lexer é **ganancioso** (`a+++b` = `(a++) + b`)
- **Identificadores:** letras, dígitos e `_`, sem começar por dígito, **sensíveis a maiúsculas**
- **Unidade de tradução** = `.c` já pré-processado; no nível de arquivo só **declarações/definições**, **instruções só dentro de funções**
- **Expressão** produz valor; **instrução** é uma ação terminada em `;`; atribuição **é** expressão (cuidado com `if (x = 5)`)
- **Bloco `{ }`** = instrução composta e **novo escopo**; declare variáveis **perto do uso** (C99); **sempre use chaves** (o `else` pertence ao `if` mais próximo)
- **Escopo** (onde o nome é visível): **arquivo, bloco, função (labels), protótipo**; blocos internos podem **sombrear** nomes (`-Wshadow`)
- **Escopo ≠ tempo de vida:** automática (stack), estática (`.data`/`.bss`), alocada (heap), de thread; `static` local mantém o valor entre chamadas
- Devolver o endereço de uma **local** gera **ponteiro pendurado** (UB)

---

# 🔗 Conexões com outras anotações

- [[O Compilador]] — análise léxica e sintática, onde os tokens nascem
- [[Fases da Compilação]] — o pré-processador e a unidade de tradução (`.i`)
- [[Como C Funciona]] — regiões de memória e UB
- [[Convenções e Estilo]] — chaves obrigatórias, `-Wshadow`, nomes reservados
- [[Sintaxe, Blocos e Escopo]] — o mesmo assunto em Go (compare: Go tem `:=` e *escape analysis*)
- [[Pilha (Stack)]] — a estrutura por trás das variáveis automáticas
- [[Variáveis e Constantes em C]] — próximo passo: declarar, definir e inicializar
