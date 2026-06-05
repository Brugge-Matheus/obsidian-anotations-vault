---
tags:
  - redes
  - redes/meios
---

# Padrões de Cores dos Cabos UTP

Entender os **padrões de cores dos cabos UTP** é fundamental para fazer conexões corretas e garantir que a rede funcione sem problemas. Esses padrões definem a ordem dos fios dentro do conector RJ-45.

---

### Padrões de Cores dos Cabos UTP (T568A e T568B)

Os cabos UTP possuem 4 pares de fios trançados, totalizando 8 fios. Cada fio tem uma cor sólida ou listrada. Para conectar o cabo ao conector RJ-45, é preciso seguir uma ordem padrão para garantir que os sinais sejam transmitidos corretamente.

Existem dois padrões oficiais definidos pela **TIA/EIA-568**:

---

### 1. Padrão T568A

Ordem dos fios do pino 1 ao 8 (da esquerda para a direita, com o clipe do conector para baixo):

| Pino | Cor do Fio |
| --- | --- |
| 1 | Branco/Verde (White/Green) |
| 2 | Verde (Green) |
| 3 | Branco/Laranja (White/Orange) |
| 4 | Azul (Blue) |
| 5 | Branco/Azul (White/Blue) |
| 6 | Laranja (Orange) |
| 7 | Branco/Marrom (White/Brown) |
| 8 | Marrom (Brown) |

---

### 2. Padrão T568B

Ordem dos fios do pino 1 ao 8:

| Pino | Cor do Fio |
| --- | --- |
| 1 | Branco/Laranja (White/Orange) |
| 2 | Laranja (Orange) |
| 3 | Branco/Verde (White/Green) |
| 4 | Azul (Blue) |
| 5 | Branco/Azul (White/Blue) |
| 6 | Verde (Green) |
| 7 | Branco/Marrom (White/Brown) |
| 8 | Marrom (Brown) |

---

### Qual padrão usar?

- **T568B** é o mais comum no Brasil e nos EUA, especialmente em redes comerciais.
- **T568A** é mais comum em instalações governamentais e residenciais.
- Ambos funcionam da mesma forma, desde que as duas extremidades do cabo sigam o mesmo padrão.

---

### Tipos de cabo UTP baseados nos padrões

- **Cabo Direto (Straight-through):**
As duas extremidades seguem o **mesmo padrão** (T568A em ambas ou T568B em ambas). Usado para conectar dispositivos diferentes, como computador para switch ou roteador.
- **Cabo Cruzado (Crossover):**
Uma extremidade segue o padrão T568A e a outra o T568B. Usado para conectar dispositivos iguais diretamente, como computador a computador ou switch a switch (embora hoje em dia a maioria dos equipamentos suporte auto-MDI/MDIX e não precise mais disso).

---

### Visualização rápida

```
Pino:  1  2  3  4  5  6  7  8
T568A: W-G G W-O B  W-B O  W-Br Br
T568B: W-O O  W-G B  W-B G  W-Br Br
```

(W = White/Branco, G = Green/Verde, O = Orange/Laranja, B = Blue/Azul, Br = Brown/Marrom)

---

### Dica prática:

> Sempre use um **testador de cabos** após crimpar para garantir que os fios estão na ordem correta e que não há falhas.

---

## Conexões com SO

- **TIA-568A vs 568B:** os mesmos 8 fios, apenas com atribuições de pino diferentes. A diferença real é nos pares 1 e 2 — os pares de TX e RX são trocados entre si.
- **Cabo direto (straight-through):** ambas as pontas com o mesmo padrão. Usado para ligar NIC ao switch (dispositivos de camadas diferentes) — TX de um lado conecta ao RX do outro via o switch — [[Dispositivos de IO]].
- **Cabo cruzado (crossover):** uma ponta A, outra B. Troca TX↔RX diretamente — necessário para ligar NIC a NIC sem switch intermediário.
- **Auto-MDI/MDIX:** switches e NICs modernos detectam automaticamente a polaridade e ajustam internamente por hardware, tornando o cabo cruzado desnecessário na prática — [[Dispositivos de IO]].
