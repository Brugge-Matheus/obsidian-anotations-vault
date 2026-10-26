---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.6"
---
# Páginas Compartilhadas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.6

---

# 🤝 3.5.6 — Páginas Compartilhadas

## Por que compartilhar?

Outra questão de projeto importante é o **compartilhamento**.

```
Em um GRANDE sistema de multiprogramação, é COMUM
que VÁRIOS USUÁRIOS estejam executando o MESMO
PROGRAMA ao mesmo tempo.

Mesmo um ÚNICO usuário pode estar executando VÁRIOS
PROGRAMAS que usam a MESMA BIBLIOTECA.
```

> ✅ É certamente **mais eficiente compartilhar as páginas** para evitar ter **duas cópias da mesma página na memória ao mesmo tempo**.

## O limite: nem tudo é compartilhável

```
Um PROBLEMA: nem TODAS as páginas são compartilháveis.

  → páginas SOMENTE LEITURA (como o CÓDIGO de
    programa) PODEM ser compartilhadas

  → o compartilhamento de páginas COM DADOS é MAIS
    COMPLICADO
```

---

## 🧱 Compartilhando código com espaços I e D separados

Se o sistema dá suporte a **espaços I e D separados** ([[Espaços Separados de Instruções e Dados]]), o compartilhamento de programas é **relativamente direto**:

```
Dois ou mais processos usam:

  → a MESMA tabela de páginas para seu ESPAÇO I
    (instruções — somente leitura, iguais)

  → tabelas de páginas DIFERENTES para seus
    ESPAÇOS D (dados — cada um tem os seus)
```

### Onde ficam as tabelas de páginas?

```
Em geral, em uma implementação que dá suporte ao
compartilhamento dessa forma, as TABELAS DE PÁGINAS
são ESTRUTURAS DE DADOS INDEPENDENTES da tabela de
processos.

Cada processo tem, então, DOIS PONTEIROS em sua tabela
de processos:
  → um para a tabela de páginas do ESPAÇO I
  → outro para a tabela de páginas do ESPAÇO D
```

> 📌 **Figura 3.25 — Dois processos que compartilham o mesmo programa compartilhando suas tabelas de páginas**

```
 Tabela de
 processos

 ┌──────────┐        ┌────────────┐
 │ Proc. 1  │───────►│  PROGRAMA  │◄───────┐   ← tabela de páginas
 │ I ───────┼──────► │ (espaço I) │        │     do espaço I
 │ D ───────┼──┐     └────────────┘        │     COMPARTILHADA
 ├──────────┤  │                            │
 │ Proc. 2  │  │     ┌────────────┐        │
 │ I ───────┼──┼─────┼────────────┼────────┘
 │ D ───────┼──┼──┐  │            │
 └──────────┘  │  │  └────────────┘
               │  │
               ▼  ▼
        ┌──────────┐  ┌──────────┐
        │ DADOS 1  │  │ DADOS 2  │   ← uma tabela de páginas
        │(espaço D)│  │(espaço D)│     de DADOS para cada
        └──────────┘  └──────────┘     processo (privadas)
```

```
QUANDO o escalonador escolhe um processo para ser
executado:

  1. ele usa esses PONTEIROS para LOCALIZAR as
     tabelas de páginas apropriadas

  2. ATIVA a MMU, que passa a utilizá-las
```

> ⚠️ **Sem espaços I e D separados**, os processos também podem compartilhar programas (ou, às vezes, bibliotecas), mas **o mecanismo é mais complicado**.

---

## ⚠️ Problemas do compartilhamento

Quando dois ou mais processos compartilham algum código, surgem problemas:

### 1. Remover um processo da memória

```
Suponha que os processos A e B estejam AMBOS
executando o EDITOR e COMPARTILHANDO suas páginas.

SE o escalonador decidir REMOVER A da memória
(tirando todas as suas páginas e preenchendo os
quadros vazios com algum outro programa):

  → isso fará B GERAR um GRANDE NÚMERO DE FALTAS DE
    PÁGINA para trazê-las de volta outra vez
```

### 2. Quando um processo termina

```
Quando A TERMINA a sua execução, é ESSENCIAL que o
SO saiba que as páginas AINDA ESTÃO EM USO (por B).

  → assim, o ESPAÇO delas no armazenamento não
    volátil NÃO será LIBERADO POR ACIDENTE
```

### 3. Como o SO descobre se uma página é compartilhada?

