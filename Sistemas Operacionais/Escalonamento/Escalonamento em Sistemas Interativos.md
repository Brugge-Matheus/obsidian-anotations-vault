---
tags:
  - sistemas-operacionais
  - so/escalonamento
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.5.3"
---
# Escalonamento em Sistemas Interativos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.5.3

---

# 🖥️ 2.5.3 — Escalonamento em Sistemas Interativos

Em [[Escalonamento em Sistemas em Lote]], vimos algoritmos que assumem tempos de execução conhecidos antecipadamente. Sistemas interativos são diferentes — usuários esperam respostas rápidas e imprevisíveis. Vamos examinar os algoritmos usados nesse contexto.

---

## 🔄 Escalonamento Circular (*Round-Robin*)

Um dos algoritmos mais antigos, simples, justos e amplamente usados.

> 💡 **Quantum:** intervalo de tempo designado a cada processo, durante o qual ele tem permissão para executar.

### Como funciona

```
A cada processo é designado um quantum de tempo.

SE o processo ainda está executando ao fim do quantum:
  → a CPU sofre uma PREEMPÇÃO
  → o processo recebe outro processo em seu lugar

SE o processo foi bloqueado ou terminado ANTES de o
quantum ter decorrido:
  → o chaveamento de CPU é feito durante o bloqueio
    desse processo, é claro
```

> 📌 **Figura 2.43 — Escalonamento circular. (a) A lista de processos executáveis. (b) A lista de processos executáveis após B usar todo seu quantum.**

```
(a) Lista original:

  Processo    Próximo
   atual      processo
  ┌───┐┌───┐┌───┐┌───┐┌───┐
  │ B ││ F ││ D ││ G ││ A │
  └───┘└───┘└───┘└───┘└───┘

(b) Após B usar todo seu quantum
    (B vai para o FIM da lista):

  Processo
   atual
  ┌───┐┌───┐┌───┐┌───┐┌───┐
  │ F ││ D ││ G ││ A ││ B │
  └───┘└───┘└───┘└───┘└───┘
```

**Implementação:** tudo o que o escalonador precisa fazer é manter uma **lista de processos executáveis**. Quando o processo usa todo o seu quantum, ele é colocado no **fim** da lista.

---

## ⏲️ A questão crucial — o tamanho do Quantum

A única questão realmente interessante em relação ao escalonamento circular é a **extensão do quantum**.

### O custo da troca de contexto

```
A passagem de um processo para o outro exige tempo
para fazer toda a administração:
  → salvar e carregar registradores
  → mapas de memória
  → atualizar várias tabelas e listas
  → descarregar e recarregar a memória CACHE

Suponha:
  Troca de contexto leva 1 ms (incluindo troca de mapas
  de memória, descarga e recarga da cache, etc.)
  Quantum estabelecido em 4 ms

Com esses parâmetros:
  Após realizar 4ms de trabalho útil, a CPU gastará
  (desperdiçará) 1ms com a troca de processo
  → 20% do tempo da CPU jogado fora em SOBRECARGA
    ADMINISTRATIVA
  → CLARAMENTE isso é demais
```

### O quantum longo demais também é ruim

```
Para melhorar a eficiência da CPU:
  quantum = 100ms
  → tempo desperdiçado cai para apenas 1% ✅

MAS considere um sistema de SERVIDORES:
  50 solicitações entram em um intervalo muito curto,
  com exigências de CPU de grande variação

  50 processos são colocados na lista de executáveis

  SE a CPU estiver ociosa:
    → o primeiro começará IMEDIATAMENTE
    → o segundo não poderá começar até 100ms mais tarde
    → e assim por diante

  O ÚLTIMO azarado talvez tenha de esperar 5s antes de
  ter uma chance (presumindo que todos os outros usem
  todo o seu quantum)

  A MAIORIA dos usuários achará demorada uma resposta
  de 5s para um comando CURTO — especialmente ruim se
  algumas das solicitações próximas do fim da fila
  exigissem apenas alguns MILISSEGUNDOS de tempo de CPU.
  Com um quantum curto, elas teriam recebido um
  serviço melhor.
```

### A conclusão sobre o tamanho do quantum

