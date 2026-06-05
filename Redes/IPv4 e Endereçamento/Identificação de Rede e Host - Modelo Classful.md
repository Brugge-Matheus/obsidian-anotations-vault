---
tags:
  - redes
  - redes/ipv4
---

# Identificação de Rede e Host — Modelo Classful

Um endereço IP não é apenas um número aleatório; ele é dividido em duas partes lógicas, como se fosse um **CEP (Rede)** e o **Número da Casa (Host)**.

Essa divisão é o que permite aos roteadores saberem se um pacote deve ser entregue na rede local ou se deve ser enviado para outra rede na internet.

---

## Os Dois Componentes do IP

### 1. ID de Rede (Network ID)

É a parte do endereço que identifica a "rua" ou o "condomínio" onde o dispositivo está. Todos os dispositivos que pertencem à mesma rede física ou lógica devem compartilhar o mesmo ID de Rede para conseguirem se comunicar diretamente.

- **Função:** Usado pelos roteadores para encaminhar pacotes entre redes diferentes.

### 2. ID de Host (Host ID)

É a parte que identifica o dispositivo específico (computador, celular, servidor) dentro daquela rede. Cada dispositivo em uma mesma rede deve ter um ID de Host exclusivo.

- **Função:** Usado para a entrega final do pacote dentro da rede local.

---

## Modelo Classful (Baseado em Classes)

O modelo **Classful** foi a arquitetura original de endereçamento da Internet (utilizada entre 1981 e 1993). Nesse sistema, a divisão entre o que é **Rede** e o que é **Host** é rígida e determinada automaticamente pelo valor do primeiro octeto do endereço IP.

Neste modelo, não era necessário informar a máscara de sub-rede separadamente, pois ela era **implícita** (o roteador "olhava" para o primeiro número e já sabia a classe).

### 1. Classe A: Grandes Redes

Destinada a governos e corporações gigantescas.

- **Intervalo do 1º Octeto:** 1 a 126 (O 127 é reservado para loopback).
- **Divisão:** O **1º octeto** identifica a Rede; os **3 octetos restantes** identificam os Hosts.
- **Máscara Padrão:** `255.0.0.0` (ou `/8`).
- **Visualização:** `[REDE] . [HOST] . [HOST] . [HOST]`
- **Exemplo:** No IP `10.50.100.1`:
  - ID de Rede: `10`
  - ID de Host: `50.100.1`

### 2. Classe B: Médias Redes

Destinada a universidades e grandes empresas.

- **Intervalo do 1º Octeto:** 128 a 191.
- **Divisão:** Os **2 primeiros octetos** identificam a Rede; os **2 octetos restantes** identificam os Hosts.
- **Máscara Padrão:** `255.255.0.0` (ou `/16`).
- **Visualização:** `[REDE] . [REDE] . [HOST] . [HOST]`
- **Exemplo:** No IP `172.16.50.1`:
  - ID de Rede: `172.16`
  - ID de Host: `50.1`

### 3. Classe C: Pequenas Redes

Destinada a redes domésticas e pequenos escritórios.

- **Intervalo do 1º Octeto:** 192 a 223.
- **Divisão:** Os **3 primeiros octetos** identificam a Rede; apenas o **último octeto** identifica o Host.
- **Máscara Padrão:** `255.255.255.0` (ou `/24`).
- **Visualização:** `[REDE] . [REDE] . [REDE] . [HOST]`
- **Exemplo:** No IP `192.168.1.10`:
  - ID de Rede: `192.168.1`
  - ID de Host: `10`

---

## Resumo de Capacidade por Classe

| Classe | Bits de Rede | Bits de Host | Nº de Redes Possíveis | Hosts por Rede |
| ------ | ------------ | ------------ | --------------------- | -------------- |
| **A**  | 8 bits       | 24 bits      | 128                   | 16.777.214     |
| **B**  | 16 bits      | 16 bits      | 16.384                | 65.534         |
| **C**  | 24 bits      | 8 bits       | 2.097.152             | 254            |

## Por que o modelo Classful entrou em desuso?

A principal falha era a **falta de flexibilidade**. Se uma empresa tivesse 300 computadores, ela não cabia em uma Classe C (máximo 254). Ela era obrigada a usar uma Classe B, que suporta 65.534 hosts. Isso significava que mais de 65.000 endereços IP ficavam "presos" a uma empresa que não os utilizava, acelerando o esgotamento dos endereços IPv4 no mundo.

### A matemática por trás da divisão Rede/Host

O **Network ID** é calculado por: `IP AND máscara` — operação bitwise AND bit a bit entre o IP e a máscara de rede.

O **Host ID** é calculado por: `IP AND (NOT máscara)` — AND com a máscara invertida (wildcard mask). Isso isola apenas os bits de host.

Exemplos:
```
IP:      10.50.100.1  →  00001010 . 00110010 . 01100100 . 00000001
Máscara: 255.0.0.0    →  11111111 . 00000000 . 00000000 . 00000000
AND:     10.0.0.0     →  00001010 . 00000000 . 00000000 . 00000000  ← Rede
```

Essa é exatamente a lógica que roteadores executam em hardware para cada pacote recebido — uma única instrução AND de 32 bits.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — O Network ID é calculado por `IP AND máscara` (operação AND bit a bit). O Host ID é `IP AND (NOT máscara)`. Essas são operações binárias fundamentais — o mesmo tipo de operação que a CPU executa em qualquer instrução lógica.
- **[[Processadores]]** — Roteadores executam esse AND em hardware para cada pacote. É uma operação de 32 bits que ocorre em um único ciclo de CPU. Toda decisão de roteamento se reduz a esse cálculo bitwise.
- **[[Operações Aritméticas]]** — O AND e NOT usados aqui são operações da ALU (Arithmetic Logic Unit) do processador — exatamente as mesmas instruções de nível de máquina estudadas em arquitetura de computadores.

## Conexão com Go

- **[[Tipos de Dados]]** — `ip.Mask(mask)` em Go executa exatamente o `IP AND máscara`. Retorna um `net.IP` com os bits de host zerados — o endereço de rede. `mask` é um `net.IPMask`, que também é `[]byte`.
- **[[Bits e Bytes]]** — Toda a operação é sobre bytes: AND byte a byte entre `ip[i]` e `mask[i]` para cada um dos 4 bytes do IPv4.
- **[[Operações Aritméticas]]** — Implementar manualmente: `networkID[i] = ip[i] & mask[i]` e `hostID[i] = ip[i] & ^mask[i]` (NOT da máscara em Go usa o operador `^`).
