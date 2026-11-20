---
tags:
  - c
  - c/introducao
  - c/compilador
source: "Documentação do GCC e do Clang/LLVM; experimentos reais com gcc 13.3 em Linux x86-64"
---
# O Compilador

📚 **Referência:** Manual do GCC, documentação do Clang/LLVM e experimentos reais (gcc 13.3, Ubuntu, x86-64) | Anterior: [[Como C Funciona]] | Próxima: [[Fases da Compilação]]

---

# 🛠️ O que é um compilador?

> 💡 **Compilador:** programa que **traduz** código escrito em uma linguagem (aqui, C) para outra forma — normalmente **código de máquina** da CPU onde o programa vai rodar. Além de traduzir, ele **verifica** o código (sintaxe e tipos) e **otimiza** o resultado.

Um compilador faz, ao mesmo tempo, **três trabalhos**:

| Trabalho | O que faz | Exemplo |
|---|---|---|
| **Traduzir** | Converte C em instruções de máquina | `a + b` → `add %edx, %eax` |
| **Verificar** | Recusa código inválido e avisa sobre suspeitos | `int x = ;` → *error: expected expression* |
| **Otimizar** | Gera código mais rápido/menor sem mudar o resultado | `soma(2, 3)` → constante `5` |

Compare com o que você já viu: em [[Como C Funciona]], o executável roda direto na CPU **porque** o compilador já fez toda a tradução antes. Em Python, essa tradução acontece **durante** a execução, por um interpretador.

---

# 🧰 Compilador, montador, linker e *toolchain*

Quando as pessoas dizem "compilador de C", normalmente falam de um **conjunto de ferramentas**. Convém separar os nomes:

| Ferramenta | Papel | Nome no GCC |
|---|---|---|
| **Pré-processador** | Resolve `#include`, `#define`, `#ifdef` | `cpp` (dentro do `cc1`) |
| **Compilador** (em sentido estrito) | Converte C em **assembly** | `cc1` |
| **Montador** (*assembler*) | Converte assembly em **código de máquina** (arquivo objeto) | `as` |
| **Ligador** (*linker*) | Junta objetos e bibliotecas em **um executável** | `ld` (via `collect2`) |
| **Driver** | O comando que você digita; **chama todos os acima** na ordem certa | `gcc` |

> 💡 **Driver:** o programa `gcc` **não compila nada sozinho** — ele é um "maestro" que dispara as ferramentas reais. Ao rodar `gcc -v ola.c`, a saída lista as chamadas internas: `cc1` (o compilador propriamente dito) e `collect2`/`ld` (o linker).

> 💡 **Toolchain (cadeia de ferramentas):** o conjunto completo — compilador, montador, linker, bibliotecas (libc), depurador e utilitários (`objdump`, `nm`, `readelf`) — necessário para transformar fonte em executável.

Por isso, quando você roda `gcc ola.c -o ola`, está disparando **quatro fases em sequência**, que são o tema da próxima nota: [[Fases da Compilação]].

---

# 🏆 Os principais compiladores de C

| Compilador | Origem | Observações |
|---|---|---|
| **GCC** (*GNU Compiler Collection*) | Projeto GNU (1987) | O padrão no Linux; compila C, C++, Fortran, Go, etc. |
| **Clang** (LLVM) | Projeto LLVM (Apple/comunidade) | Mensagens de erro muito claras; padrão no macOS; base de muitas ferramentas |
| **MSVC** | Microsoft | Compilador do Visual Studio no Windows |
| **TCC** (*Tiny C Compiler*) | Fabrice Bellard | Minúsculo e rapidíssimo para compilar; pouca otimização |
| **ICX / ICC** | Intel | Focado em desempenho em CPUs Intel |
| **Toolchains de microcontroladores** | Fabricantes (ARM, Espressif, AVR) | GCC ou Clang configurados para a CPU-alvo |

Ambos GCC e Clang entendem as mesmas opções básicas (`-O2`, `-Wall`, `-o`), então os exemplos desta pasta valem para os dois. No Linux, `cc` costuma ser um atalho para o compilador padrão do sistema.

> 💡 **Um compilador é um programa também:** GCC e LLVM são, eles mesmos, escritos em C++ (e C). Existe até a ideia de ***bootstrap*** (auto-hospedagem): compilar o compilador usando uma versão anterior dele mesmo.

---

# 🏗️ Por dentro: front-end, middle-end e back-end

Os compiladores modernos seguem uma arquitetura em **três estágios**. Entender isso explica **por que** um mesmo compilador consegue ler várias linguagens e gerar código para várias CPUs.

