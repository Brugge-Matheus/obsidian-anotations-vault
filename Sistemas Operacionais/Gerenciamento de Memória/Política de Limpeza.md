---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.3"
---
# Política de Limpeza

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.3

---

# 🧹 3.5.3 — Política de Limpeza

## O problema

A questão da **limpeza** está relacionada ao tópico do **controle de carga** ([[Controle de Carga]]). Os algoritmos de substituição de páginas, como o **envelhecimento** ([[LRU e Envelhecimento]]), funcionam melhor quando há uma **oferta abundante de quadros de página livres**, prontos para serem requisitados quando ocorrerem faltas de página.

```
E SE TODOS os quadros de página estiverem CHEIOS e,
além disso, MODIFICADOS (sujos)?

  → antes que uma página NOVA seja trazida, uma
    página ANTIGA deve ser PRIMEIRO ESCRITA no
    armazenamento não volátil

  → a falta de página fica MAIS LENTA: além de ler a
    página nova, é preciso ESPERAR a escrita da
    página suja
```

> 💡 Compare com [[Substituição de Páginas e Algoritmo Ótimo]]: remover uma página **limpa** é só sobrescrevê-la; remover uma **suja** exige uma escrita em disco/SSD **no momento da falta**. A política de limpeza tenta evitar que esse custo caia no pior momento.

## A solução: o daemon de paginação

Para assegurar uma oferta abundante de quadros livres, os sistemas de paginação geralmente têm um **processo de segundo plano**, chamado **daemon de paginação**.

```
O DAEMON DE PAGINAÇÃO:

  → DORME a maior parte do tempo

  → é DESPERTADO periodicamente para INSPECIONAR o
    estado da memória

  SE um número MUITO PEQUENO de quadros de página
  estiver disponível:

    1. o daemon começa a SELECIONAR páginas a serem
       removidas, usando algum ALGORITMO DE
       SUBSTITUIÇÃO DE PÁGINAS

    2. SE essas páginas tiverem sido MODIFICADAS
       desde que foram carregadas, elas serão
       ESCRITAS para o armazenamento não volátil
```

## Bônus: o conteúdo anterior é lembrado

```
DE QUALQUER MANEIRA, o conteúdo ANTERIOR da página é
LEMBRADO.

SE uma das páginas removidas for NECESSÁRIA NOVAMENTE
antes de seu quadro ser SOBREPOSTO por uma nova
página:

  → ela pode ser REOBTIDA, retirando-a do CONJUNTO de
    quadros de página DISPONÍVEIS
    (sem precisar ler do disco de novo)
```

Ou seja, o quadro "livre" continua guardando o conteúdo antigo até alguém sobrescrevê-lo, e uma falta de página sobre essa página sai **barata**.

```
Por que manter uma oferta de quadros disponíveis é
MELHOR do que...

  (A) usar TODA a memória e só então, na hora da
      falta, tentar ENCONTRAR um quadro
      → pode exigir escrita em disco bem NA HORA

  (B) manter uma OFERTA de quadros disponíveis
      → resulta em MELHOR DESEMPENHO
```

> ✅ **No mínimo**, o daemon de paginação assegura que **todos os quadros disponíveis estejam limpos**, assim eles **não precisam ser escritos às pressas** para o disco quando requisitados.

---

## 🕐🕐 Implementação: relógio de dois ponteiros

Uma forma de implementar essa política de limpeza é com um **relógio de dois ponteiros** (uma variação do algoritmo do relógio, veja [[FIFO, Segunda Chance e Relógio]]).

```
              A
          L       B
        K           C
        J     ●     D        ● = centro do relógio
          I       E
            H G F

  PONTEIRO DA FRENTE → controlado pelo DAEMON
  PONTEIRO DE TRÁS   → usado na SUBSTITUIÇÃO
```

| Ponteiro | Quem controla | O que faz |
|---|---|---|
| **Da frente** | o **daemon de paginação** | aponta para página **suja** → ela é **gravada de volta** no armazenamento e o ponteiro **avança**. Aponta para página **limpa** → só **avança** |
| **De trás** | o algoritmo de **substituição** | usado para substituição de página, **como no algoritmo do relógio padrão** |

```
A DIFERENÇA em relação ao relógio comum:

  Agora, a PROBABILIDADE de o ponteiro de TRÁS atingir
  uma página LIMPA é AUMENTADA devido ao TRABALHO do
  daemon de paginação.

  → o ponteiro da frente "vai na frente" limpando
    páginas sujas
  → quando o ponteiro de trás passa por elas, já
    estão limpas e podem ser reaproveitadas na hora,
    SEM esperar escrita em disco
```

---

# ✅ Resumo do Conceito

- **Problema:** se todos os quadros estão cheios e sujos, cada falta de página precisa **primeiro gravar** uma página antiga no disco, o que a deixa lenta. Algoritmos como o envelhecimento funcionam melhor com **quadros livres em abundância**
- **Solução:** o **daemon de paginação**, processo de segundo plano que dorme quase o tempo todo, acorda periodicamente e, se faltarem quadros livres, **seleciona páginas pelo algoritmo de substituição** e **grava as modificadas** no armazenamento não volátil
- O conteúdo da página removida é **lembrado**: se ela for necessária de novo antes de o quadro ser sobreposto, é **reobtida do conjunto de quadros disponíveis**, sem ir ao disco
- **No mínimo**, o daemon garante que os quadros livres estejam **limpos**, evitando escritas às pressas
- **Implementação:** **relógio de dois ponteiros**. O **da frente** (daemon) grava as páginas sujas e avança; o **de trás** faz a substituição como no relógio comum, mas agora **encontra páginas limpas com mais probabilidade**

---

## 🔗 Notas Relacionadas

- [[Controle de Carga]] — o tópico ao qual a limpeza está relacionada (3.5.2)
- [[FIFO, Segunda Chance e Relógio]] — o algoritmo do relógio que ganha o segundo ponteiro
- [[LRU e Envelhecimento]] — o algoritmo que funciona melhor com muitos quadros livres
- [[WSClock]] — também agenda escritas de páginas sujas para encontrar páginas limpas
- [[Substituição de Páginas e Algoritmo Ótimo]] — diferença de custo entre remover página limpa e suja
- [[Tabelas de Páginas]] — o bit M (modificada) que distingue páginas sujas das limpas
