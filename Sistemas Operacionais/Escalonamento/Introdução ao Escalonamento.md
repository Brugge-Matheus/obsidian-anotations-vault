---
tags:
  - sistemas-operacionais
  - so/escalonamento
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.5 e 2.5.1"
---
# Introdução ao Escalonamento

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.5 e 2.5.1

---

# ⚙️ 2.5 — Escalonamento

## O que é o escalonamento

Quando um computador é multiprogramado, ele com frequência tem múltiplos processos ou threads competindo pela CPU ao mesmo tempo. Essa situação ocorre sempre que dois ou mais processos estão simultaneamente no estado pronto. Se apenas uma CPU está disponível, é preciso escolher qual processo será executado em seguida.

> 💡 **Escalonador (*scheduler*):** parte do sistema operacional responsável por escolher qual processo (ou thread) será executado a seguir.

> 💡 **Algoritmo de escalonamento (*scheduling algorithm*):** o critério/regra que o escalonador usa para tomar essa decisão.

Muitas das mesmas questões que se aplicam ao escalonamento de processos também se aplicam ao escalonamento de threads — embora algumas sejam diferentes. Quando o núcleo gerencia threads, o escalonamento é geralmente feito por **thread**, com pouca ou nenhuma consideração sobre o processo ao qual a thread pertence. Este capítulo foca inicialmente nas questões que se aplicam tanto a processos quanto a threads; o escalonamento específico de threads (e o escalonamento em CPUs multinúcleo) é tratado mais adiante.

---

## 📜 Breve contexto histórico

```
Sistemas em lote antigos (fita magnética):
  → algoritmo simples: execute o próximo trabalho da fita
  → sem complexidade de escalonamento

Multiprogramação:
  → múltiplos usuários esperando pelo serviço simultaneamente
  → escalonamento torna-se mais complexo
  → alguns computadores de grande porte ainda misturam
    lote e tempo compartilhado: o escalonador decide se um
    trabalho em lote ou um usuário interativo em terminal
    vai em seguida

Computadores pessoais:
  → mudança de duas formas:
  
  1. Na maior parte do tempo há apenas UM processo ativo
     (é improvável que um usuário digitando um documento esteja
      simultaneamente compilando um programa em segundo plano)
     → o escalonador não precisa se esforçar muito para
       escolher o processo certo — o processo de texto é
       o único candidato quando o usuário digita
     
  2. Computadores ficaram MUITO mais rápidos ao longo dos anos
     → CPU deixou de ser um recurso escasso na maioria dos casos
     → mesmo compilações, que antes consumiam muitos ciclos,
       hoje levam apenas segundos
```

> ⚠️ **A CPU dificilmente ainda é um recurso escasso** na maioria dos PCs. A limitação atual é a **taxa na qual o usuário pode apresentar a entrada** (digitando ou clicando), não a taxa na qual a CPU pode processá-la. Mesmo quando dois programas parecem estar rodando ao mesmo tempo (um processador de texto e uma planilha), dificilmente importa qual deles vai primeiro — o usuário não notará a diferença, pois ambos completam suas tarefas rapidamente.

---

## 💻 Quando o escalonamento NÃO importa muito — e quando importa

### PCs simples — escalonamento pouco relevante

```
Como consequência da abundância de recursos:
  → o algoritmo de escalonamento não importa muito em PCs simples

Exceção: aplicações que praticamente devoram a CPU "viva"
  → reproduzir vídeo de alta resolução enquanto ajusta cores
  → NTSC: 107.892 quadros/hora | PAL: 90.000 quadros/hora
  → exige potência computacional tremenda

Mas aplicações desse tipo são a EXCEÇÃO, não a regra.
```

### Servidores em rede — escalonamento volta a importar

```
Quando voltamos aos servidores em rede, a situação muda
consideravelmente:
  → múltiplos processos competem pela CPU
  → exemplo: CPU escolhendo entre executar um processo que
    reúne estatísticas diárias VS um que serve solicitações
    de usuários
  → usuários ficarão muito mais contentes se o segundo
    processo receber a chance de acessar a CPU primeiro
```

### IoT e sensores — o argumento da "abundância" não se sustenta

