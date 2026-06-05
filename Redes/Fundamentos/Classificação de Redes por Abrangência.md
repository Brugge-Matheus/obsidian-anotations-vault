---
tags:
  - redes
  - redes/fundamentos
---

# Classificação de Redes por Abrangência

Define o "tamanho" da rede. A classificação é feita com base na distância geográfica que a rede cobre.

Aqui estão as principais siglas, do menor para o maior:

---

### 1. PAN (Personal Area Network)

**Rede de Área Pessoal.**
É a rede que envolve os dispositivos de um único indivíduo, geralmente em um raio de poucos metros.

- **Tecnologias comuns:** Bluetooth, NFC, USB.
- **Exemplo:** Seu celular conectado ao seu fone de ouvido sem fio, ou o smartwatch sincronizando com o smartphone.

### 2. LAN (Local Area Network)

**Rede de Área Local.**
É a rede mais comum, cobrindo uma área limitada como uma residência, um escritório ou um andar de um prédio.

- **Características:** Alta velocidade de transmissão e baixas taxas de erro.
- **Tecnologias comuns:** Ethernet (cabos) e Wi-Fi (neste caso, também chamada de **WLAN**).
- **Exemplo:** A rede Wi-Fi da sua casa ou os computadores interconectados de uma sala de aula.

### 3. CAN (Campus Area Network)

**Rede de Área de Campus.**
Interconecta várias LANs em uma área geográfica específica, como um complexo de prédios.

- **Abrangência:** Universidades, bases militares ou grandes complexos industriais.
- **Exemplo:** A rede que interliga o prédio da faculdade de Engenharia ao prédio da faculdade de Medicina dentro de uma mesma universidade.

### 4. MAN (Metropolitan Area Network)

**Rede de Área Metropolitana.**
Cobre uma área maior que um campus, mas menor que um país, geralmente uma cidade ou um grupo de municípios vizinhos.

- **Objetivo:** Interconectar diversas LANs espalhadas pela cidade.
- **Tecnologias comuns:** Fibra óptica de alta velocidade (Metro Ethernet).
- **Exemplo:** A rede que interliga todas as agências de um banco dentro de uma mesma cidade.

### 5. WAN (Wide Area Network)

**Rede de Área Ampla.**
Cobre grandes distâncias geográficas, como países, continentes ou o mundo inteiro.

- **Características:** Como as distâncias são longas, a velocidade pode ser menor que na LAN e o custo de manutenção é maior. Utiliza satélites, cabos submarinos e micro-ondas.
- **Exemplo:** A própria **Internet** é a maior WAN existente.

---

### Outras classificações importantes:

- **SAN (Storage Area Network):** Rede dedicada exclusivamente ao armazenamento de dados e backup, conectando servidores a dispositivos de armazenamento (storages).
- **WPAN / WLAN / WMAN / WWAN:** O prefixo **"W"** significa *Wireless* (Sem Fio). É a mesma classificação acima, mas usando tecnologias de rádio em vez de cabos.

---

### Tabela de Resumo

| Sigla | Nome | Abrangência Típica | Exemplo Prático |
| --- | --- | --- | --- |
| **PAN** | Personal | Alguns metros | Bluetooth (Fone + Celular) |
| **LAN** | Local | Prédio ou residência | Wi-Fi de casa |
| **CAN** | Campus | Complexo de prédios | Universidade |
| **MAN** | Metropolitan | Uma cidade | Rede de câmeras da prefeitura |
| **WAN** | Wide | Países ou continentes | Internet |

---

## 6. Aprofundamento: O que acontece em cada camada de abrangência no nível do SO

**PAN — Bluetooth:** opera em frequências de rádio (2,4 GHz), com alcance de poucos metros. O stack Bluetooth é gerenciado pelo kernel do SO — o driver do controlador Bluetooth é tratado como um dispositivo de I/O. Cada conexão Bluetooth cria uma interface de rede virtual no SO → [[Dispositivos de IO]].

**LAN — Switch e NIC:** dispositivos são conectados via switch (camada 2 / enlace) e roteador (camada 3 / rede). O kernel gerencia a NIC (Network Interface Card) como um dispositivo de bloco: recebe interrupções quando pacotes chegam e usa DMA para copiar dados da NIC para a RAM sem ocupar a CPU → [[Dispositivos de IO]], [[System Calls]].

**WAN — BGP e ISPs:** o ISP gerencia o roteamento entre Sistemas Autônomos usando o protocolo BGP. Nenhuma máquina individual "vê" a WAN diretamente — o roteador da borda faz a abstração. Cada pacote que sai da LAN para a WAN passa pelo gateway (roteador), que consulta sua tabela de roteamento.

**Data Centers — Spine-Leaf:** redes de data center modernas usam topologia spine-leaf para alta largura de banda e baixa latência dentro da LAN. Cada servidor leaf conecta a todos os spines — qualquer servidor alcança qualquer outro em no máximo 2 hops. Isso é essencialmente uma LAN de escala corporativa.

---

## Conexão com Sistemas Operacionais

- **PAN / Bluetooth:** o driver Bluetooth no kernel trata o controlador como dispositivo de I/O → [[Dispositivos de IO]].
- **LAN / NIC:** o kernel usa interrupções e DMA para gerenciar a placa de rede → [[Dispositivos de IO]]. System calls como `socket()`, `bind()`, `send()`, `recv()` são a interface do processo com a rede → [[System Calls]].
- **WAN / roteamento:** tabelas de roteamento no kernel (Linux: `ip route`) são manipuladas via syscalls de controle de rede → [[System Calls]].
