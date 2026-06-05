---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.5.1"
---
# Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.5.1

---

# 🔄 O que é um Processo?

Um processo é um dos conceitos mais fundamentais em sistemas operacionais. A definição mais simples é:

> **Um processo é basicamente um programa em execução.**
> 

Mas essa definição merece expansão. Um programa é apenas um arquivo estático no disco — uma sequência de instruções. Quando esse programa é carregado na memória e começa a ser executado, ele se torna um processo. A diferença é como a diferença entre uma receita de bolo (programa) e o ato de fazer o bolo (processo).

---

# 🏗️ O que Compõe um Processo?

Associado a cada processo há um conjunto completo de recursos:

## Espaço de Endereçamento

Cada processo tem seu próprio **espaço de endereçamento** — uma lista de posições de memória que vai de 0 a algum máximo, onde o processo pode ler e escrever. Esse espaço contém:

```
┌─────────────────────────────┐  ← endereço máximo
│         Pilha               │  → variáveis locais, parâmetros de funções,
│    (cresce para baixo)      │    endereços de retorno
├─────────────────────────────┤
│           ↕                 │  → espaço livre entre pilha e heap
├─────────────────────────────┤
│         Heap                │  → memória alocada dinamicamente
│    (cresce para cima)       │    (malloc, new, etc.)
├─────────────────────────────┤
│    Dados (variáveis         │  → variáveis globais e estáticas
│    globais e estáticas)     │
├─────────────────────────────┤
│    Código (texto)           │  → as instruções do programa
│    (programa executável)    │
└─────────────────────────────┘  ← endereço 0
```

---

# 🧠 Entendendo Melhor: Pilha, Heap e Memória do Processo

## A Pilha é o Contexto das Funções em Execução

A pilha representa o **estado atual das funções que estão rodando** — quem chamou quem, com quais valores, e para onde voltar. É a "história de chamadas" do programa naquele momento:

```
Pilha em um dado momento:

┌──────────────────┐  ← topo (SP aponta aqui)
│  abrirDisco()    │  ← função mais recente
│  - setor = 1024  │
├──────────────────┤
│  lerArquivo()    │  ← chamou abrirDisco
│  - nome = "a.txt"│
├──────────────────┤
│  main()          │  ← chamou lerArquivo
│  - argc = 1      │
└──────────────────┘

Quando abrirDisco() terminar → sua parte some automaticamente
Quando lerArquivo() terminar → sua parte some automaticamente
```

O contexto completo do processo inclui a pilha **mais** os registradores (PC, SP, PSW). A pilha guarda especificamente o contexto das funções ativas.

---

## A Pilha Fica na RAM

A pilha **fica na RAM**, assim como heap, código e dados. Todos são regiões dentro do espaço de endereçamento virtual do processo, mapeadas pela MMU para endereços físicos na RAM.

A confusão é natural porque os **registradores da CPU** participam — o SP (Stack Pointer) aponta para o topo da pilha — mas os dados em si ficam na RAM. O registrador é apenas o ponteiro que diz onde na RAM está o topo:

```
CPU:                         RAM:
┌────────┐                   ┌──────────────────┐
│  SP    │ ──────────────▶   │ topo da pilha    │ ← dados ficam aqui
│ 0x7FFF │                   │ variável local   │
└────────┘                   │ parâmetro        │
                             └──────────────────┘
```

---

## Código no Espaço de Endereçamento e malloc no Heap

Todas as regiões do espaço de endereçamento — código, dados, heap e pilha — são endereços virtuais do processo. Quando o processo chama `malloc()`, o fluxo é:

```
Seu código (região de Código):
  int* p = malloc(100);
          ↓
  malloc() → função da libc (também na região de Código)
          ↓
  chama brk() ou mmap() → system call ao SO
          ↓
  SO atualiza tabelas da MMU
          ↓
  retorna endereço virtual alocado no Heap
          ↓
  p aponta para uma região no Heap
```

O código do `malloc` fica na região de **Código**. O que ele gerencia é o **Heap**. São duas regiões separadas do mesmo espaço de endereçamento — o código é as instruções, o heap é onde os dados dinâmicos ficam armazenados.

