---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.1 (continuação)"
---
# Alocação de Quadros e PFF

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.1 (continuação)

> 💡 Esta nota continua [[Alocação Local versus Global]]. Se a política é **global**, o sistema precisa **decidir quantos quadros de página dar a cada processo**, e isso muda ao longo do tempo. Aqui estão as estratégias do livro para essa decisão.

---

# 🧮 O problema: quantos quadros por processo?

```
Com uma política GLOBAL, o sistema tem de decidir
CONTINUAMENTE quantos quadros de página designar
para cada processo.
```

## Tentativa 1 — monitorar o conjunto de trabalho via bits de envelhecimento

```
Uma maneira: monitorar o tamanho do conjunto de
trabalho de cada processo, como indicado pelos BITS
DE ENVELHECIMENTO (ver [[LRU e Envelhecimento]]).

MAS essa abordagem NÃO EVITA necessariamente o
THRASHING:

  → o conjunto de trabalho pode MUDAR DE TAMANHO EM
    MILISSEGUNDOS

  → os bits de envelhecimento são uma medida MUITO
    RUDIMENTAR, baseada no número de INTERRUPÇÕES DE
    RELÓGIO
```

## Tentativa 2 — repartir os quadros por um algoritmo de alocação

### Divisão igual

```
Determinar PERIODICAMENTE o número de processos em
execução e alocar uma PORÇÃO IGUAL a cada um.

EXEMPLO DO LIVRO:
  12.416 quadros de página disponíveis
  (isto é, sem contar o sistema operacional)
  10 processos

  → cada processo recebe 1.241 quadros
  → os 6 restantes vão para uma ÁREA COMUM, usada
    quando ocorrer falta de página

  (12.416 ÷ 10 = 1.241 com resto 6)
```

> ⚠️ **Parece justo, mas faz pouco sentido:** conceder porções **iguais** de memória a um processo de **10 KB** e a um de **300 KB** é um desperdício para um e uma falta para o outro.

### Divisão proporcional ao tamanho

```
Em vez disso, as páginas podem ser alocadas em
PROPORÇÃO ao TAMANHO TOTAL de cada processo:

  → um processo de 300 KB recebe 30 VEZES a
    quantidade alocada a um de 10 KB
```

### Mas é preciso garantir um MÍNIMO

```
Parece razoável dar a cada processo um número
MÍNIMO de quadros, de maneira que ele possa ser
EXECUTADO, por menor que seja.

POR QUÊ? Em algumas máquinas, uma ÚNICA instrução de
DOIS operandos pode precisar de até SEIS páginas:

  → a própria INSTRUÇÃO
  → o operando de ORIGEM
  → o operando de DESTINO

  cada um deles pode EXTRAPOLAR os limites de uma
  página, ocupando duas

COM uma alocação de apenas CINCO páginas, programas
contendo tais instruções NEM SEQUER PODERÃO ser
executados.
```

### Em algoritmos globais: alocação inicial + ajuste dinâmico

```
Se um algoritmo GLOBAL é usado, talvez seja possível
COMEÇAR cada processo com um número de páginas
PROPORCIONAL ao seu tamanho.

MAS a alocação precisa ser ATUALIZADA DINAMICAMENTE
à medida que o processo é executado.
```

---

# 📈 O algoritmo PFF (Page Fault Frequency)

Uma maneira de gerenciar a alocação dinamicamente é o algoritmo **PFF** (*page fault frequency* — **frequência de faltas de página**).

> 💡 **O que o PFF faz e o que NÃO faz:**
>
> - ✅ Diz **quando aumentar ou diminuir** a alocação de páginas de um processo
> - ❌ **Não diz nada** sobre **qual página substituir** em uma falta
>
> Ele apenas **controla o tamanho do conjunto de alocação**.

## A premissa

```
Para uma grande classe de algoritmos de substituição
(incluindo o LRU), SABE-SE que a taxa de faltas
DIMINUI à medida que MAIS páginas são designadas.

Esse é o PRESSUPOSTO por trás do PFF.
```

## 📌 Figura 3.23 — Faltas de página × quadros designados

```
 Faltas de
 página/segundo
   │
   │\
   │ \
 A ┼- -\- - - - - - - - - - - - - - - - -   ← limite SUPERIOR
   │    '.
   │      '-.
 B ┼- - - - - '--._ - - - - - - - - - - - -   ← limite INFERIOR
   │                '--.__
   │                      '-----
   └──────────────────────────────────────
        Número de quadros de página designados
```

```
LINHA TRACEJADA A (taxa INACEITAVELMENTE ALTA):
  → o processo gera faltas demais
  → ele RECEBE MAIS quadros de página, para REDUZIR a
    frequência de faltas

LINHA TRACEJADA B (taxa TÃO BAIXA que se presume
que o processo tem memória DEMAIS):
  → quadros de página PODEM SER RETIRADOS dele

CONCLUSÃO:
  O PFF tenta MANTER a frequência de paginação de
  cada processo DENTRO DE LIMITES ACEITÁVEIS
  (entre A e B).
```

| Frequência de faltas | Situação | Ação do PFF |
|---|---|---|
| **Acima de A** | processo "sufocado" (falta memória) | **dar mais quadros** |
| **Entre A e B** | região aceitável | **manter** |
| **Abaixo de B** | processo com memória sobrando | **retirar quadros** |

## Como medir a frequência de faltas

```
A medição é SIMPLES: apenas CONTE o número de faltas
POR SEGUNDO, possivelmente tomando uma MÉDIA
DESLIZANTE do número de faltas nos últimos segundos
também.

UMA MANEIRA FÁCIL:
  → somar o número de faltas durante o SEGUNDO
    IMEDIATAMENTE ANTERIOR à média de execução atual
    e DIVIDIR POR DOIS.
```

---

# ✅ Resumo do Conceito

- Com política **global**, o sistema precisa decidir **dinamicamente** quantos quadros dar a cada processo
- Monitorar o conjunto de trabalho pelos **bits de envelhecimento** é impreciso: o conjunto muda em milissegundos, e os bits são rudimentares (baseados em interrupções de relógio), então não evita o thrashing
- **Divisão igual** (ex.: 12.416 quadros ÷ 10 processos = 1.241 cada, 6 sobram para uma área comum) é injusta para processos de tamanhos diferentes; **divisão proporcional ao tamanho** é mais razoável (300 KB recebe 30× mais que 10 KB)
- É preciso um **mínimo** por processo: uma instrução de dois operandos pode exigir **até 6 páginas**, e com só 5 o programa nem executa
- Na alocação global, começa-se com um número proporcional ao tamanho e **ajusta-se dinamicamente**
- **PFF:** mede a **frequência de faltas de página** de cada processo. **Acima do limite A**, dá-se mais quadros; **abaixo do limite B**, tiram-se quadros. Ele **só controla o tamanho da alocação**, não escolhe qual página substituir. Baseia-se no fato de que as faltas **diminuem** quando há mais quadros (Figura 3.23)

---

## 🔗 Notas Relacionadas

- [[Alocação Local versus Global]] — a política que torna necessária essa decisão (3.5.1)
- [[Conjunto de Trabalho]] — o conceito de que o PFF e a alocação tentam se aproximar; thrashing
- [[LRU e Envelhecimento]] — os bits de envelhecimento e a propriedade de a taxa de faltas cair com mais páginas
- [[Resumo dos Algoritmos de Substituição de Páginas]] — o panorama dos algoritmos que o PFF complementa
