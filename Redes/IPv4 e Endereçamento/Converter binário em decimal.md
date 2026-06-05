---
tags:
  - redes
  - redes/ipv4
---

# Converter binário em decimal e vice-versa

A forma mais simples de fazer essas conversões sem usar calculadoras complexas é focar na **Tabela de 8 Bits**. Como o IPv4 é formado por octetos, você só precisa decorar 8 números.

---

### A Tabela Mágica (Pesos dos Bits)

Sempre que for converter, desenhe esta tabela. Cada posição é o dobro da anterior (da direita para a esquerda):

| 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
| --- | --- | --- | --- | --- | --- | --- | --- |

---

### 1. Binário para Decimal (Método da Soma)

**Regra:** Coloque o número binário embaixo da tabela. Onde tiver **1**, você soma o valor da tabela. Onde tiver **0**, você ignora.

**Exemplo: Converter `10100100`**

1. **Monte a tabela:**
    - **128** (bit 1) → Sim
    - 64 (bit 0) → Não
    - **32** (bit 1) → Sim
    - 16 (bit 0) → Não
    - 8 (bit 0) → Não
    - **4** (bit 1) → Sim
    - 2 (bit 0) → Não
    - 1 (bit 0) → Não
2. **Some os "Sim":** $128 + 32 + 4 = \mathbf{164}$

---

### 2. Decimal para Binário (Método do "Cabe ou Não Cabe")

**Regra:** Tente subtrair os valores da tabela (do maior para o menor). Se o valor da tabela "couber" no seu número, coloque **1** e subtraia. Se não couber, coloque **0**.

**Exemplo: Converter o número `172`**

1. **$172 \ge 128?$** Sim. Coloque **1**. (Sobra $172 - 128 = 44$)
2. **$44 \ge 64?$** Não. Coloque **0**. (Continua sobrando 44)
3. **$44 \ge 32?$** Sim. Coloque **1**. (Sobra $44 - 32 = 12$)
4. **$12 \ge 16?$** Não. Coloque **0**. (Continua sobrando 12)
5. **$12 \ge 8?$** Sim. Coloque **1**. (Sobra $12 - 8 = 4$)
6. **$4 \ge 4?$** Sim. Coloque **1**. (Sobra $4 - 4 = 0$)
7. **$0 \ge 2?$** Não. Coloque **0**.
8. **$0 \ge 1?$** Não. Coloque **0**.

**Resultado:** `10101100`

---

### Dica de Ouro: Os "Números de Máscara"

Como as máscaras de rede sempre têm os bits `1` em sequência, as conversões mais comuns são sempre as mesmas somas acumuladas:

- `10000000` = **128**
- `11000000` = **192** $(128+64)$
- `11100000` = **224** $(192+32)$
- `11110000` = **240** $(224+16)$
- `11111000` = **248** $(240+8)$
- `11111100` = **252** $(248+4)$
- `11111110` = **254** $(252+2)$
- `11111111` = **255** $(254+1)$

**Resumo Visual:**

- **Binário $\rightarrow$ Decimal:** "Ligue" os valores da tabela e some.
- **Decimal $\rightarrow$ Binário:** Vá subtraindo os valores da tabela do maior para o menor.

---

## Conexões com SO e Go

**Por que isso importa além das redes:**

- Cada bit da tabela acima é uma potência de 2 — a mesma aritmética que define tamanhos de tipos, flags de permissão e endereços de memória. Ver [[Bits e Bytes]].
- A ULA (Unidade Lógica e Aritmética) do processador executa essa conversão em hardware a cada ciclo de clock — operações de deslocamento de bits (`>>`, `<<`) e AND/OR fazem exatamente isso. Ver [[Processadores]].
- Em Go, `strconv.FormatInt` e `strconv.ParseInt` abstraem essa conversão diretamente:
  ```go
  bin := strconv.FormatInt(172, 2)   // "10101100"
  dec, _ := strconv.ParseInt("10101100", 2, 64) // 172
  ```
  O parâmetro `base` é exatamente a base posicional da tabela mágica. Ver [[Tipos de Dados]].
