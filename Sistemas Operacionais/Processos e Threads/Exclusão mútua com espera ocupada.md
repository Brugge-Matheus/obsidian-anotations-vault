---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.3"
---
# Exclusão Mútua com Espera Ocupada

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.3

---

# ⚙️ 2.4.3 — Exclusão Mútua com Espera Ocupada

Em [[Regiões Críticas]], estabelecemos o *problema*: precisamos garantir que nunca dois processos estejam simultaneamente em suas regiões críticas. Agora examinaremos **propostas concretas de implementação** dessa exclusão mútua.

Todas as soluções desta seção compartilham uma característica: enquanto um processo espera para entrar em sua região crítica, ele **fica em loop verificando ativamente** se já pode entrar. Isso é chamado de **espera ocupada** (*busy waiting*).

> 💡 **Espera ocupada (*busy waiting*):** técnica em que um processo que não pode prosseguir fica em um laço apertado, verificando continuamente uma condição até que ela se torne verdadeira. Também chamada de *spin waiting*. Desperdiça ciclos de CPU — a CPU fica "girando em falso" sem fazer trabalho útil.

> 💡 **Trava giratória (*spin lock*):** uma trava que usa espera ocupada para aguardar sua liberação. Útil apenas quando a espera será muito curta — o custo de bloquear e acordar o processo seria maior que o custo de girar por poucos ciclos.

Tanenbaum apresenta quatro abordagens, cada uma com suas falhas e méritos, culminando na solução elegante de Peterson e nas instruções de hardware TSL/XCHG.

---

## 🔇 Abordagem 1 — Desabilitando Interrupções

### A ideia

Em um sistema de processador único, a solução mais simples é fazer com que cada processo **desabilite todas as interrupções** logo após entrar em sua região crítica e as reabilite um momento antes de sair dela.

```
processo entra na região crítica:
    DISABLE_INTERRUPTS()       ← nenhuma troca de contexto pode acontecer
    
    /* acessa memória compartilhada com segurança */
    /* nenhuma outra thread pode interromper aqui */
    
    ENABLE_INTERRUPTS()        ← volta ao normal
processo sai da região crítica
```

Com as interrupções desligadas, **nenhuma interrupção de relógio pode ocorrer**, e a CPU não será chaveada de processo em processo. O processo pode examinar e atualizar a memória compartilhada sem medo de interferência.

### Por que funciona (em teoria)

A CPU só troca de processo em consequência de uma interrupção de relógio ou outra. Com as interrupções desligadas, isso não pode acontecer.

### Por que é inadequada como solução geral

```
Problema 1 — Dar poder demais ao usuário:
  Processo de usuário desabilita interrupções
         │
         ▼
  Nunca mais as reabilita  ← fim do sistema operacional
  
Problema 2 — Multiprocessadores:
  CPU 1 desabilita interrupções ──► afeta apenas a CPU 1
  CPU 2 ainda pode acessar a memória compartilhada! ← race condition persiste

Problema 3 — Chips multinúcleo modernos:
  2, 4, 8, 16, 32 núcleos são comuns hoje
  Desabilitar interrupções de um núcleo não protege acesso pelos outros
```

> ⚠️ **Conclusão:** desabilitar interrupções é uma técnica válida **dentro do próprio núcleo do SO** — o núcleo frequentemente desabilita interrupções por algumas instruções enquanto atualiza variáveis ou listas especialmente críticas. Mas **não é apropriada como mecanismo de exclusão mútua geral para processos de usuário**. O núcleo não deve desabilitar interrupções por mais do que algumas instruções.

---

## 🔢 Abordagem 2 — Variáveis do Tipo Trava (*Lock Variables*)

### A ideia

Uma segunda tentativa é uma solução de *software*: usar uma única variável compartilhada (de trava), inicialmente 0.

```
lock = 0  → nenhum processo está na região crítica
lock = 1  → algum processo está na região crítica
```

**Protocolo proposto:**

```
Antes de entrar na região crítica:
    while (lock != 0) { }   ← espera ocupada até a trava estar livre
    lock = 1                ← declara que está entrando

Ao sair da região crítica:
    lock = 0                ← libera a trava
```

### O problema fatal

Essa ideia contém **exatamente a mesma falha** do diretório de spool que vimos em [[Race Condition]]:

```
PROCESSO A                              PROCESSO B
──────────────────────────────────────────────────────────────

1. Lê lock → vê 0 (livre!)
   
   ← INTERRUPÇÃO: CPU troca para B →

                                2. Lê lock → vê 0 (livre!)
                                3. Configura lock = 1
                                4. Entra na região crítica

   ← CPU volta para A →

5. Processo A retoma onde parou
   Configura lock = 1
   Entra na região crítica  ← DOIS PROCESSOS NA REGIÃO CRÍTICA!
```