```
Outro fator: SE o quantum for configurado por um tempo
mais longo que o SURTO DE CPU MÉDIO, a preempção NÃO
acontecerá com muita frequência.

Em vez disso, a MAIORIA dos processos desempenhará uma
operação de bloqueio ANTES de o quantum acabar,
provocando uma troca de processo.

Eliminar a preempção MELHORA o desempenho, porque as
trocas de processo então acontecem apenas quando são
LOGICAMENTE necessárias — isto é, quando um processo é
bloqueado e não pode continuar.
```

> ⚠️ **A conclusão pode ser formulada:** estabelecer o quantum **curto demais** provoca **muitas trocas de processo** e reduz a eficiência da CPU. Mas estabelecê-lo **longo demais** pode provocar uma **resposta ruim** a solicitações interativas curtas. Um **quantum em torno de 20 a 50ms** é quase sempre bastante razoável.

---

## 🎖️ Escalonamento por Prioridades

O escalonamento circular pressupõe implicitamente que todos os processos têm a mesma importância. Isso frequentemente não reflete a realidade.

> 💡 **Escalonamento por prioridades:** a cada processo é designada uma prioridade, e o processo executável com a prioridade mais alta é autorizado a executar.

### Motivação — nem todos os processos importam igualmente

```
Em uma universidade: uma ordem hierárquica começaria
pelo presidente, depois reitores, professores,
secretárias, zeladores e, por fim, estudantes.

Mesmo em um PC de único proprietário, pode haver
múltiplos processos com importâncias diferentes:

  Processo daemon enviando e-mail em segundo plano
    → prioridade BAIXA

  Processo exibindo um filme de vídeo na tela em
  tempo real
    → prioridade ALTA
```

### Evitando que processos de alta prioridade executem indefinidamente

```
Para evitar que processos de prioridade mais alta
executem INDEFINIDAMENTE:

MÉTODO 1: escalonador diminui a prioridade do processo
          em execução a cada tique do relógio (cada
          interrupção de relógio)
  → se essa ação fizer a prioridade cair abaixo da do
    próximo processo com a prioridade mais alta, ocorre
    uma troca de processo

MÉTODO 2 (alternativa): designar a cada processo um
          quantum de tempo MÁXIMO no qual ele é
          autorizado a executar
  → quando esse quantum for esgotado, o processo
    seguinte na escala de prioridade recebe uma
    chance de ser executado
```

### Prioridades estáticas vs. dinâmicas

```
ESTÁTICAS:
  Exemplo — computador militar:
    Processos iniciados por generais → prioridade 100
    Processos iniciados por coronéis → prioridade 90
    Processos iniciados por majores  → prioridade 80
    Processos iniciados por capitães → prioridade 70
    Processos iniciados por tenentes → prioridade 60
    (conforme a insígnia)

  Exemplo — data center comercial:
    Tarefas de alta prioridade  → US$ 100/hora
    Tarefas de prioridade média → US$ 75/hora
    Tarefas de baixa prioridade → US$ 50/hora

  ⚠️ O sistema UNIX tem um comando, nice, que permite
  que um usuário reduza VOLUNTARIAMENTE a prioridade
  do seu processo, a fim de ser gentil com os outros
  usuários — mas NINGUÉM o utiliza.

DINÂMICAS:
  Designadas pelo próprio sistema para alcançar
  determinadas metas.

  Exemplo — processos vinculados à E/S:
    Processos altamente vinculados à E/S passam a
    maior parte do tempo esperando que a E/S seja
    concluída.

    Sempre que um processo assim deseja a posse da CPU,
    ele deve recebê-la IMEDIATAMENTE, para deixá-lo
    iniciar sua próxima solicitação de E/S, que pode
    então prosseguir EM PARALELO com outro processo
    que estiver de fato computando.

    Fazer o processo vinculado à E/S esperar muito
    tempo pela CPU significará apenas tê-lo ocupando
    a memória por um tempo desnecessariamente longo.

  Um algoritmo simples: configurar a prioridade para 1/f,
  onde f é a fração do último quantum que o processo usou.

    Processo que usou apenas 1ms de um quantum de 50ms
      → recebe prioridade 50

    Processo que usasse 25ms antes de bloquear
      → recebe prioridade 2

    Processo que usasse o quantum INTEIRO
      → recebe prioridade 1
```

---

## 📚 Múltiplas Filas

Um dos primeiros escalonadores de prioridade foi no **CTSS**, o sistema compatível de tempo compartilhado do MIT, que operava no IBM 7094.

### O problema histórico do CTSS

