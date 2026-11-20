---
tags:
  - c
  - c/introducao
source: "Padrão ISO C (C23), histórico de Dennis Ritchie (Bell Labs) e documentação de GCC/Clang"
---
# O que é C

📚 **Referência:** Padrão ISO/IEC 9899 (C23), *The C Programming Language* (Kernighan & Ritchie) e a história do Unix | Próxima nota: [[Como C Funciona]]

---

# 🧭 O que é C?

**C** é uma linguagem de programação **compilada**, **procedural** e de **propósito geral**, criada para escrever **sistemas** — isto é, programas que ficam bem perto do hardware, como sistemas operacionais, drivers e firmware.

> 💡 **Linguagem compilada:** o código-fonte que você escreve é traduzido **antes** da execução, por um programa chamado **compilador**, em **código de máquina** — instruções que a CPU entende diretamente. Veja [[O Compilador]].
> **Linguagem procedural:** o programa é organizado em **funções** (procedimentos) que operam sobre dados, sem classes nem objetos como em Java.
> **Linguagem de propósito geral:** não foi criada para um nicho (como SQL para bancos de dados); serve para qualquer tipo de programa.

Uma boa forma de resumir C: **ela é uma camada fina sobre o hardware.** Um `int` é (quase sempre) um registrador de 32 bits; um ponteiro é um endereço de memória; uma função é um endereço para onde a CPU pula. Quase nada do que você escreve é escondido de você — o que é ótimo para controle e perigoso para erros.

---

# 📜 De onde veio: uma linha do tempo

```
1960s   BCPL (Martin Richards)       ──►  linguagem sem tipos, usada em sistemas
1969    B  (Ken Thompson, Bell Labs) ──►  versão enxuta do BCPL para o primeiro Unix
1972    C  (Dennis Ritchie)          ──►  B + tipos de dados (int, char, structs...)
1973    Kernel do Unix reescrito em C ──►  primeiro SO grande em linguagem de alto nível
1978    Livro "K&R" (1ª edição)      ──►  define o "C clássico"
1989    ANSI C (C89), depois ISO C90 ──►  primeiro padrão oficial
1999    C99                          ──►  //, long long, stdint.h, VLAs...
2011    C11                          ──►  threads, _Atomic, _Generic
2018    C17                          ──►  correções (sem recursos novos)
2024    C23 (ISO/IEC 9899:2024)      ──►  nullptr, #embed, bool/true/false nativos...
```

**O ponto histórico mais importante:** em 1973 o **Unix foi reescrito em C**. Até então, sistemas operacionais eram escritos em **assembly** — preso a uma arquitetura. Com o sistema em C, bastava **escrever um compilador de C para uma nova CPU** e recompilar o SO: surgia a **portabilidade**. Foi o que espalhou o Unix e o próprio C.

> 💡 **Assembly:** a linguagem de máquina em forma legível (`mov`, `add`, `call`). Cada CPU tem o seu. Código em assembly **não é portável**.

Por isso, quando em [[A Linguagem C]] estudamos por que **sistemas operacionais são escritos em C**, a resposta está aqui: C foi **projetada junto com o Unix**.

---

# 🧠 A filosofia de C

C parte de algumas ideias que explicam **tudo** o que você vai encontrar na linguagem:

| Princípio | O que significa na prática |
|---|---|
| **Confie no programador** | O compilador **não** te impede de fazer algo perigoso (acessar fora de um array, por exemplo) |
| **Você não paga pelo que não usa** | Sem *garbage collector*, sem verificação de limites, sem runtime pesado |
| **Perto da máquina** | Tipos, ponteiros e structs mapeiam diretamente em memória e registradores |
| **Pequena e simples** | A linguagem em si tem poucas palavras-chave (algumas dezenas) e poucas regras; o resto vem da **biblioteca** |
| **Portável** | O mesmo código-fonte compila para várias arquiteturas |

> 💡 **Garbage collector (coletor de lixo):** mecanismo de runtime que **libera memória automaticamente** quando ela não é mais usada (Java, Go, Python). **C não tem:** a memória dinâmica é liberada **por você**, com `free`.

