---
tags:
  - sistemas-operacionais
  - so/hardware
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.3.1"
---
# Processadores

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.3.1

---

# 🧠 O que é um Processador?

O processador — ou **CPU** (*Central Processing Unit*) — é o "cérebro" do computador. É ele quem executa todas as instruções dos programas. Sem a CPU, nenhum software roda.

O funcionamento básico da CPU segue um ciclo contínuo e simples:

1. **Buscar** (*fetch*) — vai à memória e busca a próxima instrução
2. **Decodificar** (*decode*) — descobre o que essa instrução manda fazer e quais são seus operandos
3. **Executar** (*execute*) — realiza a operação (soma, comparação, acesso à memória, etc.)
4. Volta ao passo 1 e repete até o programa terminar

Esse ciclo é chamado de **ciclo buscar-decodificar-executar** (*fetch-decode-execute cycle*) e é a essência de como qualquer programa é rodado em um computador.

---

# 📋 Conjunto de Instruções

Cada família de CPU tem seu próprio **conjunto de instruções** (*instruction set*) — a "língua" que ela entende. Isso significa que:

- Um processador **x86** (Intel/AMD) não consegue executar programas compilados para **ARM**, e vice-versa
- Programas precisam ser compilados para a arquitetura alvo
- O termo **x86** engloba todos os processadores Intel descendentes do 8088 (286, 386, Pentium, Core i3/i5/i7 e seus clones)

> 💡 **Por que isso importa para o SO?** O sistema operacional é compilado para uma arquitetura específica. Um Linux para x86 não roda nativamente em um chip ARM — é preciso uma versão diferente do SO para cada arquitetura.
> 

---

# 📦 Registradores

Como acessar a memória RAM é muito mais lento do que executar uma instrução, todas as CPUs possuem **registradores** — pequenas memórias internas ultrarrápidas localizadas dentro do próprio chip.

Os registradores armazenam variáveis e resultados temporários durante a execução. O conjunto de instruções geralmente contém instruções para:

- **Carregar** uma palavra da memória para um registrador
- **Armazenar** o conteúdo de um registrador de volta na memória
- **Combinar** dois operandos (de registradores ou memória) para produzir um resultado

Além dos registradores de uso geral, existem **registradores especiais** visíveis para o programador:

| Registrador | Função |
| --- | --- |
| **Contador de Programa** (*Program Counter* — PC) | Contém o endereço de memória da **próxima instrução** a ser buscada. Após cada busca, é automaticamente atualizado para apontar para a instrução seguinte. |
| **Ponteiro de Pilha** (*Stack Pointer* — SP) | Aponta para o **topo da pilha** atual na memória. A pilha armazena, para cada procedimento chamado: parâmetros de entrada, variáveis locais e variáveis temporárias que não cabem em registradores. |
| **PSW** (*Program Status Word*) | Contém **bits de condição** (resultado de comparações), a **prioridade** da CPU, o **modo de execução** (usuário ou núcleo) e outros bits de controle. Tem papel fundamental nas chamadas de sistema e na E/S. |

---

# 🏗️ Arquitetura vs. Microarquitetura

Tanenbaum faz uma distinção importante entre dois conceitos:

**Arquitetura**

Tudo que é **visível para o software** — as instruções disponíveis, os registradores acessíveis, o modelo de memória. É o "contrato" entre o hardware e o programador/SO.

**Microarquitetura**

A **implementação interna** da arquitetura — como os transistores, caches, pipelines e unidades de execução estão organizados para executar as instruções. Inclui elementos que normalmente não deveriam ser visíveis para o SO:

- **Caches** de dados e instruções
- *Translation Lookaside Buffers* (TLBs)
- Preditores de desvio
- Caminho de dados em pipeline

> 💡 **Exemplo:** Dois processadores podem ter a mesma arquitetura x86 (mesmo conjunto de instruções), mas microarquiteturas completamente diferentes — um pode ter 3 estágios de pipeline e outro ter 20. O SO não precisa saber disso; ele vê a mesma arquitetura.
> 

---

# ⚡ Técnicas para Aumentar o Desempenho