---

## O Heap Representa a Memória Dinâmica do Processo

O Heap começa pequeno e cresce conforme o processo pede memória em tempo de execução — via `malloc`, `new`, ou automaticamente por um garbage collector. Em termos práticos, **quando um processo "consome mais memória" ao longo do tempo, é principalmente o Heap crescendo**.

```
Memória total do processo:
┌──────────────────────────────────┐
│ Pilha      → temporária          │ → tamanho relativamente estável
├──────────────────────────────────┤
│ Heap       → dinâmica            │ → CRESCE conforme o programa
│                                  │   pede mais memória em execução
│                                  │   é aqui que o "consumo de
│                                  │   memória" aumenta
├──────────────────────────────────┤
│ Dados globais → fixo             │ → definido na compilação
├──────────────────────────────────┤
│ Código        → fixo             │ → não muda em execução
└──────────────────────────────────┘
```

---

## free() Não Devolve Memória ao SO — Marca Como Disponível

Quando você chama `free(ptr)`, o comportamento padrão **não é devolver a memória ao SO** — é marcar aquela região do Heap como disponível para uso futuro pelo próprio processo:

```
malloc(100)  → Heap cresce, SO entrega memória
malloc(200)  → Heap cresce mais
free(ptr1)   → malloc marca aquela região como livre
               MAS não devolve ao SO
malloc(50)   → malloc reutiliza a região já disponível
               sem precisar pedir ao SO novamente
```

Os motivos são **performance e fragmentação**:

**Performance** — pedir memória ao SO via system call é caro. Se o malloc devolvesse a cada `free()` e o programa pedisse de novo logo em seguida, haveria um ciclo desnecessário de system calls. Manter uma reserva interna é muito mais eficiente.

**Fragmentação** — depois de várias alocações e liberações de tamanhos diferentes, a memória fica fragmentada em buracos:

```
Heap após várias alocações e liberações:

[LIVRE 50b][USADO 200b][LIVRE 30b][USADO 100b][LIVRE 80b]

malloc(120) → nenhum buraco tem 120b contíguos!
→ precisa pedir mais memória ao SO mesmo com
  160b livres espalhados pelo Heap
```

Devolver memória fragmentada ao SO seria inútil — a maioria dos sistemas operacionais só consegue receber de volta **páginas inteiras e contíguas**. Por isso o malloc prefere manter o controle internamente.

> ⚠️ **Implementações modernas** às vezes devolvem memória ao SO quando blocos livres grandes e contíguos se acumulam no final do Heap — mas isso é uma otimização pontual, não o comportamento padrão a cada `free()`.
> 

---

## Quando Acontece Stack Overflow?

Stack overflow acontece quando a pilha cresce além do seu limite — tipicamente **1 a 8 MB** dependendo do SO. O que causa stack overflow é o **tamanho total dos dados na pilha** — calculado como quantidade de elementos × tamanho do tipo em bytes:

```c
// NÃO causa stack overflow — int é sempre 4 bytes independente do valor
int x = 999999999;

// CAUSA stack overflow — cálculo do tamanho total:
// int array[10000000] → 10.000.000 × 4 bytes = 40 MB na stack → overflow!
// short array[10000000] → 10.000.000 × 2 bytes = 20 MB → overflow!
// double array[10000000] → 10.000.000 × 8 bytes = 80 MB → overflow!
void funcao() {
    int array[10000000];  // 40 MB → stack overflow imediato
}

// SOLUÇÃO CORRETA — array grande vai no Heap
void funcao() {
    int* array = malloc(10000000 * sizeof(int));  // 40 MB no Heap → ok
    free(array);
}

// Recursão infinita — caso mais comum de stack overflow
void recursao() {
    int x = 1;     // 4 bytes a cada chamada
    recursao();    // com ~250.000 chamadas → stack overflow
}
```

O número de elementos isoladamente não é o problema — é sempre o produto **quantidade × tamanho do tipo** que define o espaço ocupado na stack. Por isso sempre que precisar de um array grande, independente do tipo dos elementos, a solução é alocar no Heap via `malloc`.

