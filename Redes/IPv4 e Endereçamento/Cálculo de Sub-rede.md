---
tags:
  - redes
  - redes/ipv4
---

# Cálculo de Sub-rede Classe A, B e C

### Guia Prático de Cálculo de Sub-rede (Método do Salto)

Este método permite descobrir rapidamente a rede, a faixa de hosts e o broadcast de qualquer IP sem precisar converter tudo para binário o tempo todo.

### Passo 1: Identificar a Máscara e o Salto

Ao receber um IP com máscara CIDR, como **192.168.0.20/26**:

1. **Converta o CIDR para Binário/Decimal:**
    - `/26` significa 26 bits "1".
    - Isso resulta em: `11111111.11111111.11111111.11000000`
    - Em decimal: **255.255.255.192**
2. **Calcule o Salto (Número Mágico):**
    - Subtraia o valor do octeto "quebrado" de 256.
    - **Salto = 256 - 192 = 64**.
    - *O que isso significa?* Que as suas sub-redes vão começar de 64 em 64.

---

### Passo 2: Montar a Tabela de Sub-redes

Com o valor do salto (64), você consegue listar todas as redes possíveis e seus limites.

| Rede (Início) | Host (Faixa Útil) | Broadcast (Fim) |
| --- | --- | --- |
| **192.168.0.0** | 1 até 62 | **192.168.0.63** |
| **192.168.0.64** | 65 até 126 | **192.168.0.127** |
| **192.168.0.128** | 129 até 190 | **192.168.0.191** |
| **192.168.0.192** | 193 até 254 | **192.168.0.255** |

**Como preencher a tabela rapidamente:**

1. **Coluna da Rede:** Comece no `.0` e vá somando o salto (64) até chegar perto de 256.
2. **Coluna do Broadcast:** É sempre o número anterior à próxima rede. (Ex: Se a segunda rede é `.64`, o broadcast da primeira é `.63`).
3. **Coluna do Host:** É tudo o que sobra no meio (Rede + 1 até Broadcast - 1).

---

### Passo 3: Regras de Ouro para Validação

Para garantir que você não errou o cálculo, verifique estas duas regras fundamentais:

- **Não existe sub-rede ímpar:** Em máscaras que dividem o último octeto (como /25, /26, /27...), o endereço de rede será sempre um número **par** (0, 64, 128, 192...).
- **Não existe broadcast par:** O endereço de broadcast será sempre um número **ímpar** (63, 127, 191, 255).

---

### Exemplo de Aplicação (Onde está o IP 192.168.0.20?)

Olhando para a sua tabela:

- O IP **.20** está entre o **.0** (Rede) e o **.63** (Broadcast).
- Portanto, ele pertence à **primeira sub-rede**.
- A rede dele é `192.168.0.0` e o broadcast é `192.168.0.63`.

---

## Conexões com SO e Go

**Por que isso importa além das redes:**

- Subnetting é, na raiz, uma partição binária do espaço de endereços — cada bit "emprestado" da porção de host divide o espaço ao meio, como uma árvore binária. Cada bit roubado **dobra** o número de sub-redes e **divide ao meio** o número de hosts disponíveis. Ver [[Bits e Bytes]].
- O "salto" (número mágico) é sempre uma potência de 2: `2^(bits de host restantes)`. Isso é pura aritmética de potências de 2, igual à que define tamanhos de páginas de memória no SO. Ver [[Operações Aritméticas]].
- Roteadores fazem operações de AND bit a bit entre IP e máscara para determinar a qual sub-rede um pacote pertence — exatamente o mesmo mecanismo que a CPU usa para alinhar endereços de memória. Ver [[Processadores]].
