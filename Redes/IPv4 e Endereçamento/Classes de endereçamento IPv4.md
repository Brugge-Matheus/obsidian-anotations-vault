---
tags:
  - redes
  - redes/ipv4
---

# Classes de endereçamento IPv4

O sistema de classes foi a primeira forma de organizar o espaço de endereçamento IPv4, dividindo os endereços disponíveis em cinco grupos principais (A, B, C, D e E). Essa divisão é baseada no valor do primeiro octeto do endereço, sendo apenas as três primeiras classes utilizadas pela internet (A, B e C) e as outras duas (D e E) sendo redes especiais usadas apenas em casos específicos.

Aqui está o detalhamento de cada classe:

## Classe A

Esta classe foi projetada para redes extremamente grandes, como as de corporações multinacionais ou governos. No primeiro octeto, o bit mais significativo é sempre `0`.

- **Intervalo:** 1.0.0.0 a 126.255.255.255.
- **Máscara Padrão:** 255.0.0.0 (ou /8).
- **Estrutura:** O primeiro octeto identifica a rede e os outros três identificam os hosts.
- **Capacidade:** Permite poucas redes (128), mas cada uma pode ter até 16.777.214 hosts.
- **Padrão de bits:** `0xxxxxxx` — o primeiro bit é sempre 0.

## Classe B

Destinada a redes de médio a grande porte, como universidades e grandes empresas. Os dois primeiros bits do primeiro octeto são sempre `10`.

- **Intervalo:** 128.0.0.0 a 191.255.255.255.
- **Máscara Padrão:** 255.255.0.0 (ou /16).
- **Estrutura:** Os dois primeiros octetos identificam a rede e os dois últimos identificam os hosts.
- **Capacidade:** Permite cerca de 16.384 redes, com até 65.534 hosts por rede.
- **Padrão de bits:** `10xxxxxx` — os dois primeiros bits são sempre 10.

## Classe C

É a classe mais comum, utilizada em redes de pequeno porte e redes domésticas. Os três primeiros bits do primeiro octeto são sempre `110`.

- **Intervalo:** 192.0.0.0 a 223.255.255.255.
- **Máscara Padrão:** 255.255.255.0 (ou /24).
- **Estrutura:** Os três primeiros octetos identificam a rede e o último identifica os hosts.
- **Capacidade:** Permite mais de 2 milhões de redes, mas cada uma suporta apenas 254 hosts.
- **Padrão de bits:** `110xxxxx` — os três primeiros bits são sempre 110.

## Classe D (Multicast)

Diferente das anteriores, esta classe não é usada para endereçar hosts específicos, mas sim para grupos de dispositivos que recebem a mesma transmissão simultaneamente (como streaming de vídeo ou protocolos de roteamento). Os quatro primeiros bits são `1110`.

- **Intervalo:** 224.0.0.0 a 239.255.255.255.

## Classe E (Experimental)

Reservada para fins de pesquisa, testes e uso futuro pela IETF. Não é utilizada em redes comerciais ou públicas. Os quatro primeiros bits são `1111`.

- **Intervalo:** 240.0.0.0 a 255.255.255.255.

## Observação Importante: O Endereço de Loopback

Você deve ter notado que o intervalo da Classe A termina em 126 e a Classe B começa em 128. O endereço **127.0.0.1** (e todo o bloco 127.x.x.x) é reservado para o **Loopback**, que serve para o dispositivo se comunicar consigo mesmo para testes de software e conectividade local.

### Identificação de Classe por Padrão de Bits

A identificação da classe é feita examinando os primeiros bits do primeiro octeto:

| Classe | Primeiros bits | Intervalo do 1º octeto | Máscara padrão |
| ------ | -------------- | ---------------------- | -------------- |
| **A**  | `0xxxxxxx`     | 1 – 126                | /8             |
| **B**  | `10xxxxxx`     | 128 – 191              | /16            |
| **C**  | `110xxxxx`     | 192 – 223              | /24            |
| **D**  | `1110xxxx`     | 224 – 239              | —              |
| **E**  | `1111xxxx`     | 240 – 255              | —              |

Esse é reconhecimento de padrão de bits puro — hardware de roteamento faz isso com máscaras de bits em hardware, sem nenhum processamento de texto.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — As classes são definidas pelos primeiros bits do endereço (`0xxxxxxx`, `10xxxxxx`, `110xxxxx`). Identificar a classe de um IP é pura correspondência de padrão binário — exatamente o tipo de operação que hardware e compiladores executam o tempo todo.
- **[[Processadores]]** — Equipamentos de roteamento identificam a classe de um endereço com operações de máscara de bits em hardware. Uma única instrução AND com a máscara correta extrai os bits de classe. Isso acontece para cada pacote que passa pelo roteador, a velocidade de hardware.

## Conexão com Go

- **[[Bits e Bytes]]** — Em Go, verificar a classe de um IP envolve examinar o primeiro byte: `ip[0] & 0x80 == 0` (Classe A), `ip[0] & 0xC0 == 0x80` (Classe B), `ip[0] & 0xE0 == 0xC0` (Classe C). Operações bitwise diretas sobre bytes.
- **[[Tipos de Dados]]** — `net.IP` em Go é `[]byte`; acessar `ip[0]` dá o primeiro octeto, permitindo classificação manual ou para fins didáticos.