```
O argumento de "recursos abundantes" também NÃO se sustenta em:
  → dispositivos de IoT
  → nós em redes de sensores
  → talvez nem em smartphones

Por quê?
  Embora as CPUs em celulares tenham se tornado mais poderosas
  e a memória mais abundante, o tempo de vida da bateria
  NÃO acompanhou esse progresso.

  Como a duração da bateria é uma das restrições mais
  importantes nesses dispositivos, alguns escalonadores
  tentam otimizar o CONSUMO DE ENERGIA, não apenas velocidade.
```

---

## 🔄 Troca de Contexto — o custo de trocar de processo

Além de escolher o processo certo a ser executado, o escalonador também tem de se preocupar em fazer um uso eficiente da CPU, pois **a troca de processos é algo caro**.

> 💡 **Troca de contexto (*context switch*):** o processo completo de suspender a execução de um processo e retomar outro — salvar todo o estado do processo atual e carregar o estado do próximo. Às vezes as pessoas usam esse termo para se referir à troca de processo completa.

### O que acontece durante uma troca de contexto

```
1. Trocar do modo usuário para o modo núcleo (kernel)

2. Salvar o estado do processo atual:
   → registradores da CPU
   → mapa de memória (bits de referência à memória
     na tabela de páginas, em alguns sistemas)
   → tudo é salvo na tabela de processos

3. Selecionar um novo processo:
   → executar o algoritmo de escalonamento

4. Recarregar a MMU (memory management unit):
   → unidade de gerenciamento de memória precisa ser
     recarregada com o mapa de memória do NOVO processo

5. Inicializar o novo processo

6. Custo adicional — invalidação de cache:
   → a troca de processo pode invalidar a memória cache
     e as tabelas relacionadas
   → força o cache a ser dinamicamente recarregado da
     memória principal DUAS VEZES:
       (a) ao entrar no núcleo
       (b) ao deixá-lo
```

> ⚠️ **De modo geral, realizar muitas trocas de processo por segundo pode consumir um montante substancial do tempo da CPU** — recomenda-se cautela. O tempo gasto trocando de contexto é tempo que NÃO é gasto executando código útil do processo.

---

## 📊 Comportamento de Processos — surtos de CPU e E/S

Quase todos os processos alternam **surtos de computação** com **solicitações de E/S** (disco ou rede).

> 📌 **Figura 2.40 — Surtos de uso da CPU alternam-se com períodos de espera por E/S**

```
(a) Processo vinculado à CPU (poucas esperas por E/S):

  [████ surto longo de CPU ████][████ surto longo ████][████████]
                                                          ↑
                                                    espera pela E/S
                                                    (rara e breve)

(b) Processo vinculado à E/S (muitos surtos curtos):

  [██][█][██][█][███][█][███][██][███][█]
   ↑ esperas frequentes pela E/S entre surtos curtos de CPU
```

**Como interpretar a figura:** a CPU executa por um tempo sem parar; então uma chamada de sistema é feita para ler ou escrever em um arquivo. Quando a chamada de sistema é concluída, a CPU calcula novamente até precisar de mais dados ou ter de escrever mais dados, e assim por diante.

> ⚠️ **Nem toda atividade de E/S conta como espera.** Por exemplo, quando a CPU copia bits para uma RAM de vídeo para atualizar a tela, ela está **computando**, não realizando E/S — porque a CPU está em uso. E/S nesse sentido é quando um processo **entra no estado bloqueado** esperando por um dispositivo externo para concluir seu trabalho.

### Vinculado à CPU vs. Vinculado à E/S

> 💡 **Processo vinculado à computação (*CPU-bound*):** processo que passa a maior parte do tempo computando — tem longos surtos de CPU e esporádicas esperas de E/S. Também chamado de **vinculado à CPU**.

> 💡 **Processo vinculado à E/S (*I/O-bound*):** processo que passa a maior parte do tempo esperando pela E/S — tem surtos de CPU curtos e esperas de E/S frequentes.

```
QUAL VARIÁVEL IMPORTA?

O fator-chave é o COMPRIMENTO DO SURTO DE CPU,
NÃO o comprimento do surto de E/S.

Processos vinculados à E/S NÃO são vinculados à E/S porque
têm solicitações especialmente demoradas — eles levam o
MESMO TEMPO para emitir o pedido de leitura de um bloco de
disco, independentemente de quanto tempo levam para
processar os dados depois que eles chegam.

O que os torna "vinculados à E/S" é que eles NÃO computam
muito ENTRE as solicitações de E/S — seus surtos de CPU
são curtos.
```