```
PESQUISAR TODAS as tabelas de páginas para ver se uma
página está sendo compartilhada normalmente é MUITO
CARO.

Por isso, ESTRUTURAS DE DADOS ESPECIAIS são
necessárias para CONTROLAR as páginas compartilhadas,
em especial se a unidade de compartilhamento for a
PÁGINA INDIVIDUAL (ou conjunto de páginas), em vez
de uma TABELA DE PÁGINAS INTEIRA.
```

---

## 📝 Compartilhando dados: cópia na escrita (copy on write)

**Compartilhar dados é mais complicado do que compartilhar código, mas não é impossível.**

### O caso do `fork` do UNIX

```
Em UNIX, após uma chamada de sistema FORK, o
PROCESSO-PAI e o PROCESSO-FILHO são solicitados a
compartilhar TANTO o código do programa QUANTO os
DADOS.
```

Em um sistema de paginação, o que se faz muitas vezes é:

```
1. dar a CADA processo a sua PRÓPRIA TABELA DE
   PÁGINAS

2. fazer AMBOS APONTAREM para o MESMO CONJUNTO de dados

→ NENHUMA cópia de páginas é realizada no instante
  do fork

3. TODAS as páginas de dados são mapeadas em AMBOS os
   processos como SOMENTE LEITURA (read-only)
```

### O que acontece quando alguém escreve

```
ENQUANTO ambos os processos apenas LEREM seus dados,
sem modificá-los:
  → a situação pode CONTINUAR (zero cópias!)

TÃO LOGO qualquer um dos processos ATUALIZE uma palavra
da memória:

  1. a VIOLAÇÃO da proteção somente leitura causa uma
     CAPTURA (trap) para o sistema operacional

  2. uma CÓPIA dessa página então é FEITA

  3. cada processo tem, a partir disso, sua PRÓPRIA
     CÓPIA PARTICULAR

  4. AMBAS as cópias estão agora configuradas para
     LER/ESCREVER → operações de escrita SUBSEQUENTES
     prosseguem SEM capturas
```

```
 fork()                       Pai escreve na pág. X

 Pai ─┐                       Pai ──► cópia PRIVADA de X (R/W)
      ├──► págs. X, Y, Z      Filho ─► X original   (R/W)
 Filho┘    (somente leitura)  
                              As páginas Y e Z (nunca
                              modificadas) continuam
                              COMPARTILHADAS.
```

> ✅ **Resultado:** aquelas páginas que **jamais** são modificadas (incluindo **todas as páginas do programa**) **não precisam ser copiadas**. **Apenas as páginas de dados que são de fato modificadas** precisam ser copiadas. Essa abordagem, chamada de **cópia na escrita** (*copy on write*), **melhora o desempenho ao reduzir o número de cópias**.

---

# ✅ Resumo do Conceito

- Compartilhar páginas evita **duas cópias da mesma página** na memória. Só páginas **somente leitura** (como código) são facilmente compartilháveis; páginas de dados são mais complicadas
- Com **espaços I e D separados**, dois processos usam a **mesma tabela de páginas do espaço I** e **tabelas de espaço D diferentes**. As tabelas de páginas ficam independentes da tabela de processos, que guarda **dois ponteiros** por processo (Figura 3.25)
- **Problemas:** remover um processo da memória faz o outro sofrer muitas faltas de página; ao terminar, o SO não pode liberar páginas ainda em uso; e como pesquisar todas as tabelas é caro, são necessárias **estruturas de dados especiais** para rastrear páginas compartilhadas
- **Dados** podem ser compartilhados com **cópia na escrita**: após `fork`, pai e filho têm tabelas próprias apontando para as mesmas páginas, todas **somente leitura**. Quando alguém escreve, ocorre uma **trap**, o SO **copia só aquela página** e cada processo passa a ter a sua cópia em leitura/escrita. Páginas nunca modificadas nunca são copiadas

---

## 🔗 Notas Relacionadas

- [[Bibliotecas Compartilhadas]] — compartilhamento em outra granularidade (3.5.7)
- [[Arquivos Mapeados em Memória]] — mecanismo geral que também permite compartilhar memória (3.5.8)
- [[Espaços Separados de Instruções e Dados]] — o que torna simples compartilhar código
- [[Controle de Carga]] — a deduplicação também usa copy on write
- [[Criação de Processos]] — o `fork` e o que pai e filho recebem
- [[Syscalls para Gerenciamento de Processos]] — a chamada de sistema `fork`
- [[Tabelas de Páginas]] — onde estão os bits de proteção que disparam a cópia
