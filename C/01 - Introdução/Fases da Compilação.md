---
tags:
  - c
  - c/introducao
  - c/compilador
source: "Documentação do GCC/binutils; todas as saídas foram geradas com gcc 13.3 em Linux x86-64 (Ubuntu 24.04)"
---
# Fases da Compilação

📚 **Referência:** Manual do GCC e do binutils (`as`, `ld`, `objdump`, `nm`, `readelf`) | Anterior: [[O Compilador]] | Complementa [[Arquivos de Cabeçalho e o Modelo de Execução]]

---

# 🔭 Visão geral: quatro fases em cadeia

Quando você digita `gcc ola.c -o ola`, parece que um único passo acontece. Na verdade, o `gcc` (o *driver* — veja [[O Compilador]]) executa **quatro fases**, e cada uma gera um **arquivo intermediário** diferente:

```
 ola.c ──► PRÉ-PROCESSAMENTO ──► ola.i ──► COMPILAÇÃO ──► ola.s ──► MONTAGEM ──► ola.o ──► LINKAGEM ──► ola
 (fonte)       (cpp)           (fonte      (cc1)       (assembly)     (as)     (objeto)      (ld)    (executável)
                              expandido)
   gcc -E ─────────────────────►│            │               │            │           │
                          gcc -S ─────────────────────────────►│            │           │
                                                           gcc -c ──────────────────────►│
                                                                                   gcc ola.o -o ola
```

| # | Fase | Ferramenta | Entrada → Saída | Flag para parar aqui |
|---|---|---|---|---|
| 1 | **Pré-processamento** | `cpp` | `.c` → `.i` | `gcc -E` |
| 2 | **Compilação** | `cc1` | `.i` → `.s` (assembly) | `gcc -S` |
| 3 | **Montagem** | `as` | `.s` → `.o` (objeto) | `gcc -c` |
| 4 | **Linkagem** | `ld` | `.o` + bibliotecas → executável | (sem flag) |

> 💡 **`-save-temps`:** `gcc -save-temps ola.c` executa tudo **e mantém** todos os arquivos intermediários (`ola.i`, `ola.s`, `ola.o`). Ótimo para estudar.

Todo o roteiro desta nota usa o **mesmo programa** `ola.c`:

```c
#include <stdio.h>

#define SAUDACAO "Ola"

int soma(int a, int b) {
    return a + b;
}

int main(void) {
    int x = soma(2, 3);
    printf("%s, %d\n", SAUDACAO, x);
    return 0;
}
```

---

# 1️⃣ Pré-processamento — `gcc -E ola.c -o ola.i`

O **pré-processador** é um **tradutor de texto para texto**. Ele **não entende C**: apenas processa as linhas que começam com `#` (as **diretivas**) e gera um novo arquivo de texto.

> 💡 **Pré-processador:** primeira fase, que faz **substituição textual** antes de o compilador "ler" o código. Trata `#include`, `#define`, `#ifdef` e remove comentários.

## O que ele faz

| Diretiva | Ação do pré-processador |
|---|---|
| `#include <stdio.h>` | **Cola** o conteúdo inteiro do arquivo `stdio.h` naquele ponto |
| `#define SAUDACAO "Ola"` | **Substitui** toda ocorrência de `SAUDACAO` pelo texto `"Ola"` |
| `#ifdef` / `#if` / `#endif` | **Inclui ou descarta** trechos conforme condições |
| Comentários `//` e `/* */` | **Removidos** |

## O resultado real

O arquivo `ola.c` tem **13 linhas**; o `ola.i` tem **826 linhas**:

```
$ wc -l ola.i
826 ola.i
```

Quase tudo vem do `#include <stdio.h>`, que trouxe centenas de declarações de tipos e funções. As **últimas linhas** do `ola.i`:

```c
# 5 "ola.c"
int soma(int a, int b) {
    return a + b;
}

int main(void) {
    int x = soma(2, 3);
    printf("%s, %d\n", "Ola", x);     // ← SAUDACAO virou "Ola"
    return 0;
}
```