O problema: a verificação de `lock` e a configuração de `lock = 1` são **duas operações separadas** — a interrupção pode ocorrer entre elas. Ler e escrever não são atômicos juntos.

> ⚠️ **Ler o valor, verificar, e então escrever** nunca é uma operação atômica com instruções comuns de leitura/escrita. É exatamente aí que mora a race condition.

Tentar resolver lendo duas vezes também não resolve: a corrida apenas se move para outro ponto — um segundo processo pode modificar a trava logo após o primeiro ter terminado sua segunda verificação.

---

## 🔄 Abordagem 3 — Alternância Explícita (*Strict Alternation*)

### A ideia

Uma terceira abordagem usa uma variável inteira `turn` para controlar de quem é a vez de entrar na região crítica.

> 📌 **Figura 2.23 — Solução proposta para o problema da região crítica**

```c
/* Processo 0 */                    /* Processo 1 */
while (TRUE) {                      while (TRUE) {
    while (turn != 0) { }               while (turn != 1) { }
    critical_region();                  critical_region();
    turn = 1;                           turn = 0;
    noncritical_region();               noncritical_region();
}                                   }
```

**Como funciona:**

- `turn = 0` inicialmente: o processo 0 inspeciona `turn`, descobre que é 0, e entra na região crítica
- O processo 1 encontra `turn = 0` e fica no laço `while (turn != 1)` em espera ocupada
- Quando o processo 0 sai da região crítica, configura `turn = 1`, permitindo que o processo 1 entre
- Depois que o processo 1 termina sua região crítica, configura `turn = 0` de volta — e assim por diante

### O problema — viola a Condição 3

A solução funciona e evita a race condition. Mas **viola a condição 3** das quatro condições que estabelecemos em [[Regiões Críticas]]:

```
Cenário problemático:

Processo 0 termina sua região crítica → configura turn = 1
Processo 1 entra e SAI RAPIDAMENTE de sua região crítica → configura turn = 0
Ambos estão agora em suas regiões NÃO críticas, turn = 0

Processo 0 quer entrar novamente na região crítica:
  → verifica turn → turn = 0 → ENTRA! ✅

Processo 0 termina → configura turn = 1
Processo 0 quer entrar de novo:
  → verifica turn → turn = 1 → BLOQUEADO! ❌
  → fica esperando o processo 1 entrar e sair
  → mas o processo 1 está em sua região NÃO crítica e pode ser muito lento!
```

**O problema:** a solução exige que os dois processos **se alternem estritamente** na entrada nas regiões críticas. Se um é muito mais rápido que o outro, o mais rápido fica bloqueado esperando o mais lento — mesmo quando o mais lento está na sua região **não crítica**.

> ⚠️ **Alternância explícita não é uma boa solução** quando um dos processos é muito mais lento que o outro. A condição 3 diz que nenhum processo executando *fora* de sua região crítica pode bloquear qualquer processo — e essa solução viola exatamente isso.

---

## 🧮 Abordagem 4 — Solução de Peterson

### O contexto histórico

Em 1981, G. L. Peterson descobriu uma maneira muito mais simples de realizar a exclusão mútua, tornando a solução de Dekker (a primeira solução correta de software para o problema) obsoleta.

> 📌 **Figura 2.24 — A solução de Peterson para realizar a exclusão mútua**

```c
#define FALSE 0
#define TRUE  1
#define N     2          /* número de processos */

int turn;                /* de quem é a vez? */
int interested[N];       /* todos os valores inicialmente 0 (FALSE) */

void enter_region(int process)   /* processo: 0 ou 1 */
{
    int other;                   /* número do outro processo */
    other = 1 - process;         /* o oposto do processo atual */
    interested[process] = TRUE;  /* mostra que você está interessado */
    turn = process;              /* marca o flag */
    while (turn == process && interested[other] == TRUE);                                                         /* comando nulo — espera ocupada */
}

void leave_region(int process)   /* processo: quem está saindo */
{
    interested[process] = FALSE; /* indica a saída da região crítica */
}
```

### Como funciona

**Protocolo:** antes de usar as variáveis compartilhadas (antes de entrar na região crítica), cada processo chama `enter_region` com seu próprio número (0 ou 1). Após terminar, chama `leave_region`.

**Caso simples — apenas o processo 0 quer entrar:**

```
Processo 0 chama enter_region(0):
  other = 1
  interested[0] = TRUE
  turn = 0
  while (turn == 0 && interested[1] == TRUE)?
    → interested[1] é FALSE → condição é FALSA → não espera
  → Entra na região crítica ✅

Processo 0 chama leave_region(0):
  interested[0] = FALSE
```

