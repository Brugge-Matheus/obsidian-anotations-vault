---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.4"
---
# Dormir e Despertar

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.4

---

# 😴 2.4.4 — Dormir e Despertar

## O problema que motivou essa seção

Em [[Exclusão mútua com espera ocupada]], vimos que a solução de Peterson e as instruções TSL/XCHG são **corretas**, mas compartilham um defeito fundamental: a **espera ocupada** (*busy waiting*).

Quando um processo não pode entrar em sua região crítica, ele fica em um laço apertado testando a condição continuamente — desperdiçando 100% dos ciclos de CPU sem fazer nenhum trabalho útil.

Mas há um problema ainda mais grave que a simples perda de CPU:

### O problema da inversão de prioridade

Considere um computador com dois processos:
- **Processo H** — alta prioridade
- **Processo L** — baixa prioridade

As regras de escalonamento garantem que H executa sempre que estiver no estado pronto.

```
Situação:
  L está em sua região crítica (executando)
         │
         ▼
  H torna-se pronto (completa uma operação de E/S)
         │
         ▼
  H começa a executar (tem prioridade maior) ✅
         │
         ▼
  H tenta entrar na região crítica → já ocupada por L
         │
         ▼
  H entra em espera ocupada 🔄
         │
         ▼
  L nunca é escalonado enquanto H estiver executando ❌
         │
         ▼
  L nunca termina sua região crítica
         │
         ▼
  H espera para sempre → sistema travado
```

> 💡 **Problema da inversão de prioridade:** situação em que um processo de alta prioridade fica bloqueado *indiretamente* por um processo de baixa prioridade que detém um recurso que o de alta prioridade precisa. Com espera ocupada, o processo de alta prioridade monopoliza a CPU fazendo "nada", enquanto o de baixa prioridade nunca recebe tempo para liberar o recurso.

Essa situação é referida como uma variante do **problema da inversão de prioridade** e é um laço infinito de fato.

---

## A solução — primitivas de bloqueio: `sleep` e `wakeup`

A ideia é simples: em vez de ficar girando em falso, o processo que não pode prosseguir deve **bloquear** — ceder a CPU e dormir até que a condição que ele espera se torne verdadeira.

> 💡 **`sleep`:** chamada de sistema que faz com que o processo que a chamou fique **bloqueado** (suspenso), cedendo a CPU, até que outro processo o desperte. O processo para de executar e sai da fila de prontos.

> 💡 **`wakeup`:** chamada de sistema que **acorda** um processo específico que está dormindo. Recebe como parâmetro o endereço de memória que identifica qual sleep deve ser despertado — o mesmo endereço usado na chamada `sleep` correspondente.

```
Processo não pode entrar na região crítica:
  sleep()  → bloqueia, cede a CPU, não consome mais ciclos

Outro processo sai da região crítica:
  wakeup(endereço) → acorda o processo que estava dormindo
  
O processo acordado:
  → volta para a fila de prontos
  → eventualmente executa novamente
  → tenta entrar na região crítica
```

Tanto `sleep` quanto `wakeup` recebem um parâmetro — um **endereço de memória** — usado para igualar sleeps a wakeups. Esse endereço é o "canal" pelo qual produtor e consumidor se comunicam sobre a disponibilidade de itens.

---

## 🏭 O problema do produtor-consumidor

Como exemplo de como essas primitivas podem ser usadas, Tanenbaum apresenta o problema do **produtor-consumidor**, também conhecido como problema do ***buffer* limitado**.

### Descrição do problema

Dois processos compartilham um *buffer* de tamanho fixo N:
- **Produtor:** insere itens no *buffer*
- **Consumidor:** retira itens do *buffer*

```
              ┌─────────────────────────────┐
              │        Buffer (N slots)     │
Produtor ──►  │ [ ][ ][ ][x][x][x][ ][ ]  │  ──► Consumidor
  insere      └─────────────────────────────┘       retira
              
              count = número de itens no buffer
```

**Os dois problemas que surgem:**

1. **Buffer cheio:** o produtor quer inserir um item, mas o buffer já tem N itens → produtor deve dormir até o consumidor retirar algo
2. **Buffer vazio:** o consumidor quer retirar um item, mas o buffer está vazio → consumidor deve dormir até o produtor inserir algo