```
 ┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
 │  FRONT-END  │ ──► │    MIDDLE-END    │ ──► │  BACK-END   │
 │             │     │                  │     │             │
 │ Lê a        │     │ Trabalha na      │     │ Gera o      │
 │ LINGUAGEM   │     │ REPRESENTAÇÃO    │     │ código da   │
 │ (C, C++...) │     │ INTERMEDIÁRIA    │     │ CPU-ALVO    │
 │             │     │ (IR) e OTIMIZA   │     │ (x86, ARM…) │
 └─────────────┘     └──────────────────┘     └─────────────┘
      C ──┐                                   ┌── x86-64
   C++ ───┼──► [ IR ]  ──► otimizações ──►    ┼── ARM
  Fortran ┘                                   └── RISC-V
```

## 1. Front-end — entende o código-fonte

| Etapa | O que faz |
|---|---|
| **Análise léxica** | Quebra o texto em **tokens** (palavras-chave, nomes, números, símbolos). `int x = 5;` → `int`, `x`, `=`, `5`, `;` |
| **Análise sintática** | Verifica se os tokens formam uma estrutura válida e monta a **AST** (árvore sintática abstrata) |
| **Análise semântica** | Verifica **tipos**, se as variáveis foram declaradas, se as funções são chamadas com os argumentos certos |

> 💡 **Token:** a menor unidade com significado no código (uma palavra-chave como `int`, um identificador como `x`, um número, um operador).
> **AST (*Abstract Syntax Tree*):** representação em árvore da estrutura do programa. `a + b * 2` vira uma árvore em que a multiplicação fica "abaixo" da soma, respeitando a precedência de operadores.

```
      a + b * 2                 AST:
                                  (+)
                                 /   \
                                a    (*)
                                    /   \
                                   b     2
```

Os **erros de sintaxe e de tipos** que você vê (como `expected expression before ';'`) vêm daqui.

## 2. Middle-end — a representação intermediária e as otimizações

O compilador converte a AST em uma **representação intermediária (IR)** — uma forma simples e independente de CPU:

> 💡 **IR (Intermediate Representation):** linguagem interna do compilador, mais próxima do assembly que do C, mas **sem depender de uma CPU específica**. O GCC usa **GIMPLE**; o Clang usa a **LLVM IR**.

É sobre a IR que a maioria das **otimizações** acontece, pois ela é fácil de analisar e transformar. Isso permite reaproveitar as mesmas otimizações para todas as linguagens e CPUs.

## 3. Back-end — gera o código para a CPU

O back-end transforma a IR em **instruções da CPU-alvo**. Aqui acontecem decisões específicas de arquitetura:

- **Seleção de instruções** (qual `mov`/`add` usar);
- **Alocação de registradores** (quais variáveis ficam em registradores e quais vão para a pilha);
- **Escalonamento de instruções** (ordem que aproveita melhor o *pipeline* — veja [[Processadores]]);
- **Emissão de assembly** para o montador.

> 💡 **Alocação de registradores:** a CPU só tem poucos registradores (16 de uso geral no x86-64). O compilador decide **quais variáveis ficam neles** (rápido) e quais "transbordam" para a pilha (lento). Isso é uma das maiores fontes de ganho de desempenho.

---

# 🎯 O alvo: arquitetura e compilação cruzada

O compilador gera código para **uma CPU específica**. O executável `ola` que fizemos é para **x86-64 Linux**: não roda em ARM, nem no Windows.

> 💡 **Compilação cruzada (*cross-compilation*):** compilar em uma máquina (o *host*, ex.: seu PC x86-64) um programa para **outra** arquitetura (o *target*, ex.: ARM ou ESP32). Exige um compilador configurado para o alvo.

Exemplo concreto do seu dia a dia: ao gravar um programa no **ESP32** pela Arduino IDE, a IDE usa um **GCC cruzado** (para o ESP32 clássico, uma toolchain **Xtensa**; para variantes como o ESP32-C3, **RISC-V**) que roda no seu PC e gera o código de máquina do microcontrolador — que depois é gravado na placa. Veja também [[Build, Flags e Cross-compilation]] em Go.

```
        seu PC (x86-64)                              ESP32 (Xtensa)
   ┌──────────────────────┐   gera código    ┌──────────────────────┐
   │  gcc cruzado         │ ───────────────► │  executável do chip  │
   │  (xtensa-esp32-...)  │  para OUTRA CPU  │  (gravado na flash)  │
   └──────────────────────┘                  └──────────────────────┘
```

---

# 🚀 Otimização: o que o compilador faz por você

O nível de otimização é escolhido com a flag `-O`:

| Flag | Objetivo | Quando usar |
|---|---|---|
| `-O0` | **Nenhuma** otimização; compila rápido, código fácil de depurar | Desenvolvimento e depuração (é o padrão) |
| `-O1` | Otimizações básicas e rápidas | Raramente |
| `-O2` | Boa otimização geral, sem inflar o código | **Produção** (o mais comum) |
| `-O3` | Otimizações agressivas (vetorização, desenrolar laços) | Código numérico intensivo |
| `-Os` | Otimiza **tamanho** do executável | Microcontroladores com pouca flash |
| `-Og` | Otimiza **preservando a depuração** | Depuração com algum ganho |