**Caso de contenção — ambos querem entrar quase simultaneamente:**

```
Processo 0: interested[0] = TRUE; turn = 0
Processo 1: interested[1] = TRUE; turn = 1   ← sobrescreve turn!

O último a escrever em turn "perde" (dá prioridade ao outro):

Se turn = 1 (processo 1 escreveu por último):
  Processo 0: while (turn == 0 && ...)?  → turn é 1, não 0 → não espera → ENTRA ✅
  Processo 1: while (turn == 1 && interested[0] == TRUE)? → TRUE → espera ⏳

Quando processo 0 termina:
  interested[0] = FALSE
  Processo 1: condição do while torna-se FALSE → ENTRA ✅
```

> 💡 **A elegância da solução de Peterson:** o processo que escreve em `turn` por último é o que "cede" a passagem. Combinando a variável `turn` (que evita deadlock entre os dois) com o array `interested` (que evita que um processo seja bloqueado por outro que não quer entrar), Peterson resolveu o problema de forma limpa e sem hardware especial.

**A solução de Peterson satisfaz todas as 4 condições:**
1. ✅ Nunca dois processos na região crítica ao mesmo tempo
2. ✅ Não faz suposições sobre velocidade ou número de CPUs
3. ✅ Um processo fora da região crítica (`interested[i] = FALSE`) não bloqueia ninguém
4. ✅ Nenhuma espera eterna — o processo que cedeu a vez sempre entrará depois

---

## 🖥️ Abordagem 5 — Instrução TSL (*Test and Set Lock*)

### A ideia — ajuda do hardware

As soluções anteriores são de software puro. Agora veremos uma proposta que **exige um pouco de ajuda do hardware**.

Alguns computadores — especialmente aqueles projetados com múltiplos processadores em mente — têm uma instrução especial:

```
TSL RX, LOCK
```

(*Test and Set Lock* — teste e configure a trava)

**O que essa instrução faz atomicamente:**

```
TSL RX, LOCK:
  1. Lê o conteúdo da palavra de memória LOCK → salva no registrador RX
  2. Armazena um valor diferente de zero em LOCK

Tudo isso de forma INDIVISÍVEL — nenhum outro processador pode 
acessar LOCK enquanto a instrução TSL não terminar.
```

> 💡 **Atomicidade via travamento de barramento:** a CPU que executa TSL **trava o barramento de memória**, proibindo que outras CPUs acessem a memória até que a instrução termine. Isso é diferente de desabilitar interrupções — desabilitar interrupções só afeta a CPU local; travar o barramento afeta **todos os processadores** do sistema.

> ⚠️ **Importante:** travar o barramento de memória é muito diferente de desabilitar interrupções. Desabilitar interrupções e depois fazer leitura + escrita separadas **não é atômico** em multiprocessadores — um segundo processador ainda pode acessar a memória entre as duas operações. O TSL resolve isso porque a leitura e a escrita são uma única instrução indivisível.

### Usando TSL para exclusão mútua

Usando uma variável compartilhada `lock`, a solução completa fica:

> 📌 **Figura 2.25 — Entrando e saindo de uma região crítica por meio da instrução TSL**

```asm
enter_region:
    TSL REGISTER, LOCK    | copia LOCK para o registrador e define LOCK = 1
    CMP REGISTER, #0      | LOCK valia zero?
    JNE enter_region      | se não era zero, lock já estava ativado → volta ao laço
    RET                   | retorna a quem chamou → ENTROU na região crítica

leave_region:
    MOVE LOCK, #0         | coloca 0 em LOCK (libera a trava)
    RET                   | retorna a quem chamou
```

**Como funciona:**

```
LOCK = 0 → região crítica livre
LOCK = 1 → região crítica ocupada

Processo quer entrar:
  → executa TSL: lê LOCK, configura LOCK = 1 atomicamente
  → se o valor lido era 0: a trava estava livre, processo entrou → RET
  → se o valor lido era 1: a trava já estava ocupada → JNE volta ao início → espera ocupada

Processo quer sair:
  → MOVE LOCK, #0  (simples — não precisa ser atômica, só um processo pode estar aqui)
```

**Por que funciona em multiprocessadores:** mesmo que dois processadores executem TSL "ao mesmo tempo", o barramento de memória garante que apenas um deles conclua a leitura-e-escrita atômica primeiro. O segundo, ao tentar executar TSL, lê o LOCK já em 1 e fica na espera ocupada.

---

## 🔀 Abordagem 6 — Instrução XCHG