## Pipeline

Para melhorar o desempenho, os projetistas de CPU há muito tempo abandonaram o modelo simples de buscar, decodificar e executar uma instrução de cada vez. CPUs modernas usam o conceito de **pipeline**.

O pipeline é uma estrutura onde o processo de execução de uma instrução é dividido em etapas encadeadas — o **output de uma etapa vira o input da próxima**. Com isso, enquanto a instrução N está sendo executada, a instrução N+1 já está sendo decodificada e a N+2 já está sendo buscada:

```
Instrução N:    [Busca] → [Decodifica] → [Executa]
Instrução N+1:            [Busca]      → [Decodifica] → [Executa]
Instrução N+2:                           [Busca]      → [Decodifica] → [Executa]
```

> 📌 **Figura 1.7(a) — Pipeline com três estágios**
> 

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Unid.   │────▶│    Unid.     │────▶│  Unid.   │
│  busca   │     │ decodificação│     │ execução │
└──────────┘     └──────────────┘     └──────────┘

  Estágio 1         Estágio 2           Estágio 3
  (busca inst.)   (descobre o que     (realiza a
                   fazer e operandos)  operação)
```

> 💡 **Pipeline ≠ Paralelismo puro.** O paralelismo no pipeline é uma *consequência do encadeamento* — cada etapa processa uma instrução diferente ao mesmo tempo porque a instrução anterior já terminou aquela etapa. Isso é diferente de paralelismo puro (como na CPU superescalar), onde unidades completamente independentes executam instruções sem relação de dependência entre si.
> 

**Problema dos pipelines longos:** desvios condicionais (*if*, *while*) causam **bolhas no pipeline** — a CPU pode ter buscado e decodificado instruções erradas antes de saber o resultado do desvio. Isso gera grande complexidade para compiladores e projetistas de SO.

## CPU Superescalar

Ainda mais avançada que o pipeline, uma **CPU superescalar** possui múltiplas unidades de execução independentes trabalhando em paralelo:

- Uma unidade para aritmética de inteiros
- Uma unidade para aritmética de ponto flutuante
- Uma unidade para operações booleanas

Duas ou mais instruções são buscadas ao mesmo tempo, decodificadas e jogadas em um **buffer de instruções**. Quando uma unidade de execução fica disponível, ela procura no buffer uma instrução compatível e a executa.

> 📌 **Figura 1.7(b) — CPU Superescalar**
> 

```
┌──────────┐     ┌──────────────┐
│  Unid.   │────▶│    Unid.     │────▶┐
│  busca   │     │ decodificação│     │     ┌────────────────┐
└──────────┘     └──────────────┘     │     │                │
                                      ├────▶│  Buffer de     │────▶ Unid. execução 1
┌──────────┐     ┌──────────────┐     │     │  instrução     │────▶ Unid. execução 2
│  Unid.   │────▶│    Unid.     │────▶┘     │                │────▶ Unid. execução 3
│  busca   │     │ decodificação│           └────────────────┘
└──────────┘     └──────────────┘

  2 pipelines buscam e decodificam     Buffer distribui para as
  instruções simultaneamente           unidades disponíveis
