---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Introdução e Seções 3.1 e 3.1.1"
---
# Ausência de Abstração de Memória

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Introdução e Seções 3.1 e 3.1.1

---

# 🧠 Capítulo 3 — Gerenciamento de Memória: Introdução

## O que o programador gostaria de ter

A memória principal (RAM) é um recurso importante que deve ser gerenciado com cuidado. Apesar de o computador pessoal médio hoje em dia ter **100.000 vezes mais memória** do que o IBM 7094 — o maior computador do mundo no início da década de 1960 — os programas estão ficando maiores mais rapidamente do que as memórias.

> ⚠️ Parafraseando a **Lei de Parkinson**: "programas tendem a se expandir a fim de preencherem a memória disponível para conte-los."

O que todo programador gostaria de ter é uma memória **privada, infinitamente grande e rápida**, que fosse **não volátil** — isto é, que não perdesse seu conteúdo quando faltasse energia elétrica. E, aproveitando o assunto, por que não torná-la **barata** também? Infelizmente, a tecnologia ainda não produz essas memórias.

## A solução — hierarquia de memórias

Ao longo dos anos, as pessoas descobriram o conceito de **hierarquia de memórias**, em que os computadores têm:
- alguns **megabytes** de memória **cache** volátil, cara e muito rápida
- alguns **gigabytes** de memória principal volátil de velocidade e custo médios
- alguns **terabytes** de armazenamento em disco em estado sólido ou magnético não volátil, barato e lento

Sem mencionar o armazenamento removível, como os dispositivos USB. É função do sistema operacional **abstrair** essa hierarquia em um modelo útil e então **gerenciar a abstração**.

> 💡 **Gerenciador de memória:** a parte do sistema operacional que gerencia (parte da) hierarquia de memórias. Sua função é gerenciar a memória de modo eficiente: controlar quais partes estão sendo usadas, alocar memória para processos quando eles precisam dela e liberá-la quando tiverem terminado.

O gerenciamento do nível mais baixo de memória (cache) é feito normalmente pelo **hardware**, então o foco deste capítulo está no modelo de memória principal apresentado ao programador e em como ela pode ser gerenciada. As abstrações para o armazenamento permanente (o disco) serão tratadas no próximo capítulo. Examinaremos primeiro os esquemas mais simples possíveis, e então gradualmente avançaremos para os esquemas cada vez mais elaborados.

---

# 🚫 3.1 — Ausência de Abstração de Memória

## O modelo mais simples possível

A abstração de memória mais simples é **não ter abstração alguma**.

```
Os primeiros computadores de grande porte (antes de 1960)
Os primeiros minicomputadores (antes de 1970)
Os primeiros computadores pessoais (antes de 1980)

→ NÃO tinham abstração de memória
→ cada programa via apenas a memória FÍSICA
```

### Como funcionava

Quando um programa executava uma instrução como:

```asm
MOV REGISTER1, 1000
```

o computador **só movia** o conteúdo da memória física da posição **1000** para o registrador REGISTER1. Assim, o modelo de memória apresentado ao programador era apenas a **memória física** — um conjunto de endereços de **0** a algum máximo, cada endereço correspondendo a uma célula contendo algum número de **bits**, normalmente oito.

```
Endereço:  0   1   2   3   ...   MAX
Conteúdo: [8b][8b][8b][8b]  ...  [8b]
```

> ⚠️ **A consequência mais grave dessas condições:** não era possível ter **dois programas em execução na memória ao mesmo tempo**. Se o primeiro programa escrevesse um novo valor para, digamos, a posição 2000, esse valor **apagaria** qualquer valor que o segundo programa estivesse armazenando ali. Nada funcionaria, e ambos os programas deixariam de funcionar quase que imediatamente.

---

## 🗺️ Três maneiras simples de organizar a memória

Embora o modelo seja apenas de memória física, várias opções são possíveis para a **organização** dessa memória entre o sistema operacional e o processo de usuário.

> 📌 **Figura 3.1 — Três maneiras simples de organizar a memória com um sistema operacional e um processo de usuário. Também existem outras possibilidades.**

```
(a)                    (b)                    (c)
0xFFF...
┌──────────────┐      ┌──────────────┐      ┌──────────────────┐
│  Programa    │      │  Sistema     │      │  Drivers de       │
│  do usuário  │      │  operacional │      │  dispositivo      │
│              │      │  em ROM      │      │  em ROM           │
├──────────────┤      ├──────────────┤      ├──────────────────┤
│  Sistema     │      │  Programa    │      │  Programa         │
│  operacional │      │  do usuário  │      │  do usuário       │
│  em RAM      │      │              │      │                   │
├──────────────┤      ├──────────────┤      ├──────────────────┤
│              │      │              │      │  Sistema          │
│      0       │      │      0       │      │  operacional      │
│              │      │              │      │  em RAM           │
│              │      │              │      │      0            │
└──────────────┘      └──────────────┘      └──────────────────┘
```

