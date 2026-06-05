---
tags:
  - sistemas-operacionais
  - so/hardware
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.3.4"
---
# Dispositivos de I/O

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.3.4

---

# 🔌 O que são Dispositivos de I/O?

A CPU e a memória não são os únicos recursos que o sistema operacional precisa gerenciar. Existem muitos outros — teclado, mouse, monitor, disco, impressora, placa de rede, USB. Todos esses são **dispositivos de Entrada/Saída** (I/O — *Input/Output*).

Eles são a forma como o computador **se comunica com o mundo externo** e com o armazenamento. Sem eles, a CPU seria uma calculadora isolada — processando dados que nunca entram nem saem.

---

# 🏗️ Estrutura de um Dispositivo de I/O

Todo dispositivo de I/O é composto por **duas partes**:

## 1. Controlador

O **controlador** é um chip (ou conjunto de chips) que fica na placa-mãe ou na própria placa do dispositivo. Ele é o intermediário entre o SO e o dispositivo físico.

```
Sistema Operacional
        ↓ comandos de alto nível
   Controlador         ← hardware inteligente
        ↓ sinais elétricos de baixo nível
  Dispositivo físico   ← disco, teclado, monitor...
```

O controlador aceita comandos simples do SO e os traduz para as operações físicas complexas necessárias. Por exemplo, o SO manda "leia o setor 11.206 do disco 2" — o controlador converte isso em posicionamento de braço, espera de rotação, verificação de checksum, etc. Toda essa complexidade fica **escondida** atrás de uma interface simples.

> 🔧 **O controlador é essencialmente um mini processador dedicado**
> 

> Assim como a CPU principal, um controlador tem lógica de processamento, **registradores** (memória interna pequeníssima e ultrarrápida) e um **firmware** (programa gravado internamente que define seu comportamento). A diferença é que ele é mais simples e barato, pois foi projetado para fazer **uma única tarefa muito bem** — ao invés de ser de propósito geral como a CPU.
> 

> - A CPU principal → propósito geral, faz qualquer coisa, poderosa e cara
> 

> - Controlador de disco → especialista em disco, só faz isso, mais simples e barato
> 

> - Controlador USB → especialista em USB, só faz isso, mais simples e barato
> 

> 
> 

> Essa divisão é intencional — **tirar trabalho especializado e repetitivo da CPU** e delegar para controladores dedicados, liberando a CPU para executar a lógica dos programas. É a mesma lógica por trás do DMA.
> 

> 💡 Todo controlador tem um pequeno conjunto de **registradores** usados para comunicação com o SO — endereço de disco, endereço de memória, contador de setores, direção (leitura/escrita). A reunião de todos esses registradores forma o **espaço de portas de I/O**.
> 

## 2. Dispositivo Propriamente Dito

É a parte física que realiza o trabalho — os pratos giratórios do HD, as teclas do teclado, o painel do monitor. Os dispositivos têm interfaces relativamente simples justamente porque a complexidade fica no controlador.

> 💡 **Importante: sempre os dois, em camadas**
> 

> Todo dispositivo tem seu próprio controlador — não é um ou outro. O que varia é *onde* cada controlador fica fisicamente:
> 

> - **Controladores simples e padronizados** (USB, SATA, áudio, rede) ficam integrados na própria placa-mãe, ao lado das entradas físicas — o fabricante da placa-mãe os inclui
> 

> - **Dispositivos mais complexos** (HD, SSD, GPU, impressora) trazem seu próprio controlador interno embutido dentro da carcaça do dispositivo
> 

> 
> 

> E muitas vezes os dois existem simultaneamente em camadas. Por exemplo, ao conectar um HD externo via USB:
> 

> `
> 

> Placa-mãe
> 

> └── Controlador USB (integrado na placa-mãe)   ← gerencia o protocolo USB
> 

> ↓
> 

> HD externo
> 

> └── Controlador interno do HD               ← gerencia pratos, setores, checksums
> 

> ↓
> 

> Pratos magnéticos físicos
> 

> `
> 

> O controlador USB cuida da comunicação pelo cabo. O controlador interno do HD cuida da leitura/escrita física. São dois controladores diferentes trabalhando em camadas — cada um escondendo sua complexidade do nível acima.
> 

---

# 🧩 Driver de Dispositivo

