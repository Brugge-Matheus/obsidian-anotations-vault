---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.7"
---
# Monitores

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.7

---

# 🖥️ 2.4.7 — Monitores

## Por que semáforos e mutexes ainda são problemáticos

Em [[Semáforos]] e [[Mutexes]], resolvemos o problema técnico da sincronização. Mas Tanenbaum aponta um problema que persiste: **é muito fácil cometer erros**.

Observe o problema da ordem dos `down` que vimos em semáforos. Se o programador inverter a ordem:

```c
// CERTO
down(&empty);   // verifica recurso primeiro
down(&mutex);   // só então entra na RC

// ERRADO — deadlock garantido quando buffer cheio
down(&mutex);   // entra na RC primeiro
down(&empty);   // tenta verificar recurso DENTRO da RC → dorme com mutex travado
```

Uma inversão de duas linhas e o sistema trava para sempre. Sem aviso, sem erro de compilação — apenas um deadlock silencioso em produção.

> ⚠️ **Usar semáforos é como programar em linguagem de montagem, mas pior** — os erros são condições de corrida, impasses e outras formas de comportamento imprevisível e irreproduzível. Um erro ínfimo e tudo para completamente.

Em 1973 e 1974, Brinch Hansen e Hoare propuseram uma solução mais segura: **passar a responsabilidade da sincronização para o compilador**, não para o programador.

---

## 🏛️ O que é um Monitor

> 💡 **Monitor:** construção de **linguagem de programação** (não uma syscall, não uma biblioteca) que agrupa procedimentos, variáveis e estruturas de dados relacionados em um módulo especial. O compilador garante automaticamente que **apenas um processo pode estar ativo dentro do monitor em qualquer instante**.

A ideia central é simples: em vez de o programador espalhar `down`/`up` ou `lock`/`unlock` pelo código e torcer para não errar a ordem, ele **declara** que um conjunto de procedimentos pertence a um monitor — e o compilador cuida da exclusão mútua automaticamente.

> 📌 **Figura 2.34 — Um monitor (em Pidgin Pascal)**

```pascal
monitor example
    integer i;
    condition c;

    procedure producer( );
    .
    .
    end;

    procedure consumer( );
    .
    .
    end;

end monitor;
```

**O que isso significa na prática:**

```
Processo A chama ProducerConsumer.insert(item):
  Compilador verifica: há outro processo ativo no monitor?
    NÃO → processo A entra e executa ✅
    SIM → processo A é SUSPENSO até o outro sair ⏳

Processo B termina de executar no monitor:
  Compilador libera o monitor
  Processo A (que estava suspenso) pode entrar agora
```

O programador não escreve nenhum `lock` ou `down` — o compilador insere esse código automaticamente em cada chamada de procedimento do monitor. Como o compilador arranja a exclusão mútua, é muito menos provável que algo dê errado.

---

## 🔑 A analogia para entender monitores

Pense no monitor como uma **sala de reunião com uma regra rígida:**

```
Regra: apenas UMA pessoa pode estar na sala por vez.

Procedimentos do monitor = coisas que se fazem NA sala
Variáveis do monitor    = objetos que ficam NA sala (quadro, mesa)
Processos               = pessoas que querem usar a sala

Pessoa A entra na sala (chama um procedimento do monitor)
  → porta trava automaticamente

Pessoa B tenta entrar
  → porta está travada → B espera do lado de fora

Pessoa A sai da sala
  → porta destrava automaticamente
  → B pode entrar agora
```

Ninguém precisa lembrar de travar e destravar a porta — ela funciona automaticamente.

---

## ⏸️ O problema dentro do monitor — variáveis de condição

A exclusão mútua automática resolve metade do problema. Mas e quando um processo **entra no monitor e descobre que não pode prosseguir**?

Exemplo: o produtor entra no monitor e descobre que o buffer está cheio. O que ele faz? Ele não pode simplesmente ficar parado dentro do monitor — se ficar, ninguém mais consegue entrar, e o consumidor (que poderia esvaziar o buffer) nunca executa. Deadlock.

A solução são as **variáveis de condição**, com duas operações: `wait` e `signal`.

> 💡 **Variável de condição:** variável especial dentro de um monitor na qual um processo pode se bloquear quando a condição que precisa não é satisfeita. Ao contrário de semáforos, **não acumula sinais** — se ninguém está esperando quando `signal` é chamado, o sinal é simplesmente perdido.

### Operação `wait`

