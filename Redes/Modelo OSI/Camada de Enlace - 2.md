---
tags:
  - redes
  - redes/osi
---

# Camada de Enlace - 2

A **Camada 2 (Enlace de Dados)** é o "Guarda de Trânsito Local" ou o "Carteiro do Bairro". Se a Camada 3 (Rede) é o GPS que conhece o mapa do mundo, a Camada 2 é quem realmente sabe **quem é quem** dentro da sua casa ou da sua empresa.

---

## 1. Camada 2: Enlace de Dados (Data Link Layer)

A Camada de Enlace é responsável por organizar os dados para que eles possam viajar pelo meio físico (cabo ou ar) e garantir que eles cheguem ao dispositivo correto **dentro da mesma rede local**.

### 1.1. O que ela faz?

Ela realiza três funções críticas para a comunicação de curto alcance:

- **Endereçamento Físico (MAC):** Ela coloca o "carimbo" do **Endereço MAC** de origem e de destino. O MAC é o RG da sua placa de rede.
- **Enquadramento (Framing):** Ela pega o pacote da Camada 3 e o "embrulha" em uma estrutura chamada **Quadro (Frame)**, adicionando um cabeçalho no início e um verificador de erros no fim.
- **Detecção de Erros:** Ela faz um cálculo matemático (CRC) no final do quadro. Se o resultado for diferente no destino, ela sabe que o dado "corrompeu" no cabo e o descarta.

### 1.2. Exemplo da Vida Real: O Condomínio e o Número do Apartamento

Imagine que o **IP (Camada 3)** é o endereço da rua: *"Rua das Flores, nº 100"*.

- **Camada 3 (Rede):** O caminhão do correio chega no portão do condomínio (Roteador).
- **Camada 2 (Enlace):** O **Porteiro** olha a lista e vê: *"O pacote é para o Matheus, que mora no **Bloco A, Apto 12 (Endereço MAC)**"*.
    1. O porteiro não precisa mais do GPS. Ele conhece os corredores.
    2. Ele leva o pacote exatamente até a sua porta.
    3. Se ele tropeçar e rasgar a caixa no caminho, ele joga fora e pede outra (Detecção de erro).

### 1.3. O Protagonista e o Equipamento

- **Equipamento Principal:** **Switch**. O Switch é um dispositivo de Camada 2. Ele guarda uma tabela com todos os endereços MAC conectados a ele.
- **Protocolo Principal:** **Ethernet** (para cabos) e **Wi-Fi (802.11)** (para ondas de rádio).
- **Unidade de Dado (PDU):** **Quadro (Frame)**.

### 1.4. As duas Subcamadas

A Camada 2 é tão importante que é dividida em duas partes:

1. **LLC (Logical Link Control):** Identifica qual protocolo da Camada 3 está sendo usado (IP, por exemplo) e controla o fluxo.
2. **MAC (Media Access Control):** Controla o acesso físico ao cabo ou ao ar e coloca os endereços de hardware.

---

## 2. Diferença Crucial

| Camada | Endereço | Alcance | Equipamento |
| --- | --- | --- | --- |
| **Camada 3 (Rede)** | **IP** (Lógico) | Global (Internet) | Roteador |
| **Camada 2 (Enlace)** | **MAC** (Físico) | Local (Sua Rede) | Switch |

---

## 3. Exemplo Prático no Switch

Quando você envia um arquivo do seu PC para a Impressora na mesma sala:

1. O seu PC cria um **Quadro** (Camada 2).
2. Ele coloca o **MAC da Impressora** como destino.
3. O **Switch** recebe o quadro, olha o MAC e diz: *"A impressora está na porta 5!"*.
4. Ele envia o dado direto para a porta 5, sem precisar perguntar para o Roteador ou para a Internet.

---

## 4. Resumo

> **Ponto Chave:** A Camada 2 é a "camada da vizinhança". Ela garante que o dado saia da placa de rede do seu PC e chegue com segurança na placa de rede do próximo dispositivo (seja o Switch ou o Roteador).

---

## Conexão com Sistemas Operacionais

**Estrutura do frame Ethernet:** Um frame Ethernet padrão tem: MAC destino (6 bytes) + MAC origem (6 bytes) + EtherType (2 bytes) + payload (46–1500 bytes) + FCS (4 bytes CRC). Todos esses campos são uma `struct` em C no kernel, com campos de tamanho fixo e alinhamento bem definido. Entender isso exige conhecer representação binária de inteiros de múltiplos bytes (big-endian na rede vs little-endian no x86) → [[Bits e Bytes]].

**CRC (Cyclic Redundancy Check):** O FCS é um CRC-32 — divisão polinomial sobre GF(2) (aritmética em campo de Galois com dois elementos, 0 e 1). O hardware da NIC calcula o CRC de cada frame enviado e verifica o CRC de cada frame recebido. No x86, há instruções dedicadas (`crc32` desde SSE4.2) que fazem isso em hardware em um único ciclo de clock por byte, em vez de usar loops em software. Isso é aritmética binária implementada em hardware → [[Bits e Bytes]], [[Processadores]].

**ARP (Address Resolution Protocol):** Quando o kernel Linux precisa enviar um pacote IP para um host na mesma sub-rede, ele precisa descobrir o MAC desse host. Para isso, envia um broadcast ARP (*"Quem tem o IP 192.168.1.5? Diga para 192.168.1.1"*). O host responde com seu MAC, e o kernel armazena o mapeamento IP→MAC na **tabela ARP** (cache no kernel, visível em `/proc/net/arp`). Toda essa lógica é implementada no kernel e acessível via syscall (`ioctl(SIOCGARP)`) → [[System Calls]].

**CSMA/CD (Ethernet com detecção de colisão):** Em redes Ethernet antigas (half-duplex com hub), duas NICs podiam transmitir simultaneamente causando colisão. O mecanismo CSMA/CD define: (1) escuta o meio antes de transmitir (*carrier sense*), (2) detecta colisão durante transmissão, (3) para e aguarda tempo aleatório antes de retransmitir. Esse é um problema clássico de **arbitragem de acesso a recurso compartilhado** — o mesmo problema que o SO resolve com semáforos e mutexes para acesso a recursos compartilhados entre processos → [[Dispositivos de IO]].
