---
tags:
  - redes
  - redes/osi
---

# Camada de Rede - 3

A **Camada 3 (Rede)** é o "GPS" ou o "Carteiro Universal" do Modelo OSI. Se a Camada 4 é o gerente que organiza as caixas, a Camada 3 é quem conhece o **mapa do mundo** e decide qual caminho a caixa deve seguir para sair da sua casa e chegar em outro país.

---

## 1. Camada 3: Rede (Network Layer)

A Camada de Rede é responsável pelo **endereçamento lógico** e pelo **roteamento**. É aqui que o conceito de "Internet" realmente ganha vida, pois ela permite que redes diferentes se conectem entre si.

### 1.1. O que ela faz?

Ela realiza duas funções que são o pilar da comunicação global:

- **Endereçamento Lógico (IP):** Ela coloca um "rótulo" em cada pacote com o **Endereço IP de Origem** (quem envia) e o **Endereço IP de Destino** (quem recebe).
- **Roteamento:** Ela escolhe o melhor caminho (a rota) para o dado viajar. Se um caminho estiver congestionado ou um cabo for cortado, a Camada 3 encontra um desvio.
- **Fragmentação (se necessário):** Se um pacote for grande demais para uma certa rede no meio do caminho, a Camada 3 o divide em pedaços menores e a Camada 3 do destino os junta.

### 1.2. Exemplo da Vida Real: O Sistema de Correios e o GPS

Imagine que você está enviando um presente de São Paulo para Tóquio:

- **Camada 4 (Transporte):** Você desmontou o presente e numerou as peças.
- **Camada 3 (Rede):** Você escreve o **Endereço Completo (IP)** na caixa.
    1. A agência dos Correios local olha o endereço e diz: *"Isso vai para o Japão"*.
    2. Eles mandam para um centro de distribuição (Roteador).
    3. Lá, eles decidem: *"O voo por Nova York está cancelado, vamos mandar via Europa"*.
    4. O pacote pula de cidade em cidade (salto por salto) até chegar ao destino.

### 1.3. O Protagonista e o Equipamento

- **Protocolo Principal:** **IP (Internet Protocol)**. É aqui que existem o **IPv4** (ex: 192.168.1.1) e o **IPv6**.
- **Equipamento Principal:** **Roteador**. O roteador é um dispositivo de Camada 3. Ele lê o endereço IP e consulta uma "Tabela de Roteamento" para saber para onde enviar o pacote.
- **Unidade de Dado (PDU):** **Pacote (Packet)**.

### 1.4. Resumo

> **Ponto Chave:** A Camada 3 é a "camada do endereço". Ela não se importa com o que tem dentro do pacote (se é um e-mail ou um vídeo), ela só quer saber: *"Qual é o IP de destino e qual o melhor caminho para chegar lá?"*.

---

## 2. Exemplo Prático no "Ping"

Quando você usa o comando `ping google.com` no seu terminal:

1. O seu computador usa o protocolo **ICMP** (que mora na Camada 3).
2. Ele cria um **Pacote** com o seu IP e o IP do Google.
3. O pacote viaja por vários **Roteadores** pelo mundo.
4. Cada roteador olha o IP de destino e diz: *"Vá por ali!"*.
5. Quando o Google recebe, ele responde de volta para o seu IP.

---

## 3. Diferença Crucial

- **Camada 2 (Enlace):** Usa o **Endereço MAC** (endereço físico da placa, só funciona dentro da sua casa/empresa).
- **Camada 3 (Rede):** Usa o **Endereço IP** (endereço lógico, funciona no mundo inteiro).

---

## Conexão com Sistemas Operacionais

**Estrutura binária do cabeçalho IP:** O cabeçalho IPv4 tem 20 bytes mínimos com campos de tamanhos variados (alguns de 4 bits, como version e IHL; outros de 16 bits como total_length; outros de 32 bits como src_ip e dst_ip). No kernel Linux, essa estrutura é a `struct iphdr` em `<linux/ip.h>`. Interpretar esses campos exige entender **bit fields**, endianness (network byte order = big-endian), e como structs em C são mapeadas para bytes em memória → [[Bits e Bytes]], [[Memória]].

O layout completo do header IPv4 é:
```
| version(4b) | IHL(4b) | DSCP(6b) | ECN(2b) | total_length(16b) |
| identification(16b)   | flags(3b) | fragment_offset(13b)         |
| TTL(8b)     | protocol(8b)        | header_checksum(16b)         |
| src_ip(32b)                                                       |
| dst_ip(32b)                                                       |
```

**TTL (Time To Live):** O campo TTL começa com um valor (tipicamente 64 ou 128) e é decrementado por cada roteador. Quando chega a 0, o roteador descarta o pacote e envia de volta um **ICMP Time Exceeded** para o remetente. Isso evita loops de roteamento infinitos. No kernel Linux, a função `ip_forward()` é quem decrementa o TTL antes de reenviar o pacote → [[System Calls]].

**Fragmentação IP:** Quando um pacote é maior que o MTU (Maximum Transmission Unit) do próximo enlace (1500 bytes no Ethernet padrão), o kernel fragmenta o pacote: divide o payload em pedaços de até MTU-20 bytes (subtraindo o cabeçalho IP), cria um novo cabeçalho IP para cada fragmento (com o mesmo ID e incrementando o fragment_offset), e define o flag `MF` (More Fragments). O destino remonta os fragmentos em um buffer na memória usando o campo `id` e `fragment_offset` para ordenar os pedaços. Esse buffer de remontagem fica na memória do kernel enquanto os fragmentos chegam → [[Memória]].
