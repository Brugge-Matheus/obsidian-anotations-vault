---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.8"
---
# Troca de Mensagens

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.8

---

# 📨 2.4.8 — Troca de Mensagens

## Por que precisamos de algo além de semáforos e monitores

Em [[Monitores]], Tanenbaum concluiu que semáforos são de nível baixo demais (fácil de errar) e monitores são um conceito de linguagem (C não tem). Mas há um problema ainda mais fundamental que ambos compartilham:

> **Semáforos e monitores foram projetados para sistemas com memória compartilhada.**

```
Semáforos e monitores assumem:
  CPU 1 e CPU 2 acessam a MESMA memória RAM

  CPU 1 ──► [ memória compartilhada ] ◄── CPU 2
  
  Funciona em: um único computador com múltiplos cores/processos

Sistemas distribuídos:
  Máquina A ──[ rede ]──► Máquina B
  
  Cada máquina tem sua própria memória privada
  Não há memória compartilhada entre elas
  Semáforos na memória compartilhada → inaplicáveis
  Monitores → igualmente inaplicáveis
```

Além disso, **nenhuma das primitivas anteriores permite a troca de informações entre máquinas**. É necessário algo mais — um mecanismo que funcione tanto em sistemas de memória compartilhada quanto em sistemas distribuídos.

A solução é a **troca de mensagens** (*message passing*).

---

## 📬 As primitivas: `send` e `receive`

Diferente de semáforos e monitores, `send` e `receive` são **chamadas de sistema** — não construções de linguagem. Isso significa que qualquer linguagem pode usá-las, e o SO as implementa.

```c
send(destination, &message);     // envia uma mensagem para destination
receive(source, &message);       // recebe uma mensagem de source
```

> 💡 **`send`:** chamada de sistema que envia uma mensagem para um destino especificado. A mensagem é copiada do espaço de endereços do processo emissor para o destino.

> 💡 **`receive`:** chamada de sistema que recebe uma mensagem de uma origem especificada. Se nenhuma mensagem estiver disponível, o receptor pode **ficar bloqueado** até que chegue uma — ou retornar imediatamente com um código de erro, dependendo da implementação.

O parâmetro `source` do `receive` pode ser `ANY` — qualquer origem — se o receptor não se importar de quem vem a mensagem.

---

## ⚠️ Questões de projeto — os problemas que surgem

Sistemas de troca de mensagens têm muitos problemas de projeto que não surgem com semáforos ou monitores, especialmente quando os processos estão em máquinas diferentes conectadas por rede.

### 1. Mensagens perdidas — confiabilidade

```
Máquina A envia mensagem ──[ rede ]──► Máquina B

Problema: a rede pode perder a mensagem.
Solução: confirmação de recebimento (acknowledgement)

  A envia mensagem ──────────────────────────────► B recebe
                    ◄─── B envia confirmação ──── B
  A recebe confirmação → mensagem chegou ✅

  A envia mensagem ──────────────────────────────► mensagem perdida!
  A não recebe confirmação em X segundos → retransmite
```

> 💡 **Confirmação de recebimento (*acknowledgement*):** mensagem especial enviada pelo receptor de volta ao emissor, confirmando que a mensagem original chegou. Se o emissor não receber a confirmação em determinado intervalo, ele retransmite.

### 2. Mensagens duplicadas — números de sequência

Mas há um problema com a retransmissão:

```
Cenário:
  A envia mensagem ──────────────────────────────► B recebe ✅
  B envia confirmação ────── PERDIDA NA REDE ────► nunca chega em A
  A não recebe confirmação → retransmite
  B recebe a mesma mensagem DUAS VEZES ❌

Solução: números de sequência
  A coloca número crescente em cada mensagem (1, 2, 3...)
  B recebe mensagem com mesmo número da anterior → é duplicata → ignora
```

> 💡 **Número de sequência:** número inteiro crescente colocado em cada mensagem pelo emissor. Permite que o receptor identifique mensagens duplicadas (retransmissões) e as descarte.

### 3. Autenticação

