---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.4.10 e 2.4.11"
---
# Inversão de Prioridade e RCU

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.4.10 e 2.4.11

---

# 🔺 2.4.10 — Inversão de Prioridade

## O problema — revisitado com profundidade

Em [[Exclusão mútua com espera ocupada]], mencionamos brevemente o problema da inversão de prioridade como motivação para abandonar a espera ocupada. Agora Tanenbaum o examina com detalhes — é um problema clássico conhecido desde a década de 1970, com um exemplo real e dramático.

> 💡 **Inversão de prioridade:** situação em que uma thread de **alta prioridade** fica bloqueada aguardando um recurso detido por uma thread de **baixa prioridade**, enquanto uma thread de **prioridade média** — que não precisa do recurso — continua executando indefinidamente, impedindo que a thread de baixa prioridade libere o recurso. O resultado é que a thread de maior prioridade efetivamente tem a menor prioridade do sistema.

---

## 🚀 O caso do Pathfinder em Marte (1997)

O exemplo mais famoso de inversão de prioridade ocorreu em **Marte, em 1997**. A NASA pousou o robô explorador Pathfinder no planeta vermelho — um feito extraordinário de engenharia. Mas houve um problema grave:

```
As transmissões de rádio do Pathfinder
paravam de enviar dados de forma constante,
exigindo reinicializações do sistema
para fazê-lo funcionar novamente.
```

A investigação revelou que **três threads** estavam interferindo:

```
THREAD DE BAIXA PRIORIDADE:
  Coleta dados meteorológicos periodicamente
  Usa o barramento de informações (memória compartilhada)
  Precisa do mutex para acessar o barramento

THREAD DE ALTA PRIORIDADE:
  Gerencia o barramento de informações
  Precisa acessar o barramento periodicamente
  Também precisa do mutex

THREAD DE PRIORIDADE MÉDIA:
  Responsável pelas comunicações
  NÃO precisa do mutex
  Executa por longos períodos
```

**A sequência do desastre:**

```
1. Thread BAIXA adquire o mutex → começa a usar o barramento

2. Thread ALTA precisa do barramento → tenta adquirir o mutex
   → mutex já está com a thread BAIXA → BLOQUEADA

3. Escalonador procura próxima thread para executar
   → thread ALTA está bloqueada
   → thread MÉDIA está pronta e tem prioridade maior que BAIXA
   → MÉDIA começa a executar

4. Thread MÉDIA executa por um longo tempo
   → thread BAIXA nunca recebe CPU para terminar e liberar o mutex
   → thread ALTA fica bloqueada indefinidamente

RESULTADO:
  Thread ALTA — maior prioridade do sistema
  Na prática não executa porque MÉDIA monopoliza a CPU
  Sistema parece travado → watchdog timer dispara → reinicialização
```

> ⚠️ **O paradoxo:** a thread de maior prioridade é a que menos executa. A thread de prioridade média efetivamente tem mais prioridade que a de alta — daí o nome "inversão".

---

## 🛠️ Soluções para a inversão de prioridade

### 1. Desabilitar interrupções

```
Enquanto estiver na região crítica:
  desabilitar todas as interrupções

Problema:
  Não é desejável para programas de usuário
  E se esquecerem de habilitá-las novamente?
  → fim do sistema
```

### 2. Teto de prioridade (*priority ceiling*)

> 💡 **Teto de prioridade:** associar uma prioridade ao próprio mutex — a **prioridade teto**. Qualquer processo que capture o mutex herda essa prioridade teto enquanto o mantém. Desde que nenhum processo que precise capturar o mutex tenha prioridade maior que a prioridade teto, a inversão não é mais possível.

```
mutex tem prioridade teto = ALTA

Thread BAIXA adquire mutex:
  → temporariamente executa com prioridade ALTA
  → thread MÉDIA (prioridade MÉDIA) não pode preemptá-la
  → thread BAIXA termina rápido, libera mutex
  → thread ALTA adquire mutex, executa ✅
```

### 3. Herança de prioridade (*priority inheritance*)

> 💡 **Herança de prioridade:** quando uma thread de alta prioridade fica bloqueada esperando um mutex detido por uma thread de baixa prioridade, a thread de baixa prioridade **herda temporariamente** a prioridade da thread de alta prioridade. Quando libera o mutex, volta à sua prioridade original.

