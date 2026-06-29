---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.4.8 e 2.4.9"
---
# Troca de Mensagens e Barreiras

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.4.8 e 2.4.9

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

Essa solução é elegante porque **usa as próprias mensagens como mecanismo de controle de fluxo**, sem nenhum semáforo ou variável compartilhada:

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

## 🌐 Variações e usos práticos

### Endereçamento flexível

```
send(destination, &message):
  destination pode ser:
    → PID de um processo específico
    → nome de uma mailbox
    → ANY (qualquer processo que esteja esperando)
    
receive(source, &message):
  source pode ser:
    → PID de um processo específico
    → nome de uma mailbox
    → ANY (aceita mensagem de qualquer origem)
```

### MPI — Message Passing Interface

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

# 🚧 2.4.9 — Barreiras

## O problema que barreiras resolvem

Os mecanismos anteriores focam em dois processos interagindo — produtor/consumidor, leitor/escritor. Mas algumas aplicações têm um padrão diferente: **grupos de processos trabalhando em fases**.

```
Exemplo: simulação científica em N processos

Fase 1: cada processo calcula sua parte dos dados
Fase 2: cada processo usa os dados de TODOS os outros para calcular
Fase 3: cada processo usa os resultados de TODOS para a próxima etapa

Problema:
  Processo A termina a Fase 1 muito rápido
  Processo A quer avançar para a Fase 2
  Mas a Fase 2 usa dados do Processo D que ainda está na Fase 1!

  Se A avançar antes de D terminar → usa dados incompletos → resultado errado
```

A regra é clara: **nenhum processo deve prosseguir para a fase seguinte até que TODOS os processos estejam prontos**.

---

## 🚧 O que é uma barreira

> 💡 **Barreira (*barrier*):** primitiva de sincronização colocada no fim de cada fase. Quando um processo atinge a barreira, ele é **bloqueado** até que todos os demais processos também tenham atingido a barreira. Só então todos são liberados simultaneamente para a próxima fase.

> 📌 **Figura 2.38 — Uso de uma barreira**

```
(a) Processos aproximando-se da barreira:

  Processo A ────────────────────────────►|
  Processo B ──────────────────►          | BARREIRA
  Processo C ────────────────────────────►|
  Processo D ──────────►                  |
  
               Tempo ──────────────────►

(b) Todos bloqueados exceto D (ainda computando):

  Processo A ────────────────────────────►| BLOQUEADO
  Processo B ──────────────────►──────────| BLOQUEADO
  Processo C ────────────────────────────►| BLOQUEADO
  Processo D ──────────►─────────────────►| chegando...
  
(c) D chega — todos liberados simultaneamente:

  Processo A ────────────────────────────►|────────────►
  Processo B ──────────────────►──────────|────────────►
  Processo C ────────────────────────────►|────────────►
  Processo D ──────────►─────────────────►|────────────►
                                          
                                    todos avançam juntos ✅
```

---

## Onde barreiras são usadas na prática

```
Computação paralela e científica:
  → simulações de física, clima, dinâmica de fluidos
  → cada nó computa uma parte, barreira sincroniza antes de trocar dados

MPI (Message Passing Interface):
  → MPI_Barrier() é uma chamada padrão
  → usada extensivamente em clusters de supercomputadores

Computação em GPU (CUDA):
  → __syncthreads() é uma barreira entre threads do mesmo bloco
  → garante que todos os threads terminaram antes de continuar
  → fundamental para algoritmos paralelos corretos em GPU

Algoritmos de ordenação paralela:
  → cada thread ordena sua parte → barreira → merge das partes
```

---

# ✅ Resumo do Conceito

**Troca de Mensagens (2.4.8):**
- Motivação: semáforos e monitores precisam de memória compartilhada — inaplicáveis em sistemas distribuídos
- Primitivas `send`/`receive` são **syscalls**, não construções de linguagem — funcionam em qualquer contexto
- Questões de projeto únicas: mensagens perdidas (confirmação de recebimento), duplicatas (números de sequência), autenticação, desempenho
- Endereçamento por processo (direto) ou por **mailbox** (desacoplado, múltiplos emissores/receptores)
- **Rendezvous:** sem buffer — emissor e receptor bloqueiam até o encontro
- Produtor-consumidor sem memória compartilhada: N mensagens circulam entre produtor e consumidor — mensagens vazias são as "fichas de permissão" para o produtor; total de mensagens no sistema é constante
- **MPI** é o padrão industrial de troca de mensagens para computação científica paralela

**Barreiras (2.4.9):**
- Para grupos de processos trabalhando em **fases** — não apenas pares produtor/consumidor
- Nenhum processo avança para a próxima fase até que **todos** tenham chegado à barreira
- Usadas em: MPI (`MPI_Barrier`), CUDA (`__syncthreads`), algoritmos paralelos

---

## 🔗 Notas Relacionadas

- [[Monitores]] — limitações de semáforos e monitores que motivam a troca de mensagens
- [[Semáforos]] — comparação: semáforos precisam de memória compartilhada, mensagens não
- [[Dormir e Despertar]] — `receive` sem mensagem disponível bloqueia o processo, assim como `sleep`
- [[Race Condition]] — troca de mensagens elimina races ao eliminar memória compartilhada completamente
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — servidores orientados a eventos usam conceitos similares de mensagens e eventos
