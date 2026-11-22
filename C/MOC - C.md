---
tags:
  - c
  - moc
---

# C — Mapa de Conteúdo

> **Versão de referência:** C23 (ISO/IEC 9899:2024). Diferenças para C99/C11 serão destacadas quando importarem.
> **Status:** em andamento 🔄 — módulos 01 (Introdução) e 02 (Fundamentos) completos ✅ · módulo 03 (Arrays, Strings e Funções) completo ✅ · módulo 04 (Ponteiros e Memória) completo ✅

---

## 01 · Introdução

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 1 | [[O que é C]] | Origem (Unix, Ritchie), filosofia, onde C brilha e onde não | ✅ |
| 2 | [[Como C Funciona]] | Do `.c` ao processo: libc, ELF, `_start`, regiões de memória, máquina abstrata | ✅ |
| 3 | [[O Compilador]] | Driver, toolchain, front/middle/back-end, otimização, flags, erros | ✅ |
| 4 | [[Fases da Compilação]] | Pré-processador → compilador → montador → linker, com saídas reais | ✅ |
| 5 | [[Convenções e Estilo]] | Nomes, formatação, `clang-format`, avisos (`-Wall -Wextra`), boas práticas, guias de estilo | ✅ |

## 02 · Fundamentos

| #   | Nota                              | Em uma linha                                                                           | Status |
| --- | --------------------------------- | -------------------------------------------------------------------------------------- | ------ |
| 6   | [[Sintaxe, Blocos e Escopo em C]] | Tokens, `;`, blocos, escopo, tempo de vida e ponteiro pendurado                        | ✅      |
| 7   | [[Variáveis e Constantes em C]]   | Declaração × definição, valor inicial, `const`, `#define`, `enum`, literais            | ✅      |
| 8   | [[Tipos de Dados em C]]           | Inteiros, `char`, floats, `bool`, `sizeof`, `<stdint.h>`, overflow                     | ✅      |
| 9   | [[Operadores em C]]               | Aritméticos, lógicos, bit a bit e máscaras, precedência, ordem de avaliação            | ✅      |
| 10  | [[Controle de Fluxo em C]]        | `if`, `switch` (*fall-through*), laços, `break`/`continue`, `goto` e salto no assembly | ✅      |
| 11  | [[Conversões de Tipo em C]]       | Promoções inteiras, conversões usuais, `signed` × `unsigned`, *casts*, `-Wconversion`  | ✅      |

## 03 · Arrays, Strings e Funções

| #   | Nota                       | Em uma linha                                               | Status |
| --- | -------------------------- | ---------------------------------------------------------- | ------ |
| 12  | [[Arrays em C]]            | Declaração, inicialização, memória contígua, limites, VLA | ✅      |
| 13  | [[Matrizes em C]]          | Arrays de arrays, *row-major*, `i×col+j`, percurso e *cache*, matriz em funções | ✅      |
| 14  | [[Strings em C]]           | Terminador `\0`, literais, `<string.h>`, `snprintf`, UTF-8, `strtol`        | ✅      |
| 15  | [[Funções em C]]           | Protótipos, `()` × `(void)`, por valor, retorno, pilha de chamadas, `static`, `inline` | ✅      |
| 16  | [[Recursão em C]]          | Caso base, pilha de chamadas, estouro de pilha, custo, recursão de cauda | ✅      |
| 17  | [[Funções Variádicas em C]] | `<stdarg.h>`, promoções de argumento, como `printf` funciona, `format` | ✅      |

## 04 · Ponteiros e Memória

| #   | Nota                       | Em uma linha                                               | Status |
| --- | -------------------------- | ---------------------------------------------------------- | ------ |
| 18  | [[Ponteiros em C]]         | Endereços, `&` e `*`, `NULL`, `const`, `void *`, ponteiros para ponteiros | ✅      |
| 19  | [[Aritmética de Ponteiros em C]] | `p + n`, `p2 - p1`, "um além do fim", percursos com ponteiros      | ✅      |
| 20  | [[Arrays e Ponteiros em C]] | *Decay*, `v[i]` ≡ `*(v+i)`, arrays em parâmetros, `char s[]` × `char *s` | ✅      |
| 21  | [[Funções e Ponteiros em C]] | Parâmetros por ponteiro, retorno de ponteiros, ponteiros para função, *callbacks*, `qsort` | ✅      |
| 22  | [[Alocação Dinâmica em C]] | `malloc`, `calloc`, `realloc`, `free`, propriedade do bloco | ✅      |
| 23  | [[Stack vs Heap em C]]     | Onde cada coisa vive, tempo de vida, fragmentação          | ✅      |
| 24  | [[Erros Clássicos de Memória em C]] | *Leak*, *use-after-free*, *double free*, *buffer overflow*, ASan/Valgrind | ✅      |

## 05 · Tipos Compostos