```
Thread BAIXA detém mutex
Thread ALTA bloqueia esperando mutex:
  → Thread BAIXA herda a prioridade de ALTA
  → Thread MÉDIA não pode preemptá-la (BAIXA agora tem prioridade ALTA)
  → Thread BAIXA termina, libera mutex, volta à prioridade BAIXA
  → Thread ALTA adquire mutex, executa ✅
```

**Esta foi a solução adotada para corrigir o Pathfinder em Marte** — uma correção enviada via uplink de rádio da Terra.

### 4. Reforço aleatório (*random boosting*)

```
Solução do Microsoft Windows:
  Periodicamente, dar alta prioridade a threads aleatórias
  que estejam aguardando para obter um mutex, até que
  saiam da região crítica.

  É uma solução estatística — não garante correção,
  mas reduz a probabilidade do problema na prática.
```

---

# 🔄 2.4.11 — Evitando Travas: Leitura-Cópia-Atualização (RCU)

## A premissa — travas têm custos

> **As travas mais rápidas não são travas.**

A ausência de travas também significa ausência de risco de inversão de prioridade. A questão é: podemos permitir acessos de leitura e escrita **concorrentes** a estruturas de dados compartilhadas sem usar travas?

Em geral, a resposta é claramente **não** — imagine uma thread A ordenando um conjunto de números enquanto uma thread B está calculando a média. Como A desloca os valores, B pode encontrar alguns valores várias vezes e outros jamais. O resultado poderia ser qualquer um.

**Mas em alguns casos específicos, sim** — e é exatamente para esses casos que existe o RCU.

---

## 🌳 O que é RCU — Read-Copy-Update

> 💡 **RCU (*Read-Copy-Update* — leitura-cópia-atualização):** técnica de sincronização sem trava que permite que um escritor atualize uma estrutura de dados enquanto leitores ainda a estão usando, desde que: (1) leitores sempre vejam uma versão **consistente** da estrutura — ou a antiga ou a nova, nunca uma combinação, e (2) a memória antiga só seja liberada depois que **todos os leitores** que podiam estar vendo-a tenham terminado.

O truque: garantir que cada leitor leia a versão anterior dos dados, ou a nova, **mas nunca uma combinação aleatória de anterior e nova**.

> 📌 **Figura 2.39 — Leitura-cópia-atualização: inserindo um nó na árvore e então removendo uma ramificação — tudo sem travas**

---

## ➕ Adicionando um nó — como funciona

```
(a) Árvore original:
         A
        / \
       B   (nada)
      / \
     C   D   E

(b) Inicializar nó X e conectar E e X:
  → criamos X completamente fora da árvore
  → configuramos todos os seus valores, incluindo ponteiros-filhos
  → com uma escrita atômica, tornamos X um filho de A
  → leitores atualmente em A e E não são afetados

(c) X completamente inicializado, conectado:
  → leitores que estavam em E lerão a versão anterior (E sem X)
  → leitores novos que chegarem em A pegarão a nova versão (verão X)
  → NUNCA há estado inconsistente — em nenhum momento
    um leitor vê X parcialmente inicializado
```

**O ponto chave:** o nó é totalmente preparado **antes** de se tornar visível. A visibilidade é uma única escrita atômica. Leitores nunca veem um nó em estado de construção.

---

## ➖ Removendo nós — o desafio da liberação de memória

A remoção é mais complexa porque precisamos garantir que nenhum leitor ainda esteja usando o nó antes de liberar sua memória.

```
(d) Desacoplar B de A:
  → fazemos o ponteiro-filho esquerdo de A apontar para C
  → leitores que ainda estavam em B continuarão seguindo
    os ponteiros originais de B e verão a versão anterior
  → leitores novos a partir de A nunca verão B ou D

(e) Esperar até certeza de que todos os leitores
    deixaram B e D:
  → esses nós não podem mais ser acessados por novos leitores
  → mas leitores ANTIGOS podem ainda estar percorrendo B/D
  → PRECISAMOS ESPERAR que todos terminem

(f) Agora podemos remover com segurança B e D:
  → liberar a memória ✅
```

---

## ⏱️ O período de graça — quanto tempo esperar?

```
Problema: quanto tempo esperar para ter certeza de que
          nenhum leitor ainda está em B ou D?

Não podemos bloquear ou adormecer dentro da região crítica de leitura
→ um critério simples: esperar até que TODAS AS THREADS
  tenham realizado uma troca de contexto

Por quê? Como não é permitido ao código na seção crítica de leitura
bloquear ou adormecer, uma thread que trocou de contexto
certamente não está mais dentro de uma seção crítica de leitura
da estrutura de dados.
```