```
processo chama wait(c):
  → processo é BLOQUEADO e colocado na fila da condição c
  → monitor é LIBERADO automaticamente
     (outro processo pode entrar agora!)
  → processo fica dormindo até outro chamar signal(c)
```

### Operação `signal`

```
processo chama signal(c):
  → SE há processo dormindo em c: acorda UM deles
  → SE não há ninguém dormindo: sinal é PERDIDO (não acumula)
```

**A diferença crucial de `up` do semáforo:**

```
Semáforo up()    → incrementa contador, crédito guardado para depois ✅
Monitor signal() → acorda alguém OU o sinal some, sem contador       ⚠️
```

> ⚠️ **Variáveis de condição de monitor não têm memória.** Um `signal` sem ninguém esperando é descartado. Por isso um processo deve sempre verificar a condição antes de continuar — não pode assumir que o sinal garantiu que a condição está satisfeita.

---

## 🤔 O problema do signal — duas propostas

Aqui surge uma questão delicada. Quando o processo A está no monitor e chama `signal(c)`, isso acorda o processo B. Mas agora **dois processos estão prontos para executar no monitor** — o que viola a regra de apenas um ativo por vez.

O que fazer?

**Proposta de Hoare:** o processo B (recém-acordado) executa imediatamente. O processo A (que sinalizou) é suspenso.

**Proposta de Brinch Hansen:** o processo A **deve sair do monitor imediatamente** após o `signal`. `signal` só pode aparecer como o **último comando** de um procedimento de monitor.

```
Hoare:    signal() → B executa agora → A espera → A retoma depois
Brinch:   signal() → A sai do monitor → B executa → mais simples!
```

Tanenbaum adota a proposta de **Brinch Hansen** por ser conceitualmente mais simples e mais fácil de implementar. Se um sinal for realizado em uma variável de condição em que vários processos estejam esperando, apenas um deles (determinado pelo escalonador) será reativado.

> 💡 Há também uma terceira opção: deixar o sinalizador continuar a executar e o processo em espera começar a ser executado apenas depois de o sinalizador ter deixado o monitor. Esta solução não é proposta nem por Hoare nem por Brinch Hansen.

---

## 🏭 Produtor-consumidor com monitores (Figura 2.35)

Veja como o código fica **muito mais limpo e seguro** com monitores:

> 📌 **Figura 2.35 — Esqueleto do produtor-consumidor com monitores em Pidgin Pascal**

```pascal
monitor ProducerConsumer
    condition full, empty;   { variáveis de condição }
    integer count;           { itens no buffer }

    procedure insert(item: integer);
    begin
        if count = N then wait(full);    { buffer cheio → dorme }
        insert_item(item);
        count := count + 1;
        if count = 1 then signal(empty) { tinha 0 itens → acorda consumidor }
    end;

    function remove: integer;
    begin
        if count = 0 then wait(empty);  { buffer vazio → dorme }
        remove := remove_item;
        count := count - 1;
        if count = N-1 then signal(full) { tinha N itens → acorda produtor }
    end;

    count := 0;   { inicialização }
end monitor;

procedure producer;
begin
    while true do
    begin
        item := produce_item;
        ProducerConsumer.insert(item)  { chama procedimento do monitor }
    end
end;

procedure consumer;
begin
    while true do
    begin
        item := ProducerConsumer.remove;  { chama procedimento do monitor }
        consume_item(item)
    end
end;
```

**Compare com a versão de semáforos:** não há `down(&mutex)`, `down(&empty)`, `up(&mutex)`, `up(&full)` espalhados pelo código do produtor e consumidor. O produtor simplesmente chama `ProducerConsumer.insert(item)` — a sincronização é invisível para ele.

### Por que monitores não têm a race condition do sleep/wakeup?

Em [[Dormir e Despertar]], o problema era:

```
Consumidor lê count=0
← interrupção →
Produtor insere item, chama wakeup → PERDIDO (consumidor ainda acordado)
Consumidor executa sleep → dorme para sempre
```

Com monitores isso **não pode acontecer**:

```
Consumidor entra no monitor, lê count=0
  → chama wait(empty) → dorme E libera o monitor ATOMICAMENTE

Produtor entra no monitor (agora pode, consumidor liberou)
  → insere item, count=1
  → chama signal(empty) → consumidor acorda
```

A exclusão mútua automática do monitor garante que o produtor **não pode entrar no monitor enquanto o consumidor ainda está dentro** — e portanto não pode chamar `signal` antes do `wait`. A janela de race condition é eliminada estruturalmente.