Uma instrução alternativa ao TSL é a **XCHG** (exchange), que troca os conteúdos de duas posições **atomicamente** — por exemplo, um registrador e uma palavra de memória.

> 📌 **Figura 2.26 — Entrando e saindo de uma região crítica usando a instrução XCHG**

```asm
enter_region:
    MOVE REGISTER, #1         | coloca 1 no registrador
    XCHG REGISTER, LOCK       | troca conteúdo entre registrador e variável LOCK
    CMP REGISTER, #0          | LOCK valia zero?
    JNE enter_region          | se não era zero, lock já havia sido ativado → continua no laço
    RET                       | retorna a quem chamou → entrou na região crítica

leave_region:
    MOVE LOCK, #0             | coloca 0 em LOCK
    RET                       | retorna a quem chamou
```

> 💡 **XCHG é essencialmente o mesmo que TSL** para fins de sincronização — ambos garantem atomicidade na leitura + escrita. A diferença é a mecânica: TSL lê e seta; XCHG troca. O resultado prático é idêntico. **Todas as CPUs Intel x86 usam a instrução XCHG para sincronização de baixo nível.**

---

## ⚖️ O defeito comum — espera ocupada

Tanto a solução de Peterson quanto as soluções usando TSL ou XCHG estão corretas, mas ambas têm o mesmo defeito: **espera ocupada** (*busy waiting*).

```
Processo B quer entrar na região crítica:
  → verifica lock/turn/interested → não pode entrar
  → fica em laço apertado testando a variável
  → CPU gasta 100% do tempo nesse laço
  → nenhum trabalho útil é feito
```

Isso apresenta dois problemas concretos:

**Problema 1 — Desperdício de CPU:**
A CPU fica "girando em falso" — 100% ocupada sem produzir nada. Em um sistema com um único processador, isso é especialmente grave: a CPU que o processo bloqueado está ocupando é justamente a que o processo na região crítica precisaria para terminar seu trabalho e sair da região crítica.

**Problema 2 — O problema da inversão de prioridade:**

```
Situação:
  Processo H — alta prioridade
  Processo L — baixa prioridade
  
L está em sua região crítica quando H se torna pronto para executar.
H começa a executar (tem prioridade maior) e entra em espera ocupada
esperando L sair da região crítica.

Mas: L nunca recebe tempo de CPU para terminar, 
     pois H tem prioridade maior e fica girando!

Resultado: H espera para sempre → deadlock de fato.
```

Esse problema é conhecido como o **problema da inversão de prioridade** — o processo de alta prioridade fica bloqueado *indiretamente* pelo de baixa prioridade.

> ⚠️ **A solução** para o problema da espera ocupada é fazer o processo que não pode entrar na região crítica **bloquear** em vez de girar — cedendo a CPU para que outros processos possam executar. Isso é feito com **sleep** e **wakeup**, que Tanenbaum examina na seção 2.4.4.

---

# ✅ Resumo do Conceito

Cinco abordagens foram examinadas, com uma progressão clara de problemas e soluções:

| Abordagem | Funciona? | Problema |
|-----------|-----------|----------|
| Desabilitar interrupções | Só no núcleo | Inaceitável para usuário; falha em multiprocessadores |
| Variável de trava | ❌ | Mesma race condition do spool — leitura e escrita não são atômicas juntas |
| Alternância explícita | ⚠️ Parcialmente | Viola condição 3: processo lento bloqueia processo rápido fora da região crítica |
| Solução de Peterson | ✅ Software puro | Espera ocupada — desperdiça CPU |
| TSL / XCHG | ✅ Hardware | Espera ocupada — desperdiça CPU; problema da inversão de prioridade |

- **Desabilitar interrupções** é simples mas perigoso e inadequado para usuário em multiprocessadores
- **Variável de trava** falha pelo mesmo motivo das race conditions: leitura + escrita separadas não são atômicas
- **Alternância explícita** viola a condição 3 — processa lentos bloqueiam processos rápidos
- **Peterson** é elegante e correto em software, mas usa espera ocupada
- **TSL/XCHG** são soluções de hardware atômicas e funcionam em multiprocessadores, mas também usam espera ocupada
- O problema comum da espera ocupada — especialmente o **problema da inversão de prioridade** — motiva as soluções com **sleep/wakeup** vistas na seção 2.4.4

---

## 🔗 Notas Relacionadas

- [[Race Condition]] — o problema que estamos resolvendo
- [[Regiões Críticas]] — as 4 condições que toda boa solução deve satisfazer
- [[System Calls]] — TSL e XCHG são instruções de hardware; o mecanismo de syscall usa conceitos similares de atomicidade
