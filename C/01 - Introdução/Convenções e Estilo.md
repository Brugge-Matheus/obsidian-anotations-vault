---
tags:
  - c
  - c/introducao
  - c/estilo
source: "Linux kernel coding style, GNU Coding Standards, CERT C, MISRA C, manual do GCC e do clang-format; saídas geradas com gcc 13.3 e clang-format 18"
---
# Convenções e Estilo

📚 **Referência:** Linux kernel coding style, GNU Coding Standards, CERT C Coding Standard, MISRA C, manuais do GCC e do clang-format | Anterior: [[Fases da Compilação]] | Compare com [[Convenções da linguagem]] (Go)

---

# 🎯 Por que estilo importa mais em C?

Em Go, o `gofmt` impõe **um único estilo** e o compilador recusa certos descuidos. Em C **não existe nada disso**: a linguagem aceita praticamente qualquer formatação e **não protege** você de muita coisa (veja [[O que é C]]). Por isso as convenções em C não são só estética — elas são uma **ferramenta de prevenção de bugs**.

> 💡 **Convenção de código (*coding style*):** conjunto de regras combinadas sobre **nomes, formatação, organização e uso da linguagem**, que deixa o código previsível. Não muda o que o programa faz, mas muda o quanto é fácil **lê-lo** e **achar erros** nele.

## O exemplo clássico: o `goto fail`

Em 2014, uma falha de segurança grave no código de verificação de certificados SSL/TLS da Apple foi causada por uma linha **duplicada**:

```c
if ((err = SSLHashSHA1.update(&hashCtx, &signedParams)) != 0)
    goto fail;
    goto fail;          // ← linha duplicada: SEMPRE executada, sai antes de verificar a assinatura
```

Sem **chaves**, só a **primeira** linha pertence ao `if`. A segunda `goto fail;` roda **sempre**, pulando a verificação. Com a regra "**sempre use chaves**", o erro seria visível a olho nu. É por isso que muitos guias de estilo a tornam obrigatória.

---

# 🏷️ Nomes

C **não tem namespaces** nem pacotes: todos os nomes globais vivem no mesmo "espaço". As convenções de nome existem, em grande parte, para **evitar colisões** e **comunicar o tipo de coisa** que o nome representa.

## O padrão mais comum

| Elemento | Convenção | Exemplo |
|---|---|---|
| **Variáveis e funções** | `snake_case` (minúsculas com `_`) | `contador`, `ler_arquivo`, `list_push` |
| **Constantes e macros** | `MAIUSCULAS_COM_UNDERSCORE` | `BUFFER_SIZE`, `MAX_USUARIOS` |
| **Tipos** (`typedef`, `struct`) | `snake_case` ou `PascalCase`; muitos projetos usam sufixo `_t` | `node_t`, `Node` |
| **Valores de `enum`** | Maiúsculas, **com prefixo** do tipo | `COR_VERMELHO`, `COR_VERDE` |
| **Variáveis globais** | Prefixo ou nome bem descritivo (evite-as) | `g_config` |

> 💡 **`snake_case`:** palavras minúsculas separadas por `_` (`tamanho_da_lista`). É o estilo da própria **biblioteca padrão** (`strlen`, `malloc`, `fopen`) e do kernel Linux. O oposto é o `camelCase` (`tamanhoDaLista`), comum em Java e Go.

Diferente de Go, onde **maiúscula no início** define o que é exportado (veja [[Convenções da linguagem]]), em C a visibilidade vem de **`static`** e de **arquivos de cabeçalho** — o nome não decide nada, é só convenção.

## Prefixo de módulo: a "falsa namespace"

Como não há namespaces, usa-se um **prefixo por módulo** para agrupar e evitar colisões:

```c
// módulo "lista"
lista_t *lista_criar(void);
void     lista_inserir(lista_t *l, int valor);
void     lista_destruir(lista_t *l);

// módulo "arquivo" — sem conflito com "lista_*"
arquivo_t *arquivo_abrir(const char *caminho);
```

Compare com a biblioteca padrão: todas as funções de string começam com `str` (`strlen`, `strcpy`, `strcmp`), as de arquivo com `f` (`fopen`, `fclose`, `fread`).

## Nomes que você **não deve** usar

A linguagem e o sistema **reservam** alguns nomes:

| Padrão do nome | Por que evitar |
|---|---|
| Começa com `_` **+ letra maiúscula** (`_Contador`) ou **dois `_`** (`__x`) | **Reservado** para o compilador e a biblioteca, em **qualquer** escopo |
| Começa com `_` no **escopo de arquivo** (global) | Reservado para a implementação |
| Termina em **`_t`** (`meu_tipo_t`) | O **POSIX** reserva o sufixo `_t` para tipos do sistema (`size_t`, `pid_t`); funciona na prática, mas pode colidir no futuro |
| Nomes de funções da libc (`index`, `time`, `exit`, `read`) | Você **sobrescreve** a função da biblioteca sem perceber |
| `str...`, `mem...`, `is...`, `to...` + minúscula | Reservados para **futuras** funções da biblioteca padrão |

> ⚠️ Definir uma função chamada `read` ou `time` no seu programa pode conflitar com as da `libc` e gerar erros estranhos de **linkagem** (veja [[Fases da Compilação]]).

## Boas práticas de nomeação

- **Nomes descritivos**: `contagem_de_linhas` em vez de `n`; abreviações só quando **universais** (`i` em laços curtos, `buf`, `ptr`, `len`);
- **Funções são verbos** (`calcular_media`, `abrir_arquivo`); **variáveis e tipos são substantivos**;
- **Booleanos soam como perguntas**: `esta_vazio`, `tem_erro`;
- **Evite nomes de uma letra** fora de laços e coordenadas (`x`, `y`);
- **Seja consistente** com o projeto — a regra mais importante é **seguir o estilo que já existe**.

---

# 🎨 Formatação

## Indentação e largura

| Decisão | Opções comuns |
|---|---|
| **Indentação** | **4 espaços** (muito comum), **8 colunas com TAB** (kernel Linux), **2 espaços** (GNU) |
| **Largura da linha** | **80** colunas (tradicional) ou **100** (moderno) |
| **Tabs × espaços** | Escolha **um** e não misture; configure o editor |

## Estilos de chaves: onde fica a `{`?

Há três "escolas" famosas. O **mesmo código** (uma função `maximo`) formatado nas três, pelo `clang-format`:

**K&R / Linux** — a chave da **função** vai para a linha de baixo; as chaves de **`if`/`for`/`while`** ficam na **mesma linha**:

```c
int maximo(int a, int b)
{
    if (a > b) {
        return a;
    } else {
        return b;
    }
}
```

**Allman** — **toda** chave vai para a linha de baixo, sozinha:

```c
int maximo(int a, int b)
{
    if (a > b)
    {
        return a;
    }
    else
    {
        return b;
    }
}
```

**GNU** — nome da função em **linha própria**, indentação de **2** e chaves indentadas junto com o bloco:

```c
int
maximo (int a, int b)
{
  if (a > b)
    {
      return a;
    }
  else
    {
      return b;
    }
}
```

| Estilo | Quem usa | Característica |
|---|---|---|
| **K&R** (variante 1TBS) | Livro *K&R*, **kernel Linux**, a maior parte do mundo C | Compacto; fecha com `} else {` |
| **Allman** | Muitos projetos Windows e ensino | Mais "arejado" |
| **GNU** | Projetos GNU (GCC, glibc) | Distinto; indentação de 2 |

> 💡 **Não existe estilo "certo".** O que importa é **escolher um** e **ser consistente**. Neste vault, a proposta é **K&R com 4 espaços** — compacto e o mais encontrado em livros e projetos.

## Regras que valem em qualquer estilo

```c
// ✅ SEMPRE use chaves, mesmo com uma só instrução
if (erro) {
    return -1;
}

// ❌ Sem chaves: convida ao bug do "goto fail"
if (erro)
    return -1;
```

- **Uma declaração por linha:** `int a;` e `int b;` em vez de `int a, b;`
- **Espaço em volta de operadores** (`a + b`, `x = 5`) e **depois de vírgulas**; **sem** espaço entre o nome da função e o parêntese (`soma(1, 2)`), **com** espaço depois de `if`/`for`/`while` (`if (x)`)
- **Declare variáveis perto do uso** e no menor escopo possível (C99 permite declarar no meio do bloco e no `for`)
- **Linhas em branco** separando blocos lógicos

## O asterisco do ponteiro: um detalhe que engana

Onde colocar o `*`: junto do **tipo** (`char* p`) ou do **nome** (`char *p`)? A convenção em C é **junto do nome**, porque o `*` **pertence ao declarador**, não ao tipo:

```c
char* a, b;     // ⚠️ só 'a' é ponteiro! 'b' é um char comum
char *c, *d;    // c e d são ambos ponteiros
```

O resultado real, com `sizeof`:

```
a=8 b=1 c=8 d=8
```

`a` tem 8 bytes (ponteiro), mas **`b` tem 1 byte** (um `char`). O espaçamento `char* a, b;` **mente** sobre o que acontece. Por isso: escreva `char *a;` e **uma declaração por linha**. (Você verá o assunto em *Ponteiros*; compare com [[Ponteiros]] em Go.)

---

# 🛠️ Automatize: `clang-format` e `.editorconfig`

Como em C **nada** formata o código por você, use ferramentas — e **pare de discutir estilo manualmente**.

> 💡 **clang-format:** ferramenta que **reformata** código C/C++ segundo um estilo configurável. É o "gofmt do C", só que **configurável**.

Um arquivo **`.clang-format`** na raiz do projeto descreve o estilo:

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
BreakBeforeBraces: Linux      # chave da função embaixo; if/for na mesma linha
InsertBraces: true            # coloca chaves onde faltarem (cuidado: revise o resultado)
ColumnLimit: 80
```

Uso:

```bash
clang-format -i programa.c     # reformata o arquivo no lugar
clang-format --dry-run -Werror *.c   # só verifica (útil em CI)
```

Para o editor respeitar **tabs/espaços e fim de linha**, o arquivo `.editorconfig` (suportado por quase todos os editores) fixa as bases:

```ini
root = true
[*.{c,h}]
indent_style = space
indent_size = 4
end_of_line = lf
insert_final_newline = true
```

---

# 🚨 Avisos do compilador são parte do estilo

Em C, o compilador é seu **primeiro revisor de código** — mas **só se você pedir**. Compare. O mesmo arquivo `bad.c`, com vários descuidos:

```c
#include <stdio.h>

int main(void) {
    int n;                       // nunca recebe valor
    int sem_uso = 3;             // nunca usada
    unsigned int u = 10;
    int i = -1;
    if (i < u) puts("menor");    // compara int (negativo) com unsigned
    printf("%d\n", n);           // usa n sem inicializar
    printf("%s\n", 42);          // %s com um inteiro
    long big = 1L << 40;
    int small = big;             // perde bits (long → int)
    return;                      // main devolve int, mas não devolve nada
}
```

**Sem flags** (`gcc bad.c`) o compilador reclamou de **apenas 2** coisas:

```
bad.c:10:14: warning: format '%s' expects argument of type 'char *', but argument 2 has type 'int' [-Wformat=]
bad.c:13:5:  warning: 'return' with no value, in function returning non-void
```

**Com `-Wall -Wextra -Wconversion`** apareceram **mais cinco**:

```
bad.c:8:11:  warning: comparison of integer expressions of different signedness: 'int' and 'unsigned int' [-Wsign-compare]
bad.c:12:17: warning: conversion from 'long int' to 'int' may change value [-Wconversion]
bad.c:12:9:  warning: unused variable 'small' [-Wunused-variable]
bad.c:5:9:   warning: unused variable 'sem_uso' [-Wunused-variable]
bad.c:9:5:   warning: 'n' is used uninitialized [-Wuninitialized]
```

Isto mostra por que **ligar os avisos é a regra de estilo mais importante** para C: **cinco dos sete problemas** só foram vistos depois de pedir. E o `i < u` é um bug **real e comum**: `-1 < 10u` é **falso**, porque `-1` é convertido para um `unsigned` enorme (veremos em *Conversões de Tipo*).

## O conjunto de flags recomendado

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -Wshadow -Wconversion -g -O0 programa.c -o programa
```

| Flag | O que pega |
|---|---|
| `-Wall` | O conjunto "quase tudo" de avisos úteis (variáveis não usadas/não inicializadas, formatos de `printf`...) |
| `-Wextra` | Avisos adicionais (comparações com/sem sinal, parâmetros não usados) |
| `-Wpedantic` | Construções que **não seguem** o padrão ISO |
| `-Wshadow` | Variável que **esconde** outra de escopo externo |
| `-Wconversion` | Conversões implícitas que podem **perder valor** |
| `-Werror` | **Transforma avisos em erros** (para CI e projetos maduros) |
| `-std=c17` | Escolhe o **padrão** (veja [[O Compilador]]) |

