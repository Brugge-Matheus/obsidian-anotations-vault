---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.1 e 2.1.1"
---
# O modelo de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.1 e 2.1.1

---

# 🔄 2.1 — Processos

Todos os computadores modernos realizam várias tarefas ao mesmo tempo. O conceito mais central em qualquer sistema operacional é o **processo** — a abstração de um programa em execução. Tudo o mais depende desse conceito, e o projetista (e estudante) do SO deve ter uma compreensão profunda do que é um processo o mais cedo possível.

> 💡 **Processo:** uma abstração de um programa em execução. Inclui não apenas o código do programa, mas também os valores atuais do contador de programa, registradores, pilha, heap e o estado geral da computação em andamento. Não confunda *programa* (arquivo passivo em disco) com *processo* (entidade ativa em execução na memória).
> 

Processos são uma das abstrações mais antigas e importantes que os SOs oferecem. Eles dão suporte à possibilidade de haver operações **(pseudo)concorrentes** mesmo quando há apenas uma CPU disponível, transformando uma única CPU em múltiplas CPUs virtuais.

## Por que processos são necessários?

Dois exemplos ilustram a necessidade:

- Um **servidor web** recebe requisições de páginas de toda parte. Quando uma solicitação chega, o servidor verifica se a página está em *cache*. Se não estiver, uma solicitação de acesso ao disco é iniciada — e isso leva uma eternidade do ponto de vista da CPU. Enquanto espera, muitas outras solicitações podem chegar. Sem processos, a CPU ficaria ociosa esperando o disco.
- Um **PC de usuário** inicializa secretamente dezenas de processos ao ligar — agentes de antivírus, sincronização de arquivos, clientes de e-mail — tudo enquanto o usuário navega na web. Toda essa atividade precisa ser gerenciada, e um sistema de multiprogramação que dê suporte a múltiplos processos é essencial.

> 💡 **Pseudoparalelismo:** ilusão de execução simultânea criada pela alternância rápida da CPU entre processos, em sistemas com apenas uma CPU disponível. É o termo usado para diferenciar do verdadeiro paralelismo de *hardware* dos sistemas **multiprocessadores**.
> 

> 💡 **Multiprocessadores:** sistemas com duas ou mais CPUs físicas compartilhando a mesma memória. Neles ocorre **paralelismo real** — múltiplos processos executam genuinamente ao mesmo tempo. Tanenbaum examina chips multinúcleos (*multicore*) com mais profundidade no Capítulo 8.
> 

---

# 🧠 2.1.1 — O Modelo de Processo

Nesse modelo, todos os *softwares* executáveis no computador — às vezes incluindo o próprio SO — são organizados em uma série de **processos sequenciais**.

> 💡 **Processo sequencial (ou simplesmente *processo*):** instância de um programa em execução. Cada processo tem sua própria **CPU virtual** — isto é, seu próprio contador de programa lógico, registradores e fluxo de controle independente dos demais.
> 

## A CPU virtual e a multiprogramação

Na realidade, há apenas **uma CPU física** (ou poucas, em sistemas multicore). O SO alterna rapidamente entre os processos — cada um executado por dezenas ou centenas de milissegundos — criando a ilusão de paralelismo.

> 💡 **Multiprogramação:** mecanismo pelo qual a CPU troca de processo em processo rapidamente. Cada processo avança aos poucos, mas, em qualquer dado instante, apenas **um** está sendo de fato executado. Ao longo do tempo, a CPU é compartilhada por todos.
> 

Conceitualmente, cada processo tem sua própria CPU virtual. Na verdade, a CPU real troca a todo momento de processo em processo — mas para compreender o sistema, é muito mais fácil pensar a respeito de uma coleção de processos sendo executados em (pseudo)paralelo do que tentar acompanhar como a CPU troca de um programa para o outro.

> ⚠️ **Atenção:** a taxa de execução de um processo **não é uniforme nem reproduzível** entre execuções. Processos não devem ser programados com suposições rígidas sobre tempo de CPU absoluto — o escalonador (scheduler) pode trocar o processo a qualquer momento. Exemplo: um processo de áudio que faz um laço ocioso 10.000 vezes antes de executar funcionará bem com uma CPU dedicada, mas pode falhar gravemente se a CPU for trocada durante o laço.
> 

## Figura 2.1 — Multiprogramação de quatro processos

> 📌 **Figura 2.1:** (a) Multiprogramação de quatro programas. (b) Modelo conceitual de quatro processos sequenciais independentes. (c) Apenas um programa está ativo de cada vez.
> 

```
(a) Um contador de programa       (b) Quatro contadores de programa      (c) Linha do tempo

  ┌──────────────────┐              ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │   Programa  A    │              │ A │  │ B │  │ C │  │ D │         D ─────
  ├──────────────────┤              │ ↕ │  │ ↕ │  │ ↕ │  │ ↕ │    Processo
  │   Programa  B    │              └───┘  └───┘  └───┘  └───┘         C ─────
  ├──────────────────┤                                                   B ─────
  │   Programa  C    │            Cada processo possui seu               A ─────
  ├──────────────────┤            próprio PC lógico salvo          Tempo ──────►
  │   Programa  D    │            em memória
  └──────────────────┘
 CPU física executa apenas         Quando o processo retoma         Analisado num intervalo
 um programa por vez, mas          a CPU, seu PC real é             longo, todos avançaram.
 salva e restaura o PC de          restaurado a partir do           Em qualquer instante,
 cada processo                     PC lógico salvo                  só 1 está ativo.
```