```
Um problema do CTSS era que a troca de processo era
LENTA — o 7094 só conseguia armazenar UM processo na
memória.

Cada chaveamento significava trocar o processo atual
para o disco e ler um novo a partir do disco.

Os projetistas do CTSS perceberam que era mais eficiente
dar aos processos vinculados à CPU um GRANDE quantum de
vez em quando, em vez de dar a eles pequenos quanta
FREQUENTEMENTE (para reduzir as operações de troca).

Por outro lado, dar a TODOS os processos um grande
quantum significaria um tempo de resposta RUIM.
```

### A solução — classes de prioridade

> 💡 **Múltiplas filas:** sistema com várias classes de prioridade, onde processos na classe mais alta executam por um quantum, na próxima classe por dois quanta, na seguinte por quatro quanta, e assim por diante. Sempre que um processo consumia todos os quanta alocados para ele, era movido para uma classe inferior.

> 📌 **Figura 2.44 — Um algoritmo de escalonamento com quatro classes de prioridade**

```
Cabeçalhos          Processos executáveis
das filas

Prioridade 4  ──►  [ ] [ ] [ ]              (Prioridade mais alta)
Prioridade 3  ──►  [ ] [ ] [ ] [ ]
Prioridade 2  ──►  [ ]
Prioridade 1  ──►  [ ]                       (Prioridade mais baixa)
```

**Como o algoritmo funciona:**

```
Desde que existam processos executáveis na CLASSE 4:
  → apenas execute cada um por um quantum, estilo
    circular
  → jamais se importe com classes de prioridade
    mais baixa

SE a classe de prioridade 4 estiver VAZIA:
  → execute os processos de classe 3 de maneira circular

SE ambas as classes — 4 e 3 — estiverem vazias:
  → execute a classe 2 de maneira circular
  → e assim por diante

⚠️ SE as prioridades não forem ajustadas ocasionalmente,
   classes de prioridade mais baixa podem todas MORRER
   DE FOME (starvation).
```

**O resultado:** processos na classe mais alta são executados com mais frequência e com prioridade alta, mas por um tempo menor — ideal para processos interativos.

### O exemplo numérico — economia dramática de trocas

```
Processo precisando computar continuamente por 100 quanta:

  1ª execução:  recebe 1 quantum,   então é trocado
  2ª execução:  recebe 2 quanta,    então é trocado
  3ª execução:  recebe 4 quanta
  4ª execução:  recebe 8 quanta
  5ª execução:  recebe 16 quanta
  6ª execução:  recebe 32 quanta
  7ª execução:  recebe 64 quanta (mas usa apenas 37 dos
                64 finais para completar o trabalho)

APENAS SETE trocas seriam necessárias (incluindo a carga
inicial) EM VEZ DE 100 com um algoritmo circular puro.
```

### O problema — punir processos que se tornam interativos

```
Para evitar punir PARA SEMPRE um processo que precisasse
ser executado por muito tempo quando foi iniciado
primeiramente, mas se tornasse INTERATIVO mais tarde:

  POLÍTICA: sempre que a tecla Enter era digitada em um
  terminal, o processo pertencente àquele terminal era
  movido para a classe de prioridade mais alta —
  pressupondo que ele estava prestes a se tornar
  interativo.

  ⚠️ RESULTADO INESPERADO: um belo dia, algum usuário
  com um processo com uso intenso de CPU descobriu que
  apenas SENTAR em um terminal e digitar a tecla Enter
  ao acaso, de tempos em tempos, ajudava e MUITO seu
  tempo de resposta!

  Moral da história: acertar na PRÁTICA é muito mais
  difícil do que acertar no PRINCÍPIO.
```

---

## ⏱️ Processo Mais Curto em Seguida

Como o algoritmo **tarefa mais curta primeiro** sempre produz o tempo de resposta médio mínimo para sistemas em lote, seria bom se ele pudesse ser usado para processos interativos também.

### O desafio — estimar o próximo comando

```
Processos interativos geralmente seguem o padrão:

  esperar pelo comando → executar o comando →
  esperar pelo comando → executar o comando → ...

Se considerarmos a execução de cada COMANDO como uma
"tarefa" em separado, poderíamos minimizar o tempo de
resposta GERAL executando a tarefa mais curta primeiro.

O PROBLEMA: descobrir qual dos processos atualmente
executáveis é o MAIS CURTO.
```

### A técnica de estimativa — envelhecimento (*aging*)