Como cada controlador é diferente, o SO precisa de um software específico para conversar com cada um. Esse software é o **driver de dispositivo**.

```bash
Programa do usuário
        ↓
Sistema Operacional (chamada de sistema)
        ↓
   Driver de dispositivo   ← software que "fala a língua" do controlador
        ↓
     Controlador           ← hardware
        ↓
   Dispositivo físico
```

- Cada fabricante de hardware fornece drivers para os principais SOs
- O driver precisa rodar em **modo núcleo** para ter acesso direto ao hardware
- É por isso que um driver mal-feito ou malicioso pode **comprometer todo o sistema** — ele roda com privilégio total

**Como um driver é instalado no SO:**

Existem três formas:

1. **Relinkar o núcleo** com o novo driver e reinicializar — forma mais antiga, usada em UNIX antigos
2. **Entrada em arquivo do SO** — o SO encontra os drivers necessários na inicialização, carrega e instala (Windows funciona assim em versões mais antigas)
3. **Hot-plugging** — o SO carrega novos drivers dinamicamente enquanto está em execução, sem precisar reinicializar. É importante entender a distinção em duas camadas:
    
    > 💡 O SO já sabe por padrão **como se comunicar com a porta** (USB, SATA, Ethernet) — o driver do controlador vem instalado junto com o SO. O que ele **não sabe** é como lidar com o **dispositivo específico** que você conecta nela. Um pen drive SanDisk Ultra 3.0, por exemplo, pode ter comandos especiais, recursos extras e comportamentos próprios que o SO desconhece até instalar o driver daquele modelo. É como uma tomada elétrica — a tomada é padronizada e o SO sabe lidar com ela, mas cada aparelho que você liga pode ter comportamentos específicos que precisam de um software próprio.
    > 
    
    > `
    > 
    
    > Driver do controlador USB     ← já vem com o SO
    > 
    
    > "sei enviar e receber dados pela porta USB"
    > 
    
    > ↓
    > 
    
    > Driver do dispositivo          ← carregado via hot-plug ao conectar
    > 
    
    > "sei falar com este SanDisk Ultra 3.0 especificamente"
    > 
    
    > `
    > 
    
    > Sem hot-plug: instala o driver → reinicia o sistema → só então funciona
    > 
    
    > Com hot-plug: conectou → SO detecta → carrega o driver a quente → pronto em segundos
    > 
    - O **controlador USB da placa-mãe** já tem seu driver instalado junto com o SO — nunca precisa de hot-plug
    - O **dispositivo específico** que você conecta na porta (pen drive, HD externo, mouse, teclado) é que precisa de hot-plug — cada um tem seu próprio driver que precisa ser carregado no momento da conexão
    - Sem hot-plug: instala o driver → reinicia o sistema → só então funciona (era assim em sistemas Unix antigos)
    - Com hot-plug: conectou o dispositivo → SO detecta → carrega o driver "a quente" → pronto para usar em segundos
    - O Windows tornava isso visível com a mensagem *"Instalando driver do dispositivo..."* ao conectar algo novo — era o hot-plugging acontecendo em tempo real

---

# 📡 Como o SO Realiza uma Operação de I/O

O I/O pode ser realizado de **três maneiras** diferentes, cada vez mais sofisticadas:

## Método 1: Espera Ocupada (*Busy Waiting*)

É o método mais simples — e o mais ineficiente.

```jsx
Programa pede I/O
        ↓
Driver inicia a operação no dispositivo
        ↓
CPU fica em loop checando se terminou:
  while (dispositivo_ocupado) { /* aguarda */ }
        ↓
Operação termina → dados disponíveis → programa continua
```

> ⚠️ **Problema:** A CPU fica 100% ocupada esperando o dispositivo — que pode levar milissegundos (uma eternidade para a CPU). Durante todo esse tempo, nenhum outro processo pode usar a CPU. Esse método é chamado de **espera ocupada** e é um enorme desperdício.
> 

## Método 2: Interrupções (*Interrupts*)

> 📌 **Figura 1.11(a) — Os passos para iniciar um dispositivo de I/O e obter uma interrupção**
> 