## Exemplo real: `-O0` × `-O2`

O programa:

```c
int soma(int a, int b) { return a + b; }

int main(void) {
    int x = soma(2, 3);
    printf("%s, %d\n", "Ola", x);
    return 0;
}
```

**Com `-O0`** (sem otimização), o `main` chama `soma` de verdade, guarda o resultado na pilha, move valores de um lado para o outro:

```asm
main:
    pushq  %rbp
    movq   %rsp, %rbp
    subq   $16, %rsp
    movl   $3, %esi          ; 2º argumento de soma
    movl   $2, %edi          ; 1º argumento de soma
    call   soma              ; ← chamada real
    movl   %eax, -4(%rbp)    ; x = resultado (vai para a pilha)
    movl   -4(%rbp), %eax
    movl   %eax, %edx
    leaq   .LC0(%rip), %rax
    ... (mais movimentos) ...
    call   printf@PLT
```

**Com `-O2`**, o compilador faz **três otimizações de uma vez**:

```asm
main:
    subq   $8, %rsp
    movl   $5, %ecx          ; ← 2 + 3 já calculado: a CONSTANTE 5
    movl   $2, %edi
    xorl   %eax, %eax
    leaq   .LC0(%rip), %rdx
    leaq   .LC1(%rip), %rsi
    call   __printf_chk@PLT
    xorl   %eax, %eax
    addq   $8, %rsp
    ret
```

| Otimização aplicada | O que aconteceu |
|---|---|
| **Inlining** | A chamada a `soma` foi "colada" no lugar |
| **Constant folding** (dobramento de constantes) | `2 + 3` foi calculado **durante a compilação** → `5` |
| **Eliminação de armazenamento redundante** | `x` nem existe na pilha: o valor vai direto para um registrador |

> 💡 **Inlining:** substituir a **chamada** de uma função pelo **corpo** dela, eliminando o custo da chamada (empilhar, pular, voltar) e abrindo espaço para mais otimizações.
> **Constant folding:** calcular expressões de valores constantes **na compilação**, em vez de na execução.

O **resultado impresso é idêntico** (`Ola, 5`). É a regra **as-if** vista em [[Como C Funciona]]: o compilador pode fazer o que quiser, desde que o comportamento observável seja o mesmo.

> ⚠️ **Otimização revela bugs:** programas com **comportamento indefinido** (UB) podem funcionar em `-O0` e quebrar em `-O2`, porque o compilador **assume que UB nunca ocorre**. Se seu programa só falha com otimização, **suspeite do seu código**, não do compilador.

---

# 🚩 Opções (flags) essenciais do `gcc`/`clang`

```bash
gcc [opções] arquivos.c -o saida
```

| Opção | Função |
|---|---|
| `-o nome` | Define o **nome do arquivo de saída** |
| `-c` | Só **compila** até o arquivo objeto `.o` (**não** linka) |
| `-S` | Para no **assembly** (`.s`) |
| `-E` | Para após o **pré-processamento** |
| `-Wall -Wextra` | Liga os **avisos** mais importantes (use **sempre**) |
| `-Werror` | Transforma avisos em **erros** |
| `-std=c17` | Escolhe o **padrão** da linguagem (`c99`, `c11`, `c17`; `c2x` no GCC ≤ 13 e `c23` em versões novas) |
| `-pedantic` | Avisa sobre construções que **não** seguem o padrão |
| `-g` | Inclui **informações de depuração** (para o `gdb`) |
| `-O0 … -O3`, `-Os` | Nível de **otimização** |
| `-I pasta` | Onde procurar arquivos de **cabeçalho** (`.h`) |
| `-L pasta`, `-l nome` | Onde procurar **bibliotecas** e qual **biblioteca** ligar (`-lm` = math) |
| `-D NOME=valor` | Define uma **macro** pela linha de comando |
| `-fsanitize=address,undefined` | Insere **verificações** em tempo de execução (acha bugs de memória e UB) |
| `-v` | Mostra os **comandos internos** executados (`cc1`, `collect2`...) |
| `-save-temps` | **Guarda** os arquivos intermediários (`.i`, `.s`, `.o`) |

**Uma boa linha de comando para estudar:**

```bash
gcc -Wall -Wextra -std=c17 -g -O0 programa.c -o programa
```

---

# 🚦 O que o compilador comunica: erros e avisos

O compilador é a sua **primeira linha de defesa**. Aprender a ler as mensagens poupa muito tempo.

## Erro de compilação

Impede a geração do executável:

```c
int main(void) { int x = ; return 0; }
```