---

## ☕ Monitores em Java — `synchronized`

Apesar de Pidgin Pascal ser uma linguagem imaginária, Java implementa monitores de forma nativa com a palavra-chave `synchronized`.

```java
// Ao adicionar synchronized, Java garante que apenas uma thread
// executa este método por vez em um dado objeto
public synchronized void insert(int val) {
    if (count == N) go_to_sleep();   // buffer cheio → dorme
    buffer[hi] = val;
    hi = (hi + 1) % N;
    count++;
    if (count == 1) notify();        // tinha 0 → acorda consumidor
}

public synchronized int remove() {
    if (count == 0) go_to_sleep();   // buffer vazio → dorme
    val = buffer[lo];
    lo = (lo + 1) % N;
    count--;
    if (count == N-1) notify();      // tinha N → acorda produtor
    return val;
}
```

Em Java, `wait` e `notify` são o equivalente a `sleep` e `wakeup`, exceto que, ao serem usados dentro de métodos sincronizados, **não são sujeitos a condições de corrida**. Sem `synchronized`, não há garantias sobre a intercalação.

> ⚠️ **Diferença importante:** Java não tem variáveis de condição embutidas como Hoare/Brinch Hansen definiram. Em vez disso, oferece `wait` e `notify` — que funcionam de forma semelhante mas com algumas diferenças de semântica. `notify` acorda uma thread, `notifyAll` acorda todas.

---

## ⚖️ Limitações dos monitores

Monitores são uma solução elegante, mas têm problemas práticos:

**1. São um conceito de linguagem, não de SO**

```
Semáforos → dois procedimentos curtos em assembly → qualquer linguagem
Monitores → o compilador precisa reconhecê-los e gerar código especial
          → C não tem monitores
          → C++ não tem monitores nativos
          → Java tem (synchronized)
          → a maioria das linguagens NÃO tem
```

**2. Não funcionam em sistemas distribuídos**

Semáforos e monitores foram projetados para sistemas com **memória compartilhada** — uma ou mais CPUs acessando a mesma RAM. Em um sistema distribuído com múltiplas máquinas e memórias privadas:

```
CPU 1 (máquina A) ←── rede ──→ CPU 2 (máquina B)
  memória A                        memória B

Não há memória compartilhada → semáforos na memória compartilhada
                                 são inaplicáveis
                              → monitores igualmente inaplicáveis
```

> ⚠️ **Conclusão de Tanenbaum:** semáforos são de nível baixo demais (fácil de errar) e monitores não são utilizáveis (exceto em algumas poucas linguagens). Além disso, nenhuma das primitivas permite a **troca de informações entre máquinas**. É necessário algo mais — o que leva ao tópico 2.4.8: **troca de mensagens**.

---

# ✅ Resumo do Conceito

- **Monitor** é uma construção de **linguagem de programação** que agrupa procedimentos e dados em um módulo especial — o compilador garante automaticamente que só um processo está ativo dentro dele por vez
- O grande ganho: **o programador não precisa escrever `lock`/`unlock` ou `down`/`up`** — a exclusão mútua é invisível e automática, eliminando toda uma classe de bugs
- Dentro do monitor, processos podem precisar esperar por uma condição — usam **`wait(c)`** para dormir e liberar o monitor, e **`signal(c)`** para acordar quem está esperando
- Variáveis de condição **não têm memória** — `signal` sem ninguém esperando é perdido; diferente de semáforos que acumulam créditos
- **Proposta de Brinch Hansen** (adotada): `signal` deve ser o último comando do procedimento — o sinalizador sai, o acordado entra. Mais simples de implementar
- Monitores eliminam a race condition do [[Dormir e Despertar]] porque `wait` libera o monitor **atomicamente** — impossível receber `signal` antes de `wait`
- **Java** implementa monitores com `synchronized` + `wait`/`notify`
- **Limitações:** são conceito de linguagem (C não tem), e não funcionam em sistemas distribuídos sem memória compartilhada — o que motiva a **troca de mensagens** (2.4.8)

---

## 🔗 Notas Relacionadas

- [[Semáforos]] — a primitiva anterior que monitores tornam mais segura de usar
- [[Mutexes]] — implementação de baixo nível que o compilador usa internamente para implementar monitores
- [[Dormir e Despertar]] — a race condition que `wait`/`signal` dentro de monitores eliminam estruturalmente
- [[Regiões Críticas]] — monitores garantem as 4 condições automaticamente via compilador
