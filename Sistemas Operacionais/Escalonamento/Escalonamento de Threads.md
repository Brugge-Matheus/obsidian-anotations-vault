---
tags:
  - sistemas-operacionais
  - so/escalonamento
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.5.6"
---
# Escalonamento de Threads

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.5.6

---

# 🧵 2.5.6 — Escalonamento de Threads

## Dois níveis de paralelismo

Quando vários processos têm, cada um, múltiplas threads, temos **dois níveis de paralelismo** presentes: **processos** e **threads**. Escalonar nesses sistemas difere substancialmente, dependendo de existir suporte para threads de usuário ou threads de núcleo (ou ambos).

---

## 🧑‍💻 Escalonamento de Threads de Usuário

Vamos considerar primeiro as threads de usuário.

```
Tendo em vista que o núcleo NÃO tem ciência da
existência das threads, ele opera como sempre faz:
  → escolhendo um processo, digamos A
  → dando a A controle de seu quantum

O escalonador de threads DENTRO de A decide qual
thread executar, digamos, A1.

Dado que NÃO há interrupções de relógio para
multiprogramar threads, essa thread pode continuar
a ser executada por QUANTO TEMPO QUISER.

SE ela utilizar TODO o quantum do processo, o núcleo
selecionará OUTRO processo para executar.
```

### O que acontece quando o processo A volta a executar

```
Quando o processo A, por fim, executar novamente,
a thread A1 RETOMARÁ a execução.

Ela continuará a consumir todo o tempo de A até que
TERMINE.

⚠️ O comportamento "antissocial" de A1 NÃO afetará
outros processos.

Eles receberão o que quer que o escalonador considere
sua fração apropriada — não importa o que estiver
acontecendo DENTRO do processo A.
```

### Um cenário mais realista — threads de curta duração

```
Agora considere o caso em que as threads de A tenham
relativamente pouco trabalho para fazer por SURTO DE
CPU, por exemplo, 5 ms de trabalho dentro de um
quantum de 50 ms.

Em consequência:
  → cada uma executa por um tempo
  → então cede a CPU de VOLTA para o escalonador de
    threads

Isso pode levar à sequência:

  A1, A2, A3, A1, A2, A3, A1, A2, A3, A1

antes que o núcleo alterne para o processo B.
```

> 📌 **Figura 2.45(a) — Escalonamento possível de threads de usuário com quantum de processo de 50ms e threads que executam 5ms por surto de CPU**

```
Ordem na qual
as threads
executam

     Processo A                 Processo B
    ┌─────────┐                ┌─────────┐
2.  │ 1  2  3 │  ◄─ o sistema  │         │
    │         │     em tempo   │         │
1.  │  ↓      │     de exec.   │         │
    │ núcleo  │     seleciona  │         │
    │ seleciona│    uma thread │         │
    │ processo │                │         │
    └─────────┘                └─────────┘

Possível:    A1, A2, A3, A1, A2, A3
Impossível:  A1, B1, A2, B2, A3, B3
```

**A restrição fundamental:** dentro do processo A, a ordem A1, A2, A3, A1, A2, A3 é possível. Mas intercalar threads de **processos diferentes** (A1, B1, A2, B2...) é **impossível** — porque o núcleo só troca de processo quando o quantum do processo termina, e o escalonador de threads de usuário só controla threads do **mesmo** processo.

---

## ⚙️ Escalonamento de Threads de Núcleo

Agora considere a situação com **threads de núcleo**.

```
Aqui o núcleo ESCOLHE uma thread em PARTICULAR para
executar.

Ele NÃO precisa levar em conta a qual processo a
thread pertence — PORÉM ele PODE, se assim o desejar.

A thread recebe um quantum e é suspensa
COMPULSORIAMENTE se o exceder.
```

### O exemplo numérico — threads bloqueando rápido

```
Com um quantum de 50ms e threads que BLOQUEIAM após
5ms:

A ordem das threads por um período de 30ms PODE ser:

  A1, B1, A2, B2, A3, B3

⚠️ Algo que NÃO é possível com esses parâmetros e
threads de USUÁRIO (o núcleo simplesmente não sabe
que A1, A2, A3 existem separadamente — só vê o
processo A).
```

> 📌 **Figura 2.45(b) — Escalonamento possível de threads de núcleo com as mesmas características que (a)**

```
     Processo A                 Processo B
    ┌─────────┐                ┌─────────┐
    │ 1     3 │                │    2    │
1.  │  ↓      │                │         │
    │ núcleo  │                │         │
    │ seleciona│                │         │
    │ uma thread│               │         │
    └─────────┘                └─────────┘

Possível:      A1, A2, A3, A1, A2, A3
Também possível: A1, B1, A2, B2, A3, B3
```

### A grande diferença de desempenho — user space vs. kernel space

> ⚠️ **Fazer uma troca de threads de USUÁRIO exige algumas instruções de máquina.** Com threads de NÚCLEO, é necessária uma **troca de contexto COMPLETA**, alterando o mapa de memória e invalidando a **cache**, que é **VÁRIAS ORDENS DE GRANDEZA MAIS LENTO**.

