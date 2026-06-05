---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.6"
---
# Implementação de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.6

---

# ⚙️ 2.1.6 — Implementação de Processos

Para implementar o modelo de processos, o SO mantém uma tabela com uma entrada por processo. Mas como exatamente o SO consegue "congelar" um processo, guardar seu estado, e depois "descongelá-lo" como se nada tivesse acontecido? Essa seção responde essa pergunta do ponto de vista do hardware e do software.

---

## A tabela de processos e o BCP

Para implementar o modelo de processos, o SO mantém uma **tabela de processos** — um arranjo de estruturas, com uma entrada para cada processo.

> 💡 **Tabela de processos:** estrutura de dados central do SO, com uma entrada por processo ativo. Cada entrada armazena tudo que é necessário para salvar e restaurar o estado completo do processo quando ele não está em execução. Alguns autores chamam cada entrada de **bloco de controle de processo (BCP)**.
> 

Cada entrada da tabela contém informações importantes sobre o estado do processo, incluindo seu contador de programa, ponteiro de pilha, alocação de memória, estado dos arquivos abertos, informações sobre cálculos diversos e escalonamento — **tudo o que deve ser salvo quando o processo é trocado da CPU**.

### Figura 2.4 — Campos típicos de uma entrada na tabela de processos

> 📌 **Figura 2.4:** Alguns dos campos de uma entrada típica na tabela de processos.
> 

```
┌──────────────────────┬──────────────────────────────┬───────────────────────────┐
│  Gerenciamento de    │   Gerenciamento de memória   │  Gerenciamento de arquivo │
│      processo        │                              │                           │
├──────────────────────┼──────────────────────────────┼───────────────────────────┤
│ Registradores        │ Ponteiro p/ segmento de texto│ Diretório-raiz            │
│ Contador de programa │ Ponteiro p/ segmento de dados│ Diretório de trabalho     │
│ Palavra de estado    │ Ponteiro p/ segmento de pilha│ Descritores de arquivo    │
│ Ponteiro da pilha    │                              │ ID do usuário             │
│ Estado do processo   │                              │ ID do grupo               │
│ Prioridade           │                              │                           │
│ Parâmetros de escal. │                              │                           │
│ ID do processo (PID) │                              │                           │
│ Processo-pai         │                              │                           │
│ Grupo de processo    │                              │                           │
│ Sinais               │                              │                           │
│ Horário de início    │                              │                           │
│ Tempo de CPU usado   │                              │                           │
│ Tempo de CPU filhos  │                              │                           │
│ Horário próx. alarme │                              │                           │
└──────────────────────┴──────────────────────────────┴───────────────────────────┘
```

Os campos se dividem em três categorias:

- **Gerenciamento de processo** — registradores, PC, prioridade, PID, processo-pai, sinais, tempos de CPU. É tudo que precisa ser salvo e restaurado para que o processo retome exatamente de onde parou.
- **Gerenciamento de memória** — ponteiros para os segmentos de texto (código), dados e pilha do processo na memória.
- **Gerenciamento de arquivo** — diretório-raiz, diretório de trabalho atual, descritores dos arquivos abertos, IDs de usuário e grupo.

---

## A visão em camadas — Figura 2.3

Essa visão da tabela de processos dá origem ao modelo mostrado na Figura 2.3:

> 📌 **Figura 2.3:** O nível mais baixo de um sistema operacional estruturado em processos controla interrupções e escalonamento. Acima desse nível estão processos sequenciais.
> 

```
┌─────────────────────────────────────────────────────┐
│                     Processos                       │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐ │
│  │  0   │  1   │  2   │ ...  │ n-2  │ n-1  │      │ │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘ │
├─────────────────────────────────────────────────────┤
│                    Escalonador                      │
│         (controla interrupções + escalonamento)     │
└─────────────────────────────────────────────────────┘
```

O nível mais baixo do SO é o **escalonador**, com uma variedade de processos acima dele. Todo o tratamento de interrupções e os detalhes sobre início e parada de processos estão **ocultos dentro do escalonador** — que, na verdade, não tem muito código. O restante do SO é estruturado em forma de processo.

> ⚠️ Poucos sistemas reais são tão bem estruturados quanto esse modelo. Mas ele serve como guia conceitual importante para entender a separação de responsabilidades dentro do SO.
> 

---

## O mecanismo de interrupção — passo a passo

Essa é a parte mais técnica e importante da seção. Vamos construir o entendimento em camadas.

### O problema: como a CPU "lembra" o que estava fazendo?

Imagine que o processo do usuário 3 está sendo executado e, de repente, o disco termina de ler um dado. O hardware gera uma **interrupção**. A CPU precisa:

1. Parar o que está fazendo com o processo 3
2. Atender a interrupção do disco
3. Depois, retomar o processo 3 **exatamente de onde parou**

Para isso funcionar, existe um mecanismo preciso envolvendo hardware e software trabalhando juntos.

### Peças fundamentais antes de ver o fluxo

> 💡 **Vetor de interrupção:** tabela armazenada em um ponto fixo próximo à parte inferior da memória. Contém um endereço por tipo de dispositivo/interrupção — cada entrada aponta para a **ISR** correspondente àquela interrupção.
> 

> 💡 **ISR (*Interrupt Service Routine* — rotina de serviço de interrupção):** procedimento em C que executa o trabalho específico para um tipo de interrupção (ex: lê o dado do disco, coloca em um buffer, avisa o processo que aguardava).
> 

