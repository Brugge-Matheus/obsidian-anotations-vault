---
tags:
  - c
  - moc
---

# C — Mapa de Conteúdo

> **Fontes:** a definir conforme os materiais que forem enviados (sugestões: *The C Programming Language* — Kernighan & Ritchie, o padrão ISO C, [cppreference](https://en.cppreference.com/w/c) e o manual do GCC/Clang).
> **Versão de referência:** C23 (ISO/IEC 9899:2024). Diferenças para C99/C11 serão destacadas quando importarem.
> **Status:** estrutura criada 🔄 — nenhuma nota escrita ainda

> **Como este MOC funciona:** cada módulo abaixo tem uma **pasta própria** com o mesmo nome. Os tópicos aparecem como **planejados** (sem link) e viram `[[link]]` assim que a nota é escrita — assim o grafo do Obsidian não enche de nós vazios. A ordem e os tópicos são uma proposta e podem mudar conforme os assuntos que você for pedindo.

---

## 01 · Introdução

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 1 | O que é C | Origem (Unix, Ritchie), filosofia, onde C brilha e onde não | ⏳ |
| 2 | Como C Funciona | Pré-processador → compilador → montador → linker; o executável | ⏳ |
| 3 | Ambiente e Primeiro Programa | `gcc`/`clang`, `main`, compilar e executar, primeiros erros | ⏳ |
| 4 | Convenções e Estilo | Nomes, formatação, warnings (`-Wall -Wextra`), padrões da linguagem | ⏳ |

## 02 · Fundamentos

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 5 | Sintaxe, Blocos e Escopo | Tokens, `;`, blocos, escopo e tempo de vida | ⏳ |
| 6 | Variáveis e Constantes | Declaração × definição, `const`, `#define`, `enum` | ⏳ |
| 7 | Tipos de Dados | Inteiros, `char`, floats, `sizeof`, `<stdint.h>`, sinal | ⏳ |
| 8 | Operadores | Aritméticos, relacionais, lógicos, bit a bit, precedência | ⏳ |
| 9 | Controle de Fluxo | `if`, `switch`, `for`, `while`, `break`/`continue`, `goto` | ⏳ |
| 10 | Conversões de Tipo | Promoções inteiras, conversões implícitas, *casts* | ⏳ |

## 03 · Ponteiros e Memória

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 11 | Ponteiros | Endereços, `&` e `*`, `NULL`, ponteiros para ponteiros | ⏳ |
| 12 | Aritmética de Ponteiros | Somar/subtrair, relação com o tamanho do tipo | ⏳ |
| 13 | Arrays e Ponteiros | *Decay*, indexação, arrays multidimensionais | ⏳ |
| 14 | Strings em C | `char[]`, terminador `\0`, `<string.h>`, armadilhas | ⏳ |
| 15 | Alocação Dinâmica | `malloc`, `calloc`, `realloc`, `free` | ⏳ |
| 16 | Stack vs Heap | Onde cada coisa vive, tempo de vida, fragmentação | ⏳ |
| 17 | Erros Clássicos de Memória | *Leak*, *use-after-free*, *double free*, *buffer overflow* | ⏳ |

## 04 · Tipos Compostos

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 18 | Structs | Campos, inicialização, passagem por valor × ponteiro | ⏳ |
| 19 | Alinhamento e Padding | Por que `sizeof(struct)` é maior que a soma dos campos | ⏳ |
| 20 | Unions | Mesma memória, vários tipos; *type punning* | ⏳ |
| 21 | Enums e `typedef` | Constantes nomeadas, apelidos de tipos | ⏳ |
| 22 | Campos de Bits e Máscaras | *Bitfields*, operadores bit a bit, registradores de hardware | ⏳ |

## 05 · Funções

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 23 | Funções — Fundamentos | Protótipos, parâmetros por valor, retorno | ⏳ |
| 24 | Ponteiros para Função | *Callbacks*, tabelas de funções | ⏳ |
| 25 | Funções Variádicas | `<stdarg.h>`, como `printf` funciona | ⏳ |
| 26 | Recursão | Pilha de chamadas, custo, recursão de cauda | ⏳ |

## 06 · Pré-processador e Modularização

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 27 | O Pré-processador | `#include`, `#define`, macros, `#ifdef` | ⏳ |
| 28 | Arquivos de Cabeçalho | `.h` × `.c`, *header guards*, declaração × definição | ⏳ |
| 29 | `static`, `extern` e Linkagem | Visibilidade entre arquivos, tempo de vida | ⏳ |
| 30 | Bibliotecas Estáticas e Dinâmicas | `.a`, `.so`, linkagem, `-l` e `-L` | ⏳ |
| 31 | Make e Build | `Makefile`, dependências, compilação incremental | ⏳ |

## 07 · Biblioteca Padrão e IO

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 32 | `stdio`: Entrada e Saída | `printf`, `scanf`, buffers, `fgets` | ⏳ |
| 33 | Arquivos | `fopen`, `fread`, `fwrite`, modos, `FILE *` | ⏳ |
| 34 | `stdlib` e Utilitários | `qsort`, `atoi`/`strtol`, `exit`, `rand` | ⏳ |
| 35 | Tratamento de Erros | `errno`, códigos de retorno, `perror` | ⏳ |

## 08 · C e o Sistema Operacional

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 36 | Chamadas de Sistema em C | `open`, `read`, `write`, a ponte entre `libc` e kernel | ⏳ |
| 37 | Processos | `fork`, `exec`, `wait` | ⏳ |
| 38 | Threads (pthreads) | `pthread_create`, mutexes, variáveis de condição | ⏳ |
| 39 | Sinais | `signal`, `sigaction`, tratadores | ⏳ |
| 40 | Pipes e Sockets | Comunicação entre processos e pela rede | ⏳ |

## 09 · C por Dentro

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 41 | Layout de Memória de um Programa | `text`, `data`, `bss`, *heap*, *stack* | ⏳ |
| 42 | Comportamento Indefinido | UB, por que importa, como o compilador o explora | ⏳ |
| 43 | Convenção de Chamada e ABI | Como argumentos e retorno passam, *stack frame* | ⏳ |
| 44 | Do C ao Assembly | Ler a saída do compilador, otimizações (`-O2`) | ⏳ |
| 45 | Endianness e Representação | *Little/big endian*, complemento de dois, ponto flutuante | ⏳ |

## 10 · Ferramentas e Depuração

| # | Nota | Em uma linha | Status |
| --- | --- | --- | --- |
| 46 | GDB | Breakpoints, inspeção de memória, *core dumps* | ⏳ |
| 47 | Valgrind e Sanitizers | `-fsanitize=address,undefined`, achar bugs de memória | ⏳ |
| 48 | Warnings e Análise Estática | `-Wall -Wextra`, `clang-tidy`, `cppcheck` | ⏳ |
| 49 | Testes em C | Testes simples, *asserts*, frameworks leves | ⏳ |

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
