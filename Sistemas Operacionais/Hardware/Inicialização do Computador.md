---
tags:
  - sistemas-operacionais
  - so/hardware
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.3.6"
---
# Inicialização do computador

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.3.6

---

# 🔌 O que Acontece Quando Você Liga o Computador?

Ligar o computador parece simples — apertar um botão e o sistema aparece. Mas por baixo existe um processo cuidadosamente orquestrado envolvendo hardware, firmware e software trabalhando em sequência. Esse processo é chamado de **boot** ou **inicialização**.

O desafio fundamental do boot é o seguinte: o SO está no disco, mas a CPU só consegue executar código da memória. A memória está vazia quando o computador é ligado. **Como a CPU sabe o que fazer antes de qualquer coisa estar carregada?**

A resposta está em uma cadeia de pequenos programas, cada um responsável por carregar e executar o próximo — como uma corrente de revezamento.

---

# 🏗️ Os Componentes do Processo de Boot

## BIOS / UEFI — O Firmware

Toda placa-mãe tem uma pequena quantidade de **memória Flash** que guarda um programa especial chamado **firmware**. Esse programa persiste sem energia (porque é Flash) e é a primeira coisa que a CPU executa.

> 💡 **O que é firmware?**
> 

> Firmware é um **software permanente gravado em hardware** — um programa que vive dentro de um chip de memória não volátil (Flash, EEPROM) e controla diretamente o hardware em que está instalado. A palavra é uma fusão de *firm* (firme, permanente) + *software* — é mais permanente que software comum, mas mais flexível que hardware puro.
> 

> 
> 

> | | Hardware | Firmware | Software |
> 

> |---|---|---|---|
> 

> | **O que é** | Circuitos físicos | Programa gravado no chip | Programa instalado no disco |
> 

> | **Onde vive** | Silício | Memória Flash/EEPROM | HD/SSD |
> 

> | **Pode ser alterado?** | Não | Sim, mas raramente | Sim, facilmente |
> 

> | **Exemplos** | CPU, RAM | BIOS/UEFI, firmware do HD, firmware do roteador | Windows, Chrome, Word |
> 

> 
> 

> Outros exemplos de firmware que você usa sem perceber: firmware do SSD (controla wear leveling), firmware do roteador (controla funções de rede), firmware da impressora, firmware do controle do videogame.
> 

Historicamente esse firmware se chamava **BIOS** (*Basic Input Output System* — sistema básico de entrada e saída). Hoje foi substituído pela **UEFI** (*Unified Extensible Firmware Interface* — interface de firmware extensível unificada), mas muitas pessoas ainda usam o termo BIOS informalmente.

|  | BIOS (antigo) | UEFI (moderno) |
| --- | --- | --- |
| **Velocidade** | Lenta | Rápida |
| **Limite de disco** | Até 2 TB | Até 8 ZiB (8 × 2⁷⁰ bytes) |
| **Interface** | Texto simples | Gráfica, com mouse |
| **Arquiteturas** | Limitado a x86 | Múltiplas arquiteturas |
| **Funcionalidades** | Básicas | Avançadas (Secure Boot, shell, etc.) |

> 💡 Como o Tanenbaum observa, a UEFI é tão complexa que "tentar entendê-la por completo tirou a felicidade de muitos por toda a vida." É quase um pequeno sistema operacional em si — entende partições, sistemas de arquivos, executáveis e até tem um shell com comandos padrão.
> 

> ⚠️ **O que significa o limite de disco do BIOS/UEFI?**
> 

> O limite de 2 TB (BIOS) e 8 ZiB (UEFI) **não se refere ao armazenamento do firmware em si** — o firmware é pequeníssimo (alguns MB). O limite se refere ao **tamanho máximo do disco rígido/SSD que o firmware consegue endereçar para boot**.
> 

> 
> 

> O BIOS usava o esquema **MBR** que reservava apenas 32 bits para endereçar setores do disco:
> 

> `
> 

> 2³² setores × 512 bytes por setor = ~2 TB
> 

> `
> 

> Qualquer disco maior que 2 TB simplesmente não existia para o BIOS — ele não enxergava os dados além desse limite. Com um HD de 4 TB, os 2 TB restantes eram invisíveis.
> 

> 
> 

