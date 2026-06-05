---
tags:
  - sistemas-operacionais
  - so/linguagem-c
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seções 1.8.2, 1.8.3 e 1.8.4"
---
# Arquivos de cabeçalho e o modelo de Execução

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seções 1.8.2, 1.8.3 e 1.8.4

---

# 📁 1.8.2 — Arquivos de Cabeçalho

Um projeto de sistema operacional geralmente consiste em uma série de diretórios, cada um contendo muitos **arquivos .c** — que por sua vez contêm o código para alguma parte do sistema — junto com alguns **arquivos de cabeçalho .h** que contêm declarações e definições usadas por um ou mais arquivos de código.

> 💡 **O que é um arquivo de cabeçalho (.h)?** Um arquivo de cabeçalho é um arquivo que contém **declarações** — não implementações. Ele declara quais funções existem, quais tipos de dados são usados e quais constantes estão definidas. Qualquer arquivo `.c` que precise usar essas definições inclui o `.h` correspondente. É o mecanismo de C para compartilhar interfaces entre múltiplos arquivos de código.
> 

## Macros — Constantes e Funções em Tempo de Compilação

Arquivos de cabeçalho também podem incluir **macros** simples. Uma macro é uma substituição feita pelo pré-processador C antes da compilação — ela não é uma função real, mas um texto que é substituído no código antes de compilar.

**Macro de constante:**

```c
#define BUFFER_SIZE 4096
```

Isso permite ao programador nomear constantes. Assim, quando `BUFFER_SIZE` é usado no código, ele é substituído pelo número `4096` durante a compilação. Uma boa prática de programação C é nomear todas as constantes (com exceção de 0, 1 e -1 e às vezes até elas).

**Macro com parâmetros:**

```c
#define max(a, b) (a > b ? a : b)
```

Isso permite escrever `i = max(j, k+1)` e obter `i = (j > k+1 ? j : k+1)` — para armazenar o maior entre `j` e `k+1` em `i`.

**Compilação condicional:**

```c
#ifdef X86
    intel_int_ack();
#endif
```

Isso compila uma chamada para a função `intel_int_ack` somente se a macro `X86` estiver definida. A compilação condicional é intensamente usada para isolar códigos dependentes de arquitetura — um determinado código é inserido apenas quando o sistema é compilado para X86, outro código quando compilado para SPARC, e assim por diante.

## A Diretiva #include

Um arquivo `.c` pode incluir zero ou mais arquivos de cabeçalho usando a diretiva `#include`:

```c
#include "defs.h"
#include <stdio.h>
```

Há muitos arquivos de cabeçalho que são comuns a quase todos os `.c` e são armazenados em um diretório central.

---

# 🏗️ 1.8.3 — Grandes Projetos de Programação

Tendo em vista que os sistemas operacionais são muito grandes (cinco milhões de linhas de código não é incomum), ter de recompilar tudo cada vez que um arquivo é alterado seria insuportável. Por outro lado, mudar um arquivo de cabeçalho-chave que esteja incluído em milhares de outros arquivos exige recompilar esses arquivos.

## O Processo de Compilação

Para construir o sistema operacional, cada `.c` é compilado em um **arquivo-objeto** pelo compilador C.

> 💡 **O que é um arquivo-objeto (.o)?** Um arquivo-objeto contém **instruções binárias para a máquina de destino** — o código já traduzido para linguagem de máquina, mas ainda não ligado com outros módulos. Não há nada semelhante ao bytecode Java ou Python.
> 

O processo completo de compilação em C tem os seguintes passos:

> 📌 **Figura 1.30 — O processo de compilação de C e arquivos de cabeçalho para criar um arquivo executável**
> 

```
defs.h    mac.h    main.c    help.c    other.c
  │          │        │         │          │
  └──────────┘        │         │          │
        │             │         │          │
        ▼             ▼         ▼          ▼
  ┌─────────────────────────────────────────┐
  │          Pré-processador C              │
  │  (expande #include, #define, #ifdef...) │
  └──────────────────┬──────────────────────┘
                     │
  ┌──────────────────▼──────────────────────┐
  │            Compilador C                 │
  │  (traduz C para código de máquina)      │
  └───────┬──────────┬──────────────────────┘
          │          │               │
       main.o     help.o          other.o
          │          │               │
          └──────────┴───────┬───────┘
                             │
                  libc.a ────┤
                             │
  ┌──────────────────────────▼──────────────┐
  │             Ligador (linker)            │
  │  (combina todos os .o em um binário)    │
  └──────────────────┬──────────────────────┘
                     │
                   a.out
            (programa binário executável)
```

**Passo 1 — Pré-processador C:** O primeiro passo do compilador C é o **pré-processador C**. Quando lê cada arquivo `.c`, toda vez que ele atinge uma diretiva `#include`, vai e pega o arquivo de cabeçalho nomeado nele e o processa — expandindo macros, lidando com a compilação condicional e passando os resultados ao próximo passo como se estivessem fisicamente incluídos.

**Passo 2 — Compilador C:** Traduz o código C expandido pelo pré-processador para **instruções binárias** do processador alvo. Gera arquivos `.o` para cada arquivo `.c`.

