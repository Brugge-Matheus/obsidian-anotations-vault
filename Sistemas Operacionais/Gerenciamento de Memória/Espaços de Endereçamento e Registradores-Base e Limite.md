---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.2 e 3.2.1"
---
# Espaços de Endereçamento e Registradores-Base e Limite

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.2 e 3.2.1

---

# 🗂️ 3.2 — Uma Abstração de Memória: Espaços de Endereçamento

## Por que a memória física exposta é um problema

Como visto em [[Ausência de abstração de memória]], expor a memória física a processos tem várias desvantagens importantes:

```
PROBLEMA 1 — Segurança:
  Se os programas do usuário podem endereçar cada
  BYTE de memória, eles podem facilmente derrubar
  o sistema operacional, intencionalmente ou por
  acidente, provocando uma parada total no sistema
  (a não ser que exista um hardware especial como o
  esquema de bloqueio e chave do IBM 360).

  Esse problema existe mesmo que só um programa do
  usuário (aplicação) esteja executando.

PROBLEMA 2 — Multiprogramação:
  Com esse modelo, é difícil ter múltiplos programas
  executando ao mesmo tempo (revezando-se, se
  houver apenas uma CPU).

  Em computadores pessoais, é comum haver vários
  programas abertos ao mesmo tempo (processador de
  textos, um programa de e-mail, um navegador web),
  um deles tendo o foco atual, mas os outros sendo
  reativados ao clique de um mouse.

  ⚠️ Essa situação é DIFÍCIL de ser atingida quando
  não há abstração da memória física, algo tinha de
  ser feito.
```

---

## 🔢 3.2.1 — A Noção de um Espaço de Endereçamento

### Os dois problemas fundamentais a resolver

Dois problemas têm de ser solucionados para permitir que **múltiplas aplicações** estejam na memória ao mesmo tempo **sem interferir** umas com as outras:

```
1. PROTEÇÃO
2. REALOCAÇÃO
```

### A solução primitiva do IBM 360 — recapitulando

Examinamos uma solução primitiva para a **primeira** (proteção), usada no IBM 360: rotular blocos de memória com uma **chave de proteção** e comparar a chave do processo em execução com aquela de toda palavra de memória buscada.

```
⚠️ No entanto, essa abordagem em si NÃO soluciona o
SEGUNDO problema (realocação), embora ele possa ser
resolvido realocando programas à medida que eles são
carregados.

Mas essa é uma solução LENTA e COMPLICADA.
```

### A solução melhor — inventar uma nova abstração

> 💡 **Uma solução melhor é inventar uma nova abstração para a memória: o espaço de endereçamento.**

Da mesma forma que o conceito de **processo** cria uma espécie de **CPU abstrata** para executar os programas, o **espaço de endereçamento** cria uma espécie de **memória abstrata** para os programas usarem.

> 💡 **Espaço de endereçamento:** o conjunto de endereços que um processo pode usar para endereçar a memória. Cada processo tem seu próprio espaço de endereçamento, **independente** daqueles pertencentes a outros processos (exceto em algumas circunstâncias especiais nas quais os processos querem **compartilhar** seus espaços de endereçamento).

### O conceito de espaço de endereçamento é geral — vai além de memória

O conceito de um espaço de endereçamento é **muito geral** e ocorre em muitos contextos. Considere os números de telefone:

```
EXEMPLO 1 — Números de telefone:

  Nos Estados Unidos e em muitos outros países, um
  número de telefone LOCAL costuma ter sete dígitos.

  Espaço de endereçamento: 0.000.000 a 9.999.999
  (embora alguns números, como aqueles começando com
  000, não sejam usados)

EXEMPLO 2 — Portas de E/S no x86:

  Espaço de endereçamento: 0 a 16.383

EXEMPLO 3 — Endereços IPv4:

  São números de 32 bits
  Espaço de endereçamento: 0 a 2³² − 1
  (de novo, com alguns números reservados)
```

### Espaços de endereçamento não precisam ser numéricos

```
EXEMPLO — Conjunto de domínios .com da Internet:

  TAMBÉM é um espaço de endereçamento.

  Consiste em TODAS as cadeias de comprimento 2 a 63
  caracteres que podem ser feitas usando letras,
  números e hífens, seguidas por .com.
```

---

## 🔧 A Questão Prática — Dando a Cada Programa seu Próprio Espaço de Endereçamento

### O problema a resolver

Algo um tanto mais difícil é como dar a cada programa seu **próprio** espaço de endereçamento, de maneira que o **endereço 28** em um programa signifique uma localização física **diferente** do endereço 28 em outro programa.

> ⚠️ Essa é exatamente a raiz do problema da relocação que vimos com a IBM 360 — endereços iguais em programas diferentes precisam mapear para lugares **fisicamente diferentes** na memória real.

---

## 🎯 Registradores-Base e Registradores-Limite

Essa solução simples usa uma versão particularmente simples de **realocação dinâmica**.

### O mecanismo

```
O QUE ela faz:
  → mapear o espaço de endereçamento de cada
    processo em uma parte DIFERENTE da memória
    física, de uma maneira simples

A SOLUÇÃO CLÁSSICA, usada em máquinas desde o CDC
6600 (o primeiro supercomputador do mundo) até o
Intel 8088 (o coração do PC IBM original), é
EQUIPAR CADA CPU com DOIS REGISTRADORES de hardware
especiais:

  REGISTRADOR-BASE
  REGISTRADOR-LIMITE
```

