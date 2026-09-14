---
tags:
  - sistemas-operacionais
  - so/escalonamento
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.5.5"
---
# Política versus Mecanismo

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.5.5

---

# ⚖️ 2.5.5 — Política *versus* Mecanismo

## O pressuposto que fizemos até agora

Até o momento, presumimos tacitamente que todos os processos no sistema pertencem a usuários **diferentes** e estão, portanto, **competindo** pela CPU. Embora isso seja muitas vezes verdadeiro, às vezes acontece de um processo ter **muitos filhos** executados sob seu controle.

```
Exemplo:
  Um processo de sistema de gerenciamento de banco
  de dados pode ter MUITOS filhos.

  Cada filho pode estar funcionando em uma solicitação
  diferente, ou cada um pode ter alguma função
  específica para realizar:
    → análise sintática de consultas
    → acesso ao disco
    → etc.
```

---

## 🧠 O problema — quem sabe o que é importante?

```
É INTEIRAMENTE possível que o PRINCIPAL processo tenha
uma ideia EXCELENTE de qual dos filhos é o mais
importante (ou tenha tempo crítico) e qual é o menos
importante.
```

> ⚠️ **Infelizmente, nenhum dos escalonadores discutidos até aqui aceita qualquer entrada dos processos do usuário sobre decisões de escalonamento.** Como resultado, o escalonador **raramente faz a melhor escolha**.

O escalonador do núcleo, olhando de fora, não tem como saber que um filho específico está trabalhando em uma consulta urgente enquanto outro apenas faz manutenção de rotina em segundo plano. Essa informação existe — mas fica presa dentro do processo-pai, sem canal para chegar ao escalonador.

---

## 🔓 A Solução — Separar Mecanismo de Política

> 💡 **Separação de mecanismo e política:** um princípio há muito estabelecido (Levin *et al.*, 1975). O que isso significa é que o **algoritmo de escalonamento** é **parametrizado** de alguma maneira, mas os **parâmetros** podem ser preenchidos pelos **processos dos usuários**.

```
MECANISMO — vive no NÚCLEO
  → o "COMO" do escalonamento
  → o algoritmo genérico que executa as decisões

POLÍTICA — estabelecida por um processo do USUÁRIO
  → o "O QUÊ" do escalonamento
  → a informação sobre PRIORIDADES, importância,
    o que deve rodar primeiro
```

### O exemplo do banco de dados

```
Suponha que o núcleo utilize um algoritmo de
escalonamento de PRIORIDADES, mas forneça uma chamada
de sistema pela qual um processo pode ESTABELECER
(e MUDAR) as prioridades dos seus FILHOS.

Dessa maneira:
  → o PAI pode controlar COMO seus filhos são
    escalonados
  → MESMO QUE ELE PRÓPRIO NÃO REALIZE O ESCALONAMENTO

Aqui:
  O MECANISMO está no NÚCLEO
  A POLÍTICA é estabelecida por um PROCESSO DO USUÁRIO
```

> ⚠️ **A separação do mecanismo de política é uma ideia FUNDAMENTAL.** Ela aparece repetidamente em sistemas operacionais em contextos variados — não é exclusiva do escalonamento.

---

## 🎯 Por que essa separação importa

```
Sem separação:
  → o núcleo precisaria embutir TODA a lógica de
    decisão sobre importância de processos
  → impossível de generalizar para toda aplicação
    imaginável
  → cada aplicação tem sua própria noção de
    "importante"

Com separação:
  → o núcleo oferece um MECANISMO genérico e eficiente
    (ex: fila de prioridades, quantum configurável)
  → cada aplicação (através do processo-pai) define
    a POLÍTICA que faz sentido para o SEU domínio
  → o mesmo mecanismo do núcleo serve a um banco de
    dados, um servidor web, uma ferramenta de
    compilação — cada um com sua própria política
```

Essa ideia conecta diretamente com o que já vimos em [[Utilização de Threads]] sobre escalonadores específicos de aplicação: o núcleo fornece o mecanismo (quantum, preempção), mas a aplicação — através de suas próprias decisões de prioridade — define a política de quem deveria rodar primeiro.

---

# ✅ Resumo do Conceito

- O escalonador do núcleo **não tem visibilidade** sobre qual processo-filho é mais importante dentro de um grupo relacionado (ex: filhos de um sistema de banco de dados) — essa informação existe, mas fica presa no processo-pai
- A solução é **separar mecanismo de política**: o núcleo implementa o **mecanismo** genérico de escalonamento (ex: algoritmo por prioridades), mas expõe uma **chamada de sistema** que permite que um processo-pai **defina e altere** as prioridades de seus filhos
- Essa separação é um princípio **fundamental** em projeto de sistemas operacionais, reaparecendo em diversos contextos além do escalonamento
- O resultado: o núcleo continua genérico e eficiente, enquanto cada aplicação define a política que faz sentido para o seu próprio domínio

---

## 🔗 Notas Relacionadas

- [[Escalonamento em Sistemas Interativos]] — o escalonamento por prioridades cujo mecanismo é parametrizado aqui
- [[Escalonamento de Threads]] — escalonadores específicos de aplicação (ex: servidor despachante+operárias) são outra manifestação da ideia de política definida fora do núcleo
- [[Hierarquia de Processos]] — o contexto de processos-pai e processos-filho mencionado no exemplo do banco de dados