```
┌──────────┐   ┌───────────────────┐   ┌─────────────────┐   ┌──────────────┐
│   CPU  │   │   Controlador    │   │   Controlador    │   │  Unidade   │
│        │   │  de interrupção │   │    de disco      │   │  de disco  │
└──────────┘   └───────────────────┘   └─────────────────┘   └──────────────┘
      │                  │                    │                      │
      │←────────────────3│    ┌────────────────────────────────────┤
      │                  │4←────────────────────────────────────┘
      │                       │
      │────────────────────1│───────────────────────────────────2→

Passo 1: Driver escreve nos registradores do controlador de disco
Passo 2: Controlador inicia a operação e sinaliza o controlador de interrupção
Passo 3: Controlador de interrupção sinaliza a CPU (pino)
Passo 4: CPU lê o número do dispositivo do barramento
```

> 📌 **Figura 1.11(b) — O processamento da interrupção**
> 

```
┌───────────────────────────┐
│        Instrução atual       │  ← processo A executando
│       Instrução seguinte     │
├───────────────────────────┤
│                             │
│   1. Interrupção chega       │  ← CPU para o que está fazendo
│      ↓                      │     salva contexto (PC, PSW,
│   2. Despacho para           │     registradores) na pilha
│      tratador                │
│      ↓                      │  ← CPU chaveia para modo núcleo
│   [Tratador de interrupção] │     e executa o tratador (driver)
│      ↓                      │
│   3. Retorno                 │  ← restaura contexto do processo A
├───────────────────────────┤
│   processo A continua        │  ← de onde parou, sem saber
│   de onde parou              │     que foi interrompido
└───────────────────────────┘
```

> 💡 **O que é o chip controlador de interrupções?**
> 

> É um chip **separado e dedicado** na placa-mãe, cujo único objetivo é gerenciar as interrupções de todos os dispositivos e repassá-las à CPU de forma organizada. O problema que ele resolve: se 20 dispositivos diferentes tentassem chamar a atenção da CPU ao mesmo tempo — disco terminou, tecla pressionada, pacote de rede chegou — seria caos. O controlador age como um **recepcionista**:
> 

> `
> 

> Disco terminou       ──┐
> 

> Teclado pressionado  ──┤
> 

> Rede recebeu dados   ──┤→ Controlador de ──→ CPU
> 

> Mouse se moveu       ──┤  Interrupções
> 

> USB conectado        ──┘  (organiza fila,
> 

> define prioridades,
> 

> avisa um por vez)
> 

> `
> 

> Ele recebe os sinais de todos os dispositivos, decide quem tem prioridade e avisa a CPU **um de cada vez** com o número do dispositivo que gerou a interrupção. Nas arquiteturas modernas essa função é desempenhada pelo **APIC** (*Advanced Programmable Interrupt Controller*), já integrado dentro da própria CPU.
> 

A solução para o desperdício da espera ocupada. Em vez de a CPU ficar em loop, o dispositivo **avisa a CPU quando terminou**.

O processo tem **4 passos**, como mostrado na Figura 1.11(a) do Tanenbaum:

```
Passo 1: Driver escreve nos registradores do controlador
         dizendo o que fazer (ex: ler setor X)

Passo 2: Controlador inicia a operação no dispositivo
         e sinaliza o chip controlador de interrupção
         via linhas do barramento quando terminar

Passo 3: Quando o controlador de interrupção está pronto,
         ele sinaliza um pino da CPU

Passo 4: O controlador de interrupção insere o número
         do dispositivo no barramento para a CPU saber
         qual dispositivo terminou
```

Do lado da CPU, quando a interrupção chega:

```
CPU está executando o Processo A
        ↓
Interrupção chega (dispositivo terminou!)
        ↓
CPU salva o contexto atual (PC, PSW, registradores)
na pilha
        ↓
CPU chaveia para modo núcleo
        ↓
CPU consulta a tabela de vetor de interrupção
para encontrar o endereço do tratador correto
        ↓
Tratador de interrupção (parte do driver) executa:
 - indaga o dispositivo sobre sua situação
 - processa os dados recebidos
 - libera buffers ou sinaliza processos em espera
        ↓
Restaura o contexto do Processo A
        ↓
Processo A continua de onde parou
```

> 💡 **Tabela de Vetor de Interrupção:** é uma estrutura na memória que mapeia o número de cada dispositivo para o endereço do seu tratador de interrupção (*interrupt handler*). Assim a CPU sabe exatamente qual código executar para cada dispositivo que gera uma interrupção.
> 