Essas escolhas dão **velocidade e controle**, mas **transferem a responsabilidade** para você. É o contrato de C.

---

# 🏷️ Características técnicas

| Característica | Em C |
|---|---|
| **Paradigma** | Procedural (e estruturado: `if`, `for`, `while`, funções) |
| **Execução** | Compilada para **código de máquina nativo** |
| **Tipagem** | **Estática** (tipos conhecidos na compilação) e **fraca** (conversões implícitas entre tipos são comuns) |
| **Gerência de memória** | **Manual** (`malloc`/`free`) |
| **Ponteiros** | Sim, **explícitos** e com **aritmética** |
| **Orientação a objetos** | Não |
| **Exceções** | Não (erros são **códigos de retorno** e `errno`) |
| **Generics** | Não (há macros e `_Generic`) |
| **Concorrência** | Biblioteca (`pthreads`, `<threads.h>`), não faz parte do núcleo da linguagem |
| **Coletor de lixo** | Não |

> 💡 **Tipagem estática × dinâmica:** em C, cada variável tem um tipo **fixo** declarado (`int x;`) e o compilador o verifica **antes** de rodar. Em Python o tipo está no **valor**, e é checado durante a execução.
> **Tipagem fraca:** C converte tipos **automaticamente** em muitas situações (um `char` vira `int` numa conta, por exemplo). Isso é cômodo, mas é fonte de bugs sutis — veremos em *Conversões de Tipo*.

---

# 🌍 Onde C é usada

C é a **base invisível** da computação. Muita coisa que você usa todo dia é C (ou feita sobre C):

| Área | Exemplos |
|---|---|
| **Sistemas operacionais** | Núcleo do **Linux**, núcleos BSD, grande parte do Windows e do macOS |
| **Embarcados e firmware** | Microcontroladores (Arduino, **ESP32**), automóveis, eletrodomésticos |
| **Servidores e infraestrutura** | **[[O que é o NGINX\|Nginx]]**, Redis, Apache |
| **Bancos de dados** | SQLite, PostgreSQL |
| **Linguagens e interpretadores** | CPython (o `python` padrão), Ruby (MRI), Lua, PHP |
| **Ferramentas** | Git, compiladores (GCC é escrito em C/C++), shells |
| **Bibliotecas fundamentais** | A **libc**, OpenSSL, zlib, SQLite |

## C é a "língua franca" entre linguagens

Quase toda linguagem sabe **chamar C**: Python (`ctypes`), Java (JNI), Rust (FFI), Ruby, e também Go, com o **cgo** (veja [[unsafe e cgo]]). Isso porque a **ABI** (a convenção binária de chamada de funções) do C é um padrão universal em cada plataforma.

> 💡 **FFI (*Foreign Function Interface*):** mecanismo para uma linguagem chamar funções escritas em outra. Quase sempre a "outra" é C.

---

# ⚖️ Pontos fortes e fracos

| ✅ Pontos fortes | ❌ Pontos fracos |
|---|---|
| **Muito rápida**: compila para código nativo, sem camadas extras | **Sem segurança de memória**: acessar fora de limites, usar memória liberada... |
| **Controle total** de memória, layout de dados e hardware | **Memória manual** → vazamentos e *use-after-free* |
| **Pequena e estável**: aprende-se a linguagem inteira | **Comportamento indefinido** (UB): erros que o compilador não avisa |
| **Portável**: existe compilador para quase toda CPU | **Biblioteca padrão limitada** (sem listas, mapas, JSON, rede... prontos) |
| **Executáveis pequenos** e sem dependência de runtime | **Strings** frágeis (`char[]` terminado em `\0`) |
| **Interoperável** com todas as outras linguagens | Código verboso para tarefas de alto nível |

> ⚠️ **A maior fonte de vulnerabilidades de segurança** em software de sistemas é a **corrupção de memória** — algo que a própria linguagem C não impede. É por isso que linguagens como **Rust** e **Go** foram criadas, e que estudar C com seriedade inclui aprender a **detectar** esses erros (veja a futura seção de ferramentas: sanitizers e valgrind).