```
Abordagem: fazer estimativas baseadas no comportamento
PASSADO e executar o processo com o tempo de execução
ESTIMADO mais curto.

Suponha:
  Tempo estimado por comando para um processo = T₀
  Execução SEGUINTE é mensurada como sendo = T₁

Poderíamos atualizar nossa estimativa tomando a SOMA
PONDERADA desses dois números:

  a·T₀ + (1 − a)·T₁

Pela escolha de 'a' podemos decidir que o processo de
estimativa ESQUEÇA as execuções anteriores rapidamente,
ou as LEMBRE por um longo tempo.

Com a = 1/2, temos estimativas SUCESSIVAS de:

  T₀
  T₀/2 + T₁/2
  T₀/4 + T₁/4 + T₂/2
  T₀/8 + T₁/8 + T₂/4 + T₃/2

Após TRÊS novas execuções, o peso de T₀ na nova
estimativa caiu para 1/8.
```

> 💡 **Envelhecimento (*aging*):** a técnica de estimar o valor seguinte em uma série tomando a média ponderada do valor mensurado atual e a estimativa anterior. É aplicável a muitas situações em que uma previsão precisa ser feita baseada em valores anteriores. É especialmente fácil de implementar quando **a = 1/2** — basta adicionar o novo valor à estimativa atual e dividir a soma por 2 (deslocando-a 1 bit para a direita).

### A variante moderna — CFS do Linux

```
CFS (Completely Fair Scheduling — escalonamento
completamente justo) do Linux:

  → rastreia o "tempo de execução gasto" para processos
    em uma ÁRVORE RUBRO-NEGRA eficiente

  → o nó mais à esquerda na árvore corresponde ao
    processo com o MENOR tempo de execução gasto

  → o escalonador indexa a árvore por tempo de execução
    e seleciona o nó mais à esquerda para executar

  → quando o processo para de executar (esgotou seu
    intervalo de tempo, foi bloqueado ou interrompido),
    o escalonador o REINSERE na árvore com base em seu
    NOVO tempo de execução gasto
```

---

## 🎟️ Escalonamento por Loteria

Embora realizar promessas para os usuários e cumpri-las seja uma boa ideia, é difícil de implementar. Outro algoritmo gera resultados similarmente previsíveis com uma implementação muito mais simples.

> 💡 **Escalonamento por loteria (Waldspurger e Weihl, 1994):** a ideia básica é dar **bilhetes de loteria** aos processos para vários recursos do sistema, como o tempo de CPU. Sempre que uma decisão de escalonamento tiver de ser feita, um bilhete de loteria é escolhido AO ACASO, e o processo com o bilhete fica com o recurso.

### Como funciona

```
Quando aplicado ao escalonamento de CPU:
  → o sistema pode realizar um sorteio 50 vezes por
    segundo
  → cada vencedor recebe 20ms de tempo da CPU como
    prêmio

"Parafraseando George Orwell: 'Todos os processos são
iguais, mas alguns processos são mais iguais.'"

Processos mais IMPORTANTES podem receber bilhetes
EXTRAS para aumentar sua chance de vencer.

Exemplo:
  100 bilhetes emitidos, processo tem 20 deles
    → 20% de chance de vencer cada sorteio
    → a longo prazo, terá acesso a cerca de 20% da CPU
```

### Por que é melhor que prioridades — clareza matemática

```
No escalonador de PRIORIDADE:
  → é muito DIFÍCIL afirmar o que realmente significa
    ter uma prioridade de 40

No escalonamento por LOTERIA a regra é CLARA:
  → um processo que tenha uma fração f dos bilhetes
    terá aproximadamente uma fração f do recurso
    em questão
```

### Propriedades interessantes

```
1. RESPONSIVIDADE:
   Se um novo processo aparece e ganha alguns bilhetes,
   no sorteio SEGUINTE ele já teria uma chance de vencer
   na proporção do número de bilhetes que tem em mãos.
   → o escalonamento de loteria é ALTAMENTE responsivo

2. COOPERAÇÃO ENTRE PROCESSOS:
   Processos cooperativos podem TROCAR bilhetes se assim
   quiserem.

   Exemplo: processo-cliente envia mensagem para
   processo-servidor e então bloqueia
     → pode dar TODOS os seus bilhetes para o servidor,
       a fim de aumentar a chance de que o servidor
       seja executado em seguida
     → quando o servidor tiver concluído, ele devolve
       os bilhetes para que o cliente possa executar
       novamente
     → na ausência de clientes, os servidores não
       precisam de bilhete algum

3. RESOLVE PROBLEMAS DIFÍCEIS DE OUTROS MÉTODOS:
   Exemplo — servidor de vídeo com vários processos
   alimentando fluxos de vídeo em DIFERENTES taxas de
   apresentação de quadros:

     Processos precisam de quadros a 10, 20 e 25 quadros/s
     
     Ao alocar para esses processos 10, 20 e 25 bilhetes
     (nessa ordem), eles automaticamente dividirão a CPU
     na proporção CORRETA: 10:20:25
```