### O código com sleep e wakeup — Figura 2.27

> 📌 **Figura 2.27 — O problema do produtor-consumidor com uma condição de corrida fatal**

```c
#define N 100              /* número de lugares no buffer */
int count = 0;             /* número de itens no buffer */

void producer(void)
{
    int item;
    while (TRUE) {
        item = produce_item();         /* gera o próximo item */
        if (count == N) sleep();       /* se o buffer estiver cheio, vai dormir */
        insert_item(item);             /* põe um item no buffer */
        count = count + 1;            /* incrementa o contador de itens no buffer */
        if (count == 1) wakeup(consumer); /* o buffer estava vazio? */
    }
}

void consumer(void)
{
    int item;
    while (TRUE) {
        if (count == 0) sleep();       /* se o buffer estiver vazio, vai dormir */
        item = remove_item();          /* retira o item do buffer */
        count = count - 1;            /* decrementa contador de itens no buffer */
        if (count == N-1) wakeup(producer); /* o buffer estava cheio? */
        consume_item(item);           /* imprime o item */
    }
}
```

**A lógica dos wakeups:**
- O produtor chama `wakeup(consumer)` quando `count` passa de 0 para 1 — sinal de que o buffer que estava vazio agora tem algo
- O consumidor chama `wakeup(producer)` quando `count` passa de N para N-1 — sinal de que o buffer que estava cheio agora tem espaço

---

## ⚠️ A race condition fatal

Esse código parece razoável, mas contém uma **race condition fatal**. O acesso a `count` é irrestrito — nenhum mecanismo de exclusão mútua protege sua leitura e escrita.

### O cenário da falha — passo a passo

```
Estado inicial: buffer VAZIO, count = 0

CONSUMIDOR                              PRODUTOR
──────────────────────────────────────────────────────────────────

1. Lê count → vê 0
   
   ← ESCALONADOR PAUSA O CONSUMIDOR →
   ← PRODUTOR COMEÇA A EXECUTAR →

                         2. Insere um item no buffer
                         3. count = count + 1 → count = 1
                         4. Verifica: count == 1? SIM
                            → chama wakeup(consumer)
                            
                            MAS: consumidor ainda não está dormindo!
                            O sinal de wakeup é PERDIDO. ❌

   ← CONSUMIDOR RETOMA →

5. Consumidor continua de onde parou
   Executa sleep() — dorme agora
   (baseado no count = 0 que leu antes)

                         6. Produtor continua produzindo
                            Buffer vai enchendo...
                            count chega a N
                         7. Produtor: count == N → sleep()
                            PRODUTOR DORME TAMBÉM.

RESULTADO:
  Ambos dormindo para sempre.
  Produtor esperando espaço no buffer.
  Consumidor esperando item no buffer.
  ══► DEADLOCK ◄══
```

> 💡 **A essência do problema:** um sinal de `wakeup` enviado para um processo que **ainda não está dormindo é perdido**. O produtor viu `count = 1` e chamou `wakeup` — mas o consumidor ainda não havia executado o `sleep`. Quando o consumidor finalmente executa o `sleep`, ninguém mais vai acordá-lo.

Esta é exatamente a mesma categoria de race condition que vimos no [[Race Condition|spool de impressão]]: uma operação de verificação-e-ação que deveria ser atômica, mas não é.

---

## 🩹 A solução paliativa — *bit* de espera pelo sinal de despertar

Uma solução rápida para esse caso específico seria modificar as regras para acrescentar ao quadro um **bit de espera pelo sinal de despertar** (*wakeup waiting bit*).

```
Quando wakeup é enviado para um processo que ainda está ACORDADO:
  → o bit de espera é marcado como 1
  
Quando o processo tenta dormir (sleep):
  → verifica o bit de espera
  → se o bit é 1: limpa o bit, permanece ACORDADO (não dorme)
  → se o bit é 0: dorme normalmente
```

O consumidor limpa o *bit* de espera pelo sinal de despertar a cada iteração do laço.

### Por que isso não é uma solução geral