### Uma tendência com o tempo

> ⚠️ **À medida que as CPUs ficam mais rápidas, os processos tendem a ficar mais vinculados à E/S.** Isso ocorre porque as CPUs estão melhorando muito mais rápido do que os discos.

```
CPUs: melhoram drasticamente ano após ano (Lei de Moore)
Discos rígidos: praticamente não estão ficando mais rápidos
SSDs: substituindo HDs em desktop/notebook, mas em grandes
      data centers os HDs ainda são muito usados (custo por bit)

Consequência:
  → o escalonamento de processos vinculados à E/S tende a
    se tornar mais importante no futuro
  → a ideia básica: se um processo vinculado à E/S quiser
    executar, ele deve receber uma chance RAPIDAMENTE para
    poder emitir sua solicitação de disco e manter o disco
    ocupado (relembrando a Figura 2.6 — múltiplos processos
    vinculados à E/S são necessários para manter a CPU
    completamente ocupada)
```

> ⚠️ **O escalonamento depende muito do contexto.** Um algoritmo que funciona bem em um notebook pode não funcionar tão bem em um data center — e em 10 anos, tudo pode ser diferente.

---

## ⏰ Quando Escalonar — os quatro momentos-chave

Uma questão fundamental é **quando** tomar decisões de escalonamento. Há uma série de situações:

```
┌────┬──────────────────────────────────────────────────────────┐
│ 1  │ Quando um NOVO PROCESSO é criado                          │
│    │ → decidir se o pai ou o filho deve executar               │
│    │ → ambos estão prontos → decisão de escalonamento normal   │
├────┼──────────────────────────────────────────────────────────┤
│ 2  │ Quando um processo TERMINA                                │
│    │ → esse processo não pode mais executar (não existe mais)  │
│    │ → outro deve ser escolhido do conjunto de prontos         │
│    │ → se nenhum está pronto, um processo ocioso do sistema    │
│    │   normalmente é executado                                 │
├────┼──────────────────────────────────────────────────────────┤
│ 3  │ Quando um processo é BLOQUEADO                             │
│    │ (para E/S, em um semáforo, ou por outra razão)            │
│    │ → outro processo precisa ser selecionado                  │
│    │ ⚠️ às vezes a RAZÃO do bloqueio importa na escolha:        │
│    │    se A espera que B saia da região crítica, deixar B     │
│    │    executar em seguida permite que A continue depois      │
│    │    Mas o escalonador geralmente NÃO tem essa informação   │
├────┼──────────────────────────────────────────────────────────┤
│ 4  │ Quando ocorre uma INTERRUPÇÃO de E/S                       │
│    │ → dispositivo completou seu trabalho                      │
│    │ → algum processo bloqueado pode estar pronto agora        │
│    │ → escalonador decide: executar o recém-pronto, o que      │
│    │   estava executando na interrupção, ou terceiro processo  │
└────┴──────────────────────────────────────────────────────────┘
```

---

## 🔁 Não Preemptivo vs. Preemptivo

Se um **hardware** de relógio fornece interrupções periódicas (a 50 ou 60 Hz, ou possivelmente maior), uma decisão de escalonamento pode ser feita a cada interrupção ou a cada *k*-ésima interrupção de relógio.

### Algoritmo não preemptivo

> 💡 **Escalonamento não preemptivo:** o algoritmo escolhe um processo para executar e então o deixa ser executado até que ele seja **bloqueado** (seja por E/S ou esperando por outro processo) ou **libere voluntariamente a CPU**. Mesmo que execute por muitas horas, não será suspenso forçosamente.

```
Nenhuma decisão de escalonamento é feita durante o
processamento de uma interrupção de relógio.

Após o processamento da interrupção de relógio ter concluído,
o processo que estava executando ANTES da interrupção é
retomado — a não ser que um processo mais prioritário
esteja aguardando por um tempo de espera agora satisfeito.
```

### Algoritmo preemptivo

> 💡 **Escalonamento preemptivo:** o algoritmo escolhe um processo e o deixa executar por, no máximo, um certo **tempo fixo**. Se ele ainda estiver executando ao final desse intervalo, é suspenso e o escalonador escolhe outro processo para executar (se houver algum disponível).

