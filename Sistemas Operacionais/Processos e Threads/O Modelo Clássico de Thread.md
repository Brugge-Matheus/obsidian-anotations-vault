---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.2"
---
# O modelo clássico de thread

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.2

---

# 🧵 2.2.2 — O Modelo Clássico de Thread

O modelo de processo é baseado em dois conceitos independentes: **agrupamento de recursos** e **execução**. Às vezes é útil separá-los — e é exatamente onde as threads entram.

> 💡 **Agrupamento de recursos:** um processo reúne em um único lugar seu espaço de endereços (código + dados do programa), arquivos abertos, processos-filho, alarmes pendentes, tratadores de sinais, informações contábeis e mais. Gerenciar esses recursos juntos na forma de um processo é conveniente.
> 

> 💡 **Execução (thread):** o outro conceito que um processo tem é o de uma linha (*thread*) de execução — normalmente abreviado para apenas **thread**. A thread é a entidade que de fato executa instruções na CPU.
> 

A separação é poderosa: um processo pode ter **múltiplas threads** compartilhando o mesmo conjunto de recursos, mas cada uma com seu próprio fluxo de execução independente.

---

## O que cada thread tem de privado

Cada thread possui seus próprios itens privados:

> 💡 **Contador de programa da thread:** registra qual instrução a thread deve executar a seguir. Cada thread tem o seu próprio — por isso threads diferentes podem estar em pontos diferentes do código ao mesmo tempo.
> 

> 💡 **Registradores da thread:** cada thread tem seu próprio conjunto de registradores de trabalho, que armazenam suas variáveis de trabalho atuais.
> 

> 💡 **Pilha da thread:** contém o histórico de execução da thread — uma estrutura de chamada de procedimento (*stack frame*) para cada procedimento chamado mas ainda não retornado, com as variáveis locais e o endereço de retorno daquele procedimento. Como cada thread geralmente chama procedimentos diferentes e tem uma história de execução diferente, **cada thread precisa da sua própria pilha**.
> 

---

## O que as threads compartilham — Figura 2.11

### Figura 2.11 — Itens por processo vs. itens por thread

> 📌 **Figura 2.11:** A primeira coluna lista alguns itens compartilhados por todas as threads em um processo. A segunda lista alguns itens privados de cada thread.
> 

```
┌──────────────────────────────┬──────────────────────────┐
│        Itens por PROCESSO    │      Itens por THREAD    │
│     (compartilhados por      │    (privados de cada     │
│       todas as threads)      │         thread)          │
├──────────────────────────────┼──────────────────────────┤
│ Espaço de endereços          │ Contador de programa     │
│ Variáveis globais            │ Registradores            │
│ Arquivos abertos             │ Pilha                    │
│ Processos-filho              │ Estado                   │
│ Alarmes pendentes            │                          │
│ Sinais e tratadores de sinais│                          │
│ Informação de contabilidade  │                          │
└──────────────────────────────┴──────────────────────────┘
```

**Os itens compartilhados** pertencem ao **processo**, não às threads. Por exemplo: se uma thread abre um arquivo, esse arquivo fica visível a todas as outras threads do processo — elas podem ler e escrever nele. Isso é lógico, já que o processo, e não a thread, é a unidade de gerenciamento de recursos.

> ⚠️ **Sem proteção entre threads:** uma thread pode ler, escrever ou mesmo apagar a pilha de outra thread. Não há proteção entre threads porque (1) é impossível implementar sem custo enorme e (2) não é necessário — threads de um mesmo processo são criadas pelo mesmo programador para cooperar, não para disputar entre si.
> 

---

## Figura 2.10 — Processos com threads vs. processo único com múltiplas threads

> 📌 **Figura 2.10:** (a) Três processos, cada um com uma thread. (b) Um processo com três threads.
> 

```
       (a)                              (b)
Processo 1  Processo 2  Processo 3         Processo
┌────────┐  ┌────────┐  ┌────────┐      ┌──────────────┐
│Espaço  │  │Espaço  │  │Espaço  │      │ Espaço de    │
│  do    │  │  do    │  │  do    │      │  usuário     │
│usuário │  │usuário │  │usuário │      │  ┌──┐┌──┐┌──┐│
│  (λ)   │  │  (λ)   │  │  (λ)   │      │  │T1││T2││T3││
│Thread  │  │Thread  │  │Thread  │      │  └──┘└──┘└──┘│
├────────┤  ├────────┤  ├────────┤      ├──────────────┤
│ Núcleo │  │ Núcleo │  │ Núcleo │      │    Núcleo    │
└────────┘  └────────┘  └────────┘      └──────────────┘

Cada processo tem seu         Um único processo com 3 threads
próprio espaço de endereços.  compartilhando o mesmo espaço.
As threads NÃO compartilham   Todas as threads enxergam os
memória entre processos.      mesmos dados globais.
```

A organização da Figura 2.10(a) seria usada quando os três processos forem essencialmente não relacionados. A Figura 2.10(b) seria apropriada quando as três threads fazem, na realidade, parte do mesmo trabalho e precisam cooperar umas com as outras de maneira ativa e próxima.

---

## Estados de uma thread

Assim como um processo, uma thread pode estar em qualquer um de vários estados:

| Estado | Descrição |
| --- | --- |
| **Em execução** | A thread tem a CPU naquele momento e está ativa |
| **Bloqueada** | A thread está esperando por algum evento para desbloqueá-la (ex: leitura do teclado) |
| **Pronta** | A thread está programada para ser executada assim que chegar sua vez |
| **Concluída** | A thread terminou e não pode mais ser escalonada |