Repare:
- `SAUDACAO` **desapareceu** e no lugar entrou `"Ola"` — substituição **puramente textual**;
- as linhas que começam com `# 5 "ola.c"` são **marcadores de linha** (*linemarkers*): permitem que o compilador diga "erro na linha 8 de `ola.c`" mesmo depois de tudo ter sido colado.

> ⚠️ **Macros são texto, não código.** Como o pré-processador não entende C, `#define max(a, b) (a > b ? a : b)` pode produzir resultados inesperados com expressões como `max(i++, j)`. Veja a nota futura *O Pré-processador*.

**Por que `#include <stdio.h>` é necessário?** Sem ele, o compilador **não conheceria** `printf` — não saberia seu tipo de retorno nem seus parâmetros. O `.h` só traz a **declaração** (o "protótipo"); o **corpo** de `printf` está na biblioteca, que só entra na fase 4. Isso é exatamente o que você viu sobre *declaração × definição* em [[Arquivos de Cabeçalho e o Modelo de Execução]].

---

# 2️⃣ Compilação — `gcc -S ola.i -o ola.s`

Esta é a fase do **compilador propriamente dito** (`cc1`): ele lê o C já pré-processado (`.i`), executa as análises do **front-end**, as otimizações do **middle-end** e o **back-end** (veja [[O Compilador]]), e gera **assembly**.

> 💡 **Assembly:** representação **textual** do código de máquina — cada linha é (quase) uma instrução da CPU, com nomes legíveis (`mov`, `add`, `call`). Como `x86-64`, `ARM` e `RISC-V` têm instruções diferentes, o assembly **depende da arquitetura**.

## O resultado real (`-O0`, x86-64)

```asm
soma:
    pushq   %rbp               ; ┐ prólogo: salva o "frame" do chamador
    movq    %rsp, %rbp         ; ┘ e cria o frame desta função
    movl    %edi, -4(%rbp)     ; guarda o 1º argumento (a) na pilha
    movl    %esi, -8(%rbp)     ; guarda o 2º argumento (b) na pilha
    movl    -4(%rbp), %edx     ; edx = a
    movl    -8(%rbp), %eax     ; eax = b
    addl    %edx, %eax         ; eax = eax + edx  ← a SOMA; resultado em eax
    popq    %rbp               ; epílogo: restaura o frame
    ret                        ; volta para quem chamou

.LC0:   .string "Ola"          ; literal de string (vai para .rodata)
.LC1:   .string "%s, %d\n"     ; formato do printf

main:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp          ; reserva espaço na pilha (variáveis locais)
    movl    $3, %esi           ; 2º argumento de soma = 3
    movl    $2, %edi           ; 1º argumento de soma = 2
    call    soma               ; chama soma(2, 3)
    movl    %eax, -4(%rbp)     ; x = valor retornado (em eax)
    movl    -4(%rbp), %eax
    movl    %eax, %edx         ; 3º arg do printf = x
    leaq    .LC0(%rip), %rax
    movq    %rax, %rsi         ; 2º arg do printf = "Ola"
    leaq    .LC1(%rip), %rax
    movq    %rax, %rdi         ; 1º arg do printf = formato
    movl    $0, %eax           ; nº de registradores vetoriais usados (varargs)
    call    printf@PLT         ; chama printf
    movl    $0, %eax           ; valor de retorno de main = 0
    leave
    ret
```

## Como ler esse assembly

Vale reparar em alguns padrões, que reaparecem em **todo** programa C:

| Observação | Explicação |
|---|---|
| **Argumentos em registradores** (`%edi`, `%esi`, `%edx`...) | Convenção de chamada **System V AMD64**: os 6 primeiros argumentos inteiros/ponteiros vão em `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` |
| **Retorno em `%eax`** | O valor de retorno da função volta em `eax`/`rax` |
| **`pushq %rbp` / `movq %rsp,%rbp`** | O **prólogo**: monta o **stack frame** da função (a "página" da pilha onde ficam suas variáveis locais) |
| **Variáveis locais em `-4(%rbp)`** | Ficam na **pilha**, em endereços relativos ao `rbp` |
| **`call soma`** | Instrução que empilha o endereço de retorno e **pula** para a função |
| **`.string "Ola"`** | Literais de string ficam em uma seção de **dados somente leitura** |