```
Realizar escalonamento preemptivo EXIGE que uma interrupção
de relógio ocorra ao final do intervalo, para devolver o
controle da CPU ao escalonador.

Se nenhum relógio estiver disponível, o escalonamento
não preemptivo é a ÚNICA solução possível.
```

### Preempção também importa para o núcleo, não só aplicações

```
A preempção NÃO é relevante apenas para aplicações,
mas também para NÚCLEOS de sistemas operacionais,
especialmente os monolíticos.

Hoje em dia muitos núcleos são preemptivos.

Se NÃO fossem preemptivos:
  → um driver mal-implementado
  → ou uma chamada de sistema muito lenta
  poderia MONOPOLIZAR a CPU inteira

Em um núcleo preemptivo, o escalonador pode FORÇAR o
driver ou a chamada de sistema de execução longa a
trocar de contexto.
```

---

## 🎯 Categorias de Algoritmos de Escalonamento

Diferentes ambientes (e diferentes tipos de sistemas operacionais) têm metas diversas — o que deve ser otimizado pelo escalonador NÃO é o mesmo em todos os sistemas. Três ambientes valem ser destacados:

```
1. LOTE (batch)
2. INTERATIVO
3. TEMPO REAL
```

### 1. Sistemas em Lote

```
Ainda bastante utilizados no mundo dos negócios para:
  → folhas de pagamento
  → contas a receber, contas a pagar
  → cálculos de juros (em bancos)
  → processamento de pedidos de indenização (seguros)
  → outras tarefas periódicas

Característica-chave: NÃO há usuários esperando
impacientemente em seus terminais por uma resposta
rápida a uma curta solicitação.

Consequência:
  → algoritmos NÃO preemptivos são aceitáveis
  → algoritmos preemptivos com longos períodos por
    processo também são aceitáveis
  → essa abordagem reduz as trocas de processos e
    melhora o desempenho
```

### 2. Sistemas Interativos

```
A preempção é ESSENCIAL — evita que um processo tome
conta da CPU e negue serviço aos outros.

Mesmo que nenhum processo execute intencionalmente
para sempre, um ERRO em um programa pode levar um
processo a impedir indefinidamente que todos os outros
executem. A preempção evita esse comportamento.

Servidores também caem nessa categoria — servem
múltiplos usuários (remotos) simultaneamente, todos
apressados.
```

### 3. Sistemas de Tempo Real

```
Por mais estranho que pareça, a preempção às vezes
NÃO é necessária:

  → os processos SABEM que não podem executar por
    longos períodos
  → em geral realizam seu trabalho e são bloqueados
    RAPIDAMENTE

Diferença com sistemas interativos:
  → sistemas de tempo real executam APENAS programas
    que visam ao progresso da aplicação em mãos
  → sistemas interativos são de propósito geral e
    podem executar programas ARBITRÁRIOS, não
    cooperativos, e talvez até maliciosos
```

---

## 🎯 Metas do Algoritmo de Escalonamento

Para projetar um bom algoritmo de escalonamento, é necessário ter clareza sobre o que ele deve otimizar. Certas metas dependem do ambiente (lote, interativo ou tempo real).

> 📌 **Figura 2.41 — Algumas metas do algoritmo de escalonamento sob diferentes circunstâncias**

```
┌──────────────────────────────────────────────────────────┐
│ TODOS OS SISTEMAS                                        │
├──────────────────────────────────────────────────────────┤
│ Imparcialidade    → dar a cada processo uma porção       │
│                      justa da CPU                        │
│ Aplicação da       → verificar se a política estabelecida│
│ política             é cumprida                          │
│ Equilíbrio        → manter ocupadas todas as partes      │
│                      do sistema                          │
├──────────────────────────────────────────────────────────┤
│ SISTEMAS EM LOTE                                         │
├──────────────────────────────────────────────────────────┤
│ Vazão (throughput) → maximizar o número de tarefas       │
│                       por hora                           │
│ Tempo de retorno   → minimizar o tempo entre a submissão │
│                       e o término                        │
│ Utilização de CPU  → manter a CPU ocupada o tempo todo   │
├──────────────────────────────────────────────────────────┤
│ SISTEMAS INTERATIVOS                                     │
├──────────────────────────────────────────────────────────┤
│ Tempo de resposta   → responder rapidamente às           │
│                        requisições                       │
│ Proporcionalidade   → satisfazer às expectativas dos     │
│                        usuários                          │
├──────────────────────────────────────────────────────────┤
│ SISTEMAS DE TEMPO REAL                                   │
├──────────────────────────────────────────────────────────┤
│ Cumprimento dos     → evitar a perda de dados            │
│ prazos                                                   │
│ Previsibilidade     → evitar a degradação da qualidade   │
│                        em sistemas multimídia            │
└──────────────────────────────────────────────────────────┘
```

