---
tags:
  - redes
  - redes/ipv4
---

# Endereço MAC

### 1. Anatomia Detalhada do Endereço

O endereço MAC é composto por 6 grupos de dois dígitos hexadecimais. Mas há um detalhe importante no primeiro byte que define o tipo de comunicação:

- **Unicast:** Se o primeiro byte termina em um número par (bit de menor ordem é 0), o endereço refere-se a uma única placa de rede.
- **Multicast:** Se o primeiro byte termina em um número ímpar (bit de menor ordem é 1), o endereço é usado para enviar dados a um grupo de dispositivos simultaneamente.
- **Broadcast:** Quando o endereço é composto apenas por "Fs" (`FF:FF:FF:FF:FF:FF`), a mensagem é enviada para **todos** os dispositivos da rede local.

### 2. MAC Address Spoofing (Mascaramento)

Embora o endereço MAC seja "gravado" no hardware, quase todos os sistemas operacionais modernos permitem o **MAC Spoofing**.

- **O que é:** É uma técnica onde o software altera o endereço MAC que a placa de rede apresenta para o sistema ou para o roteador.
- **Uso comum:** Privacidade (evitar rastreamento em redes Wi-Fi públicas) ou para contornar filtros de segurança baseados em "Lista Branca" de endereços MAC.

### 3. Endereços MAC Aleatórios (Privacidade)

Hoje, smartphones (iOS e Android) e o Windows 10/11 usam uma função chamada **"Endereços de Hardware Aleatórios"**.

- Ao buscar redes Wi-Fi, o seu celular não mostra o MAC real dele. Ele gera um MAC falso para que lojas ou aeroportos não consigam rastrear seus movimentos físicos através dos sinais de Wi-Fi que seu aparelho emite.

### 4. Onde o MAC atua (Subcamadas da Camada 2)

No Modelo OSI, a Camada 2 (Enlace) é dividida em duas subcamadas. É importante anotar isso para entender a hierarquia:

1. **LLC (Logical Link Control):** Lida com o controle de erro e fluxo (parte lógica).
2. **MAC (Media Access Control):** É aqui que o endereço físico entra. Ele controla o acesso ao meio físico (o cabo ou o ar) e decide quem tem o direito de transmitir dados naquele momento para evitar colisões.

### 5. Tabela MAC vs. Tabela ARP

Muitos estudantes confundem essas duas tabelas. Vale criar essa distinção:

- **Tabela MAC (no Switch):** Associa o **Endereço MAC** à **Porta Física** do switch. (Ex: "O MAC AA está no cabo da porta 5").
- **Tabela ARP (no Computador/Roteador):** Associa o **Endereço IP** ao **Endereço MAC**. (Ex: "O IP 192.168.1.10 pertence ao MAC AA").

### 6. Por que não usamos apenas o MAC para a Internet toda?

Se o MAC é único no mundo, por que precisamos do IP?

- **Escalabilidade:** Imagine uma lista telefônica com bilhões de nomes sem ordem de cidade ou bairro. Seria impossível achar alguém.
- O MAC não é **hierárquico**. Ele não diz em que parte do mundo o dispositivo está. O IP, por outro lado, é organizado por redes e sub-redes, permitindo que os roteadores saibam para qual país ou cidade enviar o pacote.

---

## Conexões com SO e Go

**Por que isso importa além das redes:**

- O MAC é um endereço de 48 bits (6 bytes) gravado no hardware — o SO mantém um **cache ARP** no espaço do kernel que mapeia IP → MAC. No Linux, você pode inspecionar essa tabela diretamente em `/proc/net/arp` sem nenhuma system call extra. Isso é um exemplo de como o kernel expõe estado interno via sistema de arquivos virtual. Ver [[System Calls]] e [[Processos]].
- Sockets do tipo `SOCK_RAW` permitem que um processo acesse diretamente os **frames Ethernet com cabeçalhos MAC** — sem que o kernel filtre ou remonte os pacotes. Isso é usado em ferramentas como `tcpdump` e Wireshark. Ver [[System Calls]].
- Em Go, o tipo `net.HardwareAddr` representa um endereço MAC como `[]byte` de 6 elementos — cada byte é um dos grupos hexadecimais. É uma aplicação direta de arrays de bytes para dados binários estruturados:
  ```go
  mac, _ := net.ParseMAC("AA:BB:CC:DD:EE:FF")
  // mac é []byte{0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF}
  ```
  Ver [[Tipos de Dados]].
