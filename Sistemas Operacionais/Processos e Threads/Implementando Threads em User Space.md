---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.4"
---
# Implementando Threads em User Space

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.4

---

# 🏗️ 2.2.4 — Implementando Threads no Espaço do Usuário

Há dois lugares principais para implementar threads: **no espaço do usuário** e **no núcleo**. Uma implementação híbrida também é possível. Esta seção trata do primeiro método.

---

## O que significa implementar threads no espaço do usuário?

> 💡 **Threads no espaço do usuário:** o pacote de threads é colocado inteiramente no espaço do usuário. O núcleo não sabe nada a respeito delas — ele continua gerenciando processos comuns de uma única thread. As threads são implementadas e gerenciadas por uma **biblioteca** que roda no espaço do usuário, sem qualquer suporte do SO.
> 

Essa abordagem tem uma vantagem histórica importante: pode ser implementada em um SO que **não dá suporte nativo a threads**. Todos os SOs costumavam cair nessa categoria, e mesmo hoje alguns ainda caem.

---

## Figura 2.15 — Threads no espaço do usuário vs. threads no núcleo

> 📌 **Figura 2.15:** (a) Um pacote de threads em nível de usuário. (b) Um pacote de threads gerenciado pelo núcleo.
> 

```
        (a) Threads no espaço do usuário
┌──────────────────────────────────────┐
│           Espaço do usuário          │
│  ┌──────────────┐ ┌──────────────┐  │
│  │   Processo   │ │   Processo   │  │
│  │  ┌────────┐  │ │  ┌────────┐  │  │
│  │  │Thread  │  │ │  │Thread  │  │  │
│  │  │ (λ)(λ) │  │ │  │ (λ)(λ) │  │  │
│  │  └────────┘  │ │  └────────┘  │  │
│  │ Sist. tempo  │ │ Sist. tempo  │  │
│  │  execução    │ │  execução    │  │
│  │ Tab.threads  │ │ Tab.threads  │  │
│  └──────┬───────┘ └──────┬───────┘  │
│         │  Tab. processos│          │
├─────────┴────────────────┴──────────┤
│              Núcleo                 │
│          Tab. de processos          │
└─────────────────────────────────────┘

        (b) Threads no núcleo
┌──────────────────────────────────────┐
│           Espaço do usuário          │
│  ┌──────────────┐ ┌──────────────┐  │
│  │   Processo   │ │   Processo   │  │
│  │  ┌────────┐  │ │  ┌────────┐  │  │
│  │  │Thread  │  │ │  │Thread  │  │  │
│  │  │ (λ)(λ) │  │ │  │ (λ)(λ) │  │  │
│  │  └────────┘  │ │  └────────┘  │  │
│  └──────┬───────┘ └──────┬───────┘  │
├─────────┴────────────────┴──────────┤
│              Núcleo                 │
│    Tab. processos + Tab. threads    │
└─────────────────────────────────────┘
```

Na figura (a), cada processo tem seu próprio **sistema de tempo de execução** e sua própria **tabela de threads** no espaço do usuário. O núcleo só vê processos normais — desconhece completamente a existência das threads. Na figura (b), o núcleo conhece e gerencia as threads diretamente.

---

## Como funciona internamente — a tabela de threads

Quando as threads são gerenciadas no espaço do usuário, cada processo precisa da sua própria **tabela de threads** privada para controlá-las.

> 💡 **Tabela de threads (espaço do usuário):** estrutura análoga à tabela de processos do núcleo, mas mantida pelo sistema de tempo de execução no espaço do usuário. Armazena para cada thread: contador de programa, ponteiro de pilha, registradores, estado e assim por diante. É gerenciada pelo sistema de tempo de execução, não pelo núcleo.
> 

> 💡 **Sistema de tempo de execução (*runtime system*):** coleção de procedimentos que gerencia as threads no espaço do usuário — inclui o escalonador de threads, as chamadas `thread_create`, `thread_exit`, `thread_join`, `thread_yield` e outros. É uma biblioteca que fica dentro do processo.
> 

