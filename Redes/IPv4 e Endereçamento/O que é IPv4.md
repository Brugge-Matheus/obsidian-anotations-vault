---
tags:
  - redes
  - redes/ipv4
---

# O que é IPv4

O endereçamento IP é a base da comunicação na internet, funcionando como o "endereço residencial" de cada dispositivo conectado a uma rede.

## O que é IP?

IP significa **Internet Protocol** (Protocolo de Internet). Ele é um conjunto de regras que rege o formato dos dados enviados pela rede ou pela internet. Na prática, o endereço IP é um identificador numérico exclusivo atribuído a cada dispositivo (computador, smartphone, servidor, impressora) que participa de uma rede de computadores que utiliza o Protocolo de Internet para comunicação. Sem um endereço IP, os roteadores não saberiam para onde enviar as informações que você solicita.

## O que é IPv4?

O **IPv4** (Internet Protocol version 4) é a quarta versão do protocolo e a mais utilizada atualmente. Ele utiliza um sistema de endereçamento de **32 bits**, o que permite aproximadamente 4,3 bilhões de endereços únicos.

$$
2^{32}
$$

Um endereço IPv4 é escrito em notação decimal por pontos, consistindo em quatro números (octetos) separados por pontos, variando de 0 a 255. Por exemplo: `192.168.1.1`.

### IPv4 é um inteiro de 32 bits

Por baixo da notação `192.168.1.1`, um endereço IPv4 é simplesmente um inteiro sem sinal de 32 bits (4 bytes). Em C seria `uint32_t`. Em Go, `net.IP` é um `[]byte` — um slice de 4 bytes para IPv4 ou 16 bytes para IPv6.

Os bytes são transmitidos na rede em **big-endian** (byte mais significativo primeiro), chamado de *network byte order*. Processadores x86 são **little-endian**, por isso existe a necessidade de conversão `htonl()`/`ntohl()` em C ao trabalhar com sockets — a CPU armazena ao contrário do que a rede espera.

## Diferenças entre IPv4 e IPv6

O IPv6 foi criado principalmente porque o estoque de endereços IPv4 disponíveis no mundo praticamente se esgotou devido ao crescimento explosivo da internet e de dispositivos conectados (Internet das Coisas). O esgotamento do IPv4 levou também ao surgimento do NAT (Network Address Translation), que permite que múltiplos dispositivos compartilhem um único IP público.

As principais diferenças incluem:

- **Capacidade de Endereçamento:** Enquanto o IPv4 usa 32 bits (4,3 bilhões de endereços), o IPv6 utiliza **128 bits**. Isso permite um número virtualmente infinito de endereços, garantindo que cada grão de areia na Terra pudesse ter seu próprio IP.

$$
2^{128}
$$

- **Formato da Escrita:** O IPv4 usa números decimais e pontos (ex: `172.16.254.1`). O IPv6 usa **hexadecimal** e dois-pontos, sendo dividido em oito grupos de quatro dígitos (ex: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`).
- **Segurança e Configuração:** O IPv6 foi projetado com o protocolo IPsec (segurança) integrado de forma nativa, enquanto no IPv4 ele é opcional. Além disso, o IPv6 suporta autoconfiguração de endereços de forma mais eficiente (Stateless Address Autoconfiguration).
- **Eficiência de Roteamento:** O cabeçalho do pacote IPv6 é mais simples e hierárquico, o que permite que os roteadores processem os pacotes de forma mais rápida em comparação ao IPv4.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — Um endereço IPv4 é literalmente um inteiro de 32 bits (4 bytes). Entender representação binária e hexadecimal é essencial para trabalhar com IPs em baixo nível.
- **[[Processadores]]** — Endianness: processadores x86 são little-endian, mas a rede usa big-endian (network byte order). As funções `htonl()` / `ntohl()` existem exatamente para fazer essa conversão antes de colocar dados em sockets. O hardware da CPU processa os 4 bytes do IP em uma única instrução de 32 bits.

## Conexão com Go

- **[[Tipos de Dados]]** — Em Go, `net.IP` é do tipo `[]byte`. Para IPv4, são 4 bytes; para IPv6, são 16 bytes. Isso reflete diretamente a natureza de "inteiro de N bits" de cada versão do protocolo.
- **[[Bits e Bytes]]** — Manipular IPs em Go envolve trabalhar com slices de bytes, conversões entre representações, e operações bitwise para cálculos de rede.
- **[[HTTP (net-http)]]** — O pacote `net` de Go abstrai toda a manipulação de endereços IP ao criar conexões TCP/UDP, mas internamente opera sobre esses bytes.
