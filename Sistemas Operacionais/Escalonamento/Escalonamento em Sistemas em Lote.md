---
tags:
  - sistemas-operacionais
  - so/escalonamento
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.5.2"
---
# Escalonamento em Sistemas em Lote

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.5.2

---

# 📦 2.5.2 — Escalonamento em Sistemas em Lote

Em [[Introdução ao Escalonamento]], vimos as metas gerais e as categorias de sistemas. Agora chegou o momento de passar das questões gerais para **algoritmos de escalonamento específicos**.

Vale destacar: alguns algoritmos vistos aqui para sistemas em lote também são usados em sistemas interativos — eles serão estudados novamente mais adiante.

---

## 🎟️ Primeiro a Chegar, Primeiro a Ser Servido (FCFS)

> 💡 **FCFS (*First-Come, First-Served*):** provavelmente o mais simples de todos os algoritmos de escalonamento já projetados. É **não preemptivo**. A CPU é atribuída aos processos na ordem em que eles a requisitam.

### Como funciona

```
Estrutura: uma ÚNICA fila de processos prontos (lista encadeada)

Quando a primeira tarefa entra no sistema de manhã:
  → é iniciada IMEDIATAMENTE
  → é deixada executar por quanto tempo ela quiser
  → NÃO é interrompida por ter sido executada por
    tempo demais

À medida que outras tarefas chegam:
  → são colocadas no FINAL da fila

Quando o processo em execução é bloqueado:
  → o PRIMEIRO processo da fila é executado em seguida

Quando um processo bloqueado fica pronto novamente:
  → assim como uma tarefa recém-chegada, ele é colocado
    no FIM da fila, atrás dos processos em espera
```

### Vantagens

```
✅ Fácil de compreender
✅ Igualmente fácil de programar
✅ Uma única lista encadeada controla TODOS os processos
✅ Escolher um processo para executar = remover um da
   frente da fila
✅ Acrescentar uma nova tarefa ou desbloquear um processo
   = apenas colocá-lo no final da fila
✅ É tão "justo" quanto alocar ingressos para um show ou
   iPhones novos por ordem de chegada na fila
```

### A grande desvantagem

```
Cenário problemático:
  1 processo VINCULADO À COMPUTAÇÃO
    → executado por 1s de cada vez
  Muitos processos VINCULADOS À E/S
    → usam pouco tempo de CPU
    → mas cada um precisa fazer 1.000 leituras de
      disco para ser concluído

Sequência dos eventos:
  1. Processo vinculado à computação executa por 1s
  2. Lê um bloco de disco (bloqueia)
  3. TODOS os processos de E/S executam agora
     → começam suas leituras de disco
  4. Quando o processo vinculado à computação obtém
     seu bloco de disco, ele é executado por mais 1s
  5. Depois disso, todos os processos vinculados à
     E/S ficam em fila atrás dele NOVAMENTE

RESULTADO LÍQUIDO:
  Cada processo vinculado à E/S lê apenas 1 BLOCO DE
  DISCO POR SEGUNDO
  → levará 1.000s para terminar (1.000 leituras necessárias)

COM PREEMPÇÃO a cada 10ms:
  → processos vinculados à E/S terminariam em 10s
    em vez de 1.000s
  → e não retardariam MUITO o processo vinculado
    à computação
```

> ⚠️ **O problema central do FCFS:** um único processo vinculado à computação pode fazer com que dezenas de processos vinculados à E/S — que individualmente usam muito pouca CPU — esperem absurdamente, simplesmente por estarem atrás na fila toda vez que ficam prontos.

---

## ⏱️ Tarefa Mais Curta Primeiro (SJF)

Agora um algoritmo em lote **não preemptivo** que presume que os **tempos de execução são conhecidos antecipadamente**.

```
Contexto de uso:
  Companhia de seguros — as pessoas podem prever com
  bastante precisão quanto tempo levará para executar
  um lote de 1.000 reivindicações (tendo em vista que
  um trabalho similar é realizado todos os dias)
```

> 💡 **SJF (*Shortest Job First* — tarefa mais curta primeiro):** quando há vários trabalhos igualmente importantes esperando na fila de entrada para serem iniciados, o escalonador escolhe a **tarefa mais curta primeiro**.

> 📌 **Figura 2.42 — Um exemplo do escalonamento tarefa mais curta primeiro. (a) Executando quatro tarefas na ordem original. (b) Executando-as na ordem da tarefa mais curta primeiro.**

### O exemplo — 4 tarefas

```
Tarefas A, B, C, D com tempos de execução: 8, 4, 4, 4 minutos

(a) ORDEM ORIGINAL (A, B, C, D):

  ┌────┬────┬────┬────┐
  │  A │  B │  C │  D │
  │  8 │  4 │  4 │  4 │
  └────┴────┴────┴────┘

  Tempo de retorno de A = 8   (termina em 8)
  Tempo de retorno de B = 12  (termina em 8+4)
  Tempo de retorno de C = 16  (termina em 8+4+4)
  Tempo de retorno de D = 20  (termina em 8+4+4+4)

  Média = (8+12+16+20)/4 = 14 minutos

(b) ORDEM SJF (B, C, D, A):

  ┌────┬────┬────┬────┐
  │  B │  C │  D │  A │
  │  4 │  4 │  4 │  8 │
  └────┴────┴────┴────┘

  Tempo de retorno de B = 4
  Tempo de retorno de C = 8
  Tempo de retorno de D = 12
  Tempo de retorno de A = 20

  Média = (4+8+12+20)/4 = 11 minutos ✅ MELHOR
```

### Por que SJF é ótimo — a prova matemática