**Passo 3 — Ligador (linker):** Uma vez que todos os arquivos `.o` estejam prontos, eles são passados para o programa chamado de **ligador** (*linker*) para combinar todos eles em um único **arquivo binário executável**. Quaisquer funções de biblioteca chamadas também são incluídas nesse ponto, referências interfuncionais resolvidas e endereços de máquinas relocados conforme a necessidade. Quando o ligador é encerrado, o resultado é um programa executável — tradicionalmente chamado de `a.out` em sistemas UNIX.

## O Papel do make

O acompanhamento de quais arquivos-objeto dependem de quais arquivos de cabeçalho seria completamente impraticável sem ajuda. Por sorte, os computadores são muito bons justamente nesse tipo de coisa.

> 💡 **O que é o make?** Nos sistemas UNIX, há um programa chamado `make` (com inúmeras variantes como `gmake`, `pmake` etc.) capaz de ler o **Makefile**, que informa quais arquivos são dependentes de quais outros arquivos.
> 

O que o `make` faz é ver quais arquivos-objeto são necessários para construir o binário do sistema operacional e, para cada um, conferir se algum dos arquivos dos quais ele depende (o código e os cabeçalhos) foi modificado depois da última vez que o arquivo-objeto foi criado. Se isso ocorreu, esse arquivo-objeto deve ser recompilado. Quando `make` determina quais arquivos `.c` precisam ser recompilados, ele então invoca o compilador C para compilá-los novamente, reduzindo assim o número de compilações ao mínimo possível.

Em grandes projetos, a criação do `Makefile` é propensa a erros — portanto, existem ferramentas que fazem isso automaticamente.

---

# ⚙️ 1.8.4 — O Modelo de Execução

Uma vez que os binários do sistema operacional tenham sido ligados, o computador pode ser reinicializado e o novo sistema operacional iniciado.

## Como o SO é Executado

Ao ser executado, o SO pode carregar dinamicamente partes que não foram estaticamente incluídas no sistema binário — como **drivers** de dispositivo e **sistemas de arquivos**. No tempo de execução, o sistema operacional pode consistir em múltiplos segmentos:

```
Espaço de memória do processo (seção 1.5 revisitada):

Endereço alto ┌──────────────────────────────┐
              │         Pilha                │ ← começa vazia, cresce e
              │      (cresce para baixo)     │   diminui conforme funções
              │                              │   são chamadas e retornadas
              ├──────────────────────────────┤
              │        (lacuna)              │
              ├──────────────────────────────┤
              │         Dados                │ ← começa com determinados
              │      (cresce para cima)      │   valores, pode mudar e
              │                              │   crescer conforme a necessidade
              ├──────────────────────────────┤
              │         Texto                │ ← imutável — não se altera
              │   (código do programa)       │   durante a execução
Endereço zero └──────────────────────────────┘
```

- **Segmento de texto** — o código do programa. É em geral **imutável**, não se alterando durante a execução. Muitas vezes é colocado próximo à parte inferior da memória
- **Segmento de dados** — começa logo acima do texto, com a capacidade de crescer para cima. Inicializado com determinados valores, mas **pode mudar e crescer** conforme a necessidade
- **Pilha** — começa em um endereço virtual alto, com a capacidade de crescer para baixo. **Começa vazia**, mas cresce e diminui conforme as funções são chamadas e retornadas

> ⚠️ Sistemas diferentes funcionam diferentemente em relação ao layout de memória — o diagrama acima representa um modelo comum, mas não universal.
> 

## Execução Direta no Hardware

Em todos os casos, **o código do sistema operacional é diretamente executado pelo hardware**, sem interpretadores ou compilação *just-in-time* como é normal com Java. Isso é fundamental para a performance — cada instrução do kernel vai direto para a CPU sem camada intermediária.

```
Java:
Código Java → bytecode → JVM interpreta → CPU executa
(duas camadas intermediárias)

Sistema Operacional em C:
Código C → compilador → binário → CPU executa diretamente
(zero camadas intermediárias em tempo de execução)
```

Essa execução direta é uma das razões pelas quais SOs em C são muito mais rápidos do que seriam se escritos em linguagens interpretadas — e por que a ausência de GC e a latência previsível são tão importantes no contexto de sistemas operacionais.

---

# ✅ Resumo do Conceito

- **Arquivos de cabeçalho (.h)** — contêm declarações, constantes e macros compartilhadas entre múltiplos arquivos `.c`. São incluídos com a diretiva `#include`
- **Macros** — substituições feitas pelo pré-processador antes da compilação. Podem ser constantes (`#define BUFFER_SIZE 4096`), funções em linha (`#define max(a,b)...`) ou compilação condicional (`#ifdef X86`)
- **O processo de compilação** tem 3 passos: **pré-processador** (expande includes e macros) → **compilador C** (gera arquivos `.o`) → **ligador** (combina tudo em um executável `a.out`)
- **make** — ferramenta que lê o `Makefile` e recompila apenas os arquivos que foram modificados desde a última compilação, minimizando o trabalho em projetos grandes
- **Modelo de execução** — o SO consiste em segmentos de texto (imutável), dados (cresce para cima) e pilha (cresce para baixo). Todo o código é **executado diretamente pelo hardware** — sem interpretador, sem JIT — o que garante latência mínima e performance máxima