> A UEFI resolve isso com o esquema **GPT** que usa 64 bits:
> 

> `
> 

> 2⁶⁴ setores × 512 bytes = ~8 ZiB (8 bilhões de terabytes)
> 

> `
> 

> Um limite tão absurdamente alto que nenhum disco fabricado hoje chegará perto.
> 

> 
> 

> `
> 

> BIOS + MBR:              UEFI + GPT:
> 

> Disco de 4 TB            Disco de 4 TB
> 

> ┌────────────┐           ┌────────────┐
> 

> │  2 TB ✅   │           │  4 TB ✅   │
> 

> │  visível   │           │  visível   │
> 

> ├────────────┤           │            │
> 

> │  2 TB ❌   │           │            │
> 

> │ invisível  │           │            │
> 

> └────────────┘           └────────────┘
> 

> `
> 

---

# 🔄 O Processo de Boot Passo a Passo

## Passo 1 — Energia estabilizada e CPU acorda

Quando você pressiona o botão de ligar, a fonte de alimentação começa a fornecer energia. A placa-mãe aguarda até que a tensão esteja **estável** antes de liberar a CPU para executar.

Quando a CPU começa a executar, ela busca sua primeira instrução em um **endereço fixo e pré-determinado** — chamado de **vetor de reset**. Esse endereço está mapeado para a memória Flash que contém o firmware. Ou seja, a CPU foi projetada para sempre começar pelo firmware, sem precisar saber de nada antes.

```
CPU liga → busca instrução no vetor de reset (endereço fixo)
                    ↓
         Endereço aponta para a memória Flash
                    ↓
              Firmware começa a executar
```

## Passo 2 — POST e inicialização do hardware

O firmware executa o **POST** (*Power-On Self Test* — autoteste de inicialização), que detecta e inicializa os recursos do sistema:

- Testa a RAM (verifica se está funcionando)
- Inicializa o hub controlador da plataforma (chipset)
- Detecta e inicializa os controladores de interrupção
- Varre os barramentos PCI e PCIe para encontrar e inicializar todos os dispositivos conectados
- Se encontrar dispositivos novos (diferentes da última vez), os configura

## Passo 3 — Determinar o dispositivo de boot

O firmware precisa decidir **de qual dispositivo carregar o SO**. Para isso, consulta uma **lista de dispositivos de boot** armazenada na memória CMOS (mantida pela bateria da placa-mãe). Essa lista define a ordem de tentativa — por exemplo: primeiro USB, depois SSD, depois rede.

Você pode mudar essa ordem entrando na configuração do firmware logo após ligar o computador (geralmente pressionando Del, F2 ou F12).

## Passo 4 — Carregar o bootloader

Aqui o BIOS e o UEFI tomam caminhos diferentes:

### Caminho BIOS → MBR

O BIOS lê o **primeiro setor** do dispositivo de boot — exatamente 512 bytes. Esse setor é chamado de **MBR** (*Master Boot Record* — registro mestre de inicialização). O MBR contém:

1. Um pequeno programa de boot
2. A tabela de partições do disco (informações sobre as partições existentes)

O programa do MBR examina a tabela de partições, encontra a partição **ativa** (marcada como bootável), lê o primeiro setor dessa partição e executa um **bootloader secundário** que sabe como carregar o SO.

```
Firmware BIOS
    ↓ lê 512 bytes
   MBR
    ↓ examina tabela de partições
   Partição ativa
    ↓ lê bootloader secundário
   Bootloader (ex: GRUB)
    ↓ carrega
   Sistema Operacional
```

### Caminho UEFI → GPT

A UEFI não depende do MBR. Em vez disso, ela procura a **GPT** (*GUID Partition Table* — tabela de partição de identificadores globalmente únicos) no segundo setor do dispositivo. A GPT contém informações completas sobre todas as partições.

A UEFI procura uma partição especial chamada **ESP** (*EFI System Partition* — partição do sistema EFI), formatada em FAT-32. Essa partição contém os bootloaders como arquivos normais — o firmware consegue lê-los diretamente porque entende sistemas de arquivos.

```
Firmware UEFI
    ↓ lê GPT
   Tabela de partições (GPT)
    ↓ encontra
   ESP (partição EFI, FAT-32)
    ↓ executa bootloader como arquivo
   Bootloader (ex: bootmgfw.efi, grubx64.efi)
    ↓ carrega
   Sistema Operacional
```