### Como funciona

```
Quando esses registradores são usados, os programas
são carregados em posições de memória CONSECUTIVAS
sempre que houver espaço e SEM realocação durante o
carregamento (como mostrado na Figura 3.2(c)).

Quando um processo é EXECUTADO:
  → o registrador-BASE é carregado com o endereço
    onde SEU programa começa na memória
  → o registrador-LIMITE é carregado com o
    COMPRIMENTO do programa
```

### O exemplo numérico (retomando a Figura 3.2)

```
Valores carregados nos registradores quando o
PRIMEIRO programa é executado:
  base  = 0
  limite = 16.384

Valores carregados quando o SEGUNDO programa é
executado:
  base  = 16.384
  limite = 16.384

SE um terceiro programa de 16 KB fosse carregado
DIRETAMENTE acima do segundo e executado:
  base  = 32.768
  limite = 16.384
```

### O que acontece a cada referência de memória

> 📌 **Figura 3.3 — Registradores-base e registradores-limite podem ser usados para dar a cada processo um espaço de endereçamento em separado.**

```
Toda vez que um processo REFERENCIA a memória, seja
para buscar uma instrução ou ler ou escrever uma
palavra de dados:

  1. o HARDWARE da CPU AUTOMATICAMENTE SOMA o valor-
     base ao endereço gerado pelo processo ANTES de
     enviá-lo para o barramento de memória

  2. AO MESMO TEMPO, ele CONFERE se o endereço
     oferecido é IGUAL ou MAIOR do que o valor no
     registrador-limite

     SE FOR: uma FALTA é gerada e o acesso é
     ABORTADO
```

### Rastreando o exemplo — a instrução que antes quebrava agora funciona

```
No caso da PRIMEIRA instrução do SEGUNDO programa na
Figura 3.2(c), o processo executa a instrução:

    JMP 28

MAS o HARDWARE a TRATA como se ela fosse:

    JMP 16412
                    (16.384 + 28 = 16.412)

Portanto, ela CHEGA à instrução CMP, como esperado.
```

### Vantagem central — espaço de endereçamento PRIVADO automaticamente

> 💡 **Usar registradores-base e limite é uma maneira fácil de dar a cada processo seu próprio espaço de endereçamento privado**, pois cada endereço de memória gerado **automaticamente** tem o conteúdo do registrador-base somado a ele antes de ser enviado para a memória.

```
Em MUITAS implementações, os registradores-base e
os limite são PROTEGIDOS de tal maneira que APENAS
o sistema operacional pode modificá-los.
```

### A diferença entre o CDC 6600 e o Intel 8088

```
CDC 6600:
  → registradores-base e limite eram protegidos,
    apenas o SO podia modificá-los

Intel 8088:
  → NÃO tinha sequer um registrador-limite

  → tinha MÚLTIPLOS registradores-base, permitindo
    que o TEXTO do programa e os DADOS fossem
    realocados INDEPENDENTEMENTE

  → MAS NÃO oferecia proteção CONTRA referências à
    memória fora do intervalo válido
```

---

## ⚠️ A Desvantagem — o Custo do Registrador-Base e Limite

Uma desvantagem da realocação usando registradores-base e limite é a necessidade de realizar uma **adição** e uma **comparação** a **cada referência de memória**.

```
COMPARAÇÕES:
  → podem ser feitas RAPIDAMENTE

ADIÇÕES:
  → são LENTAS devido ao tempo de propagação do
    transporte (carry-propagation)
  → A NÃO SER que sejam usados circuitos especiais
    para adição
```

> ⚠️ Isso significa que EVERY memory access carries a small hardware cost — a soma não é gratuita, exigindo hardware dedicado para não se tornar um gargalo de desempenho.

---

# ✅ Resumo do Conceito

- Expor memória física diretamente traz dois problemas: **segurança** (programas podem derrubar o SO) e **dificuldade de multiprogramação** (múltiplos programas simultâneos)
- Dois problemas fundamentais precisam ser resolvidos: **proteção** e **realocação**
- A solução da IBM 360 (chaves de proteção) resolveu proteção, mas não realocação de forma geral
- A solução definitiva foi inventar uma nova abstração: o **espaço de endereçamento** — cada processo tem o seu, de forma independente (análogo ao processo criar uma CPU abstrata)
- Espaços de endereçamento são um conceito geral, aparecendo em números de telefone, portas de E/S, endereços IPv4, e até domínios .com — não precisam ser numéricos
- **Registradores-base e registradores-limite** implementam **realocação dinâmica** simples: o hardware soma automaticamente o valor-base a cada endereço gerado e verifica contra o limite, gerando uma falta se ultrapassado
- Usado desde o CDC 6600 até o Intel 8088 — o 8088 tinha múltiplos registradores-base (para texto e dados separadamente), mas nenhum registrador-limite, sem proteção contra acesso fora do intervalo
- A desvantagem é o custo de hardware: **adição** (lenta, devido à propagação de transporte) e **comparação** a **cada** referência de memória

---

## 🔗 Notas Relacionadas

- [[Ausência de abstração de memória]] — o problema da relocação (Figura 3.2) que os registradores-base e limite resolvem definitivamente
- [[Troca de Processos (Swapping)]] — próximo tópico, sobre como lidar quando processos não cabem todos simultaneamente na memória
- [[Hardware de Proteção]] — contexto de mecanismos de proteção de memória em geral
