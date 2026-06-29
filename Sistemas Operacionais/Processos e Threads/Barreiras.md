---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.9"
---
# Barreiras

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.9

---

# 🚧 2.4.9 — Barreiras

## O problema que barreiras resolvem

Os mecanismos de sincronização anteriores — semáforos, mutexes, monitores, troca de mensagens — focam em **pares de processos** interagindo: produtor/consumidor, leitor/escritor, emissor/receptor.

Mas algumas aplicações têm um padrão diferente: **grupos de processos trabalhando em fases**.

```
Exemplo: simulação física em N processos paralelos
         (ex: temperatura de uma lâmina de metal ao longo do tempo)

Fase 1 (iteração n):
  Processo A calcula sua parte da matriz
  Processo B calcula sua parte da matriz
  Processo C calcula sua parte da matriz
  Processo D calcula sua parte da matriz

Fase 2 (iteração n+1):
  Cada processo precisa dos resultados de TODOS os outros
  para calcular os novos valores

Problema:
  Processo A termina rápido → quer avançar para a Fase 2
  Mas a Fase 2 usa dados do Processo D que ainda está na Fase 1!
  Se A avançar antes de D terminar → usa dados incompletos → resultado errado
```

A regra é clara: **nenhum processo deve prosseguir para a fase seguinte até que TODOS os processos estejam prontos**.

> 💡 **Barreira (*barrier*):** primitiva de sincronização colocada no fim de cada fase. Quando um processo atinge a barreira, ele é **bloqueado** até que todos os demais processos também tenham atingido a barreira. Só então todos são liberados simultaneamente para a próxima fase.

---

## 📊 Funcionamento — Figura 2.38

> 📌 **Figura 2.38 — Uso de uma barreira. (a) Processos aproximando-se de uma barreira. (b) Todos os processos, exceto um, bloqueados na barreira. (c) Quando o último processo chega à barreira, todos podem passar.**

```
(a) Processos aproximando-se da barreira — ainda computando:

  Processo A ──────────────────────────────────────►|
  Processo B ──────────────────────────────►        | BARREIRA
  Processo C ──────────────────────────────────────►|
  Processo D ──────────────────────►                |

                          Tempo ──────────────────►

(b) A, B e C chegaram — bloqueados esperando D:

  Processo A ──────────────────────────────────────►| ⏸ BLOQUEADO
  Processo B ──────────────────────────────►────────| ⏸ BLOQUEADO
  Processo C ──────────────────────────────────────►| ⏸ BLOQUEADO
  Processo D ──────────────────────►────────────────| computando...

(c) D chega — TODOS liberados simultaneamente:

  Processo A ────────────────────────────────────►|──────────────►
  Processo B ──────────────────────────────►──────|──────────────►
  Processo C ────────────────────────────────────►|──────────────►
  Processo D ──────────────────────►──────────────|──────────────►
                                                  
                                            todos avançam ✅
```

O processo executa a primitiva `barrier` ao terminar sua parte da iteração atual. Se não for o último, é suspenso. Quando o último processo finalmente atinge a barreira, **todos são liberados simultaneamente** para começar a próxima fase.

---

## 🔧 Barreiras de memória — um tipo diferente

Além das barreiras de sincronização entre processos, há outro tipo relacionado mas com propósito diferente:

> 💡 **Barreira de memória (*memory barrier* ou *memory fence*):** instrução especial de CPU que garante que **todas as operações de memória iniciadas antes da barreira sejam concluídas antes** que qualquer operação de memória posterior à barreira seja iniciada.

Por que isso é necessário? CPUs modernas são **superescalares** — executam instruções **fora de ordem** para maximizar o uso das unidades de execução. Isso pode causar problemas em código concorrente:

```
THREAD 1:                    THREAD 2:
while (turn != 1) { }        x = 100;
printf("%d\n", x);           turn = 1;

Situação esperada com turn == 0 inicial:
  Thread 2 executa: x = 100, depois turn = 1
  Thread 1 sai do loop, imprime 100 ✅

Problema — execução fora de ordem na Thread 2:
  CPU pode executar turn = 1 ANTES de x = 100
  (se as instruções não dependem uma da outra, a CPU pode reordenar)
  
  Thread 1 vê turn = 1, sai do loop
  Lê x → mas x ainda não foi atualizado!
  Imprime valor lixo ❌
```

A solução é inserir uma instrução de barreira de memória entre as duas linhas da Thread 2:

```
THREAD 2 corrigida:
  x = 100;
  [MEMORY BARRIER]   ← garante que x=100 chegou à memória
  turn = 1;          ← só depois disso turn é atualizado
```