> ⚠️ As transições entre estados de thread são **exatamente as mesmas** que entre estados de processos — as mesmas quatro transições do diagrama da Figura 2.2 (seção 2.1.5) se aplicam aqui.
> 

---

## Figura 2.12 — Cada thread tem sua própria pilha

> 📌 **Figura 2.12:** Cada thread tem a sua própria pilha.
> 

```
              Thread 2
   Thread 1             Thread 3
      │                    │
┌─────▼────────────────────▼─────┐
│          Processo              │
│  ┌──────┐  ┌──┐  ┌──────┐      │
│  │Pilha │  │  │  │Pilha │      │
│  │  T1  │  │T2│  │  T3  │      │
│  └──────┘  └──┘  └──────┘      │
│       (espaço de endereços)    │
├────────────────────────────────┤
│             Núcleo             │
└────────────────────────────────┘
```

Cada pilha contém uma estrutura para cada procedimento chamado mas ainda não retornado. Se o procedimento X chama Y e Y chama Z, enquanto Z está executando, as estruturas de X, Y e Z estarão todas na pilha. Como cada thread geralmente chama procedimentos diferentes, cada uma terá uma história de execução diferente — por isso cada thread **precisa da sua própria pilha**.

---

## Criação e término de threads

Quando existe multithreading, os processos normalmente começam com uma única thread presente. Essa thread pode criar novas chamando um procedimento de biblioteca:

> 💡 **`thread_create`:** cria uma nova thread. Um parâmetro especifica o nome do procedimento que a nova thread deve executar. Não é necessário especificar o espaço de endereços — a nova thread executa automaticamente no espaço de endereços da thread criadora. Às vezes threads são hierárquicas (relação pai-filho); outras vezes não há relação de hierarquia, e todas as threads são iguais — dependendo do sistema.
> 

> 💡 **`thread_exit`:** chamada quando uma thread termina seu trabalho. A thread desaparece e não pode mais ser escalonada.
> 

> 💡 **`thread_join`:** permite que uma thread espere pela conclusão de uma thread específica antes de continuar. A thread que chamou `thread_join` fica bloqueada até que a thread alvo termine. O identificador da thread alvo é passado como parâmetro.
> 

> 💡 **`thread_yield`:** permite que uma thread abra mão voluntariamente da CPU para que outra thread seja executada. Essa chamada não existe para processos — o pressuposto lá é que processos são competitivos e cada um quer o máximo de CPU possível. Já threads de um mesmo processo são escritas pelo mesmo programador e podem cooperar voluntariamente, dando chance às outras de executar.
> 

---

## A questão do fork com threads

Threads introduzem uma complicação importante no modelo UNIX:

> ⚠️ **fork + threads = problema complexo:** se o processo-pai tem múltiplas threads, o filho deveria tê-las também? Se não, pode não funcionar adequadamente. Se sim, o que acontece se uma thread no pai estava bloqueada numa chamada de leitura do teclado — duas threads ficam bloqueadas no teclado (uma no pai, uma no filho)? Quando uma linha é digitada, ambas recebem uma cópia? Apenas a thread pai? Apenas a thread filho? O mesmo problema existe com conexões de rede abertas. Não há resposta universalmente correta — cada sistema faz escolhas de projeto diferentes. Em Linux, por exemplo, um `fork` de um processo multithreaded cria apenas uma única thread no filho.
> 

---

## Problemas de compartilhamento entre threads

O compartilhamento de dados entre threads é poderoso, mas introduz complicações. Dois exemplos do livro:

- **Fechamento de arquivo:** o que acontece se uma thread fecha um arquivo enquanto outra ainda está lendo dele?
- **Alocação de memória:** se uma thread observa que há pouca memória e começa a alocar mais, e no meio do caminho ocorre uma troca de threads, a nova thread também observa pouca memória e começa a alocar mais — a memória provavelmente será alocada duas vezes.

> ⚠️ Com certo esforço esses problemas podem ser solucionados, mas **programas multithreaded devem ser pensados e projetados com muito cuidado** para funcionarem corretamente.
> 

---

# ✅ Resumo do Conceito

- **Dois conceitos no modelo de processo:** agrupamento de recursos (espaço de endereços, arquivos, sinais) pertence ao **processo**; execução pertence à **thread**
- **O que cada thread tem de privado:** contador de programa, registradores e pilha próprios — suficientes para representar seu estado de execução independente
- **O que as threads compartilham:** tudo que pertence ao processo — espaço de endereços, variáveis globais, arquivos abertos, processos-filho, alarmes, sinais
- **Sem proteção entre threads:** uma thread pode acessar a pilha de outra; não há isolamento porque as threads cooperam, não disputam
- **Estados de thread:** em execução, bloqueada, pronta e concluída — mesmas transições do diagrama de estados de processos
- **Pilha própria por thread:** cada thread chama procedimentos diferentes e tem histórico de execução diferente — por isso precisa de pilha independente
- **`thread_create`** — cria nova thread no mesmo espaço de endereços; **`thread_exit`** — termina a thread; **`thread_join`** — espera outra thread terminar; **`thread_yield`** — cede a CPU voluntariamente para outra thread
- **fork + threads** — relação complexa sem resposta universal; cada SO define sua própria semântica
- **Cuidado com compartilhamento:** fechar arquivos e alocar memória em ambientes multithreaded exige sincronização cuidadosa para evitar condições de corrida