### O fluxo de uma troca de thread no espaço do usuário

Quando uma thread quer parar voluntariamente (via `thread_yield`):

```
Thread A chama thread_yield
        │
        ▼
Sistema de tempo de execução salva
registradores de A na tabela de threads
        │
        ▼
Escalonador de threads escolhe Thread B
        │
        ▼
Registradores de B são recarregados
na máquina (pilha + PC trocados)
        │
        ▼
Thread B ressurge e continua de onde parou
```

O procedimento que salva o estado da thread e o escalonador são apenas **procedimentos locais** — invocar um procedimento local é muito mais eficiente do que fazer uma chamada de sistema ao núcleo. Não é preciso realizar captura, chaveamento de contexto, esvaziamento de cache etc.

---

## Vantagens das threads no espaço do usuário

**1. Troca de contexto extremamente rápida**

Uma troca de thread requer apenas salvar alguns registradores e carregar outros — isso pode ser feito com poucas instruções, sem envolver o núcleo. É pelo menos uma ordem de grandeza mais rápido do que desviar o controle para o núcleo.

**2. Escalonamento customizável por processo**

Cada processo pode ter seu próprio algoritmo de escalonamento de threads. Para algumas aplicações — por exemplo, aquelas com uma thread coletora de lixo — é vantagem não ter de se preocupar com uma thread sendo parada em um momento inconveniente.

**3. Escalabilidade**

Threads de usuário não exigem espaço de tabela e de pilha no núcleo, o que pode ser um problema se houver um número muito grande de threads.

**4. Funcionam em SOs sem suporte a threads**

Como o núcleo não está envolvido, o mecanismo funciona em qualquer SO.

---

## Problemas das threads no espaço do usuário

### Problema 1 — Chamadas de sistema bloqueantes

> ⚠️ **O problema mais crítico:** suponha que uma thread leia de um teclado antes que quaisquer teclas tenham sido acionadas. Deixar que a thread realmente faça a chamada de sistema `read` é algo inaceitável — isso parará **todas** as threads do processo, pois o núcleo bloqueará o processo inteiro (ele não sabe que existem outras threads).
> 

Uma das principais razões para ter threads é justamente permitir que cada uma utilize chamadas bloqueantes enquanto evita que uma thread bloqueada afete as outras. Com chamadas de sistema bloqueantes no espaço do usuário, essa meta é difícil de alcançar.

**Soluções possíveis (e suas limitações):**

- **Modificar chamadas para não bloqueantes** — por exemplo, um `read` no teclado retornaria 0 bytes se nenhum caractere estivesse disponível no buffer. Exige mudanças no SO, não é atraente, e mudaria a semântica de `read` para muitos programas.
- **Usar `select` antes do `read`** — em muitas versões UNIX existe `select`, que permite saber antecipadamente se um `read` futuro será bloqueado. Se a chamada `read` puder ser bloqueada, outra thread é executada em vez disso; se for segura, `read` é chamado. Isso exige reescrever partes da biblioteca de chamadas de sistema. O código colocado em torno da chamada de sistema é chamado de **wrapper**.

> 💡 **Wrapper:** código que envolve uma chamada de sistema original para interceptá-la e modificar seu comportamento — por exemplo, verificar via `select` se a chamada bloquearia antes de executá-la de fato.
> 

### Problema 2 — Faltas de página

> ⚠️ **Falta de página (*page fault*):** ocorre quando um programa acessa uma instrução ou dado que não está na memória principal no momento. O SO busca a instrução do disco (operação de E/S), bloqueando o processo durante a busca.
> 

Se uma thread causa uma falta de página, o núcleo — sem saber da existência das outras threads — bloqueia o **processo inteiro**, embora outras threads pudessem ser executadas enquanto a E/S ocorre.

### Problema 3 — Thread que nunca cede a CPU voluntariamente

> ⚠️ Dentro de um único processo, **não há interrupções de relógio** que forcem o escalonador de threads do espaço do usuário a agir. A menos que uma thread chame voluntariamente `thread_yield` ou execute uma chamada de sistema, o escalonador jamais terá chance de rodar — e uma thread que nunca cede a CPU monopolizará o processo inteiro.
> 