### (a) — Sistema operacional na parte inferior, em RAM

```
O SO fica na parte BAIXA da memória, em RAM
(random access memory — memória de acesso aleatório)

O programa do usuário fica ACIMA dele
```

### (b) — Sistema operacional em ROM, no topo

```
O SO fica no TOPO da memória, em ROM
(read-only memory — memória apenas para leitura)

O programa do usuário fica ABAIXO dele
```

### (c) — Drivers em ROM no topo, SO em RAM embaixo

```
Os drivers de dispositivo talvez estejam no TOPO
da memória, em ROM

O restante do sistema operacional fica em RAM,
bem abaixo

O programa do usuário fica entre os dois
```

### O uso histórico e atual de cada modelo

```
MODELO (a):
  → usado FORMALMENTE em computadores de grande porte
    e minicomputadores
  → ATUALMENTE quase não é mais usado

MODELO (b):
  → utilizado em ALGUNS computadores portáteis e
    sistemas embarcados

MODELO (c):
  → estava presente nos PRIMEIROS computadores
    pessoais (ex: executando o MS-DOS)
  → a porção do sistema na ROM era chamada de
    BIOS (basic input output system —
    sistema básico de E/S)
```

> ⚠️ **A desvantagem dos modelos (a) e (c):** um erro no programa do usuário pode **apagar por completo** o sistema operacional, possivelmente com resultados **desastrosos** — já que não há proteção de hardware separando o programa do SO.

---

## 🖥️ Como o sistema é usado quando organizado dessa maneira

Quando o sistema está organizado dessa maneira, geralmente **só um processo de cada vez** pode estar executando.

```
FLUXO DE EXECUÇÃO:

1. Usuário digita um comando

2. O sistema operacional copia o programa solicitado
   do armazenamento não volátil para a memória

3. O sistema operacional o executa

4. Quando o processo termina, o sistema operacional
   exibe um PROMPT (ou aviso) de comando e espera
   por um novo comando do usuário

5. Quando o sistema operacional recebe o comando,
   ele carrega um programa NOVO para a memória,
   SOBRESCREVENDO o primeiro
```

> 💡 **Este conceito é chamado de *swapping* (troca de processos)** e será discutido em detalhes mais adiante. Desde que exista apenas um programa de cada vez na memória, não há conflitos.

---

## 🔀 3.1.1 — Executando Múltiplos Programas sem uma Abstração de Memória

### Paralelismo via múltiplas threads

Uma maneira de se conseguir algum **paralelismo** em um sistema sem abstração de memória é programá-lo com **múltiplas threads**.

```
Como todas as threads em um processo DEVEM ver a
MESMA imagem da memória, o fato de elas serem
FORÇADAS a fazê-lo NÃO é um problema.

Embora essa ideia funcione, ela é de uso LIMITADO,
pois o que muitas vezes as pessoas QUEREM é que
programas NÃO RELACIONADOS estejam executando ao
mesmo tempo — algo que a abstração de threads NÃO
oferece.
```

> ⚠️ Além disso, qualquer sistema que seja tão primitivo a ponto de não proporcionar qualquer abstração de memória **provavelmente não** proporciona uma abstração de threads.

### A solução real — salvar e restaurar a memória inteira

No entanto, mesmo sem uma abstração de memória, é possível executar **múltiplos programas ao mesmo tempo**.

```
O que um sistema operacional precisa fazer:

1. Salvar o conteúdo INTEIRO da memória em um
   arquivo de disco

2. Depois trazer e executar o programa SEGUINTE

3. Desde que exista APENAS UM de cada vez na
   memória, não há conflitos

Esse conceito (SWAPPING — troca de processos)
será discutido a seguir.
```

### Múltiplos programas SIMULTANEAMENTE — usando hardware especial

Com a adição de algum **hardware especial**, é possível executar múltiplos programas **simultaneamente**, mesmo sem *swapping*.

#### A solução dos primeiros modelos da IBM 360

```
A MEMÓRIA foi dividida em BLOCOS DE 2 KB

A CADA BLOCO foi designada uma CHAVE DE PROTEÇÃO
de 4 BITS

Essas chaves eram armazenadas em REGISTRADORES
ESPECIAIS dentro da CPU

Exemplo — máquina com 1 MB de memória:
  → necessita de apenas 512 desses registradores
    de 4 bits
  → para um total de 256 bytes de armazenamento
    de chaves
```

