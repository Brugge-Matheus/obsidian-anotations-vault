---
tags:
  - c
  - c/introducao
source: "Padrão ISO C (C23), documentação do GCC/glibc e experimentos reais com gcc 13.3 em Linux x86-64"
---
# Como C Funciona

📚 **Referência:** Padrão ISO/IEC 9899 (C23), documentação do GCC e da glibc | Anterior: [[O que é C]] | Próxima: [[O Compilador]]

---

# 🗺️ A grande visão: do texto ao processo em execução

Um arquivo `.c` é **só texto**. Para virar um programa rodando, ele passa por uma cadeia de transformações — e termina na CPU executando instruções de máquina:

```
 ola.c ──► [ COMPILADOR ] ──► ola (executável) ──► [ SO carrega ] ──► [ CPU executa ]
```

Três atores, três momentos diferentes:

| Momento | Quem age | O que acontece |
|---|---|---|
| **Tempo de compilação** | O **compilador** (e o *linker*) | Traduz e verifica o código; gera o executável |
| **Tempo de carga** | O **sistema operacional** | Lê o executável, monta o espaço de endereçamento do processo |
| **Tempo de execução** | A **CPU** | Executa as instruções, usando a memória do processo |

> 💡 **Tempo de compilação × tempo de execução:** muita confusão em C vem de misturar os dois. Erros de **sintaxe** e **tipos** aparecem na compilação. Erros como acessar um array fora dos limites **só** aparecem (se aparecerem) na execução — C **não verifica** isso para você.

---

# ⚙️ C não tem máquina virtual nem interpretador

Compare o caminho de um programa em diferentes linguagens:

```
C        .c ──compila──► código de máquina ─────────────────────► CPU
Go       .go ─compila──► código de máquina + runtime embutido ──► CPU
Java     .java ─compila─► bytecode ──► JVM (interpreta/JIT) ────► CPU
Python   .py ──────────► bytecode ──► interpretador CPython ───► CPU
```

| | **C** | **Go** | **Java** | **Python** |
|---|---|---|---|---|
| **Intermediário** | Nenhum | Nenhum | Bytecode + JVM | Bytecode + interpretador |
| **Runtime** | Mínimo (a `libc`) | Embutido (GC, scheduler) | JVM | Interpretador |
| **O executável roda direto na CPU?** | ✅ | ✅ | ❌ (via JVM) | ❌ (via interpretador) |

O resultado: **nada entre o seu código e a CPU**. Por isso C é rápida e por isso é a linguagem usada para **escrever** os interpretadores e máquinas virtuais das outras (veja [[Paravirtualização e a Máquina Virtual Java]]; a JVM e o CPython são, em boa parte, escritos em C/C++).

Em [[Como Go Funciona]] você viu que o runtime de Go é como um "mini SO" dentro do binário (scheduler, GC). Em C, esse mini SO **não existe**: o que sobra é a **biblioteca padrão**.

---

# 📚 O "runtime" de C: a libc

C não tem runtime pesado, mas isso **não significa que não há nada além do seu código**. Ao compilar o programa de exemplo e rodar `ldd`, aparece:

```
$ ldd ola
    linux-vdso.so.1
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
    /lib64/ld-linux-x86-64.so.2
```

| Componente | O que é |
|---|---|
| **`libc.so.6`** | A **biblioteca C padrão** (aqui, a **glibc**): `printf`, `malloc`, `strlen`, `fopen`... |
| **`ld-linux-x86-64.so.2`** | O **carregador dinâmico**: carrega as bibliotecas compartilhadas e resolve os endereços de funções antes do `main` |
| **`linux-vdso.so.1`** | Um pedaço do **núcleo** mapeado no processo para acelerar algumas syscalls (como obter a hora) |

> 💡 **libc (biblioteca C padrão):** conjunto de funções definido pelo padrão ISO C (`stdio.h`, `stdlib.h`, `string.h`, `math.h`...) mais extras do sistema (POSIX). Ela é a **ponte entre o seu programa e o kernel**: `printf` formata o texto e, no fim, chama a *syscall* `write`. Veja [[System Calls]].

> 💡 **Biblioteca compartilhada (dinâmica):** código de biblioteca que **fica fora do executável** (`.so` no Linux, `.dll` no Windows) e é carregado em tempo de execução, sendo **compartilhado** entre vários processos. É a mesma ideia de [[Bibliotecas Compartilhadas]] no gerenciamento de memória.

Ou seja: em C, "o runtime" **é a libc** — e ela é, ela mesma, escrita em C.

---

# 🚀 O que acontece quando você roda `./ola`

Quando você digita o comando no shell ([[Interpretador de Comandos Shell]]), acontece uma sequência bem definida:

```
1. O shell chama  fork()          ──► cria um processo-filho         (veja Criação de Processos)
2. O filho chama  execve("./ola") ──► pede ao kernel para trocar o programa
3. O KERNEL abre o arquivo ELF, lê o cabeçalho e MAPEIA os segmentos na memória
4. Como o executável é dinâmico, o kernel também carrega o CARREGADOR DINÂMICO (ld.so)
5. O ld.so carrega libc.so.6 e resolve os endereços (ex.: printf)
6. O controle vai para o ponto de entrada do programa: _start
7. _start prepara o ambiente e chama  __libc_start_main
8. __libc_start_main chama  main(argc, argv)    ◄── AQUI começa o SEU código
9. main retorna → o valor vira o argumento de exit() → o processo termina
```