---

# 🆚 C × outras linguagens

| | **C** | **Go** | **Java** | **Python** |
|---|---|---|---|---|
| **Execução** | Código de máquina | Código de máquina + runtime | Bytecode na JVM | Interpretada |
| **Memória** | Manual | Coletor de lixo | Coletor de lixo | Coletor de lixo/contagem de referências |
| **Ponteiros** | Sim, com aritmética | Sim, **sem** aritmética | Referências | Referências |
| **Velocidade** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **Segurança de memória** | ❌ | ✅ | ✅ | ✅ |
| **Concorrência nativa** | Não (biblioteca) | Sim (goroutines) | Sim (threads) | Limitada (GIL) |

Você já conhece [[Ponteiros]] em Go: em C eles existem com **muito mais poder** (aritmética) e **menos proteção**.

## C × C++

Frequentemente confundidos, mas **são linguagens diferentes**. O C++ nasceu como "C com classes" (Bjarne Stroustrup, anos 1980) e hoje é uma linguagem enorme, com orientação a objetos, templates, exceções e muito mais. **Todo código C válido não é necessariamente C++ válido**, e as duas evoluem separadamente. Aqui estudamos **C puro**.

---

# 🏗️ Como um programa C se parece

O famoso "Olá, mundo" — será dissecado em detalhes em *Ambiente e Primeiro Programa*:

```c
#include <stdio.h>

int main(void) {
    printf("Ola, mundo!\n");
    return 0;
}
```

Já dá para ver as marcas de C:

- `#include <stdio.h>` — trazer declarações da **biblioteca padrão** (veja o pré-processador em [[Fases da Compilação]]);
- `int main(void)` — a **função** onde tudo começa;
- `printf(...)` — função da biblioteca, **não** da linguagem;
- `return 0;` — o **código de saída** devolvido ao sistema operacional (0 = sucesso);
- ponto e vírgula e chaves delimitam instruções e blocos.

---

# ✅ Resumo do Conceito

- **C** é uma linguagem **compilada**, **procedural**, de **baixo nível relativo** (perto do hardware), criada por **Dennis Ritchie** nos Bell Labs por volta de 1972, descendente de **BCPL → B**
- Em **1973 o Unix foi reescrito em C**, criando a ideia de **sistema operacional portável**; por isso C e SOs andam juntos ([[A Linguagem C]])
- Padrões principais: **C89/C90** (primeiro padrão), **C99**, **C11**, **C17** e **C23** (ISO/IEC 9899:2024)
- Filosofia: **confie no programador**, **não pague pelo que não usa**, **fique perto da máquina**
- Tipagem **estática e fraca**, memória **manual**, **ponteiros com aritmética**, **sem** GC, exceções, classes ou generics
- Usada em **SOs, embarcados (como o ESP32), servidores (Nginx), bancos de dados, interpretadores e ferramentas**; é a **língua franca** entre linguagens
- Dá **velocidade e controle**, mas **não protege** contra erros de memória e **comportamento indefinido**
- **C ≠ C++**: duas linguagens distintas

---

# 🔗 Conexões com outras anotações

- [[Como C Funciona]] — o que acontece entre escrever `main.c` e ver o programa rodando
- [[O Compilador]] — a ferramenta central da linguagem
- [[Fases da Compilação]] — pré-processador, compilador, montador e linker, com saídas reais
- [[A Linguagem C]] — por que SOs são escritos em C; introdução a ponteiros (visão do Tanenbaum)
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — macros, `#include`, `make` e linker (visão do Tanenbaum)
- [[Ponteiros]] — o mesmo conceito em Go, sem aritmética
- [[unsafe e cgo]] — como Go chama código C
- [[MOC - Protocolos de Hardware]] — C é a linguagem dos microcontroladores que falam esses protocolos
- [[Processos]] — todo programa C em execução vira um processo