> ⚠️ **Regra de ouro:** o código **só** é considerado pronto quando compila **sem nenhum aviso** com essas flags. Silenciar um aviso sem entender é **esconder um bug**.

---

# 💬 Comentários e documentação

```c
// Comentário de linha (C99 em diante)
/* Comentário de bloco: pode ocupar várias linhas */
```

Boas práticas:

- Comente o **porquê**, não o **quê**. `i++; // incrementa i` é ruído; `// pula o cabeçalho de 4 bytes` explica uma decisão;
- **Não deixe código comentado** no repositório (o Git guarda o histórico). Para desativar temporariamente um trecho, use `#if 0 ... #endif`, que **aninha** corretamente, ao contrário de `/* */`;
- Documente **funções públicas** no `.h`: o que fazem, parâmetros, **quem libera a memória**, o que retornam em caso de erro;
- Estilo **Doxygen** (muito usado) gera documentação automática:

```c
/**
 * @brief Soma dois inteiros.
 * @param a Primeira parcela.
 * @param b Segunda parcela.
 * @return A soma a + b.
 */
int soma(int a, int b);
```

> 💡 **Quem é dono da memória?** Em C, a pergunta mais importante de uma função que devolve um ponteiro é: *quem chama `free`?* **Documente isso.** Como não há GC (veja [[O que é C]]), é a única forma de o chamador saber.

---

# ✅ Boas práticas de escrita de C

Pequenos hábitos que o estilo de C recomenda:

| Prática | Por quê |
|---|---|
| **`int main(void)`**, não `int main()` | Em C **antes do C23**, `()` significa "argumentos **não especificados**"; `(void)` diz "**sem** parâmetros". No C23 as duas formas passam a ser equivalentes |
| **Inicialize toda variável** na declaração | Variáveis locais em C **não** são zeradas: contêm **lixo** da pilha |
| **Prefira `const`/`enum`/`static const` a `#define`** para constantes | Têm **tipo**, **escopo** e aparecem no depurador; macros são só **texto** |
| **Use `size_t`** para tamanhos e índices; **`<stdint.h>`** (`int32_t`, `uint8_t`) quando o tamanho importar | `int` varia entre plataformas (veja [[Como C Funciona]] e [[Bits e Bytes]]) |
| **`sizeof *p`** em vez de `sizeof(tipo)` | `p = malloc(n * sizeof *p);` continua certo se o tipo de `p` mudar |
| **Verifique o retorno** de `malloc`, `fopen`, `scanf`... | Eles **falham** — e C não lança exceção, só devolve `NULL` ou erro |
| **`static`** em funções e globais usadas **só** no arquivo | Esconde-as do resto do programa e **evita colisões** de nomes |
| **`NULL`** para ponteiros nulos (ou `nullptr` no C23) | Deixa a intenção explícita |
| **Parênteses em macros** e em expressões ambíguas | `#define DOBRO(x) ((x) * 2)` evita surpresas de precedência |
| **Funções pequenas**, com **uma responsabilidade** | Mais fáceis de testar e de ler |
| **Evite globais mutáveis** | Dificultam raciocínio e concorrência |

```c
// ❌ Estilo frágil
int main()
{
    int *p = malloc(10 * sizeof(int));
    p[0] = 1;
    ...
}

// ✅ Estilo recomendado
int main(void)
{
    int *p = malloc(10 * sizeof *p);
    if (p == NULL) {
        return 1;           // tratou a falha de alocação
    }
    p[0] = 1;
    ...
    free(p);
    return 0;
}
```

---

# 📁 Organização de arquivos (prévia)

Convenções que serão aprofundadas no módulo de Modularização:

- Um **módulo = um par `nome.h` + `nome.c`**: o `.h` declara a **interface**, o `.c` guarda a **implementação** (veja [[Arquivos de Cabeçalho e o Modelo de Execução]]);
- **Sempre** use **header guards** nos `.h`, para evitar inclusão dupla:

```c
#ifndef LISTA_H
#define LISTA_H

/* declarações ... */

#endif /* LISTA_H */
```

- **Ordem dos `#include`**: o **próprio** cabeçalho primeiro (prova que ele é autossuficiente), depois os do sistema `<...>`, depois os do projeto `"..."`;
- **Não coloque definições** (corpos de função, variáveis) em `.h` — só **declarações**.