### Detalhando as metas universais

**Imparcialidade (*fairness*):**
```
Processos COMPARÁVEIS devem receber serviços comparáveis.
Conceder a um processo muito mais tempo de CPU do que a
um processo equivalente NÃO é justo.

⚠️ Categorias diferentes de processos PODEM ser tratadas
diferentemente — pense em controle de segurança vs.
folha de pagamento em uma usina nuclear.
```

**Aplicação da política:**
```
Se a política local é que os processos de controle de
segurança são executados sempre que quiserem — mesmo que
isso signifique atraso de 30 segundos da folha de
pagamento — o escalonador precisa se certificar de que
essa política seja cumprida. Pode exigir esforço extra.
```

**Equilíbrio:**
```
Manter TODAS as partes do sistema ocupadas quando possível.
Se a CPU e todos os dispositivos de E/S podem ser mantidos
em execução o tempo inteiro, mais trabalho é realizado
por segundo do que se alguns componentes estivessem ociosos.

Exemplo em sistema em lote: o escalonador controla quais
tarefas são trazidas à memória. É melhor ter alguns
processos vinculados à CPU e alguns vinculados à E/S
juntos na memória do que carregar e executar todas as
tarefas vinculadas à CPU primeiro (fazendo o disco ficar
ocioso) e só depois carregar e executar as vinculadas à
E/S (fazendo a CPU ficar ociosa).
```

### Detalhando as três métricas de sistemas em lote

> 💡 **Vazão (*throughput*):** número de tarefas que o sistema completa por hora. Considerados todos os fatores, terminar 50 tarefas por hora é melhor que terminar 40 por hora.

> 💡 **Tempo de retorno (*turnaround time*):** o tempo médio (estatístico) desde o momento em que a tarefa em lote é submetida até o momento em que ela é concluída. Mede quanto tempo o usuário normal tem de esperar pela saída. Regra geral: **menos é mais**.

> ⚠️ **Vazão e tempo de retorno podem entrar em conflito:** um algoritmo que tenta maximizar a vazão talvez não minimize o tempo de retorno.

```
Exemplo: combinação de tarefas curtas e tarefas longas

Escalonador que SEMPRE executa tarefas curtas primeiro
e nunca as longas:
  → pode conseguir excelente vazão (muitas tarefas
    curtas por hora)
  → MAS causa tempo de retorno TERRÍVEL para as tarefas
    longas

Se tarefas curtas continuarem chegando a uma taxa
aproximadamente uniforme:
  → as tarefas longas talvez NUNCA sejam executadas
  → tempo de retorno médio se torna INFINITO
  → mesmo alcançando uma alta vazão
```

> 💡 **Utilização de CPU:** métrica frequentemente usada em sistemas em lote, mas **NÃO é uma boa métrica**. O que de fato importa é quantas tarefas por hora saem do sistema (vazão) e quanto tempo leva para receber uma tarefa de volta (tempo de retorno). Usar utilização de CPU como métrica é como **classificar carros com base no giro do motor**. No entanto, saber quando a utilização da CPU está próxima de 100% é útil para saber quando chegou o momento de obter mais poder computacional.

### Detalhando as metas de sistemas interativos

> 💡 **Tempo de resposta:** o tempo entre emitir um comando e receber o resultado. Em um computador pessoal onde um processo de segundo plano está sendo executado (ex: lendo e armazenando e-mail da rede), uma solicitação do usuário para abrir um arquivo ou começar um programa deve ter **precedência** sobre o trabalho de segundo plano. Atender primeiro todas as solicitações interativas será percebido como bom serviço.

> 💡 **Proporcionalidade:** usuários têm uma ideia inerente (porém muitas vezes incorreta) de quanto tempo as coisas "devem" levar.