> 💡 **PSW (*Program Status Word* — palavra de estado do programa):** um registrador especial que, além de conter o estado do programa, também continha uma **chave de 4 bits**.

```
O HARDWARE do 360 gerava uma CAPTURA (trap) em
qualquer tentativa de um processo em execução de
acessar a memória com um código de proteção
DIFERENTE daquele da chave PSW.

Visto que APENAS o sistema operacional podia mudar
as chaves de proteção, os processos do usuário eram
IMPEDIDOS de interferir uns com os outros e com o
próprio sistema operacional.
```

---

## ⚠️ O Problema Importante — Relocação

No entanto, essa solução (chaves de proteção da IBM 360) tinha um problema importante, descrito com dois programas de mesmo tamanho carregados consecutivamente na memória.

### O cenário

```
DOIS programas, cada um com 16 KB, como mostrado
na Figura 3.2(a) e (b).

O PRIMEIRO está sombreado para indicar que ele tem
uma chave de memória DIFERENTE da do segundo.

O primeiro programa COMEÇA com um salto para o
endereço 24, que contém uma instrução MOV.

O segundo INICIA saltando para o endereço 28, que
contém uma instrução CMP.
```

> 📌 **Figura 3.2 — Exemplo do problema de relocação. (a) Um programa de 16 KB. (b) Outro programa de 16 KB. (c) Os dois programas carregados consecutivamente na memória.**

```
(a) Programa 1            (b) Programa 2           (c) Ambos carregados
                                                     consecutivamente

  0        32764             0        32764           0        32764
  │           │               │           │             │           │
 CMP      16412              CMP      16412            CMP      16412
  ⋮                           ⋮                          ⋮
JMP 28    16384              JMP 28    16384          JMP 28    16384
  0       16380               0       16380             0       16380
  ⋮                           ⋮                          ⋮
ADD          28              CMP          28           ADD          28
MOV          24                            24           MOV          24
                                                         
JMP 24        0              JMP 28        0           JMP 24        0
```

**Quando os dois programas são carregados consecutivamente na memória, começando no endereço 0**, temos a situação da Figura 3.2(c). Presumimos que o sistema operacional está na região **alta** da memória e, portanto, não aparece.

### O que acontece após os programas serem carregados

```
Após os programas terem sido carregados, eles podem
ser EXECUTADOS.

Considerando que eles têm chaves de memória
DIFERENTES, NENHUM dos dois pode danificar o outro.

MAS o problema é de uma natureza DIFERENTE.
```

### A execução do PRIMEIRO programa — funciona bem

```
Quando o primeiro programa INICIALIZA, ele executa
a instrução JMP 24, que salta para a instrução
esperada, como esperado.

Esse programa funciona NORMALMENTE.
```

### A execução do SEGUNDO programa — DESASTRE

```
No entanto, após o primeiro programa ter executado
por tempo suficiente, o sistema operacional pode
decidir executar o SEGUNDO programa, que foi
carregado ACIMA do primeiro, no endereço 16.384.

A PRIMEIRA instrução executada é JMP 28, que SALTA
para a instrução ADD no PRIMEIRO programa, EM VEZ
DE para a instrução CMP ESPERADA.

⚠️ É MUITO PROVÁVEL que o programa QUEBRE bem
antes de 1 segundo.
```

### O problema fundamental

> ⚠️ **O problema fundamental aqui é que ambos os programas referenciam a memória FÍSICA ABSOLUTA, e não é isso que queremos, de forma alguma.** Queremos que cada programa possa referenciar um conjunto **privado** de endereços **local a ele**.

---

## 🔧 A Solução Temporária da IBM 360 — Realocação Estática

O que a IBM 360 utilizou como solução temporária foi **modificar o segundo programa dinamicamente enquanto o carregava na memória**, usando uma técnica conhecida como **realocação estática**.

> 💡 **Realocação estática:** técnica em que, quando um programa é carregado em um endereço diferente de 0 (digamos, no endereço 16.384), uma constante correspondente a esse endereço-base é **somada a cada endereço de programa durante o processo de carregamento**. Assim, "JMP 28" tornou-se "JMP 16.412" e assim por diante.

```
Quando um programa estava carregado no endereço
16.384:
  a constante 16.384 era SOMADA a cada endereço de
  programa DURANTE o processo de carregamento

  "JMP 28" → torna-se → "JMP 16.412"
```

### As limitações da realocação estática

```
Embora esse mecanismo FUNCIONE se feito de maneira
CORRETA, ele NÃO é uma solução muito GERAL e deixa
o carregamento LENTO.

Além disso, exige INFORMAÇÕES ADICIONAIS em TODOS
os programas executáveis para indicar quais
palavras contêm endereços (RELOCÁVEIS) e quais NÃO.
```