**Solução possível:** obrigar o sistema de tempo de execução a solicitar um **sinal de relógio** (interrupção) a cada segundo para ganhar controle. Mas isso é grosseiro e confuso para o programa. Além disso, interrupções periódicas em frequência mais alta nem sempre são possíveis — e mesmo que fossem, o custo adicional poderia ser substancial. Uma thread talvez precise também de uma interrupção de relógio, interferindo com o uso do relógio pelo sistema de tempo de execução.

> 💡 **Sinal de relógio:** evento gerado pela interrupção periódica do chip timer de hardware (a cada N milissegundos). No contexto de threads no espaço do usuário, é a única forma do sistema de tempo de execução retomar o controle de uma thread que não cede voluntariamente a CPU — mas seu uso é limitado e introduz complexidade.
> 

### Problema 4 — Aplicações que fazem chamadas de sistema com frequência

> ⚠️ O argumento mais devastador: programadores geralmente desejam threads precisamente em aplicações que são bloqueadas com frequência — como servidores web com múltiplas threads. Essas threads estão constantemente fazendo chamadas de sistema. Uma vez que ocorra a captura para o núcleo para executar uma chamada, o núcleo poderia simplesmente trocar a thread em execução — eliminando a necessidade de `select` e wrappers. Para aplicações vinculadas à CPU e raramente bloqueadas, threads de usuário fazem sentido. Mas para aplicações com E/S intensiva, threads no núcleo são mais adequadas.
> 

---

## Resumo comparativo — Threads de usuário vs. Threads de núcleo

| Aspecto | Threads de usuário | Threads de núcleo |
| --- | --- | --- |
| **O núcleo sabe das threads?** | Não | Sim |
| **Troca de contexto** | Muito rápida (sem syscall) | Mais lenta (envolve syscall) |
| **Escalonamento** | Customizável por processo | Definido pelo núcleo |
| **Chamada bloqueante** | Bloqueia o processo inteiro | Bloqueia só a thread |
| **Falta de página** | Bloqueia o processo inteiro | Bloqueia só a thread |
| **Thread que monopoliza CPU** | Problema real (sem interrupção) | SO retoma controle via timer |
| **Suporte a SO antigo** | Funciona sem suporte do SO | Requer suporte do SO |
| **Escalabilidade** | Melhor (sem tabela no núcleo) | Limitada pelo espaço do núcleo |

---

# ✅ Resumo do Conceito

- **Threads no espaço do usuário** — implementadas inteiramente por uma biblioteca; o núcleo desconhece sua existência e gerencia apenas processos; cada processo tem seu próprio sistema de tempo de execução e tabela de threads
- **Troca de contexto ultra-rápida** — salvar/restaurar registradores é feito via procedimentos locais, sem envolver o núcleo; pelo menos uma ordem de grandeza mais rápido que threads de núcleo
- **Escalonamento customizável** — cada processo pode definir seu próprio algoritmo de escalonamento de threads
- **Problema crítico — chamadas bloqueantes:** uma thread que faz `read` bloqueante paralisa o processo inteiro; soluções via `select` e wrappers existem mas são ineficientes
- **Wrapper** — código que envolve uma chamada de sistema para verificar se ela bloquearia antes de executá-la
- **Problema — falta de página:** thread que acessa memória ausente bloqueia o processo inteiro, mesmo que outras threads estejam prontas
- **Problema — monopolização da CPU:** sem interrupção de relógio interna ao processo, uma thread que não cede voluntariamente a CPU impede todas as outras de rodar
- **Sinal de relógio** — gerado pela interrupção periódica do timer de hardware; única forma do sistema de tempo de execução retomar controle de threads não cooperativas, mas seu uso introduz complexidade e limitações
- **Caso de uso ideal:** aplicações vinculadas à CPU com poucas chamadas bloqueantes; para E/S intensiva, threads de núcleo são mais adequadas