> 💡 **Por que isso é melhor?** Enquanto o dispositivo está trabalhando (podendo levar milissegundos), a CPU pode executar outro processo. Ela só é "interrompida" quando o dispositivo realmente terminou — sem desperdício.
> 

## Método 3: DMA (*Direct Memory Access*)

As interrupções são ótimas, mas ainda exigem que a CPU transfira os dados do controlador para a memória byte a byte. Para grandes volumes de dados (como leitura de disco), isso ainda ocupa muito a CPU.

A solução é o chip **DMA** (*Direct Memory Access* — Acesso Direto à Memória):

```
CPU configura o chip DMA:
 - quantos bytes transferir
 - endereço do dispositivo
 - endereço de memória de destino
 - direção (leitura ou escrita)
        ↓
CPU deixa o DMA trabalhar e vai fazer outra coisa
        ↓
DMA transfere os dados diretamente entre
dispositivo e memória, SEM envolver a CPU
        ↓
Quando termina, DMA gera uma interrupção
        ↓
CPU é notificada: transferência concluída
```

```
Sem DMA:  Dispositivo → CPU → Memória   (CPU no meio de tudo)
Com DMA:  Dispositivo → DMA → Memória   (CPU liberada)
```

> 💡 **Analogia:** Sem DMA é como se o chefe (CPU) tivesse que carregar cada caixa do caminhão (dispositivo) para o depósito (memória) pessoalmente. Com DMA, o chefe contrata um ajudante (DMA) para fazer o transporte, e o chefe só é avisado quando tudo estiver no depósito.
> 

---

# 🗺️ Como os Registradores do Controlador São Acessados

Existem duas abordagens que os sistemas usam para o driver se comunicar com os registradores do controlador:

**Esquema 1: I/O Mapeado em Memória**

Os registradores dos dispositivos são mapeados no espaço de endereços de memória do SO. O driver lê e escreve neles como palavras de memória normais — sem instruções especiais. O endereçamento é simples, mas consome parte do espaço de endereços.

**Esquema 2: Espaço de Porta de I/O**

Os registradores ficam em um espaço separado de "portas de I/O". Instruções especiais `IN` e `OUT` (disponíveis em modo núcleo) são usadas para ler e escrever nesses registradores. Não consome espaço de endereços de memória, mas exige instruções especiais.

> Ambos os esquemas são amplamente usados. Muitos sistemas modernos usam os dois ao mesmo tempo para dispositivos diferentes.
> 

---

# 🔄 Visão Geral: O Caminho Completo de uma Operação de I/O

Juntando tudo, o que acontece quando um programa lê um arquivo do disco:

```
1. Programa chama read() ← chamada de sistema

2. SO entra em modo núcleo e aciona o driver de disco

3. Driver escreve nos registradores do controlador:
   "leia o setor X, coloque em memória Y"

4. Controlador inicia a operação física no disco

5. DMA transfere os dados do disco para a RAM
   enquanto a CPU executa outros processos

6. DMA termina → gera interrupção

7. CPU para o que estava fazendo, salva contexto

8. Tratador de interrupção do driver executa,
   confirma os dados, sinaliza o SO

9. SO marca o processo original como pronto

10. Escalonador devolve a CPU ao processo

11. Programa continua com os dados disponíveis
```

---

# ✅ Resumo do Conceito

- Dispositivos de I/O têm duas partes: o **controlador** (hardware inteligente que esconde a complexidade) e o **dispositivo físico**
- O **driver** é o software em modo núcleo que fala a língua do controlador — cada fabricante fornece o seu
- Drivers podem ser instalados de 3 formas: relinkando o núcleo, na inicialização, ou dinamicamente (*hot-plug*)
- **Espera ocupada** — CPU fica em loop esperando o dispositivo: simples, mas desperdiça 100% da CPU
- **Interrupções** — dispositivo avisa a CPU quando terminou: CPU fica livre enquanto o dispositivo trabalha. A **tabela de vetor de interrupção** mapeia cada dispositivo para seu tratador
- **DMA** — chip especializado transfere dados diretamente entre dispositivo e memória sem envolver a CPU: ideal para grandes volumes. Gera uma interrupção ao terminar
- Os registradores do controlador são acessados via **I/O mapeado em memória** ou **espaço de porta de I/O** — ambos amplamente usados