Embora o *bit* de espera salve a situação nesse exemplo simples com dois processos, **não escala**:

```
Com 3 processos: precisaríamos de 1 bit de espera? Não basta.
Com N processos: precisaríamos de múltiplos bits de espera.
                 Em princípio, o problema ainda existe.
```

Poderíamos acrescentar um segundo *bit* de espera, ou talvez 8 ou 32 deles — mas em princípio o problema permanece. Essa é uma solução ad hoc, não uma solução estruturada.

---

## 📊 Comparação: espera ocupada vs. sleep/wakeup

```
┌─────────────────────────┬────────────────────────┬──────────────────────────┐
│      Característica     │    Espera Ocupada      │      Sleep/Wakeup        │
├─────────────────────────┼────────────────────────┼──────────────────────────┤
│ Uso de CPU enquanto     │ 100% (girando em       │ 0% (processo bloqueado,  │
│ aguarda                 │ falso)                 │ cedeu a CPU)             │
├─────────────────────────┼────────────────────────┼──────────────────────────┤
│ Inversão de prioridade  │ Sim — H bloqueia L     │ Não — H dorme,          │
│                         │ indefinidamente        │ L pode executar          │
├─────────────────────────┼────────────────────────┼──────────────────────────┤
│ Race condition           │ Não (se implementado   │ Sim — wakeup antes do   │
│ própria                 │ corretamente)          │ sleep é perdido          │
├─────────────────────────┼────────────────────────┼──────────────────────────┤
│ Quando usar             │ Esperas muito curtas   │ Esperas longas ou        │
│                         │ (spin locks no kernel) │ indeterminadas           │
└─────────────────────────┴────────────────────────┴──────────────────────────┘
```

---

## 🚦 O caminho para semáforos

O problema do sinal de wakeup perdido revelou que precisamos de uma solução mais robusta — uma que **não perca sinais** e **não exija espera ocupada**.

Em 1965, E. W. Dijkstra propôs exatamente isso: usar uma variável inteira para **contar** o número de sinais de despertar salvos para uso futuro. Essa variável foi chamada de **semáforo** — o tema da seção 2.4.5 ([[Semáforos]]).

> 💡 A ideia central dos semáforos: em vez de perder um wakeup enviado para um processo que ainda está acordado, **acumulá-lo** em um contador. O processo pode "sacar" esse crédito depois quando precisar dormir — e se já houver crédito disponível, simplesmente não dorme.

---

# ✅ Resumo do Conceito

- A **espera ocupada** das soluções anteriores (Peterson, TSL, XCHG) desperdiça CPU e cria o **problema da inversão de prioridade** — onde um processo de alta prioridade pode travar o sistema ao monopolizar a CPU esperando por um de baixa prioridade
- As primitivas **`sleep`** e **`wakeup`** resolvem a espera ocupada: o processo que não pode prosseguir **bloqueia**, cedendo a CPU, e é **acordado** quando a condição se torna verdadeira
- O **problema do produtor-consumidor** (*buffer* limitado) é o exemplo clássico de uso dessas primitivas: produtor dorme quando buffer cheio, consumidor dorme quando buffer vazio
- Porém, `sleep`/`wakeup` introduzem uma nova race condition: um **wakeup enviado antes do sleep correspondente é perdido**, podendo causar deadlock com ambos os processos dormindo para sempre
- O **bit de espera pelo sinal de despertar** é uma solução paliativa que funciona para dois processos, mas não escala para N processos
- A solução robusta e generalizada para esse problema são os **semáforos** (seção 2.4.5), que acumulam sinais de wakeup em um contador em vez de perdê-los

---

## 🔗 Notas Relacionadas

- [[Exclusão mútua com espera ocupada]] — as soluções anteriores (Peterson, TSL, XCHG) que motivaram esta seção
- [[Race Condition]] — o problema fundamental do wakeup perdido é uma race condition sobre o estado de `count`
- [[Regiões Críticas]] — o acesso a `count` no produtor-consumidor é uma região crítica não protegida
- [[Estados de Processos]] — `sleep` move um processo para o estado bloqueado; `wakeup` o move de volta para pronto
