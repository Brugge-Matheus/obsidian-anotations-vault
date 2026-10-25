---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.4.9"
---
# WSClock

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.4.9

---

# 🕐⚙️ 3.4.9 — O Algoritmo de Substituição de Página WSClock

## Por que existe

O algoritmo básico do [[Conjunto de Trabalho]] é **árduo**: a tabela de páginas **inteira** precisa ser varrida a cada falta de página até que uma candidata adequada seja localizada.

> 💡 O **WSClock** (Carr e Hennessey, 1981) é uma versão melhorada: baseia-se no **algoritmo do relógio** ([[FIFO, Segunda Chance e Relógio]]), mas **também usa a informação do conjunto de trabalho**. Por sua simplicidade de implementação e bom desempenho, é **muito utilizado na prática**.

```
RELÓGIO  → estrutura circular simples + bit R
CONJUNTO DE TRABALHO → idade da página vs. τ

WSCLOCK  → estrutura circular do relógio
           + critério de idade do conjunto de trabalho
           + tratamento de páginas SUJAS (bit M)
```

## A estrutura de dados

```
Uma LISTA CIRCULAR de quadros de páginas, como no
algoritmo do relógio (Figura 3.20(a)).

  → De início, a lista está VAZIA
  → Quando a primeira página é carregada, é
    adicionada à lista
  → À medida que mais páginas são adicionadas, elas
    vão até a lista para formar um ANEL

Cada ENTRADA contém:
  → o campo "Instante do último uso" (do algoritmo
    do conjunto de trabalho básico)
  → o bit R (referenciada)
  → o bit M (modificada)
```

## 📌 Figura 3.20 — Operação do WSClock

Tempo virtual atual: **2204**. Cada quadro mostra `instante do último uso | bit R`.

```
(a) R = 1: o ponteiro aponta para 2014|1

          1620|0
   2084|1         2032|1

 2003|1                  2020|1
              ↘
   1980|1       [2014|1]
          1213|0

(b) Após tratar: a entrada recebe o tempo atual e R = 0
    (2204|0); o ponteiro avança para 1213|0

          1620|0
   2084|1         2032|1

 2003|1                  2020|1
                ↓
   1980|1       2204|0
          [1213|0]

(c) R = 0: o ponteiro está em 1213|0
    (idade 2204 − 1213 = 991, maior que τ, e página LIMPA)

(d) O quadro é reivindicado: a NOVA página entra ali
    (2204|1, sombreada) e o ponteiro avança para 1980|1
```

## Como funciona, passo a passo

Assim como no relógio, **a cada falta de página, a página apontada é examinada primeiro**.

### Caso R = 1 (Figura 3.20 (a) e (b))

```
→ a página foi usada durante a interrupção de relógio
  ATUAL → NÃO é candidata ideal à remoção

→ o bit R é colocado em 0

→ o ponteiro AVANÇA para a próxima página

→ o algoritmo é REPETIDO para aquela página
```

### Caso R = 0 (Figura 3.20 (c) e (d))

```
Calcular a IDADE (tempo virtual atual − instante do
último uso) e compará-la com τ.

  SE idade > τ  E  a página está LIMPA (M = 0):
    → ela NÃO está no conjunto de trabalho
    → há uma CÓPIA VÁLIDA no disco ou SSD
    → o quadro é simplesmente REIVINDICADO e a
      nova página é colocada ali (Figura 3.20(d))

  SE idade > τ  MAS a página está SUJA (M = 1):
    → NÃO pode ser reivindicada imediatamente,
      pois NENHUMA cópia válida está no
      armazenamento não volátil
    → para evitar uma TROCA DE PROCESSO, a ESCRITA
      EM DISCO É ESCALONADA
    → o ponteiro AVANÇA e o algoritmo continua
      com a página SEGUINTE
    → afinal, pode haver uma página VELHA e LIMPA
      mais adiante, que pode ser usada
      imediatamente
```

> 💡 **A grande vantagem sobre o algoritmo básico:** em vez de **esperar** a escrita de uma página suja terminar (o que bloquearia o processo), o WSClock **agenda** a escrita e **segue em frente** procurando uma página limpa.

## Limite de escritas

```
Em PRINCÍPIO, todas as páginas podem ser escalonadas
para E/S em disco a cada ciclo do relógio.

Para REDUZIR o tráfego de disco ou SSD, pode-se
ESTABELECER UM LIMITE:

  → no MÁXIMO n páginas podem ser reescritas
  → uma vez ATINGIDO o limite, NÃO são escalonadas
    mais escritas novas
```

## E se o ponteiro der uma volta completa?

Se o ponteiro deu uma volta completa e voltou ao ponto de partida, há **dois casos**:

```
CASO 1 — PELO MENOS UMA ESCRITA FOI ESCALONADA:

  → o ponteiro apenas CONTINUA a se mover,
    procurando por uma página LIMPA

  → como uma ou mais escritas foram escalonadas,
    EVENTUALMENTE alguma será COMPLETADA e sua
    página será marcada como LIMPA

  → a PRIMEIRA página limpa encontrada é REMOVIDA

  → essa NÃO é necessariamente a primeira escrita
    escalonada, porque o DRIVER do disco pode
    REORDENAR escritas para otimizar o desempenho
    do armazenamento não volátil

CASO 2 — NENHUMA ESCRITA FOI ESCALONADA:

  → TODAS as páginas estão no conjunto de trabalho
    (de outra maneira, pelo menos uma escrita teria
    sido escalonada)

  → por falta de informações adicionais, o mais
    simples é REIVINDICAR QUALQUER PÁGINA LIMPA e
    usá-la

  → a localização de uma página limpa pode ser
    REGISTRADA durante a varredura

  → SE NÃO EXISTIR NENHUMA, a página ATUAL é
    escolhida como VÍTIMA e será REESCRITA no
    armazenamento não volátil
```

---

# ✅ Resumo do Conceito

- O **conjunto de trabalho** básico exige varrer a tabela inteira a cada falta; o **WSClock** combina a **lista circular do relógio** com a **idade vs. τ** do conjunto de trabalho
- Cada entrada tem **instante do último uso**, bit **R** e bit **M**
- **R = 1:** a página foi usada agora → zera R (e atualiza o instante) e o ponteiro avança
- **R = 0, idade > τ, página limpa:** o quadro é reivindicado imediatamente
- **R = 0, idade > τ, página suja:** agenda a escrita em disco e **segue em frente**, sem bloquear. Pode haver um limite de n escritas agendadas
- **Volta completa:** se houve escritas agendadas, continua até uma ser concluída e usa a primeira página limpa; se nenhuma foi agendada (todas no conjunto de trabalho), reivindica qualquer página limpa, ou, se não houver, reescreve a página atual
- Simples, eficiente e **muito usado na prática**

---

## 🔗 Notas Relacionadas

- [[Conjunto de Trabalho]] — o modelo e o algoritmo básico que o WSClock melhora
- [[FIFO, Segunda Chance e Relógio]] — a estrutura circular com ponteiro herdada pelo WSClock
- [[Resumo dos Algoritmos de Substituição de Páginas]] — comparação de todos os algoritmos (3.4.10)
- [[Tabelas de Páginas]] — bits R e M
