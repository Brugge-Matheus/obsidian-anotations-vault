---
tags:
  - nginx
  - nginx/performance
---

# Cache HTTP

### O que é cache HTTP?

**Cache HTTP** é o mecanismo de armazenamento temporário de respostas HTTP para que elas possam ser reutilizadas em requisições futuras, sem a necessidade de processar tudo novamente.

Em termos simples:

- um cliente pede um recurso;
- o servidor responde;
- essa resposta pode ser armazenada por algum tempo;
- se o mesmo recurso for solicitado novamente, a resposta pode ser reaproveitada.

---

### Para que serve o cache HTTP?

O cache existe para melhorar:

### 1. Performance

Reduz o tempo de resposta, porque o conteúdo pode ser entregue mais rapidamente.

### 2. Redução de carga no servidor

Evita que a aplicação, banco de dados ou backend processem a mesma informação repetidas vezes.

### 3. Redução de consumo de banda

Se o recurso já está armazenado em cache, menos dados precisam ser transferidos novamente.

### 4. Melhor experiência do usuário

Páginas e recursos carregam mais rápido.

### 5. Escalabilidade

Ajuda a suportar muito mais tráfego com a mesma infraestrutura.

---

### Onde o cache HTTP pode existir?

Um ponto importante é que o cache HTTP não existe em um lugar só. Ele pode aparecer em várias camadas.

### 1. Cache no navegador

O navegador guarda arquivos como:

- CSS
- JavaScript
- imagens
- fontes
- até algumas respostas de API, dependendo dos headers

### 2. Cache em proxies intermediários

CDNs, proxies corporativos ou gateways podem armazenar respostas e servi-las sem consultar o servidor de origem.

### 3. Cache no proxy reverso

O próprio Nginx pode guardar respostas de um backend e servi-las diretamente.

### 4. Cache na aplicação

A aplicação pode armazenar resultados em memória, Redis ou outros mecanismos, mas isso já não é exatamente cache HTTP, e sim cache de aplicação.

---

### Ideia central do cache HTTP

A lógica é simples:

> se a resposta ainda é válida, não há motivo para gerar ou transferir tudo de novo

Isso pode ocorrer de duas formas principais:

### Reutilização direta

A resposta é servida diretamente do cache.

### Revalidação

O cliente ou proxy pergunta ao servidor se a versão armazenada ainda é válida. Se estiver, o servidor responde algo como "continue usando a cópia que você já tem".

---

### O que pode ser armazenado em cache?

Depende da configuração, mas normalmente:

### Muito comum armazenar:

- imagens
- CSS
- JavaScript
- fontes
- arquivos estáticos
- páginas públicas
- respostas de API que mudam pouco

### Mais delicado armazenar:

- páginas com login
- carrinhos de compra
- dashboards personalizados
- respostas que dependem de cookies ou autenticação

Porque nesses casos o conteúdo pode variar por usuário.

---

### Como o HTTP controla cache?

O protocolo HTTP usa **headers** para informar como o cache deve se comportar.

Os principais conceitos são:

### 1. Cache-Control

É o cabeçalho mais importante hoje para controle de cache.

Ele define políticas como:

- se pode armazenar ou não;
- por quanto tempo;
- se o cache é público ou privado;
- se precisa revalidar antes de usar.

Exemplos conceituais:

- `public` → pode ser armazenado por caches compartilhados
- `private` → apenas o navegador do usuário deve armazenar
- `no-cache` → pode armazenar, mas precisa revalidar antes de usar
- `no-store` → não deve armazenar
- `max-age` → tempo de validade em segundos

### 2. Expires

É uma forma mais antiga de definir validade com uma data/hora específica.

Hoje geralmente é complementado ou substituído por `Cache-Control`.

### 3. ETag

É um identificador da versão de um recurso.

Se o cliente já tem uma cópia com certa ETag, ele pode perguntar ao servidor se ela ainda é válida.

### 4. Last-Modified

Indica quando o recurso foi modificado pela última vez.