> 💡 **Stack frame (quadro de pilha):** região da pilha reservada para **uma chamada de função**: guarda argumentos salvos, variáveis locais e o endereço de retorno. Cada `call` cria um; cada `ret` o desfaz. É o mecanismo que viabiliza a **recursão**, e a pilha de chamadas segue a mesma lógica LIFO da estrutura [[Pilha (Stack)]].
> **Convenção de chamada (ABI):** conjunto de regras que define **como** argumentos e retorno trafegam entre funções (quais registradores, o que vai na pilha). Precisa ser a mesma entre o código chamador e o chamado — inclusive entre linguagens, o que torna C a **língua franca** (veja [[O que é C]]).

Neste ponto **ainda não é código de máquina**: é texto. O compilador também ainda **não sabe** onde `printf` fica na memória — apenas anotou `call printf@PLT`.

---

# 3️⃣ Montagem — `gcc -c ola.s -o ola.o`

O **montador** (`as`) converte o assembly em **código de máquina binário**, o que a CPU realmente entende. O produto é um **arquivo objeto** (`.o`).

> 💡 **Arquivo objeto (`.o`):** arquivo binário com o **código de máquina** de **um único arquivo-fonte**, mas **incompleto**: ele ainda tem "buracos" (referências a funções ou variáveis de outros arquivos) a serem preenchidos pelo linker. **Não é executável.**

```
$ file ola.o
ola.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

A palavra **relocatable** diz tudo: o código ainda pode ser **realocado** — seus endereços finais **não foram definidos**.

## A tabela de símbolos

O `ola.o` guarda uma **tabela de símbolos** — nomes de funções e variáveis que ele **define** ou **precisa**. Com o comando `nm`:

```
$ nm ola.o
0000000000000014 T main
                 U printf
0000000000000000 T soma
```

| Letra | Significado |
|---|---|
| **T** | Símbolo **definido** na seção de código (*text*) deste arquivo — `main` e `soma` estão aqui |
| **U** | Símbolo **indefinido** (*undefined*) — este arquivo **usa**, mas **não contém**; vai ter de vir de outro lugar. É o `printf` |

> 💡 **Símbolo:** nome de uma função ou variável global que aparece no arquivo objeto. O **linker** usa esses nomes para **conectar** quem usa e quem define.

Os números à esquerda (`0x14`, `0x0`) são **deslocamentos** dentro do arquivo: `soma` começa em `0x0`, e `main` em `0x14` — ainda **não são endereços finais**.

## As relocações: os "buracos" a preencher

Com `objdump -dr` vemos o código de máquina desmontado **e** os pontos a serem corrigidos:

```
0000000000000014 <main>:
  14:  push   %rbp
  15:  mov    %rsp,%rbp
  18:  sub    $0x10,%rsp
  1c:  mov    $0x3,%esi
  21:  mov    $0x2,%edi
  26:  call   2b <main+0x17>          ← "call" com endereço PROVISÓRIO
           27: R_X86_64_PLT32   soma-0x4     ← relocação: "preencher com o endereço de soma"
  ...
  4c:  call   51 <main+0x3d>
           4d: R_X86_64_PLT32   printf-0x4   ← relocação: "preencher com o endereço de printf"
```

O `call` foi montado com um endereço **provisório** (`2b`, a própria próxima instrução) porque o montador **não sabe** onde `soma` e `printf` ficarão. Ele deixa um **registro de relocação** (`R_X86_64_PLT32 ... printf`) dizendo ao linker: *"nesta posição, coloque o endereço de `printf`"*.

> 💡 **Relocação:** instrução para o linker corrigir um endereço no código quando o endereço **final** for conhecido. É o mecanismo que permite compilar cada `.c` **separadamente** e juntar tudo depois.

É por causa dessa fase que é possível compilar um projeto grande **arquivo por arquivo** e **recompilar apenas o que mudou** — a base do `make` (veja [[Arquivos de Cabeçalho e o Modelo de Execução]]).

---

# 4️⃣ Linkagem — `gcc ola.o -o ola`

O **linker** (`ld`, chamado via `collect2`) junta os arquivos objeto e as bibliotecas e produz o **executável**. Seu trabalho tem quatro partes:

1. **Resolver símbolos** — ligar cada `U` ao `T` que o define (em outro `.o` ou numa biblioteca);
2. **Combinar seções** — juntar os `.text` de todos os objetos num só, os `.data` num só, etc.;
3. **Atribuir endereços finais** e **aplicar as relocações**;
4. **Incluir o código de inicialização** (`_start`, o *crt0*) e ligar à biblioteca.

> 💡 **Linker (ligador):** programa que **une** vários arquivos objeto e bibliotecas em um único executável (ou biblioteca), resolvendo as referências entre eles.

## O resultado real

```
$ gcc ola.o -o ola && ./ola
Ola, 5

