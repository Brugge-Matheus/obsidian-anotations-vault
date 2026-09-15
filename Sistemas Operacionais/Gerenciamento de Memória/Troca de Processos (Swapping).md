---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.2.2"
---
# Troca de Processos (Swapping)

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.2.2

---

# 💾 3.2.2 — Troca de Processos (*Swapping*)

## Quando os esquemas anteriores bastam — e quando não bastam

Se a memória física do computador for **grande o suficiente** para armazenar todos os processos, os esquemas descritos em [[Espaços de Endereçamento e Registradores-Base e Limite]] em parte bastarão.

> ⚠️ **Mas, na prática, a quantidade total de RAM demandada por todos os processos é muitas vezes maior do que pode ser colocada na memória.**

### O exemplo concreto — sistemas típicos hoje

```
Em sistemas típicos Windows, MacOS ou Linux:

  ALGO COMO 50 a 100 processos podem ser INICIADOS
  tão logo o computador é ligado.

Exemplo — instalação de uma aplicação do Windows:
  → emite comandos de tal forma que, em
    inicializações SUBSEQUENTES do sistema, um
    processo será iniciado SOMENTE para conferir se
    existem atualizações para as aplicações
  → um processo desses pode facilmente ocupar
    5 a 10 MB de memória

Outros processos de segundo plano conferem:
  → se há e-mails
  → conexões de rede chegando
  → muitas outras coisas

⚠️ E TUDO ISSO antes de o PRIMEIRO programa do
usuário ter sido iniciado.
```

### Programas de aplicação sérios exigem ainda mais

```
Programas de aplicação sérios do usuário, como o
Photoshop, podem facilmente exigir:

  500 MB apenas para serem INICIALIZADOS

  e muitos GIGABYTES assim que começam a
  PROCESSAR dados

Em consequência: manter TODOS os processos na
memória o TEMPO INTEIRO exige uma quantidade
ENORME de memória e é algo que NÃO pode ser feito
se ela for INSUFICIENTE.
```

---

## 🔀 Duas Abordagens Gerais para o Custo Adicional de Memória

Duas abordagens gerais para lidar com o excesso de demanda por memória foram desenvolvidas ao longo dos anos:

```
1. SWAPPING (troca de processos)
2. MEMÓRIA VIRTUAL
```

### 1 — Swapping

> 💡 **Swapping (troca de processos):** a estratégia mais simples, que consiste em trazer cada processo em sua **totalidade**, executá-lo por um tempo e então colocá-lo de volta no armazenamento **não volátil** (disco ou SSD).

```
Processos OCIOSOS estão armazenados em disco em
sua maior parte, portanto NÃO ocupam qualquer
memória quando NÃO estão sendo executados (embora
alguns "despertem" periodicamente para fazer seu
trabalho e então voltem a "dormir").
```

### 2 — Memória Virtual

> 💡 **Memória virtual:** a outra estratégia, que permite que os programas possam ser executados mesmo quando estão **parcialmente** na memória principal.

```
A seguir estudaremos a troca de processos;
na Seção 3.3 examinaremos a memória virtual.
```

---

## 🎬 A Operação do Sistema de Troca de Processos

A operação de um sistema de troca de processos está ilustrada na **Figura 3.4**.

```
                          Tempo ─────────────►

(a)          (b)          (c)          (d)          (e)          (f)          (g)
┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────┐
│////│      │////│      │////│      │////│      │////│      │////│      │////│
│////│      │////│      │////│      │////│      │////│      │////│      │////│
├────┤      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤
│  A │      │  B │      │  C │      │  C │      │  C │      │  C │      │  C │
├────┤      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤
│Sis-│      │  A │      │  B │      │////│      │  D │      │  D │      │  A │
│tema│      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤
│Ope-│      │Sis-│      │  A │      │  D │      │  B │      │////│      │  D │
│rac.│      │tema│      ├────┤      ├────┤      ├────┤      ├────┤      ├────┤
│    │      │Ope-│      │Sis-│      │Sis-│      │Sis-│      │Sis-│      │Sis-│
│    │      │rac.│      │tema│      │tema│      │tema│      │tema│      │tema│
│    │      │    │      │Ope-│      │Ope-│      │Ope-│      │Ope-│      │Ope-│
│    │      │    │      │rac.│      │rac.│      │rac.│      │rac.│      │rac.│
└────┘      └────┘      └────┘      └────┘      └────┘      └────┘      └────┘
```

> 📌 **Figura 3.4 — Mudanças na alocação de memória à medida que processos entram e saem dela. As regiões sombreadas são regiões não utilizadas da memória.**

### Passo a passo

```
DE INÍCIO:
  → somente o processo A está na memória

DEPOIS:
  → os processos B e C são criados ou trazidos do
    armazenamento não volátil

NA FIGURA 3.4(d):
  → o processo A é DEVOLVIDO ao armazenamento não
    volátil

ENTÃO:
  → o processo D é INSERIDO
  → o processo B é TIRADO

POR FIM:
  → o processo A é NOVAMENTE trazido
```

### A questão da realocação — endereços precisam mudar

```
Como A está agora em uma posição DIFERENTE:

  Os endereços contidos nele DEVEM ser REALOCADOS,

  seja pelo SOFTWARE quando ele é trazido,

  OU (mais provável) pelo HARDWARE durante a
  EXECUÇÃO do programa.

  Por exemplo, REGISTRADORES-BASE e LIMITE
  funcionariam bem nesse caso.
```

---

## 📈 O Problema do Crescimento — Segmentos de Dados que Podem Crescer

Se os segmentos de dados dos processos podem crescer, por exemplo, alocando dinamicamente memória de um **heap**, como em muitas linguagens de programação, um problema acontece sempre que um processo tenta crescer.