```

**Implicação importante:** as instruções são frequentemente **executadas fora de ordem** (*out-of-order execution*). Cabe ao hardware garantir que o resultado final seja equivalente ao de uma execução sequencial — e parte dessa complexidade é empurrada para o sistema operacional.

---

# 🔐 Modos de Execução: Núcleo e Usuário

A maioria das CPUs modernas opera em dois modos distintos, controlados por um bit no **PSW**:

## Modo Núcleo (*Kernel Mode*)

- Usado pelo **sistema operacional**
- A CPU pode executar **todas as instruções** do conjunto de instruções
- Tem acesso a **todo o hardware**
- Em desktops, notebooks e servidores, o SO normalmente opera nesse modo

## Modo Usuário (*User Mode*)

- Usado pelos **programas do usuário**
- Apenas um **subconjunto das instruções** pode ser executado
- Instruções envolvendo E/S e proteção de memória são **inacessíveis**
- Alterar o bit de modo do PSW para modo núcleo também é proibido

```
┌─────────────────────────────────────┐
│  MODO NÚCLEO (Kernel)               │
│  • Acesso total ao hardware         │
│  • Todas as instruções disponíveis  │
│  • Sistema Operacional roda aqui    │
├─────────────────────────────────────┤
│  MODO USUÁRIO (User)                │
│  • Acesso restrito                  │
│  • Subconjunto de instruções        │
│  • Programas do usuário rodam aqui  │
└─────────────────────────────────────┘
```

> ⚠️ Em sistemas embarcados, uma parte pequena opera em modo núcleo, com o restante do sistema operacional em modo usuário.
> 

---

# 📞 Chamadas de Sistema e Traps

Para obter serviços do SO, um programa em modo usuário precisa fazer uma **chamada de sistema** (*system call*). O mecanismo funciona assim:

1. O programa executa uma instrução especial de **captura** (*trap*, ex.: `syscall` nos processadores x86 de 64 bits)
2. Essa instrução **chaveia do modo usuário para o modo núcleo** automaticamente
3. O sistema operacional assume o controle e executa o serviço solicitado
4. Ao terminar, o controle retorna para o programa do usuário na instrução seguinte

Além das chamadas de sistema (traps **intencionais**), existem traps gerados **automaticamente pelo hardware** para situações excepcionais:

| Situação | Exemplo |
| --- | --- |
| Divisão por zero | `int x = 10 / 0;` |
| *Underflow* de ponto flutuante | Número muito pequeno para ser representado |
| Acesso a endereço inválido | *Segmentation fault* |
| Instrução ilegal | Tentativa de executar dado como código |

> 💡 **Distinção importante:** nas chamadas de sistema, o programa age *de propósito* para pedir um serviço. Nas exceções de hardware, o trap é *reativo* — algo deu errado e o SO precisa intervir para decidir o que fazer (encerrar o programa, ignorar o erro ou repassar o controle de volta ao programa).
> 

---

# 🧵 Multithreading e Chips Multinúcleo

## A Lei de Moore e seus Limites

A **Lei de Moore** observa que o número de transistores em um chip dobra a cada 18 meses. Essa tendência se mantém há meio século, mas já mostra sinais de desaceleração — à medida que os transistores se tornam menores, a mecânica quântica começa a impedir reduções ainda maiores. Superar esse limite será um desafio enorme.

Com a abundância de transistores, os projetistas precisam decidir o que fazer com todos eles. Além de aumentar o número de unidades funcionais (superescalar), surgiram duas abordagens fundamentais: **multithreading** e **múltiplos núcleos**.

## Multithreading (Hyperthreading)

O **multithreading** — também chamado de *hyperthreading* pela Intel — permite que uma CPU mantenha o estado de **duas threads diferentes** e alterne entre elas em uma escala de tempo de nanossegundos.

Uma **thread** é um tipo de processo leve — essencialmente um fluxo de execução dentro de um programa.

**Como funciona na prática:** se um processo precisa ler uma palavra da memória (o que leva muitos ciclos de relógio, ciclos de clock), uma CPU com multithreading pode simplesmente chavear para outra thread enquanto espera, aproveitando o tempo ocioso.

> ⚠️ **Multithreading não é paralelismo real.** Apenas um processo de cada vez é executado — o que muda é que o tempo de chaveamento entre threads é reduzido para a ordem de um nanossegundo, tornando o aproveitamento da CPU muito mais eficiente.
> 

**Implicação para o SO:** cada thread aparece para o sistema operacional como uma CPU separada. Um sistema com uma CPU com duas threads é visto pelo SO como se tivesse duas CPUs. Isso pode levar o escalonador a cometer erros — por exemplo, escalonar duas threads para a *mesma* CPU física com multithreading enquanto outra CPU fica ociosa, o que é muito menos eficiente.

## Chips Multinúcleo

Além do multithreading, muitos chips modernos possuem **múltiplos núcleos** — processadores completos e independentes dentro do mesmo chip.

Um chip de quatro núcleos efetivamente traz quatro *minichips*, cada um com sua própria CPU independente. Existem duas organizações comuns de cache nesses chips:

> 📌 **Figura 1.8 — Chip de quatro núcleos**
> 

```
(a) Cache L2 compartilhada              (b) Caches L2 separadas por núcleo