```
e.c: In function 'main':
e.c:1:25: error: expected expression before ';' token
    1 | int main(void){ int x = ; return 0; }
      |                         ^
```

A mensagem indica **arquivo : linha : coluna**, o **tipo** (`error`), a descrição e uma seta apontando o ponto exato.

## Erro de linkagem

Acontece **depois** da compilação, quando o linker não encontra algo declarado:

```c
int soma(int, int);                          // só a DECLARAÇÃO, sem corpo
int main(void) { return soma(1, 2); }
```

```
/usr/bin/ld: l.c:(.text+0x13): undefined reference to `soma'
collect2: error: ld returned 1 exit status
```

Repare que o erro vem do **`ld`** (linker), não do compilador: o código **estava correto**, mas faltava a **definição** de `soma`. Saber **qual ferramenta** reclamou ajuda a diagnosticar:

| Mensagem | Quem reclamou | Causa comum |
|---|---|---|
| `error: expected ...`, `error: unknown type name` | Compilador (front-end) | Erro de sintaxe, tipo não declarado |
| `warning: implicit declaration of function` | Compilador | Esqueceu um `#include` ou o protótipo |
| `undefined reference to ...` | **Linker** | Função/variável declarada mas **não definida**, ou biblioteca `-l` faltando |
| `multiple definition of ...` | **Linker** | Mesmo símbolo definido em dois arquivos |

## Avisos (*warnings*)

O código **compila**, mas o compilador suspeita de um erro. Alguns exemplos que `-Wall -Wextra` capturam:

- variável usada **sem ser inicializada**;
- **comparação** entre inteiros com e sem sinal;
- `printf` com **formato errado** para o argumento;
- variável declarada e **nunca usada**.

> ⚠️ **Trate avisos como erros.** Em C, um aviso quase sempre aponta um bug real, porque a linguagem permite muita coisa que "compila, mas está errada". Use sempre `-Wall -Wextra` e corrija **todos** os avisos.

---

# 📏 Padrões da linguagem e extensões

Cada versão do padrão ISO acrescenta recursos. O compilador escolhe um **dialeto**:

| Opção | Significado |
|---|---|
| `-std=c17` | C17 **estrito** (sem extensões do GNU quando combinado com `-pedantic`) |
| `-std=gnu17` | C17 **+ extensões GNU** (o padrão do GCC) |

> 💡 **Extensões do compilador:** recursos que **não fazem parte** do padrão ISO mas o GCC/Clang oferecem (`__attribute__`, *statement expressions*, `typeof` antes do C23...). Úteis, mas **prendem o código** àquele compilador.

---

# ✅ Resumo do Conceito

- **Compilador** = traduz C em código de máquina, **verifica** o código e **otimiza**
- `gcc` é um **driver**: dispara o **pré-processador**, o **compilador** (`cc1`), o **montador** (`as`) e o **linker** (`ld`); o conjunto é a **toolchain**
- Arquitetura: **front-end** (tokens → AST → verificação de tipos), **middle-end** (IR + otimizações) e **back-end** (instruções da CPU, alocação de registradores)
- O código gerado é **específico da CPU-alvo**; **compilação cruzada** gera código para outra arquitetura (ex.: GCC cruzado para o **ESP32**)
- `-O0` = sem otimização (depuração); `-O2` = padrão de produção; `-Os` = menor tamanho
- Otimizações reais vistas: **inlining**, **constant folding** (`2+3` → `5`) e **eliminação de variáveis na pilha**
- **Mensagens importam**: quem reclamou (compilador ou **linker**) indica a causa; `undefined reference` é erro do **linker**
- Use sempre **`-Wall -Wextra`** e leve os avisos a sério
- UB pode se manifestar **só com otimização**; o compilador assume que UB **não acontece**

---

# 🔗 Conexões com outras anotações

- [[O que é C]] — a linguagem que o compilador entende
- [[Como C Funciona]] — onde o compilador entra no ciclo de vida de um programa
- [[Fases da Compilação]] — as quatro fases em detalhe, com saídas reais de cada uma
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — o processo de compilação e o `make` na visão do Tanenbaum
- [[Processadores]] — pipeline, registradores e arquitetura (o que o back-end otimiza)
- [[Build, Flags e Cross-compilation]] — o equivalente em Go (`go build`, `GOOS`, `GOARCH`)
- [[Como Go Funciona]] — o compilador de Go e seu runtime embutido
- [[Bits e Bytes]] — o que é código de máquina na essência: bytes interpretados pela CPU
- [[MOC - Protocolos de Hardware]] — o código dos microcontroladores que usam esses protocolos é compilado com toolchains cruzadas
- [[Convenções e Estilo]] — como usar as flags de aviso (`-Wall -Wextra`) como parte do estilo, com exemplos reais