```
Por outro lado, com threads de NÚCLEO, ter uma thread
BLOQUEADA por E/S NÃO SUSPENDE O PROCESSO INTEIRO —
como acontece com as threads de USUÁRIO.
```

Esse é exatamente o trade-off que já vimos em [[Implementando Threads em User Space]] e [[Implementando Threads em Kernel Space]]: threads de usuário são baratas de trocar, mas uma syscall bloqueante trava o processo inteiro; threads de núcleo são caras de trocar, mas bloquear uma não afeta as outras.

---

## 🤔 Quando o Núcleo Considera Dependências Entre Threads

```
Visto que o núcleo SABE mudar de uma thread no
processo A para uma thread no processo B é mais CARO
do que executar uma SEGUNDA thread no processo A
(devido a ter que alterar o mapa de memória e ter a
memória cache invalidada):

  → ele PODE levar em consideração essas informações
    ao tomar uma decisão

Exemplo:
  Dadas DUAS threads que são, de outra forma,
  IGUALMENTE importantes, com uma delas pertencendo
  ao MESMO processo que uma thread que foi bloqueada
  HÁ POUCO, e outra pertencendo a um processo
  DIFERENTE:

  → a PREFERÊNCIA poderia ser dada à PRIMEIRA
    (a do mesmo processo — evita o custo extra de
     trocar de processo)
```

---

## 🎯 Escalonadores de Threads Específicos de Aplicação

Outro fator importante: as threads de **usuário** podem empregar um escalonador de threads **específico para uma aplicação**.

### O exemplo do servidor web

```
Considere o servidor WEB da Figura 2.8 (a arquitetura
multithreaded despachante + operárias):

Suponha que:
  → uma thread OPERÁRIA tenha sido bloqueada há pouco
  → a thread DESPACHANTE e DUAS threads OPERÁRIAS
    estejam PRONTAS

QUEM deve ser executada em seguida?

O SISTEMA DE TEMPO DE EXECUÇÃO, sabendo o que TODAS
as threads fazem, pode FACILMENTE escolher a
despachante para ser executada em seguida, de maneira
que ELA possa colocar outra operária para executar.

Essa estratégia MAXIMIZA o montante de paralelismo
em um ambiente onde operárias FREQUENTEMENTE são
bloqueadas pela E/S de disco.
```

### Por que threads de núcleo não conseguem fazer isso tão bem

```
Com threads de NÚCLEO:
  → o núcleo JAMAIS saberia o que cada thread FEZ
  → (embora prioridades DIFERENTES pudessem ser
    atribuídas a elas)

NO GERAL, entretanto:
  → escalonadores de threads ESPECÍFICOS de
    aplicações são capazes de ajustar uma aplicação
    MELHOR do que o núcleo
```

> 💡 **A vantagem central das threads de usuário aqui:** o escalonador vive DENTRO da aplicação e tem conhecimento semântico sobre o PAPEL de cada thread (despachante vs. operária) — algo que o núcleo, agnóstico ao propósito de cada thread, não pode replicar tão bem.

---

# ✅ Resumo do Conceito

- Sistemas com múltiplas threads por processo têm **dois níveis de paralelismo**: entre processos e entre threads dentro de cada processo
- **Threads de usuário:** o núcleo escolhe um processo e lhe dá o quantum inteiro; o escalonador de threads DENTRO do processo decide qual thread executa. Sem interrupção de relógio para threads, uma thread pode monopolizar todo o tempo do processo sem afetar outros processos. A ordem A1,B1,A2,B2... (intercalando processos diferentes) é **impossível**
- **Threads de núcleo:** o núcleo escolhe threads individualmente, podendo intercalar threads de processos diferentes (A1,B1,A2,B2...) — algo impossível com threads de usuário
- **Trade-off de desempenho:** trocar threads de usuário exige poucas instruções de máquina; trocar threads de núcleo exige troca de contexto completa (mapa de memória + invalidação de cache) — ordens de grandeza mais lenta. Em compensação, com threads de núcleo, uma thread bloqueada por E/S não suspende o processo inteiro
- O núcleo, ciente do custo de trocar de processo, PODE favorecer threads do mesmo processo que acabou de bloquear, para evitar a troca mais cara
- **Escalonadores específicos de aplicação** (possíveis com threads de usuário) podem ter conhecimento semântico sobre o papel de cada thread — ex: no servidor web despachante+operárias, o sistema de tempo de execução escolhe a despachante para executar em seguida, maximizando o paralelismo. Threads de núcleo não têm esse conhecimento (embora possam usar prioridades diferentes)

---

## 🔗 Notas Relacionadas

- [[Implementando Threads em User Space]] — a implementação que fundamenta por que threads de usuário são baratas de trocar mas bloqueiam o processo inteiro
- [[Implementando Threads em Kernel Space]] — a implementação que fundamenta o custo da troca de contexto completa
- [[Utilização de Threads]] — a arquitetura despachante + operárias do servidor web (Figura 2.8) usada como exemplo de escalonador específico de aplicação
- [[Escalonamento em Sistemas Interativos]] — o escalonamento circular e por prioridades mencionados como os mais comuns para threads na prática
