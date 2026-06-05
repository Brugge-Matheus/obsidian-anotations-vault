---
tags:
  - redes
  - redes/ipv4
---

# Determinando Rede, Host e Broadcast

Como identificar os três componentes fundamentais de qualquer rede (Rede, Host e Broadcast) para as três classes principais. O segredo está em observar onde a **Máscara de Rede** faz o "corte" entre o que pertence à vizinhança (Rede) e o que pertence ao dispositivo (Host).

---

## Regra Geral de Identificação

Para qualquer classe, a lógica de cálculo é sempre a mesma:

1. **Endereço de Rede:** Pegue a parte do IP que é rede e preencha toda a parte de host com **bits 0**.
2. **Endereço de Broadcast:** Pegue a parte do IP que é rede e preencha toda a parte de host com **bits 1** (decimal 255).
3. **Primeiro Host Útil:** Endereço de Rede + 1.
4. **Último Host Útil:** Endereço de Broadcast - 1.

### Em termos de operações bitwise:

- **Rede** = `IP AND máscara` (zera todos os bits de host)
- **Broadcast** = `rede OR (NOT máscara)` = `rede OR wildcard_mask` (coloca 1 em todos os bits de host)
- **Primeiro host** = rede + 1
- **Último host** = broadcast - 1

---

## 1. Classe A (Máscara `/8` ou `255.0.0.0`)

- **Divisão:** `[REDE] . [HOST] . [HOST] . [HOST]`
- **Exemplo:** IP `10.50.100.20`

| Componente             | Cálculo                           | Resultado         |
| ---------------------- | --------------------------------- | ----------------- |
| **Rede**               | Mantém o 1º octeto, zera os outros 3 | **10.0.0.0**   |
| **Primeiro Host útil** | Rede + 1                          | **10.0.0.1**      |
| **Último Host útil**   | Broadcast - 1                     | **10.255.255.254**|
| **Broadcast**          | Mantém o 1º octeto, 255 nos outros 3 | **10.255.255.255**|

---

## 2. Classe B (Máscara `/16` ou `255.255.0.0`)

- **Divisão:** `[REDE] . [REDE] . [HOST] . [HOST]`
- **Exemplo:** IP `172.16.80.50`

| Componente        | Cálculo                               | Resultado          |
| ----------------- | ------------------------------------- | ------------------ |
| **Rede**          | Mantém os 2 primeiros, zera os outros 2 | **172.16.0.0**   |
| **Primeiro Host** | Rede + 1                              | **172.16.0.1**     |
| **Último Host**   | Broadcast - 1                         | **172.16.255.254** |
| **Broadcast**     | Mantém os 2 primeiros, 255 nos outros 2 | **172.16.255.255**|

---

## 3. Classe C (Máscara `/24` ou `255.255.255.0`)

- **Divisão:** `[REDE] . [REDE] . [REDE] . [HOST]`
- **Exemplo:** IP `192.168.1.15`

| Componente        | Cálculo                               | Resultado          |
| ----------------- | ------------------------------------- | ------------------ |
| **Rede**          | Mantém os 3 primeiros, zera o último  | **192.168.1.0**    |
| **Primeiro Host** | Rede + 1                              | **192.168.1.1**    |
| **Último Host**   | Broadcast - 1                         | **192.168.1.254**  |
| **Broadcast**     | Mantém os 3 primeiros, 255 no último  | **192.168.1.255**  |

---

## Tabela Comparativa

Esta tabela ajuda a visualizar onde os valores "0" e "255" entram conforme a classe:

| Classe | Máscara | Exemplo de IP | Rede         | Broadcast         | Hosts Úteis |
| ------ | ------- | ------------- | ------------ | ----------------- | ----------- |
| **A**  | `/8`    | `10.x.y.z`   | `10.0.0.0`   | `10.255.255.255`  | 16.777.214  |
| **B**  | `/16`   | `172.16.y.z` | `172.16.0.0` | `172.16.255.255`  | 65.534      |
| **C**  | `/24`   | `192.168.1.z`| `192.168.1.0`| `192.168.1.255`   | 254         |

---

## Resumo de Regras de Ouro

1. **O Endereço de Rede** sempre termina em **0** (ou múltiplos dependendo da sub-rede) e nunca pode ser usado em um PC.
2. **O Endereço de Broadcast** sempre termina em **255** (em máscaras padrão) e é usado para "gritar" para todos na rede.
3. **O Gateway Padrão** (seu roteador) geralmente recebe o **Primeiro Host Útil** (`.1`), embora tecnicamente pudesse ser qualquer um da faixa útil.
4. **O Número de Hosts** é sempre calculado pela fórmula $2^n - 2$, onde $n$ é o número de bits de host (zeros na máscara). Subtraímos 2 justamente para excluir a Rede e o Broadcast.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — Rede = `IP AND máscara` (AND bit a bit). Broadcast = `rede OR (NOT máscara)` — operação OR com a wildcard mask (todos os bits de host em 1). Essas são as operações binárias fundamentais que a CPU executa para qualquer decisão de roteamento.
- **[[Operações Aritméticas]]** — Broadcast usa NOT (complemento bit a bit) da máscara seguido de OR. Em termos de código: `broadcast[i] = network[i] | ^mask[i]` — OR com o complemento da máscara.
- **[[System Calls]]** — O kernel usa exatamente esses cálculos ao processar a syscall `sendto()`: ele verifica se o destino está na mesma sub-rede (mesmo resultado de `IP AND máscara`) para decidir se entrega localmente via ARP ou encaminha para o gateway. Essa decisão acontece no kernel para cada pacote enviado.

## Conexão com Go

- **[[Bits e Bytes]]** — Em Go: `network := ip.Mask(mask)` calcula o endereço de rede. Para o broadcast: `for i := range broadcast { broadcast[i] = network[i] | ^mask[i] }`.
- **[[Operações Aritméticas]]** — `net.IPNet.Contains(ip)` verifica se um IP está na sub-rede — internamente faz `(ip & mask) == network`. Isso retorna verdadeiro se o IP pertence à rede, falso caso contrário.
- **[[Tipos de Dados]]** — `net.IPNet` encapsula o par `{IP net.IP, Mask net.IPMask}`. O método `Contains` usa exatamente a comparação bitwise descrita acima.
