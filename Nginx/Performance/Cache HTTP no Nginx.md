---
tags:
  - nginx
  - nginx/performance
---

# Cache HTTP no Nginx

## 1. Cache de arquivos estáticos no Nginx

Esse é o uso mais comum e mais simples.

Quando você tem arquivos como:

- `.css`
- `.js`
- imagens
- fontes
- ícones

o ideal é deixar o navegador guardar esses arquivos por um tempo, para evitar downloads repetidos.

---

### Exemplo básico

```
server {
    listen 80;
    server_name meusite.com;

    root /var/www/html;

    location ~* \\.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

---

### O que isso faz?

### `location ~* \\.(...)$`

Essa regra captura arquivos com essas extensões.

- `~*` significa regex case-insensitive
- ou seja, tanto faz `.JPG` ou `.jpg`

### `expires 30d;`

Diz ao cliente que esse recurso pode ser considerado válido por 30 dias.

O Nginx vai enviar um header `Expires` e também ajuda no controle de cache.

### `add_header Cache-Control "public";`

Indica que o conteúdo pode ser armazenado por caches públicos e privados.

---

### Melhorando o exemplo

Na prática, normalmente você quer colocar também o `max-age`.

```
location ~* \\.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

### `max-age=2592000`

2592000 segundos = 30 dias.

Isso dá mais clareza para navegadores e proxies modernos.

---

### Quando isso é seguro?

É seguro quando os arquivos estáticos usam **versionamento**.

Exemplo:

- `app.v1.js`
- `app.v2.js`
- `style.abc123.css`

Ou, mais comum em bundlers modernos:

- `app.8f3a1c.js`

Se o arquivo mudar, o nome muda.
Assim o navegador baixa a nova versão normalmente.

---

### Problema se não houver versionamento

Se você fizer isso:

```
expires 30d;
```

em um arquivo chamado sempre `app.js`, o usuário pode continuar usando a versão antiga por muito tempo.

Por isso:

> cache longo em arquivos estáticos só é seguro quando existe versionamento no nome do arquivo

---

## 2. Desabilitando cache para conteúdo sensível

Nem tudo deve ficar em cache.

Exemplo: páginas de login, painel administrativo, respostas personalizadas.

```
location /admin/ {
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0";
    add_header Pragma "no-cache";
    expires off;
}
```

---

### O que isso faz?

### `no-store`

Não armazene a resposta em cache.

### `no-cache`

Pode armazenar, mas deve revalidar antes de usar.

### `must-revalidate`

Se expirar, o cache precisa consultar o servidor antes de reutilizar.

### `expires off;`

Desliga o comportamento de expiração configurado ali.

### `Pragma "no-cache"`

É um header legado, usado para compatibilidade com clientes antigos.

## Boas práticas

### 1. Sempre diferencie conteúdo público e privado

Nem toda resposta deve ser cacheada.

### 2. Use cache longo apenas com arquivos versionados

Se o nome não muda, cuidado com conteúdo antigo.

### 3. Teste com `curl -I`

Verifique os headers reais enviados.

### 4. Adicione headers de debug

`X-Cache-Status` ajuda muito a entender o comportamento.

### 5. Evite cachear respostas com sessão sem critério

Isso pode causar bugs e vazamento de conteúdo entre usuários.

### 6. Cache é estratégia, não mágica

Se a lógica de invalidação estiver errada, você ganha performance e perde consistência.

---

## Conexão com Sistemas Operacionais

- **Diretiva `expires`** — o Nginx adiciona `Cache-Control: max-age=N` e o header `Expires` (data absoluta) nas respostas de arquivos estáticos. Esses headers instruem o browser e CDNs a manter uma cópia local pelo período configurado, evitando requisições ao servidor → [[Memória]] (cache no lado cliente)

- **`open_file_cache`** — o Nginx mantém em memória um cache de file descriptors abertos, tamanhos de arquivo e timestamps de modificação. Sem isso, cada requisição de arquivo estático exige `stat()` + `open()` + leitura; com `open_file_cache`, o fd permanece aberto e os metadados ficam em memória → [[System Calls]] (stat, open), [[Arquivos]]

- **`sendfile on`** — ativa a syscall `sendfile()`: o kernel transfere dados diretamente da page cache para o socket buffer sem copiar para o espaço do usuário (zero-copy). O arquivo não passa pela memória do processo Nginx, reduzindo ciclos de CPU e uso de memória → [[System Calls]], [[Memória]] (zero-copy, page cache)

- **`tcp_nopush` + `sendfile`** — com `sendfile` ativo, `tcp_nopush` faz o kernel acumular dados antes de enviar o pacote TCP, reduzindo o número total de system calls e pacotes de rede para entrega de arquivos estáticos → [[System Calls]]

---

## Conexão com Go

- **Go: http.ServeFile e http.FileServer** — `http.ServeFile` e `http.FileServer` definem automaticamente `Last-Modified` e respondem `304 Not Modified` quando o arquivo não mudou. Internamente, usam `os.File` com operações que aproveitam o page cache do kernel de forma similar ao `sendfile` do Nginx → [[HTTP (net-http)]], [[Leitura e Escrita de Arquivos]]