> 💡 **Vantagem da UEFI:** como o bootloader é um arquivo em uma partição FAT-32, você pode ter vários bootloaders para vários SOs, gerenciá-los facilmente, e até atualizá-los sem ferramentas especiais.
> 

## Passo 5 — O Bootloader escolhe o SO

O **bootloader** (ex: GRUB no Linux, Windows Boot Manager no Windows) é um programa cuja única função é **carregar o sistema operacional**. Quando há múltiplos SOs instalados, ele exibe um menu para o usuário escolher.

O bootloader pode ser configurado para mudar a ordem, o SO padrão, o tempo de espera antes de iniciar automaticamente. Essa configuração pode ser feita a partir do próprio SO em execução.

## Passo 6 — O SO assume o controle

O bootloader carrega o **kernel** do SO na memória e transfere o controle para ele. A partir daqui, o firmware sai de cena e o SO assume completamente:

1. **Consulta o firmware** para obter informações de configuração de hardware
2. **Verifica drivers** — para cada dispositivo encontrado, verifica se tem o driver. Se não tiver, pede ao usuário para instalar ou baixa da internet
3. **Carrega todos os drivers** no núcleo
4. **Inicializa suas tabelas internas** — tabela de processos, sistema de arquivos, etc.
5. **Cria processos de segundo plano** (*daemons*) — serviços que ficam rodando em background
6. **Inicia o programa de login** ou a interface gráfica (GUI)

```
Kernel carregado
    ↓
Inicializa hardware (via drivers)
    ↓
Monta sistema de arquivos raiz
    ↓
Inicia processo init (Linux) / smss.exe (Windows)
    ↓
Inicia serviços e daemons em background
    ↓
Exibe tela de login / área de trabalho
    ↓
Computador pronto para uso
```

---

# 🔐 Secure Boot

Um recurso importante da UEFI mencionado pelo Tanenbaum é o **Secure Boot** (inicialização segura). Ele garante que apenas software **assinado digitalmente** e confiável seja executado durante o boot — impedindo que malware substitua o bootloader ou o kernel por versões maliciosas.

O firmware verifica a assinatura digital de cada componente antes de executá-lo. Se a assinatura não for válida ou estiver ausente, o boot é interrompido. Isso é especialmente importante contra ataques de **bootkits** — malwares que se instalam antes do SO para escapar de antivírus.

---

# 🔋 O Papel da Bateria da Placa-Mãe

Durante todo esse processo, a memória **CMOS** (alimentada pela bateria da placa-mãe) guarda configurações persistentes:

- Lista de dispositivos de boot e ordem de prioridade
- Hora e data atuais
- Configurações de hardware personalizadas

Quando a bateria morre, o computador "esquece" essas configurações e volta aos padrões de fábrica — hora volta para 01/01/2000, ordem de boot volta ao padrão, configurações personalizadas se perdem. É o "Alzheimer" que o Tanenbaum menciona.

---

# ✅ Resumo do Processo

```
1. Energia estabiliza → CPU busca instrução no vetor de reset
2. Firmware (BIOS/UEFI) executa → POST → inicializa hardware
3. Firmware determina dispositivo de boot (via CMOS)
4. BIOS → lê MBR → bootloader secundário
   UEFI → lê GPT → ESP → bootloader como arquivo
5. Bootloader exibe menu (se múltiplos SOs) → carrega kernel
6. Kernel inicializa → drivers → serviços → login/GUI
```

| Componente | Função | Onde fica |
| --- | --- | --- |
| **Firmware (BIOS/UEFI)** | Primeiro código executado, inicializa hardware | Memória Flash da placa-mãe |
| **CMOS** | Guarda configurações e hora | RAM especial com bateria |
| **MBR** | Primeiro setor do disco, contém bootloader e tabela de partições | Primeiro setor do HD/SSD |
| **GPT** | Tabela de partições moderna usada pela UEFI | Segundo setor do HD/SSD |
| **ESP** | Partição FAT-32 com bootloaders | Partição no HD/SSD |
| **Bootloader** | Carrega o kernel do SO escolhido | ESP ou partição ativa |
| **Kernel** | Núcleo do SO — assume controle total | HD/SSD, carregado na RAM |