## Tipos Têm Limite de Valor — Overflow Aritmético vs. Stack Overflow

Um ponto importante: **tipos inteiros têm um intervalo máximo de valores** — não é possível guardar números infinitos em 4 bytes. O número de combinações possíveis é sempre 2ᴺ (onde N é o número de bits):

```
short  (16 bits): com sinal → -32.768 até 32.767
int    (32 bits): com sinal → -2.147.483.648 até 2.147.483.647
long   (64 bits): com sinal → -9.223.372.036.854.775.808 até 9.223.372.036.854.775.807
```

Existem dois tipos de overflow completamente diferentes que é importante não confundir:

```c
// OVERFLOW ARITMÉTICO — valor estoura o limite do tipo
int x = 2147483647;  // valor máximo de int
x = x + 1;           // overflow! vira -2147483648
                     // não causa crash, mas dá resultado errado

// STACK OVERFLOW — pilha estoura o limite de memória
int array[10000000]; // 40 MB na pilha → crash garantido
```

|  | Overflow Aritmético | Stack Overflow |
| --- | --- | --- |
| **O que estoura** | O valor máximo do tipo | O limite de memória da pilha |
| **Causa** | Número grande demais para o tipo | Dados grandes demais para a stack |
| **Resultado** | Valor incorreto (wrap around) | Crash do programa |
| **Exemplo** | `int x = MAX_INT + 1` | `int arr[10000000]` |

Para números verdadeiramente grandes — maiores que qualquer tipo primitivo suporta — é necessário usar bibliotecas de **big integer** que alocam no Heap conforme necessário. Python resolve isso nativamente — o tipo `int` em Python cresce no Heap automaticamente conforme o número aumenta:

```python
# Python: int cresce no Heap automaticamente
x = 99999999999999999999999999999999  # funciona! sem limite prático

# C: precisa de biblioteca externa (GMP) para números grandes
mpz_t x;
mpz_init(x);  # aloca no Heap
mpz_set_str(x, "99999999999999999999999999999999", 10);
```

de Memória são Conceitos Separados

Uma confusão comum é achar que "variável local = stack" sempre. Mas **escopo e localização são coisas diferentes**:

- **Escopo** → define por quanto tempo a variável existe no programa
- **Localização** → define onde na memória ela fica armazenada

Uma variável pode ser **local em escopo mas viver no Heap**:

```c
void funcao() {
    // ptr é local em escopo — mas os DADOS estão no Heap
    int* ptr = malloc(500000 * sizeof(int));
    //   ↑                ↑
    //   stack (8 bytes)   heap (2 MB)

    // usa ptr...
    free(ptr);  // sem isso → memory leak!
}              // ptr some da stack, mas os dados no Heap
               // só somem com o free()
```

Quando a função termina, o ponteiro `ptr` some da stack. Mas os dados alocados no Heap **continuam lá** até alguém chamar `free()`. Se esquecer o `free()`, os dados ficam presos no Heap enquanto o processo rodar — isso é um **memory leak**.

---

## Por que o Heap é Obrigatório Quando o Tamanho é Desconhecido

Além do tamanho grande, existe outro motivo fundamental para usar o Heap: quando o **tamanho só é conhecido em tempo de execução**, não na compilação.

Imagine uma função que recebe um parâmetro `n` e precisa alocar um array desse tamanho:

```c
void funcao(int n) {
    // n só é conhecido quando a função é chamada
    // a função pode ser chamada com n=10 ou n=500.000
    // não tem como alocar na stack sem saber o tamanho
    // exato em tempo de compilação

    int* arr = malloc(n * sizeof(int));  // Heap — obrigatório
    // usa arr...
    free(arr);
}
```

Isso é fundamental: a mesma função pode ser chamada diversas vezes com valores de `n` completamente diferentes — cada chamada alocando um tamanho diferente no Heap. A stack não conseguiria lidar com isso porque ela precisa saber o tamanho **antes** de executar.

As duas razões para usar o Heap são portanto:

| Motivo | Exemplo |
| --- | --- |
| **Dado grande demais para a stack** | Array de 500.000 elementos |
| **Tamanho desconhecido na compilação** | Array cujo tamanho vem de um parâmetro ou input do usuário |

---

## Registradores

Cada processo tem associado um conjunto de registradores, incluindo:

- **Contador de programa (PC)** — aponta para a próxima instrução a executar
- **Ponteiro de pilha (SP)** — aponta para o topo da pilha atual
- Registradores gerais com variáveis e resultados temporários

## Outros Recursos

- Lista de **arquivos abertos** — com ponteiros indicando a posição atual de leitura/escrita em cada arquivo
- **Alarmes pendentes**
- Listas de **processos relacionados**
- Todas as demais informações necessárias para executar o programa

> 💡 Um processo é, na essência, um **contêiner** que armazena todas as informações necessárias para executar um programa. É como uma "instância" do programa em execução — você pode ter dois processos rodando o mesmo programa simultaneamente, cada um com seu próprio espaço de endereçamento, registradores e recursos independentes.
> 

---

# 🌐 Multiprogramação e Processos

A razão pela qual o conceito de processo é tão importante é a **multiprogramação** — a capacidade do SO de ter múltiplos processos ativos ao mesmo tempo.

**Exemplo prático:** você pode ter simultaneamente:

- Um editor de vídeo convertendo um arquivo de 2 horas (processo pesado, em segundo plano)
- Um navegador web aberto (processo interativo)
- Um cliente de email checando mensagens (processo periódico)

Periodicamente, o SO decide parar de executar um processo e começar a executar outro — talvez porque o primeiro utilizou mais do que sua parcela de tempo da CPU no último segundo. Isso é o **timesharing** que já estudamos.

---

# ⏸️ Suspensão e Retomada de Processos

Quando um processo é **suspenso temporariamente**, ele deve ser reiniciado mais tarde exatamente no mesmo estado em que estava quando foi parado. Isso significa que todas as informações sobre o processo precisam ser explicitamente salvas.

Por exemplo, um processo pode ter vários arquivos abertos para leitura ao mesmo tempo, com um ponteiro associado a cada um indicando a posição atual (o número do byte a ser lido em seguida). Quando o processo é suspenso, todos esses ponteiros precisam ser salvos para que, quando retomado, as leituras continuem do ponto correto.

---

# 📋 Tabela de Processos

Em muitos sistemas operacionais, todas as informações sobre cada processo — exceto o conteúdo do seu próprio espaço de endereçamento — são armazenadas em uma estrutura chamada **tabela de processos**. É um arranjo de estruturas, uma para cada processo existente no momento.

```
Tabela de Processos (simplificada):

┌────────┬────────────┬────────┬──────────┬───────────┬─────────┐
│ PID    │ Estado     │ PC     │ SP       │ Arquivos  │ UID     │
│        │            │        │          │ abertos   │         │
├────────┼────────────┼────────┼──────────┼───────────┼─────────┤
│ 1001   │ executando │ 0x4A20 │ 0x7FFF   │ [0,1,2,5] │ 1000    │
│ 1002   │ bloqueado  │ 0x9F10 │ 0x6EFF   │ [0,1,3]   │ 1001    │
│ 1003   │ pronto     │ 0x3B80 │ 0x5DCC   │ [0,1]     │ 1000    │
└────────┴────────────┴────────┴──────────┴───────────┴─────────┘
```

Um processo suspenso consiste em:

- Seu **espaço de endereçamento** — chamado de **imagem do núcleo** (*core image*, em homenagem às memórias de núcleo magnético usadas antigamente)
- Sua **entrada na tabela de processos** — com os registradores e outros itens necessários para reiniciar o processo

---

# 👶 Processos Filhos e Árvore de Processos

Um processo pode criar um ou mais **processos-filhos** (*child processes*), que por sua vez podem criar seus próprios processos-filhos, formando uma **árvore de processos**.

> 📌 **Figura 1.13 — Árvore de processos**
> 