```
Problema:
  Cliente conecta ao "servidor de arquivos"
  Mas como saber se é o servidor real e não um impostor?
  
Solução: autenticação criptográfica
  → o cliente verifica a identidade do servidor
    antes de enviar dados sensíveis
```

### 4. Desempenho

```
Problema:
  Copiar uma mensagem de um processo para outro é sempre
  mais lento do que uma operação de semáforo ou entrar
  num monitor.
  
  Muito trabalho foi dispendido para que a transmissão
  de mensagens seja eficiente — especialmente quando
  emissor e receptor estão na mesma máquina.
```

---

## 📮 Endereçamento — como as mensagens chegam ao destino

Há duas abordagens principais para endereçar mensagens:

### Endereçamento por processo

```
Cada processo tem um endereço único.
Mensagens são endereçadas diretamente ao processo.

send(processo_B, &msg)  → vai diretamente para o processo B
```

Simples, mas rígido — o emissor precisa saber exatamente qual processo vai receber.

### Mailbox (caixa postal)

> 💡 **Mailbox (caixa postal):** estrutura de dados que armazena um número fixo de mensagens, funcionando como uma fila. Processos enviam para a mailbox e recebem da mailbox — não diretamente uns dos outros.

```
Processo A ──► [ mailbox ] ◄── Processo B lê daqui
Processo C ──►             
(múltiplos emissores podem escrever na mesma mailbox)

Vantagem: desacoplamento — o emissor não precisa saber
          quem vai ler, e o receptor não precisa saber
          quem enviou

Comportamento quando mailbox está cheia:
  → processo emissor é suspenso até que uma mensagem
    seja removida da mailbox, abrindo espaço
```

Para o problema produtor-consumidor com mailboxes: tanto produtor quanto consumidor criariam mailboxes grandes o suficiente para conter N mensagens. O produtor enviaria mensagens contendo dados reais para a mailbox do consumidor, e o consumidor enviaria mensagens vazias para a mailbox do produtor. A mailbox de destino armazena mensagens que foram enviadas para o processo de destino, mas que ainda não foram aceitas.

### Rendezvous — sem buffer

O extremo oposto de mailboxes com buffer é **eliminar todo o armazenamento**:

> 💡 **Rendezvous (encontro):** estratégia de troca de mensagens sem buffer. Se `send` é realizado antes do `receive`, o emissor é **bloqueado** até que o receptor faça `receive`. Se `receive` é feito primeiro, o receptor é **bloqueado** até que o emissor faça `send`. Os dois processos se "encontram" para transferir a mensagem diretamente.

```
Com rendezvous:
  A chama send() → bloqueia esperando B chamar receive()
  B chama receive() → encontro! mensagem copiada diretamente
  Ambos continuam ✅

Vantagem:  mais simples, sem gerenciamento de buffer
Desvantagem: menos flexível, emissor e receptor
             precisam estar prontos ao mesmo tempo
```

---

## 🏭 Produtor-consumidor com troca de mensagens

Sem memória compartilhada nenhuma — usando apenas `send` e `receive`:

> 📌 **Figura 2.37 — O problema do produtor-consumidor com N mensagens**

```c
#define N 100   /* número de posições no buffer */

void producer(void)
{
    int item;
    message m;              /* buffer de mensagens */

    while (TRUE) {
        item = produce_item();      /* gera algo para colocar no buffer */
        receive(consumer, &m);      /* espera que um processo chegue vazio */
        build_message(&m, item);    /* monta uma mensagem para enviar */
        send(consumer, &m);         /* envia item ao consumidor */
    }
}

void consumer(void)
{
    int item, i;
    message m;

    for (i = 0; i < N; i++)
        send(producer, &m);         /* envia N mensagens vazias */

    while (TRUE) {
        receive(producer, &m);      /* pega mensagem contendo item */
        item = extract_item(&m);    /* extrai o item da mensagem */
        send(producer, &m);         /* envia uma mensagem vazia como resposta */
        consume_item(item);         /* faz algo com o item */
    }
}
```

### Como funciona — a lógica das mensagens vazias