O cliente pode usar isso para perguntar: "mudou desde tal data?"

---

### Requisição condicional e revalidação

Nem sempre o cache significa servir tudo sem falar com o servidor.

Às vezes o cliente faz uma **requisição condicional**.

Exemplo conceitual:

1. navegador tem uma cópia do arquivo;
2. envia uma requisição com um header como:
    - `If-None-Match`
    - ou `If-Modified-Since`
3. o servidor verifica;
4. se nada mudou, responde **304 Not Modified**;
5. o navegador reutiliza o conteúdo que já tinha.

Isso economiza banda e processamento.

---

### Diferença entre cache de navegador e cache reverso

### Cache de navegador

Fica no cliente.
O objetivo é evitar downloads repetidos do mesmo recurso.

### Cache reverso

Fica no Nginx ou em outro proxy.
O objetivo é evitar chamadas repetidas ao backend.

Esses dois podem coexistir.

---

## Como o cache HTTP funciona no Nginx?

No Nginx, o tema de cache aparece em dois contextos principais:

### 1. Controle de cache para o cliente

O Nginx pode enviar headers HTTP para instruir navegadores e proxies sobre como armazenar recursos.

Aqui ele atua dizendo:

- esse arquivo pode ficar em cache por 30 dias;
- essa resposta não deve ser armazenada;
- esse conteúdo é público;
- esse recurso precisa de revalidação.

### 2. Cache reverso de respostas do backend

Quando o Nginx está na frente de uma aplicação, ele pode guardar a resposta dessa aplicação e responder diretamente nas próximas requisições iguais.

Aqui ele atua como um **cache intermediário** entre o cliente e o backend.

---

### 1. Controle de cache de arquivos estáticos no Nginx

Esse é um dos usos mais comuns.

O Nginx é excelente para servir arquivos estáticos, então é natural usá-lo para definir políticas de cache para:

- imagens
- CSS
- JavaScript
- fontes
- PDFs
- vídeos estáticos

### Conceito

Quando o cliente pede um arquivo estático, o Nginx pode responder com headers dizendo por quanto tempo aquele recurso pode ser reutilizado.

Isso reduz novas requisições completas ao servidor.

### Exemplo conceitual

Se um arquivo `app.js` muda raramente e usa versionamento no nome, ele pode ficar em cache por muito tempo sem problema.

---

### 2. Cache reverso no Nginx

Esse é um uso mais poderoso.

Quando o Nginx está como proxy reverso, ele pode armazenar respostas vindas do backend.

Fluxo conceitual:

1. cliente pede `/produtos`;
2. Nginx encaminha ao backend;
3. backend processa e responde;
4. Nginx guarda essa resposta em cache;
5. próximo cliente pede `/produtos`;
6. Nginx entrega do cache sem chamar o backend.

Isso reduz drasticamente:

- uso de CPU no backend;
- consultas ao banco;
- tempo de resposta;
- carga total da aplicação.

---

### O que o Nginx pode usar para decidir o cache?

O Nginx pode considerar coisas como:

- URL
- query string
- método HTTP
- headers
- cookies
- status code da resposta
- tempo de expiração
- regras configuradas manualmente

Ou seja, o cache não é apenas "guardar qualquer coisa".
Ele depende de uma **chave de cache** e de regras de validade.

---

### Conceitos importantes no cache reverso do Nginx

### Cache key

É a identidade usada para saber se duas requisições podem reutilizar a mesma resposta.

Exemplo conceitual:

- `/produtos?page=1`
- `/produtos?page=2`

São chaves diferentes, porque as respostas são diferentes.

### Cache hit

Quando a resposta é encontrada no cache e servida diretamente.

### Cache miss

Quando a resposta não está no cache e o Nginx precisa consultar o backend.

### Cache bypass

Quando o Nginx é instruído a ignorar o cache para determinada requisição.

### Cache revalidation

Quando o Nginx verifica se a resposta armazenada ainda é válida.

### Expiração / TTL

Define por quanto tempo uma entrada de cache é considerada válida.