### O problema de distinguir endereço de constante

```
Afinal, o "28" na Figura 3.2(b) DEVE ser realocado,
mas uma instrução como:

    MOV REGISTER1, 28

que move o número 28 para REGISTER1 NÃO deve ser
realocada.

O CARREGADOR precisa, de alguma maneira, DISTINGUIR
o que é um ENDEREÇO e o que é uma CONSTANTE.
```

---

## 🕰️ A Repetição da História e o Estado Atual

Por fim, como já foi destacado no Capítulo 1, a história tende a se repetir no mundo dos computadores.

```
Embora o endereçamento direto de memória física seja
apenas uma memória DISTANTE (lamentamos) nos
computadores de grande porte, minicomputadores,
computadores DESKTOP, NOTEBOOKS e SMARTPHONES:

  → a FALTA de uma abstração de memória AINDA é
    comum em SISTEMAS EMBARCADOS e CARTÕES
    INTELIGENTES

Dispositivos como:
  rádios
  máquinas de lavar roupas
  fornos de micro-ondas

→ estão TODOS cheios de SOFTWARE (em ROM)
→ na maioria dos casos, o SOFTWARE endereça
  MEMÓRIA ABSOLUTA

⚠️ Isso FUNCIONA porque todos os programas são
CONHECIDOS ANTECIPADAMENTE e os usuários NÃO são
livres para executar seu próprio software na sua
torradeira.
```

### A ironia moderna — o retorno das chaves de proteção

```
Os modernos processadores Intel x86 têm formas MUITO
MAIS AVANÇADAS de gerenciamento e ISOLAMENTO de
memória do que a simples combinação de chaves de
proteção e realocação estática do IBM 360.

Apesar disso, a Intel começou a adicionar essas
chaves de proteção EXATAS (e aparentemente
ANTIQUADAS) às suas CPUs apenas em 2017 — mais de
50 ANOS após o primeiro IBM 360 entrar em uso.

Agora elas são apresentadas como uma IMPORTANTE
INOVAÇÃO para aumentar a SEGURANÇA.
```

---

# ✅ Resumo do Conceito

- A abstração de memória **mais simples é não ter abstração alguma**: cada programa vê e acessa diretamente a **memória física**, com endereços que vão de 0 até algum máximo — sistema usado nos primeiros computadores de grande porte, minicomputadores e PCs
- Nesse modelo, **só um processo pode estar na memória de cada vez** — dois programas simultâneos se sobrescreveriam mutuamente
- Três organizações são possíveis (Figura 3.1): **(a)** SO em RAM na parte baixa da memória; **(b)** SO em ROM no topo; **(c)** drivers em ROM no topo, SO em RAM embaixo (usado pelo MS-DOS, com a porção em ROM chamada **BIOS**)
- Modelos (a) e (c) têm a desvantagem de um erro no programa do usuário poder **apagar completamente** o sistema operacional
- **Paralelismo por threads** funciona nesse modelo (threads compartilham a mesma imagem de memória por natureza), mas é limitado — não permite programas **não relacionados** rodando simultaneamente
- **Swapping** (troca de processos) permite múltiplos programas "ao mesmo tempo" salvando o conteúdo inteiro da memória em disco antes de carregar o próximo
- A IBM 360 conseguiu execução realmente **simultânea** com **chaves de proteção de hardware** (4 bits por bloco de 2KB, armazenadas na **PSW**) — o hardware gerava uma **captura (trap)** em acessos com chave incompatível
- O **problema da relocação** surgia quando dois programas eram carregados em endereços diferentes — ambos referenciavam **memória física absoluta**, causando quebras (ex: `JMP 28` saltando para o endereço errado quando o programa é carregado em outra posição)
- A solução temporária foi a **realocação estática**: somar o endereço-base a cada endereço de programa **durante o carregamento** — funcional, mas lenta, não geral, e exigindo informação adicional para distinguir endereços de constantes
- A ausência de abstração de memória ainda é comum em **sistemas embarcados e cartões inteligentes**, onde o software é conhecido antecipadamente
- Ironicamente, a Intel só adicionou chaves de proteção semelhantes às da IBM 360 em **2017** — mais de 50 anos depois — apresentadas como inovação de segurança moderna

---

## 🔗 Notas Relacionadas

- [[Memória]] — o contexto de hardware sobre RAM, ROM e hierarquia de memórias
- [[Hardware de Proteção]] — os mecanismos de proteção entre processos que evoluíram a partir das chaves da IBM 360
- [[Espaços de Endereçamento]] — a abstração que resolve definitivamente o problema de relocação visto aqui
- [[Memória Virtual]] — a evolução completa do gerenciamento de memória a partir destes modelos primitivos