```
              ┌─────┐
              │  A  │  ← processo pai
              └──┬──┘
        ┌────────┴────────┐
     ┌──┴──┐           ┌──┴──┐
     │  B  │           │  C  │  ← processos-filhos de A
     └──┬──┘           └─────┘
   ┌────┴────┐
┌──┴──┐   ┌──┴──┐   ┌──┴──┐
│  D  │   │  E  │   │  F  │  ← processos-filhos de B
└─────┘   └─────┘   └─────┘

O processo A criou dois processos-filhos, B e C.
O processo B criou três processos-filhos, D, E e F.
```

**Exemplo concreto com shell:** quando um usuário digita um comando no terminal, o **shell** (interpretador de comandos) cria um novo processo para executar aquele comando. Quando o processo termina, ele executa uma chamada de sistema para se autofinalizacionar, e o controle volta ao shell.

Processos relacionados que cooperam para finalizar alguma tarefa muitas vezes precisam se **comunicar entre si e sincronizar suas atividades** — isso é chamado de **comunicação entre processos** (*interprocess communication*) e será analisado detalhadamente no Capítulo 2.

---

# 👤 UIDs, GIDs e Superusuário

## UID (User Identification)

Cada pessoa autorizada a usar um sistema é designada uma **UID** (*User Identification* — identificação do usuário) pelo administrador do sistema. Todo processo iniciado tem a UID da pessoa que o iniciou. Um processo-filho tem a mesma UID que seu processo-pai.

## GID (Group Identification)

Usuários podem ser membros de grupos, cada qual com uma **GID** (*Group Identification* — identificação do grupo). Grupos permitem compartilhar recursos entre conjuntos de usuários.

## Superusuário / Root / Administrador

Uma UID especial, chamada de **superusuário** (ou **root** no UNIX, ou **administrador** no Windows), tem poder especial e pode passar por cima de muitas das regras de proteção do sistema.

- Em grandes instalações, apenas o administrador do sistema conhece a senha necessária
- Muitos usuários comuns — especialmente estudantes — dedicam esforço considerável tentando encontrar falhas no sistema que permitam tornar-se superusuários sem a senha

---

# ⏰ Sinais de Alarme

Há ocasionalmente necessidade de transmitir informação para um processo em execução que não está parado esperando por ela. Por exemplo, um processo comunicando-se com outro em um computador diferente envia mensagens — e pode haver perda de mensagem ou resposta.

Para lidar com isso, o processo pode pedir ao SO para notificá-lo após um número especificado de segundos — configurando um **temporizador**. Quando o número de segundos expira, o SO envia um **sinal de alarme** para o processo:

```
Processo em execução normalmente...
        ↓
Temporizador expira (ou sinal chega)
        ↓
SO suspende o processo
        ↓
Processo salva registradores na pilha
        ↓
Executa rotina especial de tratamento do sinal
(ex: retransmitir mensagem perdida)
        ↓
Rotina encerra
        ↓
Processo reiniciado no estado anterior ao sinal
```

> 💡 No software, os sinais correspondem às interrupções no hardware — e podem ser gerados por uma série de causas além de temporizadores expirando. Desvios de controle detectados por hardware (como executar uma instrução ilegal ou utilizar um endereço inválido) também são convertidos em sinais para o processo causador.
> 

---

# ✅ Resumo do Conceito

- Um **processo** é um programa em execução — um contêiner com espaço de endereçamento, registradores, arquivos abertos e demais recursos
- O **espaço de endereçamento** contém código, dados globais, heap e pilha — todos isolados dos outros processos pela MMU
- A **tabela de processos** mantém o estado de todos os processos existentes, incluindo registradores e ponteiros de arquivos
- Quando suspenso, o estado completo do processo é salvo — ao ser retomado, continua exatamente de onde parou
- Processos podem criar **processos-filhos**, formando uma **árvore de processos**
- Cada processo tem uma **UID** do usuário que o criou e pode pertencer a **grupos** (GID)
- O **superusuário/root** tem privilégios especiais que passam por cima das regras de proteção normais
- **Sinais de alarme** permitem notificar processos de eventos assíncronos — o equivalente software das interrupções de hardware