> 💡 **Palavra de estado do programa (PSW):** registrador especial da CPU que contém flags de status, modo de execução (núcleo/usuário), e outras informações de controle. É salva junto com o PC durante uma interrupção.
> 

### Figura 2.5 — Os 8 passos do tratamento de uma interrupção

> 📌 **Figura 2.5:** O esboço do que o nível mais baixo do sistema operacional faz quando ocorre uma interrupção. Os detalhes podem variar conforme o sistema operacional.
> 

```
CONTEXTO: Processo do usuário 3 está em execução. Disco termina leitura → interrupção.

HARDWARE faz (automático, sem código de SO):
┌─────────────────────────────────────────────────────────────────┐
│ Passo 1: Hardware empilha o contador de programa (PC),          │
│          a palavra de estado (PSW) e, às vezes, registradores   │
│          na pilha ATUAL do processo interrompido                │
│                                                                 │
│ Passo 2: Hardware carrega o novo PC a partir do vetor de        │
│          interrupções — aponta para o início da ISR do disco    │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
ASSEMBLY faz (rotina em linguagem de montagem — não pode ser C):
┌─────────────────────────────────────────────────────────────────┐
│ Passo 3: Rotina em assembly SALVA os registradores              │
│          (geralmente na entrada da tabela de processos          │
│           do processo atual — o processo 3)                     │
│                                                                 │
│ Passo 4: Rotina em assembly configura uma nova pilha            │
│          temporária usada pelo tratador de processos            │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
SOFTWARE em C faz (o trabalho real da interrupção):
┌─────────────────────────────────────────────────────────────────┐
│ Passo 5: Procedimento em C é executado — lê a entrada do disco, │
│          armazena no buffer, marca o processo que aguardava     │
│          como PRONTO                                            │
│                                                                 │
│ Passo 6: Escalonador decide qual é o próximo processo a         │
│          executar (pode ser o processo 3, pode ser outro)       │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
RETORNO (assembly novamente):
┌─────────────────────────────────────────────────────────────────┐
│ Passo 7: Procedimento em C retorna para o código em assembly    │
│                                                                 │
│ Passo 8: Rotina em assembly carrega registradores e mapa de     │
│          memória do processo escolhido e inicia sua execução    │
└─────────────────────────────────────────────────────────────────┘
```

### Por que os passos 3, 4, 7 e 8 precisam ser em assembly e não em C?

Porque ações como **salvar todos os registradores** e **trocar a pilha** não podem ser expressas em linguagens de alto nível como C. C não dá acesso direto ao conjunto completo de registradores da CPU nem permite trocar a pilha de execução no meio do caminho. Por isso, há sempre uma pequena **rotina em linguagem de montagem** — normalmente a mesma para todas as interrupções, já que o trabalho de salvar os registradores é idêntico independente da causa da interrupção.

### Por que a informação empurrada pelo hardware para a pilha precisa ser removida?

No passo 3, a rotina em assembly **remove** da pilha a informação que o hardware empilhou (PC, PSW) e a salva na tabela de processos. Isso é necessário porque a pilha usada pelo hardware no momento da interrupção é a **pilha do processo interrompido** — ela não pode ficar "suja" com dados da interrupção, pois o processo precisará retomá-la normalmente.

### A ideia fundamental

Um processo pode ser interrompido **milhares de vezes** durante sua execução. Mas a ideia central é que, após cada interrupção, o processo retorne **exatamente ao mesmo estado em que se encontrava antes de ter sido interrompido**. Do ponto de vista do processo, a interrupção simplesmente não aconteceu.

> ⚠️ **Em vez de pensar sobre interrupções**, podemos pensar sobre **processos de usuários, processos de disco, processos de terminais** e assim por diante, que ficam bloqueados quando estão esperando algo acontecer. Quando o disco foi lido ou o caractere foi digitado, o processo esperando por ele é desbloqueado e fica disponível para ser executado novamente. Essa abstração é o poder do modelo de processos.
> 

---

# ✅ Resumo do Conceito

- **Tabela de processos / BCP** — estrutura central do SO com uma entrada por processo; armazena tudo necessário para congelar e restaurar um processo: registradores, PC, PSW, ponteiros de memória, descritores de arquivo
- **Vetor de interrupção** — tabela em endereço fixo na memória; mapeia cada tipo de interrupção para o endereço da ISR correspondente
- **ISR** — rotina em C que executa o trabalho específico de cada tipo de interrupção (ler disco, processar teclado etc.)
- **Os 8 passos do tratamento de interrupção:** hardware salva PC+PSW → hardware carrega novo PC via vetor → assembly salva registradores na tabela → assembly configura nova pilha → C executa a ISR → escalonador decide próximo processo → C retorna para assembly → assembly restaura registradores e inicia novo processo
- **Assembly é obrigatório** para salvar/restaurar registradores e trocar pilhas — C não tem acesso direto a essas operações de baixo nível
- **Transparência total ao processo** — um processo pode ser interrompido milhares de vezes; após cada interrupção, ele retoma exatamente do mesmo estado, como se nada tivesse ocorrido
- **Visão em camadas** — o escalonador fica na camada mais baixa do SO, controlando interrupções e trocas de contexto; acima dele ficam todos os processos sequenciais