$ file ola
ola: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
     interpreter /lib64/ld-linux-x86-64.so.2, ...
```

Agora é **executable** (e não mais *relocatable*) e **dynamically linked**. A tabela de símbolos mudou:

```
$ nm ola | grep -E ' (soma|main)$|printf'
000000000000114d T main
                 U printf@GLIBC_2.2.5       ← continua indefinido... mas agora com versão
0000000000001139 T soma
```

- `soma` e `main` ganharam **endereços finais** (`0x1139`, `0x114d`) — as relocações foram aplicadas;
- `printf` **continua `U`**: ele **não está dentro do executável**. Ele virá da `libc.so` em **tempo de execução**. O `@GLIBC_2.2.5` indica a **versão** da glibc que fornece essa função.

## Linkagem estática × dinâmica

| | **Estática** | **Dinâmica** (o padrão) |
|---|---|---|
| **Onde fica o código de `printf`** | **Copiado para dentro** do executável | Em `libc.so`, **fora** do executável |
| **Tamanho do executável** | Grande | Pequeno |
| **Dependências na execução** | Nenhuma | Precisa da biblioteca instalada |
| **Atualização da biblioteca** | Exige recompilar | Basta atualizar a `.so` |
| **Memória** | Cada programa tem sua cópia | A `.so` é **compartilhada** entre processos |
| **Como pedir** | `gcc -static` | (padrão) |

```
$ ldd ola
    linux-vdso.so.1
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
    /lib64/ld-linux-x86-64.so.2
```

O `ldd` lista as **bibliotecas dinâmicas** de que o executável depende. Veja [[Bibliotecas Compartilhadas]] para a ideia de compartilhar código entre processos e [[Como C Funciona]] para o carregamento em tempo de execução.

## O cabeçalho do executável

```
$ readelf -h ola | grep -E 'Type|Machine|Entry'
  Type:     DYN (Position-Independent Executable file)
  Machine:  Advanced Micro Devices X86-64
  Entry point address:  0x1050
```

- **Machine** diz para qual CPU este binário foi feito (x86-64);
- **Entry point** `0x1050` é o `_start` — **não** `main` (`0x114d`);
- **PIE** (*Position-Independent Executable*): o executável pode ser carregado em **qualquer endereço**, útil para a segurança (**ASLR**, que randomiza os endereços a cada execução).

## O tamanho das seções

```
$ size ola
   text    data     bss     dec     hex filename
   1318     600       8    1926     786 ola
```

Aqui você vê as seções `.text` (código), `.data` (globais inicializadas) e `.bss` (globais não inicializadas) descritas em [[Como C Funciona]].

---

# 🔍 Os erros de cada fase

Cada fase tem seus próprios tipos de erro — saber **onde** o erro nasceu é metade da solução:

| Fase | Tipo de erro | Exemplo |
|---|---|---|
| **1. Pré-processamento** | Arquivo não encontrado, macro mal formada | `fatal error: foo.h: No such file or directory` |
| **2. Compilação** | Sintaxe, tipos, declarações | `error: expected expression before ';' token` |
| **3. Montagem** | Raro (o compilador gera assembly válido) | — |
| **4. Linkagem** | Símbolo não definido ou duplicado | `undefined reference to 'soma'` |
| **Execução** | Fora do alcance do compilador | *Segmentation fault*, comportamento indefinido |

Exemplo real de erro na **fase 4** — o código **compilou** corretamente, mas faltou a definição:

```
/usr/bin/ld: l.c:(.text+0x13): undefined reference to `soma'
collect2: error: ld returned 1 exit status
```