> ⚠️ **Barreiras de memória e vulnerabilidades de execução transitória:** as barreiras de memória desempenham papel importante na mitigação de vulnerabilidades de CPU conhecidas como **vulnerabilidades de execução transitória** — como **Meltdown** e **Spectre**, descobertas em 2018. Invasores podem explorar o fato de que CPUs executam instruções fora de ordem para vazar informações sensíveis. As correções geralmente envolvem inserir barreiras de memória estratégicas para impedir que a CPU execute especulativamente instruções que cruzam limites de segurança. Tanenbaum examina ataques de execução transitória em detalhes no Capítulo 9.

---

## 🌐 Onde barreiras são usadas na prática

```
MPI (Message Passing Interface):
  MPI_Barrier()  → sincroniza todos os processos de um comunicador
  Amplamente usado em clusters e supercomputadores
  
  Exemplo: simulação climática em 10.000 nós
    Cada nó calcula sua fatia do globo terrestre
    MPI_Barrier() garante que todos terminem antes de trocar dados
    Próxima iteração começa com dados completos de todos os nós

CUDA (computação em GPU):
  __syncthreads()  → barreira entre threads do mesmo bloco
  Fundamental para algoritmos paralelos corretos na GPU
  
  Exemplo: soma paralela (reduction)
    Cada thread some parte dos dados na memória compartilhada
    __syncthreads() garante que todas terminaram
    Próxima fase combina os resultados parciais

OpenMP (paralelismo em CPU):
  #pragma omp barrier  → sincroniza threads OpenMP
  Usado em loops paralelos com dependências entre fases

Hardware (barreiras de memória):
  x86:  MFENCE (memory fence), LFENCE (load), SFENCE (store)
  ARM:  DMB (Data Memory Barrier), DSB (Data Synchronization Barrier)
  Usadas pelo compilador e pelo programador em código de baixo nível
```

---

## 📐 Exemplo concreto — simulação de lâmina de metal

Tanenbaum usa o exemplo de relaxamento físico para ilustrar onde barreiras são necessárias:

```
Problema: calcular como o calor se propaga por uma lâmina de metal

Dados: matriz N×N de temperaturas nos pontos de amostragem
       (valores iniciais conhecidos nas bordas, desconhecidos no interior)

Algoritmo iterativo:
  Para cada ponto (i,j) no interior:
    nova_temp[i][j] = média dos 4 vizinhos da iteração anterior

Paralelizando com P processos:
  Cada processo cuida de N/P linhas da matriz
  
  ITERAÇÃO n:
    Processo 0: calcula linhas 0 a N/P-1
    Processo 1: calcula linhas N/P a 2N/P-1
    ...
    Processo P-1: calcula linhas (P-1)N/P a N-1
    
    [BARRIER]  ← nenhum avança sem que todos terminem
    
  ITERAÇÃO n+1:
    Cada processo precisa dos resultados dos vizinhos (outras linhas)
    Como todos passaram pela barreira, os dados estão completos ✅
    
    Processo 0: calcula suas linhas usando valores atualizados
    ...
    
    [BARRIER]  ← próxima sincronização

  Repete até convergir
```

---

# ✅ Resumo do Conceito

- **Barreira de sincronização:** primitiva para grupos de processos trabalhando em fases — nenhum processo avança até que **todos** tenham chegado à barreira; todos são liberados simultaneamente
- Diferente das primitivas anteriores (que coordenam pares), barreiras coordenam **grupos inteiros**
- Uso clássico: algoritmos iterativos paralelos onde cada iteração depende dos resultados completos da iteração anterior (simulações físicas, processamento de matrizes)
- **Barreira de memória** (*memory fence*) é um conceito diferente: instrução de CPU que garante ordenação das operações de memória, necessária porque CPUs modernas executam instruções fora de ordem — importante para corretude em código concorrente de baixo nível
- Barreiras de memória também são relevantes para mitigação de vulnerabilidades de execução transitória como Meltdown e Spectre (Capítulo 9)
- Implementações práticas: `MPI_Barrier()` em computação distribuída, `__syncthreads()` em CUDA, `#pragma omp barrier` em OpenMP, `MFENCE`/`DMB` em hardware

---

## 🔗 Notas Relacionadas

- [[Troca de Mensagens]] — mecanismo anterior na sequência do capítulo; barreiras complementam a troca de mensagens em sistemas paralelos
- [[Semáforos]] — semáforos podem ser usados para implementar barreiras manualmente
- [[Race Condition]] — execução fora de ordem sem barreiras de memória é uma fonte de races em baixo nível
- [[Exclusão mútua com espera ocupada]] — barreiras de memória são implementadas com instruções especiais de hardware similares ao TSL/XCHG