Ponto surpreendente: **o programa não começa em `main`**. Ao inspecionar o executável com `readelf -h`, o *entry point* é `0x1050` — não é o endereço de `main` (`0x114d`), mas o do `_start`, um código fornecido pelo sistema de compilação (o chamado **crt0**), que prepara a pilha, os argumentos e as bibliotecas **antes** de chamar `main`.

> 💡 **Ponto de entrada (*entry point*):** endereço da primeira instrução executada pelo processo. Fica no cabeçalho do arquivo ELF. Em C, é `_start`, **não** `main`.

> 💡 **ELF (*Executable and Linkable Format*):** formato dos arquivos executáveis, objetos e bibliotecas no Linux. Guarda o código, os dados, tabelas de símbolos e as instruções para o kernel/ld.so carregarem tudo. No Windows o equivalente é o **PE**; no macOS, o **Mach-O**.

Isso liga diretamente com [[Criação de Processos]] e [[Syscalls para Gerenciamento de Processos]] (`fork` + `execve`) que você já estudou.

## O fim: `return` de `main` e `exit`

```c
int main(void) {
    return 0;       // equivalente a chamar exit(0)
}
```

O valor retornado vira o **código de saída** do processo, que o shell lê (`echo $?`). Por convenção, **0 = sucesso**; qualquer outro valor = erro. Em [[Término de Processos]] você viu como o pai recebe esse valor via `waitpid`.

---

# 🧠 O que o seu programa enxerga: o espaço de endereçamento

Quando o kernel carrega o ELF, ele monta o **espaço de endereçamento** do processo ([[Espaços de Endereçamento]]) em **regiões**:

```
  endereços altos
 ┌──────────────────────┐
 │       STACK          │ ← variáveis locais, chamadas de função (cresce para baixo)
 │          ↓           │
 │                      │
 │          ↑           │
 │        HEAP          │ ← memória dinâmica (malloc) (cresce para cima)
 ├──────────────────────┤
 │   .bss               │ ← globais NÃO inicializadas (zeradas)
 ├──────────────────────┤
 │   .data              │ ← globais INICIALIZADAS
 ├──────────────────────┤
 │   .rodata            │ ← constantes e literais de string ("Ola")
 ├──────────────────────┤
 │   .text              │ ← o CÓDIGO (instruções), somente leitura
 └──────────────────────┘
  endereços baixos
```

O comando `size` mostra o tamanho de cada parte no nosso exemplo:

```
$ size ola
   text    data     bss     dec     hex filename
   1318     600       8    1926     786 ola
```

| Região | O que guarda | Exemplo |
|---|---|---|
| **text** | Instruções de máquina | O código de `main` e `soma` |
| **data** | Variáveis globais com valor inicial | `int contador = 10;` |
| **bss** | Variáveis globais sem valor inicial (o SO zera) | `int total;` |
| **stack** | Variáveis locais e endereços de retorno | `int x = soma(2, 3);` dentro de `main` |
| **heap** | Memória alocada em tempo de execução | `malloc(100)` |

Esse mapa é a base para entender **ponteiros**, **escopo**, **tempo de vida** e **erros de memória**. Cada região será aprofundada na nota *Layout de Memória de um Programa* e em [[Stack vs Heap em C]]. Para a parte de **memória virtual** por trás disso, veja [[Memória Virtual]] e [[Paginação]].

---

# 📐 O que o padrão C promete (e o que não promete)

O padrão ISO C **não descreve uma CPU real** — descreve uma **máquina abstrata**: um computador imaginário e simplificado que define **o que o programa deve fazer**, não **como**.

> 💡 **Máquina abstrata:** modelo teórico usado pelo padrão para especificar o comportamento observável do programa. O compilador tem **liberdade total** de gerar qualquer código de máquina, desde que o **resultado visível** seja o mesmo (regra *as-if*).

> 💡 **Regra *as-if*:** o compilador pode reorganizar, remover ou fundir instruções livremente, **contanto que o comportamento observável** (saídas, acessos a dispositivos, etc.) seja o mesmo que o da máquina abstrata.

Um exemplo concreto, com o mesmo programa compilado em dois níveis de otimização — a função `soma` e a chamada `soma(2, 3)`:

```
-O0 (sem otimização):  chama soma de verdade (call soma)
-O2 (otimizado):       soma DESAPARECE — o compilador calculou 2+3 = 5 em tempo de compilação
                       e passou  movl $5, %ecx  direto para o printf
```

O resultado impresso é o mesmo (`Ola, 5`), mas o **código gerado** é totalmente diferente. Detalhes em [[O Compilador]].

## Comportamento definido, não especificado e indefinido

O padrão classifica o que um programa pode fazer:

| Categoria | Significado | Exemplo |
|---|---|---|
| **Bem definido** | O padrão determina o resultado | `2 + 3` → `5` |
| **Dependente de implementação** | Varia entre compiladores/plataformas, mas **deve ser documentado** | Tamanho de `int`, `long` |
| **Não especificado** | Há várias opções válidas; o compilador não precisa documentar | Ordem de avaliação dos argumentos de uma função |
| **Indefinido (UB)** | **Qualquer coisa** pode acontecer: travar, "funcionar", dar lixo, ser otimizado de modo inesperado | Acessar `arr[10]` num array de 5; overflow de `int` com sinal; usar ponteiro liberado |

> ⚠️ **Comportamento indefinido** é o conceito mais importante para entender C **com seriedade**. O compilador **assume que UB nunca acontece** e otimiza com base nisso — por isso um programa com UB pode funcionar em `-O0` e quebrar em `-O2`. Terá nota própria em *Comportamento Indefinido*.

## O tamanho dos tipos depende da plataforma

Em C, `int` **não** tem tamanho fixo garantido. Neste ambiente (Linux x86-64, gcc), o programa

```c
printf("%zu %zu %zu %zu %zu\n", sizeof(char), sizeof(short),
       sizeof(int), sizeof(long), sizeof(void*));
```

imprimiu:

```
1 2 4 8 8
```

ou seja: `char` = 1 byte, `short` = 2, `int` = 4, `long` = **8** e ponteiro = **8** bytes. Já no **Windows 64 bits**, `long` tem 4 bytes; num microcontrolador de 8 bits, `int` pode ter 2. Para tamanhos exatos, usa-se `<stdint.h>` (`int32_t`, `uint8_t`...). Veja [[Bits e Bytes]] para relembrar por que o tamanho dos tipos define o que dá para guardar e endereçar.

> 💡 **Portabilidade de C:** o **código-fonte** é portável (compila em várias plataformas); o **executável** **não** é — ele é específico de **CPU + sistema operacional** (um binário Linux x86-64 não roda no Windows nem num ESP32).

---

# 🔄 Visão geral do ciclo de vida de um programa C

```
┌───────────────────────── Desenvolvimento ─────────────────────────┐
│                                                                   │
│  Você escreve   ola.c ──► gcc ola.c -o ola ──► ola (executável)   │
│  o código            (pré-processa, compila,                      │
│                       monta e linka)                              │
│                           │                                       │
│        erros de sintaxe ◄─┤ erros de link ("undefined reference") │
└───────────────────────────┼───────────────────────────────────────┘
                            ▼
┌───────────────────────── Execução ────────────────────────────────┐
│  shell: fork ► execve ► kernel carrega o ELF ► ld.so carrega a    │
│  libc ► _start ► main ► ... ► return/exit ► processo termina      │
│                                                                   │
│  erros de execução: segfault, UB, vazamento de memória            │
└───────────────────────────────────────────────────────────────────┘
```

---

# ✅ Resumo do Conceito

- Um programa C passa por **três momentos**: **compilação** (compilador + linker geram o executável), **carga** (o SO monta o processo) e **execução** (a CPU roda o código)
- **Não há interpretador nem máquina virtual**: o executável é código de máquina que roda direto na CPU
- O "runtime" de C é a **libc** (glibc no Linux): `printf`, `malloc`, etc.; ela faz a ponte entre o programa e o **kernel** via syscalls
- Ao executar `./programa`: `fork` → `execve` → kernel carrega o **ELF** → **ld.so** carrega bibliotecas → `_start` → `__libc_start_main` → **`main`** → `exit`
- **O programa não começa em `main`**, e sim no `_start` (ponto de entrada do ELF)
- O processo vive em **regiões de memória**: `.text`, `.rodata`, `.data`, `.bss`, **heap** e **stack**
- O padrão C descreve uma **máquina abstrata**; o compilador pode transformar o código livremente (regra **as-if**), por exemplo calculando `2+3` na compilação
- **Comportamento indefinido (UB)** = qualquer coisa pode acontecer; é o conceito de risco central da linguagem
- O tamanho dos tipos é **dependente da plataforma** (aqui: `int` = 4, `long` = 8, ponteiro = 8 bytes); código-fonte é portável, executável **não**

---

# 🔗 Conexões com outras anotações

- [[O que é C]] — origem, filosofia e usos da linguagem
- [[O Compilador]] — a ferramenta que gera o executável
- [[Fases da Compilação]] — o que acontece dentro do `gcc`, passo a passo, com saídas reais
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — a visão do Tanenbaum do processo de compilação e execução
- [[Criação de Processos]] e [[Syscalls para Gerenciamento de Processos]] — `fork` e `execve`, que lançam o programa
- [[Término de Processos]] — como o código de saída de `main` chega ao shell
- [[Espaços de Endereçamento]], [[Memória Virtual]] e [[Paginação]] — o mapa de memória do processo
- [[System Calls]] — como a libc fala com o kernel
- [[Bibliotecas Compartilhadas]] — o conceito de `.so` e `.dll`
- [[Como Go Funciona]] — compare: Go embute um runtime; C depende da libc
- [[Bits e Bytes]] — tamanho de tipos e endereços