---

# 📚 Guias de estilo conhecidos

| Guia | Foco | Característica marcante |
|---|---|---|
| **Linux kernel coding style** | Kernel Linux | Tab de **8 colunas**, K&R, linhas de ~80 colunas, "se precisa de mais de 3 níveis de indentação, refatore" |
| **GNU Coding Standards** | Projetos GNU | Estilo GNU, convenções de portabilidade e documentação |
| **CERT C Coding Standard** | **Segurança** | Regras e recomendações para evitar vulnerabilidades (buffer overflow, UB) |
| **MISRA C** | **Sistemas críticos** (automotivo, aeroespacial) | Subconjunto **restrito** da linguagem; muito usado em firmware (compare com [[CAN]] e [[MOC - Protocolos de Hardware]]) |
| **Barr Group Embedded C** | Firmware | Regras práticas para sistemas embarcados |

Não é preciso decorar nenhum: o essencial é **escolher um estilo**, **automatizar** com `clang-format` e **ligar os avisos**.

---

# 🧾 Estilo adotado neste estudo (proposta)

Para os exemplos deste vault, o padrão proposto é:

| Item | Escolha |
|---|---|
| **Nomes** | `snake_case` para funções/variáveis; `MAIUSCULAS` para macros e constantes; prefixo de módulo |
| **Indentação** | 4 espaços |
| **Chaves** | K&R (`{` da função embaixo; das demais estruturas na mesma linha), **sempre** presentes |
| **Compilação** | `gcc -std=c17 -Wall -Wextra -Wpedantic -g` e, nos exemplos finais, `-Wshadow -Wconversion` |
| **Declarações** | Uma por linha; `*` junto do nome (`char *p`) |
| **Formatação** | `clang-format` com o `.clang-format` acima |

---

# ✅ Resumo do Conceito

- Em C **não há formatador nem proteção embutidos**; convenções funcionam como **prevenção de bugs**, não só estética (exemplo: o `goto fail` sem chaves)
- **Nomes:** `snake_case` para funções e variáveis, `MAIUSCULAS` para macros; **prefixo de módulo** substitui namespaces; evite nomes **reservados** (`_X`, `__x`, sufixo `_t` do POSIX, nomes da libc)
- **Chaves:** K&R/Linux, Allman e GNU são os estilos clássicos; o que importa é **consistência** e **sempre usar chaves**
- **`char *a, *b;`** — o `*` pertence ao **declarador**: em `char* a, b;` só `a` é ponteiro (visto com `sizeof`: 8 × 1). Declare **uma variável por linha**
- Automatize com **`clang-format`** (`.clang-format`) e **`.editorconfig`**
- **Ligue os avisos:** sem flags, o `gcc` achou **2** problemas; com `-Wall -Wextra -Wconversion` achou **7**. Meta: **zero avisos** com `-std=c17 -Wall -Wextra -Wpedantic`
- Comente o **porquê**; documente **quem libera a memória**; use `#if 0` para desativar código
- Hábitos: `int main(void)`, inicializar tudo, `const`/`enum` em vez de `#define`, `size_t` e `<stdint.h>`, `sizeof *p`, **checar retornos**, `static` para o que é interno
- Guias conhecidos: **Linux kernel**, **GNU**, **CERT C** (segurança), **MISRA C** (críticos)

---

# 🔗 Conexões com outras anotações

- [[O Compilador]] — as flags `-Wall -Wextra -std=` e o que cada fase reclama
- [[Fases da Compilação]] — onde nascem erros (pré-processador, compilador, linker) e por que nomes colidem
- [[O que é C]] — a filosofia "confie no programador" que torna o estilo tão importante
- [[Como C Funciona]] — tamanho de tipos dependente de plataforma e comportamento indefinido
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — `#include`, macros e `make` (a visão do Tanenbaum)
- [[A Linguagem C]] — ponteiros e a falta de coleta de lixo
- [[Convenções da linguagem]] — o equivalente em Go (`gofmt`, visibilidade por maiúscula)
- [[Ponteiros]] — o mesmo conceito em Go, sem a armadilha do `char* a, b`
- [[Bits e Bytes]] — por que tipos de tamanho fixo (`stdint.h`) importam
- [[MOC - Protocolos de Hardware]] — firmware de microcontroladores costuma seguir **MISRA C**