---

## ⚖️ Escalonamento por Fração Justa

Até agora presumimos que cada processo é escalonado **por si próprio**, sem levar em consideração **quem é o seu dono**.

### O problema

```
SE o usuário 1 inicia NOVE processos e o usuário 2
inicia UM processo, com chaveamento circular ou com
prioridades iguais:

  Usuário 1 receberá 90% da CPU
  Usuário 2 receberá apenas 10% dela
  
  ⚠️ INJUSTO em relação aos USUÁRIOS (não aos processos)
```

### A solução

> 💡 **Escalonamento por fração justa:** alguns sistemas levam em conta qual usuário é dono de um processo antes de escaloná-lo. Nesse modelo, a cada usuário é alocada alguma fração da CPU, e o escalonador escolhe processos de uma maneira que garanta essa fração — não importa quantos processos aquele usuário tenha em execução.

```
Exemplo: sistema com dois usuários, cada um tendo a
         promessa de 50% da CPU

  Usuário 1 tem QUATRO processos: A, B, C, D
  Usuário 2 tem APENAS UM processo: E

Se o escalonamento circular for usado, uma sequência
POSSÍVEL que atende a TODAS as restrições é:

  A E B E C E D E A E B E C E D E ...

Por outro lado, se o usuário 1 tem direito a DUAS VEZES
o tempo de CPU do usuário 2, talvez tenhamos:

  A B E C D E A B E C D E ...
```

---

# ✅ Resumo do Conceito

- **Escalonamento circular (round-robin):** cada processo recebe um **quantum**; se ainda executa ao fim do quantum, sofre preempção e vai para o fim da fila
- O tamanho do quantum é uma decisão crítica: **curto demais** = muitas trocas de contexto e desperdício de CPU; **longo demais** = resposta ruim para solicitações interativas curtas; **20-50ms** é geralmente razoável
- **Escalonamento por prioridades:** cada processo tem uma prioridade — pode ser **estática** (definida externamente, ex: hierarquia militar/comercial) ou **dinâmica** (ajustada pelo sistema, ex: 1/f para favorecer processos vinculados à E/S)
- **Múltiplas filas** (origem no CTSS): classes de prioridade onde cada classe recebe quanta progressivamente maiores (1, 2, 4, 8...); processos que consomem todo o quantum descem de classe — economiza drasticamente o número de trocas de processo, mas exige cuidado (ex: o hack do Enter) para não punir permanentemente processos que se tornam interativos
- **Processo mais curto em seguida:** aplica a lógica do SJF a processos interativos usando **envelhecimento (aging)** — média ponderada de execuções passadas para estimar a próxima; o Linux usa uma variante (**CFS**) com árvore rubro-negra
- **Escalonamento por loteria:** processos recebem bilhetes; sorteios periódicos decidem quem executa — a fração de bilhetes = fração aproximada da CPU; permite **troca de bilhetes** entre processos cooperantes (cliente/servidor) e resolve elegantemente problemas de proporção (ex: servidor de vídeo com taxas de quadro diferentes)
- **Escalonamento por fração justa:** garante que a CPU seja dividida de forma justa entre **usuários**, não apenas entre processos — evita que um usuário com muitos processos monopolize a CPU às custas de outro com poucos

---

## 🔗 Notas Relacionadas

- [[Escalonamento em Sistemas em Lote]] — SJF e sua relação com o "processo mais curto em seguida" e a técnica de envelhecimento
- [[Introdução ao Escalonamento]] — as metas de imparcialidade e tempo de resposta que estes algoritmos tentam otimizar
- [[Inversão de Prioridade e RCU]] — problemas relacionados a prioridades e como o Windows usa reforço aleatório de forma análoga ao escalonamento por loteria
- [[Estados de Processos]] — a fila de processos executáveis manipulada pelo escalonamento circular