---

# 🔎 Espiando o `gcc` trabalhando

Com `gcc -v ola.c -o ola`, o `gcc` revela as ferramentas que ele chama por baixo:

```
/usr/libexec/gcc/x86_64-linux-gnu/13/cc1 ... ola.c ...      ← fases 1 e 2 (cpp + compilador)
as ...                                                      ← fase 3 (montador)
/usr/libexec/gcc/x86_64-linux-gnu/13/collect2 ...           ← fase 4 (linker)
```

Para guardar todos os intermediários e examiná-los:

```bash
gcc -save-temps ola.c -o ola
ls
# ola  ola.c  ola.i  ola.o  ola.s
```

---

# 🧪 Ferramentas de inspeção (para guardar)

| Comando | O que mostra |
|---|---|
| `gcc -E arquivo.c` | Resultado do pré-processamento |
| `gcc -S arquivo.c` | Assembly gerado |
| `gcc -c arquivo.c` | Apenas o objeto |
| `nm arquivo.o` | Tabela de **símbolos** (`T`, `U`, `D`, `B`...) |
| `objdump -d arquivo` | **Desmonta** o código de máquina |
| `objdump -dr arquivo.o` | Desmonta **e** mostra as relocações |
| `readelf -h arquivo` | **Cabeçalho** ELF (tipo, CPU, ponto de entrada) |
| `size arquivo` | Tamanho de `text`, `data` e `bss` |
| `ldd arquivo` | Bibliotecas **dinâmicas** de que depende |
| `file arquivo` | Que **tipo** de arquivo é |

---

# ✅ Resumo do Conceito

- `gcc ola.c -o ola` executa **quatro fases**: **pré-processamento → compilação → montagem → linkagem**, cada uma gerando um arquivo intermediário (`.i`, `.s`, `.o`, executável)
- **Pré-processador**: **substituição de texto** (`#include`, `#define`, `#ifdef`); o `ola.c` de 13 linhas virou um `.i` de **826 linhas**; macros são **texto**, não código
- **Compilador (`cc1`)**: converte C em **assembly** da CPU-alvo, aplicando análises e otimizações; argumentos vão em registradores (`edi`, `esi`...), retorno em `eax`, cada função cria um **stack frame**
- **Montador (`as`)**: gera o **objeto `.o`** (código de máquina **relocável**, ainda **incompleto**); `nm` mostra `T` (definido) e `U` (indefinido); **relocações** marcam os endereços que o linker preencherá
- **Linker (`ld`)**: **resolve símbolos**, combina seções, atribui **endereços finais** e produz o executável; `printf` continua `U` porque vem da **libc dinâmica**
- **Linkagem estática** copia a biblioteca para dentro do executável; **dinâmica** (padrão) a carrega em tempo de execução e a **compartilha**
- O **ponto de entrada** do ELF é `_start`, não `main`
- **Qual ferramenta reclamou** (compilador × linker) indica a causa: `undefined reference` é **erro de linkagem**
- Compilar arquivo por arquivo + linkar no final é o que torna possível **projetos grandes** e **compilação incremental** (`make`)

---

# 🔗 Conexões com outras anotações

- [[O Compilador]] — a arquitetura (front-end, IR, back-end) e as flags usadas aqui
- [[Como C Funciona]] — o que acontece **depois**: carga do ELF, `_start`, `main` e `exit`
- [[O que é C]] — a linguagem; por que o C permite essa compilação separada
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — a visão do Tanenbaum: `#include`, macros, `make`, linker e segmentos
- [[Bibliotecas Compartilhadas]] — o princípio das `.so` e da linkagem dinâmica
- [[Pilha (Stack)]] — a estrutura da pilha; aqui aplicada ao **stack frame** das funções
- [[Memória]] e [[Espaços de Endereçamento]] — onde `.text`, `.data`, heap e stack ficam
- [[Processadores]] — registradores (`rax`, `rdi`...) e a CPU que executa o código gerado
- [[System Calls]] — `printf` termina chamando `write`; a libc faz a ponte com o kernel
- [[Build, Flags e Cross-compilation]] — o equivalente em Go