```
Quando uma solicitação PERCEBIDA como complexa leva
muito tempo, o usuário aceita.

Quando uma solicitação PERCEBIDA como simples leva
muito tempo, o usuário fica irritado.

Exemplo — enviar um vídeo de 500 MB para um servidor
na nuvem:
  → demora 60 segundos → usuário provavelmente aceita
    (não espera que a transferência leve menos tempo)

Por outro lado, desconectar do servidor após o vídeo
já ter sido enviado:
  → usuário tem expectativas DIFERENTES
  → se a desconexão não estiver completa após 30s,
    o usuário provavelmente já estará impaciente
  → após 60s, "espumando de raiva"

Por quê a diferença? Percepção comum dos usuários de que
enviar um monte de dados SUPOSTAMENTE leva muito mais
tempo do que apenas desconectar uma conexão.

⚠️ Às vezes o escalonador NÃO pode fazer nada a
respeito do tempo de resposta percebido — mas em
outros casos, especialmente quando o atraso é
causado por uma escolha RUIM da ORDEM dos processos,
o escalonador pode ajudar.
```

### Detalhando as metas de sistemas de tempo real

> 💡 **Cumprimento dos prazos:** a principal exigência de um sistema de tempo real. Por exemplo, se um computador está controlando um dispositivo que produz dados a uma taxa regular, deixar de executar o processo de coleta de dados a tempo pode resultar em **perda de dados**.

> 💡 **Previsibilidade:** especialmente importante em sistemas de tempo real envolvendo multimídia. Descumprir um prazo ocasional não é fatal, mas se o processo de áudio executar de maneira errática demais, a qualidade do som deteriorará rapidamente. O vídeo também é uma questão, mas **o ouvido é muito mais sensível a atrasos do que o olho**. Para evitar esse problema, o escalonamento de processos deve ser altamente previsível e regular.

---

# ✅ Resumo do Conceito

- **Escalonamento** é necessário sempre que múltiplos processos/threads competem pela CPU simultaneamente no estado pronto
- Historicamente, a **relevância** do escalonamento mudou: em PCs simples com CPU abundante, importa pouco (exceto casos como vídeo em alta resolução); mas volta a importar muito em **servidores em rede** e em **IoT/dispositivos com bateria limitada**
- **Troca de contexto** é cara: salvar/restaurar registradores, recarregar a MMU, e invalidar caches — trocas frequentes desperdiçam tempo de CPU
- Processos alternam **surtos de CPU** com **esperas por E/S**; o que classifica um processo como **vinculado à CPU** ou **vinculado à E/S** é o comprimento do surto de CPU, não o da E/S
- Com CPUs cada vez mais rápidas em relação a discos, processos tendem a ficar mais **vinculados à E/S** — tornando o escalonamento desses processos mais importante
- Decisões de escalonamento ocorrem em **4 momentos**: criação de processo, término de processo, bloqueio de processo, e interrupção de E/S
- **Não preemptivo:** processo roda até bloquear ou ceder voluntariamente. **Preemptivo:** processo pode ser suspenso à força após um intervalo de tempo fixo — exige hardware de relógio
- Preempção importa também para o **núcleo do SO**, não só para aplicações — evita que drivers ou syscalls mal-comportados monopolizem a CPU
- Três categorias de sistemas com metas diferentes: **lote** (não preemptivo aceitável, vazão/tempo de retorno importam), **interativo** (preempção essencial, tempo de resposta/proporcionalidade importam), **tempo real** (preempção às vezes desnecessária, cumprimento de prazos/previsibilidade são críticos)
- Metas universais: **imparcialidade**, **aplicação da política**, **equilíbrio** entre componentes do sistema
- **Vazão** e **tempo de retorno** podem entrar em conflito — maximizar um pode prejudicar o outro; **utilização de CPU** não é boa métrica por si só

---

## 🔗 Notas Relacionadas

- [[Estados de Processos]] — os estados pronto/bloqueado/executando que o escalonador gerencia
- [[Modelando a Multiprogramação]] — o motivo pelo qual múltiplos processos vinculados à E/S mantêm a CPU ocupada
- [[Dispositivos de IO]] — contexto de hardware sobre a diferença de velocidade entre CPU e dispositivos de E/S
- [[Implementando Threads em Kernel Space]] — escalonamento de threads pelo núcleo, mencionado como próximo tópico do capítulo