┌───────────────────────────┐           ┌───────────────────────────┐
│  ┌──────────┬──────────┐  │           │  ┌──────────┬──────────┐  │
│  │ Núcleo 1 │ Núcleo 2 │  │           │  │ Núcleo 1 │ Núcleo 2 │  │
│  │  [L1]    │  [L1]    │  │           │  │ [L1][L2] │ [L1][L2] │  │
│  └────┬─────┴─────┬────┘  │           │  └──────────┴──────────┘  │
│       └─────┬─────┘       │           │  ┌──────────┬──────────┐  │
│          [Cache L2]       │           │  │ Núcleo 3 │ Núcleo 4 │  │
│       ┌─────┴─────┐       │           │  │ [L1][L2] │ [L1][L2] │  │
│  ┌────┴─────┬─────┴────┐  │           │  └──────────┴──────────┘  │
│  │ Núcleo 3 │ Núcleo 4 │  │           └───────────────────────────┘
│  │  [L1]    │  [L1]    │  │
│  └──────────┴──────────┘  │           Cada núcleo tem sua própria
└───────────────────────────┘           cache L2 — mais independência,
                                        mas mais difícil manter
Todos os núcleos compartilham          consistência entre elas
a mesma cache L2 — mais simples,
mas exige controlador mais complexo
```

Alguns processadores populares como o **Intel Xeon** e o **AMD Ryzen** já vêm com mais de 50 núcleos, e há CPUs com centenas de núcleos. Fazer uso eficiente de um chip com múltiplos núcleos exige um **sistema operacional de multiprocessador**.

## GPU (*Graphics Processing Unit*)

Em termos de número absoluto de núcleos, nada bate uma **GPU** moderna. Uma GPU é um processador com literalmente **milhares de núcleos minúsculos**, projetados para realizar muitos pequenos cálculos em paralelo — como renderizar polígonos em aplicações gráficas.

|  | CPU | GPU |
| --- | --- | --- |
| **Número de núcleos** | Poucos (4–128) | Milhares |
| **Tipo de tarefa** | Tarefas complexas e sequenciais | Muitas tarefas simples em paralelo |
| **Ponto forte** | Lógica geral, controle de fluxo | Cálculo massivamente paralelo |
| **Uso em SO** | Execução do sistema operacional | Codificação, processamento de rede |

> ⚠️ Embora GPUs possam ser úteis para sistemas operacionais em tarefas específicas (como codificação ou processamento de tráfego de rede), não é provável que grande parte do SO em si seja executada em GPUs — elas são difíceis de programar para tarefas de controle geral e têm dificuldade com execução em série.
> 

---

# ✅ Resumo do Conceito

- A CPU executa programas por meio do ciclo **buscar → decodificar → executar**
- Cada família de CPU tem seu próprio **conjunto de instruções** — programas não são portáveis entre arquiteturas sem recompilação
- **Registradores** são memórias ultrarrápidas dentro da CPU; os mais importantes são o **PC**, o **SP** e o **PSW**
- **Arquitetura** é o que o software vê; **microarquitetura** é a implementação interna
- **Pipelines** aumentam o desempenho pelo encadeamento de etapas; CPUs **superescalares** adicionam unidades de execução paralelas e independentes
- A CPU opera em **modo núcleo** (acesso total, SO) ou **modo usuário** (acesso restrito, programas)
- Programas acessam serviços do SO via **chamadas de sistema** (traps intencionais); **exceções de hardware** são traps reativos gerados automaticamente
- **Multithreading** permite chavear entre threads em nanossegundos, aproveitando a CPU durante esperas — mas não é paralelismo real
- **Chips multinúcleo** trazem múltiplos processadores independentes no mesmo chip, exigindo SO de multiprocessador
- **GPUs** têm milhares de núcleos para cálculo paralelo massivo, mas não são adequadas para executar a lógica geral do SO