```
Considere 4 tarefas com tempos de execução a, b, c, d
(nessa ordem de execução).

A primeira tarefa termina no tempo a
A segunda termina no tempo a+b
A terceira termina no tempo a+b+c
A quarta termina no tempo a+b+c+d

Tempo de retorno médio = (4a + 3b + 2c + d) / 4

Observe os PESOS:
  a aparece 4 vezes  → 'a' contribui MAIS para a média
  b aparece 3 vezes
  c aparece 2 vezes
  d aparece 1 vez    → 'd' contribui MENOS

CONCLUSÃO:
  Para minimizar a média, 'a' deve ser a TAREFA MAIS CURTA
  → depois 'b' (a segunda mais curta)
  → depois 'c'
  → e 'd' deve ser a MAIS LONGA
    (porque afeta apenas o SEU PRÓPRIO tempo de retorno,
     não o de ninguém mais)

O mesmo argumento se aplica igualmente bem a qualquer
número de tarefas.
```

> 💡 **SJF é comprovadamente ótimo** — nenhum outro ordenamento produz tempo de retorno médio menor, DADO um conjunto fixo de tarefas com tempos conhecidos.

### A ressalva importante — SJF só é ótimo quando todas as tarefas estão disponíveis simultaneamente

> ⚠️ **Contraexemplo:** SJF é ótimo apenas quando todas as tarefas estão disponíveis SIMULTANEAMENTE.

```
5 tarefas A, B, C, D, E:
  Tempos de execução: 2, 4, 1, 1, 1
  Tempos de CHEGADA:  0, 0, 3, 3, 3

No início (tempo 0):
  Apenas A ou B podem ser escolhidas
  (as outras três tarefas ainda NÃO chegaram)

Usando SJF PURO (executando A, B, C, D, E na ordem
de chegada disponível):
  → tempo de espera médio = 4,6

Executando na ordem B, C, D, E, A:
  → tempo de espera médio = 4,4  ✅ MELHOR

Por que a segunda ordem é melhor mesmo "quebrando"
a regra de tarefa mais curta primeiro?
  → ao adiar A (a tarefa mais longa disponível
    inicialmente) e deixar as tarefas curtas que
    chegam depois (C, D, E) passarem à frente,
    o tempo de espera total é reduzido
```

> ⚠️ **Conclusão prática:** quando tarefas chegam em momentos diferentes, escolher greedy (sempre a mais curta disponível no momento) nem sempre é ótimo globalmente — mas isso normalmente não pode ser conhecido antecipadamente no mundo real.

---

## ⏳ Tempo Restante Mais Curto em Seguida (SRTN)

Uma versão **preemptiva** do SJF.

> 💡 **SRTN (*Shortest Remaining Time Next* — tempo restante mais curto em seguida):** o escalonador escolhe o processo cujo **tempo de execução restante** é o mais curto. O tempo de execução precisa ser conhecido antecipadamente, como no SJF.

### Como funciona a preempção

```
Quando uma NOVA tarefa chega:
  1. Seu tempo TOTAL é comparado com o tempo RESTANTE
     do processo ATUAL

  2. SE a nova tarefa precisa de MENOS tempo para
     terminar do que o processo atual:
     → o processo atual é SUSPENSO
     → a nova tarefa é INICIADA imediatamente

  3. SE a nova tarefa precisa de MAIS (ou igual) tempo:
     → o processo atual continua normalmente

BENEFÍCIO:
  Esse esquema permite que TAREFAS CURTAS NOVAS
  tenham um bom desempenho — não precisam esperar
  atrás de uma tarefa longa já em execução.
```

```
┌─────────────────┬──────────────┬──────────────────────┐
│    Algoritmo     │  Preemptivo  │  Conhece tempo?       │
├─────────────────┼──────────────┼──────────────────────┤
│ FCFS             │ Não          │ Não                    │
│ SJF              │ Não          │ Sim (tempo total)      │
│ SRTN             │ Sim          │ Sim (tempo restante)   │
└─────────────────┴──────────────┴──────────────────────┘
```

---

# ✅ Resumo do Conceito

- **FCFS (Primeiro a Chegar, Primeiro a Ser Servido):** algoritmo não preemptivo com uma única fila; simples de implementar, mas o grande problema é que um único processo vinculado à computação pode fazer com que muitos processos vinculados à E/S esperem absurdamente, mesmo usando pouca CPU individualmente
- **SJF (Tarefa Mais Curta Primeiro):** não preemptivo, presume tempos de execução conhecidos antecipadamente (comum em sistemas em lote previsíveis, como processamento de reivindicações de seguro); é **matematicamente ótimo** para minimizar o tempo de retorno médio quando todas as tarefas estão disponíveis ao mesmo tempo — porque a tarefa mais curta pesa mais vezes na soma do tempo de retorno médio
- **A ressalva do SJF:** quando tarefas chegam em momentos diferentes, a estratégia gulosa de sempre pegar a mais curta disponível no momento pode NÃO ser globalmente ótima — mas essa informação futura raramente é conhecida na prática
- **SRTN (Tempo Restante Mais Curto em Seguida):** versão preemptiva do SJF — uma tarefa nova com tempo total menor que o tempo restante do processo atual pode preemptá-lo; favorece tarefas curtas recém-chegadas

---

## 🔗 Notas Relacionadas

- [[Introdução ao Escalonamento]] — as metas gerais (vazão, tempo de retorno, utilização de CPU) que estes algoritmos tentam otimizar
- [[Modelando a Multiprogramação]] — contexto sobre processos vinculados à CPU/E/S e por que múltiplos processos de E/S mantêm o sistema ocupado
- [[Estados de Processos]] — os estados pronto/bloqueado envolvidos na fila de escalonamento do FCFS