```
INICIALIZAÇÃO:
  Consumidor envia N mensagens VAZIAS para o produtor
  → essas N mensagens representam as N vagas do buffer
  → é como o produtor ter N "fichas" de permissão para produzir

PRODUTOR quer inserir item:
  receive(consumer, &m)    → pega uma mensagem vazia (uma "ficha")
  Se não há mensagens vazias → BLOQUEIA (buffer cheio — sem fichas)
  Se há → build_message e send(consumer, &m)  → envia item cheio

CONSUMIDOR recebe item:
  receive(producer, &m)    → pega mensagem cheia (o item)
  extract_item             → retira o item
  send(producer, &m)       → devolve a mensagem como vazia (devolve a "ficha")
  consume_item             → usa o item

RESULTADO:
  Total de mensagens no sistema = N (constante)
  Produtor mais rápido: todas mensagens ficam cheias → ele bloqueia
  Consumidor mais rápido: todas mensagens ficam vazias → ele bloqueia
```

> 💡 **A chave desta solução:** o número total de mensagens no sistema permanece constante — N — e elas podem ser armazenadas em um montante de memória previamente conhecido. Não há memória compartilhada, não há semáforos, não há risco de race condition. Tudo é coordenado pela troca de mensagens.

---

## 🌐 MPI — Message Passing Interface

```
MPI (Message Passing Interface):
  → padrão de troca de mensagens para computação científica paralela
  → amplamente usado em supercomputadores e clusters
  → permite que programas rodem em centenas ou milhares de nós
    coordenados via troca de mensagens
  → base de simulações científicas, weather forecasting, etc.
```

---

## Comparação final — todas as primitivas de sincronização

```
┌──────────────────┬────────────┬──────────────┬──────────────┬──────────────┐
│                  │ Semáforos  │  Monitores   │   Mutexes    │    Troca de  │
│                  │            │              │              │  Mensagens   │
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Tipo             │ Syscall/   │ Construção   │ Biblioteca/  │ Syscall      │
│                  │ biblioteca │ de linguagem │ Syscall      │              │
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Mem. compartilh. │ Necessária │ Necessária   │ Necessária   │ NÃO necessár.│
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Funciona em rede │ ❌         │ ❌           │ ❌           │ ✅           │
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Facilidade de uso│ Baixa      │ Alta         │ Média        │ Média        │
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Risco de erro    │ Alto       │ Baixo        │ Médio        │ Médio        │
├──────────────────┼────────────┼──────────────┼──────────────┼──────────────┤
│ Desempenho       │ Alto       │ Alto         │ Alto         │ Menor        │
│                  │            │              │              │ (cópia de msg│
└──────────────────┴────────────┴──────────────┴──────────────┴──────────────┘
```

---

# ✅ Resumo do Conceito

- Motivação: semáforos e monitores precisam de memória compartilhada — inaplicáveis em sistemas distribuídos onde cada máquina tem sua própria memória
- Primitivas `send`/`receive` são **syscalls**, não construções de linguagem — funcionam em qualquer contexto e qualquer linguagem
- Questões de projeto exclusivas da troca de mensagens em rede: mensagens perdidas (confirmação de recebimento), duplicatas (números de sequência), autenticação e desempenho
- Endereçamento direto por processo ou via **mailbox** (desacoplada, múltiplos emissores/receptores)
- **Rendezvous:** sem buffer — emissor e receptor bloqueiam até o encontro; simples mas menos flexível
- Produtor-consumidor sem memória compartilhada: N mensagens circulam entre produtor e consumidor — mensagens vazias funcionam como "fichas de permissão", total no sistema é sempre constante
- **MPI** é o padrão industrial de troca de mensagens para computação científica paralela em clusters

---

## 🔗 Notas Relacionadas

- [[Monitores]] — limitações de semáforos e monitores que motivam a troca de mensagens
- [[Semáforos]] — comparação: semáforos precisam de memória compartilhada, mensagens não
- [[Dormir e Despertar]] — `receive` sem mensagem disponível bloqueia o processo, assim como `sleep`
- [[Race Condition]] — troca de mensagens elimina races ao eliminar memória compartilhada completamente
- [[Barreiras]] — próximo mecanismo de sincronização, para grupos de processos em fases
