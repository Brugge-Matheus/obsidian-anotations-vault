---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.5"
---
# Espaços Separados de Instruções e Dados

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.5

---

# 🗂️ 3.5.5 — Espaços Separados de Instruções e Dados

## O modelo comum: um espaço de endereçamento único

A maioria dos computadores tem **um único espaço de endereçamento** tanto para **programas** (instruções) quanto para **dados** (Figura 3.24(a)).

```
SE esse espaço de endereçamento for GRANDE o
suficiente:
  → o restante funciona bem

SE for PEQUENO DEMAIS:
  → força os programadores a encontrarem uma SAÍDA
    para fazer com que TUDO caiba no espaço de
    endereçamento
```

> 💡 **Contexto histórico:** hoje os espaços de endereçamento são enormes (32 ou 64 bits), mas antigamente (computadores de **16 bits**, com no máximo 64 KB endereçáveis) o tamanho era um **problema sério**: programa e dados disputavam os mesmos 64 KB.

## 📌 Figura 3.24 — Um espaço único vs. espaços I e D independentes

```
(a) ESPAÇO DE ENDEREÇAMENTO ÚNICO

        2³² ┌──────────────┐
            │              │
            │    Dados     │
            │              │
            ├──────────────┤
            │   Programa   │
          0 └──────────────┘

     Programa e dados dividem o MESMO espaço.
     Os dados começam DEPOIS do programa.


(b) ESPAÇOS I e D INDEPENDENTES

        Espaço I                 Espaço D
   2³² ┌────────────┐      2³² ┌────────────┐
       │ Página não │          │ Página não │
       │ utilizada  │          │ utilizada  │
       │            │          ├────────────┤
       │            │          │            │
       │            │          │   Dados    │
       │            │          │            │
       ├────────────┤          │            │
       │  Programa  │          │            │
     0 └────────────┘        0 └────────────┘

     Cada um vai de 0 até seu próprio valor máximo.
```

## A solução: dois espaços de endereçamento

Uma solução, apresentada inicialmente no **PDP-11** (16 bits), é ter **dois espaços de endereçamento diferentes**:

```
ESPAÇO I (Instruction space) → para as INSTRUÇÕES
                               (código do programa)

ESPAÇO D (Data space)        → para os DADOS
```

```
Cada espaço de endereçamento se situa entre 0 e um
VALOR MÁXIMO, em geral:

  2¹⁶ − 1   (computadores de 16 bits)
  2³² − 1   (computadores de 32 bits)

Resultado: o espaço de endereçamento DISPONÍVEL é
DOBRADO:
  → antes: 64 KB para programa + dados juntos
  → depois: 64 KB para o programa E 64 KB para os dados
```

### O papel do linker (ligador)

```
O LINKER precisa saber QUANDO endereços I e D
SEPARADOS estão sendo usados.

POR QUÊ? Porque, nesse caso, os DADOS são realocados
para o endereço virtual 0, em vez de começarem
DEPOIS do programa.
```

Veja a diferença na Figura 3.24: em (a) os dados começam **depois** do programa; em (b) os dados começam em **0** no espaço D, independentemente do tamanho do programa.

## Como funciona a paginação com espaços I e D

Em um computador com esse tipo de projeto, **ambos os espaços de endereçamento podem ser paginados de forma independente**.

```
Cada um tem:
  → sua PRÓPRIA tabela de páginas
  → seu PRÓPRIO mapeamento de páginas virtuais para
    quadros de página física

QUANDO o HARDWARE quer BUSCAR UMA INSTRUÇÃO:
  → ele sabe que deve usar o ESPAÇO I
  → e a TABELA DE PÁGINAS do espaço I

DA MESMA FORMA, os DADOS precisam passar pela
TABELA DE PÁGINAS do ESPAÇO D.
```

```
              CPU
               │
     ┌─────────┴─────────┐
     │                   │
 busca de             acesso a
 INSTRUÇÃO              DADOS
     │                   │
     ▼                   ▼
 tabela de páginas   tabela de páginas
   do espaço I         do espaço D
     │                   │
     └─────────┬─────────┘
               ▼
        memória física
```

> ✅ **Fora essa distinção**, ter espaços I e D separados **não apresenta quaisquer complicações especiais para o sistema operacional** e **dobra o espaço de endereçamento disponível**.

---

## Hoje: ainda comuns, mas para outro fim

Embora os espaços de endereçamento atuais sejam grandes, o espaço I/D separado **continua comum**, só que com **outro propósito**:

```
Em vez de serem usados para os ESPAÇOS DE
ENDEREÇAMENTO normais, agora eles são usados para
DIVIDIR A CACHE L1.

POR QUÊ? Afinal de contas, a memória AINDA é bastante
ESCASSA na cache L1.
```

```
E em alguns processadores, descobriu-se ainda que, nos
bastidores, o TLB também é PARTICIONADO:

  TLB  →  L1  +  L2

  e o TLB L1 é dividido ainda em:
     → um TLB para INSTRUÇÕES
     → um TLB para DADOS
```

> 📝 **Observação minha (não está no livro):** essa divisão em instruções e dados é a mesma ideia da chamada **arquitetura Harvard** (memórias/caminhos separados para código e dados), em contraste com a arquitetura de **von Neumann** (tudo na mesma memória). Hoje os processadores modernos são "von Neumann" na memória principal, mas "Harvard" nas **caches L1** (cache de instruções + cache de dados).

---

# ✅ Resumo do Conceito

- Normalmente, **programa e dados dividem um único espaço de endereçamento**; se ele é pequeno, os programadores precisam de truques para fazer tudo caber
- **Solução (PDP-11, 16 bits):** dois espaços independentes, **espaço I** (instruções) e **espaço D** (dados), cada um de 0 até seu máximo (2¹⁶−1 ou 2³²−1). Isso **dobra** o espaço de endereçamento disponível
- O **linker** precisa saber que há espaços I e D separados, pois os dados passam a ser realocados para o endereço virtual **0**
- Ambos os espaços podem ser paginados de forma **independente**, cada um com sua **própria tabela de páginas**: o hardware usa a tabela do espaço I para buscar instruções e a do espaço D para acessar dados. Para o SO, isso **não traz complicações especiais**
- Hoje a ideia sobrevive para **dividir a cache L1** (e o **TLB L1**, em um TLB de instruções e um de dados), porque a memória da cache L1 ainda é escassa

---

## 🔗 Notas Relacionadas

- [[Espaços de Endereçamento e Registradores-Base e Limite]] — o conceito de espaço de endereçamento
- [[Ausência de abstração de memória]] — o problema de relocação que o linker precisa tratar
- [[Tabelas de Páginas]] — a estrutura que agora existe em duplicata (uma por espaço)
- [[Acelerando a Paginação]] — o TLB, que também é dividido em instruções e dados
- [[Memória]] — hierarquia de memória e caches L1/L2
- [[Tamanho de Página]] — outra questão de projeto do sistema de paginação