### Se houver espaço adjacente disponível

```
SE houver um espaço ADJACENTE ao processo:
  → ele poderá ser ALOCADO
  → e o processo será autorizado a crescer naquele
    espaço
```

### Se o espaço adjacente estiver ocupado

```
SE o processo for ADJACENTE a outro:
  → AQUELE que cresce terá de ser MOVIDO para um
    espaço na memória grande o suficiente para ele,

  OU

  → um ou mais processos terão de ser TROCADOS para
    o disco para criar um espaço grande o suficiente
```

### Se não houver como acomodar o crescimento

```
SE um processo NÃO puder crescer na memória e a
área de troca no disco estiver CHEIA:
  → ele terá de ser SUSPENSO até que espaço seja
    liberado (ou ele pode ser MORTO)
```

---

## 📐 Alocando Espaço Extra Antecipadamente

Se o esperado for que **a maioria dos processos** cresça à medida que são executados, provavelmente seja uma boa ideia **alocar um pouco de memória extra** sempre que um processo for trocado ou movido.

```
POR QUÊ?
  → para reduzir o custo adicional associado com a
    TROCA e a MOVIMENTAÇÃO dos processos que não
    cabem mais em sua memória alocada
```

### Mas cuidado ao transferir para o disco

```
No entanto, ao transferir processos para o
armazenamento não volátil, APENAS a memória que
está EM USO deve ser TRANSFERIDA.

⚠️ É um DESPERDÍCIO levar a memória EXTRA também.
```

> 📌 **Figura 3.5(a) — mostra uma configuração de memória na qual o espaço para o crescimento foi alocado para dois processos**

```
Espaço para expansão
┌──────────────────┐
│  Espaço para      │
│  expansão          │
├──────────────────┤
│  B                │
│  Realmente em uso │
├──────────────────┤
│////////////////  │  (área não utilizada)
├──────────────────┤
│  Espaço para      │
│  expansão          │
├──────────────────┤
│  A                │
│  Realmente em uso │
├──────────────────┤
│  Sistema          │
│  operacional      │
└──────────────────┘
        (a)
```

---

## 🔄 Uma Alternativa — Pilha e Dados em Extremos Opostos

Se os processos podem ter **dois** segmentos em expansão — por exemplo, os segmentos de **dados** usados como uma área temporária para variáveis alocadas/liberadas dinamicamente, e uma área de **pilha** para as variáveis locais normais e os endereços de retorno — uma solução alternativa se apresenta, como a da Figura 3.5(b).

> 📌 **Figura 3.5(b) — Alocação de espaço para uma pilha e um segmento de dados em expansão**

```
┌──────────────────┐
│  Pilha B          │
│  Espaço para       │
│  expansão          │
├──────────────────┤
│  Dados B          │
├──────────────────┤
│  Programa B        │
├──────────────────┤
│////////////////  │  (área não utilizada)
├──────────────────┤
│  Pilha A          │
│  Espaço para       │
│  expansão          │
├──────────────────┤
│  Dados A           │
├──────────────────┤
│  Programa A        │
├──────────────────┤
│  Sistema           │
│  operacional        │
└──────────────────┘
        (b)
```

### Como funciona

```
Cada processo ilustrado TEM:

  → uma PILHA no TOPO de sua memória alocada, que
    CRESCE PARA BAIXO

  → um segmento de DADOS logo além do texto do
    programa, que CRESCE PARA CIMA

A memória ENTRE eles pode ser usada por QUALQUER
segmento.

SE ela ACABAR:
  → o processo poderá ser TRANSFERIDO para outra
    área com espaço suficiente

  → ser TRANSFERIDO para o disco até que um espaço
    de tamanho suficiente possa ser criado

  → ou ser MORTO
```

---

# ✅ Resumo do Conceito

- Quando a memória total demandada por todos os processos excede a RAM disponível (algo muito comum — 50-100 processos podem existir só na inicialização do sistema, e aplicações sérias como Photoshop podem exigir centenas de MB), duas estratégias gerais lidam com isso: **swapping** e **memória virtual**
- **Swapping:** traz cada processo em sua TOTALIDADE, executa por um tempo, e o devolve ao armazenamento não volátil (disco/SSD) — processos ociosos ficam armazenados em disco, sem ocupar memória
- A **Figura 3.4** ilustra a mudança dinâmica de alocação conforme processos entram e saem — quando um processo retorna em posição diferente, seus endereços precisam ser **realocados** (via software ou, mais comum, via hardware como registradores-base e limite)
- **Segmentos que crescem** (heap dinâmico) criam um problema: se há espaço adjacente, o processo cresce nele; senão, precisa ser movido ou outros processos precisam ser trocados para o disco; se nada disso for possível, o processo é suspenso (ou morto)
- É comum **alocar espaço extra antecipadamente** ao trocar/mover um processo, esperando crescimento futuro e reduzindo o custo de operações repetidas — mas apenas a memória **realmente em uso** deve ser transferida para o disco, nunca a extra alocada
- Uma alternativa elegante (Figura 3.5b) coloca a **pilha** no topo do espaço alocado (crescendo para baixo) e os **dados** logo após o programa (crescendo para cima) — a memória entre os dois é compartilhada por qualquer um dos dois segmentos, maximizando a flexibilidade sem desperdício

---

## 🔗 Notas Relacionadas

- [[Espaços de Endereçamento e Registradores-Base e Limite]] — o mecanismo de realocação (base/limite) usado quando um processo retorna em posição diferente após ser trocado
- [[Ausência de abstração de memória]] — o conceito original de swapping mencionado ali, agora detalhado
- [[Gerenciando a Memória Livre]] — próximo tópico, sobre como o SO rastreia quais partes da memória estão livres para alocar processos trocados