---

### Vantagens do cache no Nginx

### 1. Reduz latência

O Nginx pode responder muito mais rápido do que a aplicação gerando tudo do zero.

### 2. Protege o backend

Menos requisições chegam à aplicação e ao banco.

### 3. Aumenta a capacidade do sistema

O mesmo backend consegue atender muito mais usuários.

### 4. Excelente para conteúdo repetitivo

Páginas públicas, APIs de catálogo, páginas institucionais e arquivos estáticos se beneficiam muito.

---

### Cuidados e limitações

Cache é excelente, mas precisa ser aplicado com critério.

### 1. Risco de servir conteúdo desatualizado

Se a invalidação não for bem pensada, o usuário pode receber dados antigos.

### 2. Risco de servir conteúdo de um usuário para outro

Se uma resposta personalizada for cacheada de forma incorreta, isso pode causar problemas graves de segurança.

### 3. Nem tudo deve ser cacheado

Conteúdo sensível, autenticado ou muito volátil geralmente exige cuidado.

### 4. Cache não substitui otimização de aplicação

Ele reduz trabalho repetido, mas não corrige código ruim nem banco mal projetado.

---

### Cache HTTP e conteúdo dinâmico

Um erro comum é pensar:

> "Se a página é dinâmica, então não posso usar cache."

Na verdade, muitas páginas dinâmicas possuem partes repetidas ou respostas que não mudam a cada segundo.

Exemplos:

- listagem pública de produtos;
- página inicial institucional;
- resultados de busca com curta validade;
- endpoints de API com dados que mudam pouco.

Então mesmo conteúdo dinâmico pode ser cacheado, desde que:

- a validade seja bem definida;
- a personalização seja controlada;
- o contexto do usuário seja considerado.

---

### Relação entre cache HTTP e performance

O cache melhora performance porque evita os custos mais caros da pilha web:

- abrir conexão com banco;
- executar queries;
- rodar lógica de aplicação;
- renderizar templates;
- serializar resposta;
- transferir conteúdo repetido.

Em muitas arquiteturas, o maior ganho de performance não vem de "código mais rápido", mas de **fazer menos trabalho**.
Cache é justamente isso: **evitar trabalho repetido**.

---

## Conexão com Sistemas Operacionais

- **Cache-Control e parsing de strings** — os headers `Cache-Control`, `Expires`, `ETag` e `Last-Modified` são strings no protocolo HTTP. O servidor precisa fazer parsing desses campos a cada requisição: comparações de strings, parsing de datas, hashing de ETags. Memória para buffers dessas strings é alocada no heap do processo → [[Memória]]

- **ETag: hash como identificador de versão** — uma ETag é tipicamente um hash (MD5, SHA-1) ou um número de versão do recurso. O servidor compara a ETag recebida no header `If-None-Match` com a ETag atual; se iguais, retorna 304 Not Modified sem transmitir o corpo, economizando banda

- **Cache no navegador: sistema de arquivos do cliente** — o browser cache armazena respostas em disco no perfil do usuário. Cada recurso em cache é um arquivo; o browser gerencia um índice de metadados (URL → arquivo, tempo de expiração) → [[Arquivos]]

- **CDN cache: anycast routing** — CDNs distribuem cópias geograficamente; o roteamento anycast direciona o cliente ao servidor mais próximo via BGP. Cada nó da CDN usa mecanismos de I/O similares ao Nginx para servir respostas cacheadas → [[Dispositivos de IO]]

- **Cache-Control: no-store** — impede qualquer armazenamento em cache, forçando cada requisição a percorrer toda a pilha até o origem. Sem cache, nenhuma resposta fica em memória intermediária → [[Memória]]

---

## Conexão com Go

- **Go: definindo headers de cache** — usa-se `w.Header().Set("Cache-Control", "public, max-age=86400")` antes de `w.WriteHeader()`. Padrão de middleware permite aplicar headers de cache a grupos de rotas de forma centralizada → [[HTTP (net-http)]]