| #   | Nota                       | Em uma linha                                               | Status |
| --- | -------------------------- | ---------------------------------------------------------- | ------ |
| 25  | Structs                    | Campos, inicialização, passagem por valor × ponteiro      | ⏳      |
| 26  | Alinhamento e Padding      | Por que `sizeof(struct)` é maior que a soma dos campos    | ⏳      |
| 27  | Unions                     | Mesma memória, vários tipos; *type punning*                | ⏳      |
| 28  | Enums e `typedef`          | Constantes nomeadas, apelidos de tipos                     | ⏳      |
| 29  | Campos de Bits e Máscaras  | *Bitfields*, operadores bit a bit, registradores de hardware | ⏳      |

## 06 · Pré-processador e Modularização

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 30 | O Pré-processador | `#include`, `#define`, macros, `#ifdef` | ⏳ |
| 31 | Arquivos de Cabeçalho | `.h` × `.c`, *header guards*, declaração × definição | ⏳ |
| 32 | `static`, `extern` e Linkagem | Visibilidade entre arquivos, tempo de vida | ⏳ |
| 33 | Bibliotecas Estáticas e Dinâmicas | `.a`, `.so`, linkagem, `-l` e `-L` | ⏳ |
| 34 | Make e Build | `Makefile`, dependências, compilação incremental | ⏳ |

## 07 · Biblioteca Padrão e IO

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 35 | `stdio`: Entrada e Saída | `printf`, `scanf`, buffers, `fgets` | ⏳ |
| 36 | Arquivos | `fopen`, `fread`, `fwrite`, modos, `FILE *` | ⏳ |
| 37 | `stdlib` e Utilitários | `qsort`, `atoi`/`strtol`, `exit`, `rand` | ⏳ |
| 38 | Tratamento de Erros | `errno`, códigos de retorno, `perror` | ⏳ |

## 08 · C e o Sistema Operacional

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 39 | Chamadas de Sistema em C | `open`, `read`, `write`, a ponte entre `libc` e kernel | ⏳ |
| 40 | Processos | `fork`, `exec`, `wait` | ⏳ |
| 41 | Threads (pthreads) | `pthread_create`, mutexes, variáveis de condição | ⏳ |
| 42 | Sinais | `signal`, `sigaction`, tratadores | ⏳ |
| 43 | Pipes e Sockets | Comunicação entre processos e pela rede | ⏳ |

## 09 · C por Dentro

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 44 | Layout de Memória de um Programa | `text`, `data`, `bss`, *heap*, *stack* | ⏳ |
| 45 | Comportamento Indefinido | UB, por que importa, como o compilador o explora | ⏳ |
| 46 | Convenção de Chamada e ABI | Como argumentos e retorno passam, *stack frame* | ⏳ |
| 47 | Do C ao Assembly | Ler a saída do compilador, otimizações (`-O2`) | ⏳ |
| 48 | Endianness e Representação | *Little/big endian*, complemento de dois, ponto flutuante | ⏳ |

## 10 · Ferramentas e Depuração

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 49 | GDB | Breakpoints, inspeção de memória, *core dumps* | ⏳ |
| 50 | Valgrind e Sanitizers | `-fsanitize=address,undefined`, achar bugs de memória | ⏳ |
| 51 | Warnings e Análise Estática | `-Wall -Wextra`, `clang-tidy`, `cppcheck` | ⏳ |
| 52 | Testes em C | Testes simples, *asserts*, frameworks leves | ⏳ |

---

## Legenda

⏳ planejado · 🔄 em andamento · ✅ escrito

---

## Conexões com outros assuntos do vault

**Sistemas Operacionais** (C é a linguagem de sistemas por excelência)
Por que SOs são escritos em C, ponteiros → [[A Linguagem C]]
Macros, `#include`, `make`, linker e segmentos → [[Arquivos de Cabeçalho e o Modelo de Execução]]
`fork`, `execve`, `waitpid` → [[Syscalls para Gerenciamento de Processos]], [[System Calls]]
pthreads → [[Threads POSIX]]
Espaço de endereçamento, pilha, heap → [[Espaços de Endereçamento]], [[Memória Virtual]], [[Paginação]]
Processos e sua memória → [[Processos]]
Bits, bytes, endianness → [[Bits e Bytes]]

**Go** (comparação direta com C)
Ponteiros e a diferença para C (sem aritmética, com GC) → [[Ponteiros]]
Chamar código C a partir de Go → [[unsafe e cgo]]
Layout de structs, alinhamento e padding → [[Structs]]

**Hardware e embarcados** (C é a linguagem dos microcontroladores)
Protocolos que você programa em C → [[MOC - Protocolos de Hardware]], [[UART]], [[I2C]], [[SPI]]
Registradores e máscaras de bits → [[Dispositivos de IO]]

**Redes**
Sockets em C → [[Socket]]