- **(a)**: Quatro programas coexistem na memória. Há apenas **um contador de programa físico** — a CPU executa um por vez.
- **(b)**: Cada processo tem seu próprio **contador de programa lógico**. Quando o processo é suspenso, o PC físico é salvo em memória; quando retoma, é restaurado.
- **(c)**: Numa janela longa de tempo, todos os processos avançaram. Mas em qualquer instante dado, apenas **um** está sendo executado de fato.

## Processo ≠ Programa

A diferença é sutil, mas criticamente importante. Tanenbaum usa uma analogia:

> *Um cientista de computação prepara um bolo de aniversário. Ele é o processador (CPU); a receita é o programa; os ingredientes são os dados de entrada. O processo é a atividade de ler a receita, buscar os ingredientes e preparar o bolo.*
> 

> 
> 

> *Se o filho aparece chorando com uma picada de abelha, o cientista registra onde estava na receita (salva o estado do processo atual), pega o livro de primeiros socorros (outro programa, de maior prioridade) e segue as orientações. Aqui vemos o processador sendo trocado de um processo para outro. Quando o filho estiver cuidado, o cientista volta ao bolo do ponto exato onde parou.*
> 

|  | Programa | Processo |
| --- | --- | --- |
| **Natureza** | Passivo — arquivo em disco | Ativo — em execução na memória |
| **Componentes** | Código (algoritmo) e dados estáticos | Código + dados + PC + registradores + pilha + heap |
| **Instâncias** | Um programa pode gerar N processos | Cada processo é uma instância independente |
| **Exemplo** | Arquivo `firefox` no disco | Duas janelas do Firefox abertas = dois processos |
| **Persistência** | Permanece no disco indefinidamente | Existe apenas enquanto está em execução |

> ⚠️ Se o mesmo programa for iniciado duas vezes, existem **dois processos distintos** — mesmo que o SO compartilhe o código em memória (otimização). O fato de dois processos em execução estarem operando o mesmo programa não importa; eles são processos distintos. O SO pode ser capaz de compartilhar o código entre eles de maneira que apenas uma cópia esteja na memória, mas isso é um detalhe técnico que não muda a situação conceitual.
> 

## Pseudoparalelismo vs. Paralelismo real

| Conceito | Descrição | Quando ocorre |
| --- | --- | --- |
| **Pseudoparalelismo** | Uma única CPU alterna entre processos; ilusão de simultaneidade | Sistemas com 1 CPU (ou 1 núcleo em uso) |
| **Paralelismo real** | Múltiplas CPUs executam processos genuinamente ao mesmo tempo | Sistemas **multiprocessadores** (≥ 2 CPUs compartilhando memória) |

## Processos de primeiro e segundo plano

Quando um SO é inicializado, em geral uma série de processos é criada automaticamente:

- **Primeiro plano (*foreground*):** processos que interagem diretamente com usuários e realizam trabalho para eles.
- **Segundo plano (*background*):** processos não associados a usuários específicos, mas com alguma função definida. Um processo de segundo plano pode aceitar *e-mails* chegando, ficando inativo a maior parte do dia e entrando em ação quando um *e-mail* chega.

> 💡 **Daemon:** processo de segundo plano que aguarda eventos (chegada de e-mail, requisição web, tarefa agendada) para executar sua função. Grandes sistemas possuem normalmente dezenas deles. Em sistemas UNIX, o programa `ps` lista os processos em execução; no Windows, o equivalente é o Gerenciador de Tarefas.
> 

---

# ✅ Resumo do Conceito

- **Processo** — abstração de um programa em execução; inclui código, dados, PC, registradores, pilha e heap. É a abstração mais central de qualquer SO
- **CPU virtual** — cada processo age como se tivesse a CPU para si, graças à alternância rápida gerenciada pelo SO
- **Multiprogramação** — mecanismo que alterna a CPU entre processos rapidamente, criando pseudoparalelismo; em qualquer instante dado, apenas um processo está ativo de fato
- **Pseudoparalelismo** — ilusão de execução simultânea em sistemas de uma CPU; oposto do paralelismo real de multiprocessadores
- **Paralelismo real** — execução verdadeiramente simultânea em sistemas com múltiplas CPUs físicas
- **Processo ≠ Programa** — programa é passivo (arquivo em disco); processo é ativo (em execução); o mesmo programa pode originar múltiplos processos independentes
- **Daemon** — processo de segundo plano aguardando eventos para executar; sistemas UNIX possuem dezenas deles
- **Taxa de execução não reproduzível** — processos não devem assumir velocidade constante; o escalonador pode interrompê-los a qualquer momento