> 💡 **Período de graça (*grace period*):** período de tempo após uma atualização RCU durante o qual o escritor aguarda antes de liberar a memória antiga. O período de graça termina quando todas as threads do sistema realizaram pelo menos uma troca de contexto — garantindo que nenhuma pode estar ainda na seção crítica de leitura usando os dados antigos.

> 💡 **Seção crítica do lado do leitor (*read-side critical section*):** região de código que acessa a estrutura de dados RCU. Pode conter qualquer código, desde que não bloqueie ou adormeça. Leitores que estão fora dessa seção não são problema — só os que estão dentro durante o período de graça precisam terminar antes da liberação.

---

## 🔍 Onde RCU é usado na prática

```
Núcleo do Linux:
  → milhares de usos da API RCU espalhados pelos subsistemas
  → pilha de rede: tabelas de roteamento (muitas leituras, poucas escritas)
  → sistema de arquivos: estruturas de diretório
  → drivers: listas de dispositivos
  → gerenciamento de memória: estruturas de mapeamento

Por que o kernel usa tanto RCU?
  → estruturas de dados do kernel são acessadas por muitas threads
    simultaneamente e exigem alta eficiência
  → leituras são muito mais frequentes que escritas na maioria dos casos
  → RCU elimina completamente a contenção de trava para leitores
```

---

## ⚖️ RCU vs. travas tradicionais

```
┌─────────────────────────┬───────────────────────┬───────────────────────┐
│   Característica        │   Mutex / Semáforo    │        RCU            │
├─────────────────────────┼───────────────────────┼───────────────────────┤
│ Leitores simultâneos    │ Bloqueados por escrita│ Nunca bloqueados ✅   │
│ Escritores              │ Exclusão mútua normal │ Um por vez, mais caro │
│ Inversão de prioridade  │ Possível              │ Impossível para leitor│
│ Complexidade            │ Baixa                 │ Alta                  │
│ Casos de uso            │ Qualquer              │ Leituras >> escritas  │
│ Liberação de memória    │ Imediata              │ Após período de graça │
│ Leitores veem estado    │ Sempre atual          │ Pode ser versão antiga│
└─────────────────────────┴───────────────────────┴───────────────────────┘
```

> ⚠️ **RCU não é uma solução geral.** É específico para estruturas de dados onde leituras são muito mais frequentes que escritas, e onde é aceitável que leitores vejam temporariamente a versão anterior dos dados. Para casos onde todos precisam ver sempre o estado mais atual, travas tradicionais são necessárias.

---

# ✅ Resumo do Conceito

**Inversão de Prioridade (2.4.10):**
- Ocorre quando thread de prioridade MÉDIA impede que thread de prioridade BAIXA (que detém um mutex) libere o recurso que thread de prioridade ALTA precisa
- Exemplo clássico: Pathfinder em Marte (1997) — resolvido via uplink com **herança de prioridade**
- Três soluções: desabilitar interrupções (perigoso), **teto de prioridade** (mutex tem prioridade predefinida), **herança de prioridade** (thread baixa herda prioridade da alta temporariamente), reforço aleatório (Windows)
- Herança de prioridade foi a solução aplicada ao Pathfinder

**RCU — Read-Copy-Update (2.4.11):**
- Técnica sem trava para estruturas de dados com **leituras muito mais frequentes que escritas**
- Leitores nunca bloqueiam — acessam a estrutura diretamente sem trava
- Escritores fazem a atualização em uma cópia, tornam-na visível com uma **escrita atômica**, e só liberam a memória antiga após o **período de graça**
- Período de graça = esperar até que todas as threads façam pelo menos uma troca de contexto
- Leitores sempre veem uma versão consistente — ou a antiga ou a nova, nunca uma mistura
- Amplamente usado no kernel Linux: rede, sistema de arquivos, drivers, gerenciamento de memória

---

## 🔗 Notas Relacionadas

- [[Exclusão mútua com espera ocupada]] — onde inversão de prioridade foi mencionada pela primeira vez como problema da espera ocupada
- [[Mutexes]] — teto de prioridade e herança de prioridade são propriedades de mutexes
- [[Dormir e Despertar]] — threads bloqueadas em mutex são semelhantes ao sleep; herança de prioridade resolve o problema de quem executa
- [[Race Condition]] — RCU elimina races para leitores garantindo consistência de versão
- [[Semáforos]] — comparação: semáforos bloqueiam leitores; RCU não
