---
tags:
  - nginx
  - nginx/performance
---

# Cache no Servidor (Cache Reverso)

## Cache Reverso no Nginx (Proxy Cache)

Antes de tudo, vamos alinhar a nomenclatura para não confundir:

| Termo | O que é |
| --- | --- |
| **Cache HTTP** | Controle de cache via headers HTTP, geralmente para o navegador ou proxies |
| **Cache Reverso** | O Nginx armazena respostas do backend e serve sem consultar a aplicação |
| **FastCGI Cache** | Igual ao cache reverso, mas específico para comunicação via FastCGI (PHP) |

Os três são "cache", mas atuam em camadas diferentes.

---

## O problema que o cache reverso resolve

Imagine o seguinte fluxo **sem cache reverso**:

```
Usuário → Nginx → Aplicação → Banco de Dados → Aplicação → Nginx → Usuário
```

Cada requisição percorre esse caminho completo.

Se 1000 usuários pedirem a mesma página ao mesmo tempo:

- 1000 chamadas chegam na aplicação;
- 1000 queries no banco;
- 1000 renderizações de template;
- 1000 respostas geradas do zero.

Mesmo que a resposta seja **idêntica** para todos.

---

## O que o cache reverso faz

Com cache reverso:

```
Usuário 1 → Nginx → Aplicação → Banco → Nginx (salva resposta) → Usuário 1
Usuário 2 → Nginx (serve do cache) → Usuário 2
Usuário 3 → Nginx (serve do cache) → Usuário 3
...
Usuário 1000 → Nginx (serve do cache) → Usuário 1000
```

O backend é chamado **uma vez**.
As outras 999 requisições são respondidas pelo Nginx diretamente.

---

## Onde o cache fica armazenado?

No cache reverso do Nginx, as respostas ficam salvas em **disco**, com metadados em **memória RAM**.

- **Disco** → corpo da resposta (HTML, JSON, etc.)
- **RAM** → chaves de cache, metadados, controle de expiração

Isso é diferente de soluções como Redis, que ficam inteiramente em memória.

---

## Proxy Cache vs FastCGI Cache

Esses dois são conceitualmente iguais, mas usados em contextos diferentes:

|  | **proxy_cache** | **fastcgi_cache** |
| --- | --- | --- |
| **Quando usar** | Backend via HTTP (Node.js, Python, Go, etc.) | Backend via FastCGI (PHP-FPM) |
| **Diretiva de ativação** | `proxy_cache` | `fastcgi_cache` |
| **Configuração de path** | `proxy_cache_path` | `fastcgi_cache_path` |
| **Validade** | `proxy_cache_valid` | `fastcgi_cache_valid` |
| **Bypass** | `proxy_cache_bypass` | `fastcgi_cache_bypass` |

A lógica é a mesma. Só muda o protocolo de comunicação com o backend.

---

## 1. Proxy Cache (backend HTTP)

### Configuração completa comentada

```
http {

    # Define onde o cache será armazenado e como será gerenciado
    proxy_cache_path /var/cache/nginx/proxy
        levels=1:2                  # organiza em subdiretórios
        keys_zone=cache_backend:20m # zona de memória RAM para metadados
        max_size=2g                 # limite máximo em disco
        inactive=60m                # remove entradas não acessadas em 60min
        use_temp_path=off;          # escreve direto no diretório final

    server {
        listen 80;
        server_name meusite.com;

        location / {
            proxy_pass <http://127.0.0.1:3000>;

            # Ativa o cache
            proxy_cache cache_backend;

            # Define validade por status HTTP
            proxy_cache_valid 200 10m;
            proxy_cache_valid 301 302 5m;
            proxy_cache_valid 404 1m;

            # Ignora cache se houver cookie (usuário logado)
            proxy_cache_bypass $http_cookie;
            proxy_no_cache $http_cookie;

            # Header de debug
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

---

### Entendendo `levels=1:2`

Sem organização em subdiretórios, todos os arquivos de cache ficariam em uma única pasta.

Com milhares de entradas, isso degrada a performance do sistema de arquivos.

O `levels=1:2` cria uma estrutura como:

```
/var/cache/nginx/proxy/a/bc/abcdef1234567890
```

Isso distribui os arquivos em subdiretórios e melhora a leitura em disco.

---

### Entendendo `keys_zone`

A `keys_zone` é uma área de memória compartilhada entre todos os Workers.

Ela guarda:

- as chaves de cache (identificadores das entradas);
- metadados como tempo de expiração;
- status de cada entrada.

O número após o nome (`20m`) define o tamanho dessa área em RAM.

`20m` é suficiente para armazenar metadados de dezenas de milhares de entradas.

---

### Entendendo `inactive`

Se uma entrada de cache não for acessada pelo tempo definido em `inactive`, ela pode ser removida mesmo que ainda não tenha expirado.

Isso evita que o cache fique cheio de respostas que ninguém mais pede.

---

## 2. FastCGI Cache (PHP-FPM)

A lógica é idêntica ao proxy cache, mas as diretivas têm o prefixo `fastcgi_`.

```
http {

    fastcgi_cache_path /var/cache/nginx/fastcgi
        levels=1:2
        keys_zone=cache_php:20m
        max_size=1g
        inactive=60m
        use_temp_path=off;

    server {
        listen 80;
        server_name meusite.com;

        root /var/www/html;

        location ~ \\.php$ {
            fastcgi_pass unix:/var/run/php-fpm.sock;
            fastcgi_index index.php;
            include fastcgi_params;

            # Ativa o cache
            fastcgi_cache cache_php;

            # Validade por status
            fastcgi_cache_valid 200 10m;
            fastcgi_cache_valid 404 1m;

            # Ignora cache se houver cookie ou sessão
            fastcgi_cache_bypass $http_cookie;
            fastcgi_no_cache $http_cookie;

            # Header de debug
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

---

## 3. A Cache Key

A **cache key** é o identificador que o Nginx usa para saber se duas requisições podem compartilhar a mesma resposta.

Por padrão, a chave é algo como:

```
scheme + method + host + uri
```

Exemplo:

```
http GET meusite.com /produtos?page=1
http GET meusite.com /produtos?page=2
```

São chaves diferentes, então terão entradas de cache separadas.

---

### Customizando a cache key

Você pode definir a chave manualmente:

```
proxy_cache_key "$scheme$request_method$host$request_uri";
```

Isso é útil quando você quer incluir ou excluir partes da requisição da chave.

Exemplo: se você quer que o cache ignore a query string:

```
proxy_cache_key "$scheme$request_method$host$uri";
```

Cuidado: isso pode fazer o Nginx servir a mesma resposta para URLs com query strings diferentes.

---

## 4. Variável `$upstream_cache_status`

Essa variável é essencial para debug e monitoramento.

| Valor | Significado |
| --- | --- |
| `HIT` | Resposta veio do cache |
| `MISS` | Não estava no cache, foi ao backend |
| `BYPASS` | Cache foi ignorado por regra |
| `EXPIRED` | Estava no cache mas expirou |
| `STALE` | Serviu conteúdo expirado (em modo stale) |
| `REVALIDATED` | Cache foi revalidado com o backend |
| `UPDATING` | Cache está sendo atualizado |

---

## 5. Stale Cache

Uma funcionalidade muito útil é servir conteúdo expirado enquanto o backend está sendo consultado ou está com problema.

```
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
proxy_cache_background_update on;
proxy_cache_lock on;
```

---

### O que cada diretiva faz?

### `proxy_cache_use_stale`

Define em quais situações o Nginx pode servir conteúdo expirado:

- `error` → backend retornou erro
- `timeout` → backend demorou demais
- `updating` → cache está sendo atualizado
- `http_500`, `http_502`, etc. → backend retornou esses status

Isso é excelente para **resiliência**.
Se o backend cair, o Nginx continua servindo a última versão conhecida.

---

### `proxy_cache_background_update on;`

Quando o cache expira, em vez de bloquear a requisição atual para atualizar, o Nginx:

- serve o conteúdo expirado imediatamente;
- atualiza o cache em segundo plano.

Isso evita que o usuário espere pela atualização.

---

### `proxy_cache_lock on;`

Quando múltiplas requisições chegam ao mesmo tempo para uma entrada que não está no cache, apenas **uma** vai ao backend.

As outras esperam o resultado e reutilizam a resposta.

Sem isso, todas as requisições simultâneas iriam ao backend ao mesmo tempo, causando o chamado **cache stampede** ou **thundering herd**.

---

## 6. Cache Stampede

Esse é um problema clássico que o `proxy_cache_lock` resolve.

### O que é?

Imagine que uma entrada de cache expira.

Naquele exato momento, 500 requisições chegam para aquela URL.

Sem proteção:

- todas as 500 vão ao backend simultaneamente;
- o backend recebe uma explosão de carga;
- pode travar ou degradar.

Com `proxy_cache_lock on`:

- apenas 1 vai ao backend;
- as outras 499 esperam;
- quando a resposta chega, todas são servidas.

---

## 7. Ignorando headers do backend

Por padrão, o Nginx respeita headers de cache enviados pelo backend, como:

- `Cache-Control: no-store`
- `Set-Cookie`

Se o backend enviar esses headers, o Nginx pode não cachear a resposta.

Para ignorar isso e forçar o cache independente do que o backend diz:

```
proxy_ignore_headers Cache-Control Expires Set-Cookie;
```

Use com cuidado. Isso pode cachear respostas que não deveriam ser cacheadas.

---

## 8. Invalidação de cache

Um dos pontos mais delicados do cache é a **invalidação**.

Como fazer o Nginx remover uma entrada de cache antes de ela expirar?

### Opção 1: Esperar expirar

A mais simples. Você define um TTL curto e aceita que o conteúdo pode estar desatualizado por aquele tempo.

### Opção 2: `proxy_cache_purge` (Nginx Plus)

A versão paga do Nginx tem suporte nativo a purge de cache via requisição HTTP.

### Opção 3: Módulo `ngx_cache_purge` (open source)

Existe um módulo de terceiros que adiciona essa funcionalidade ao Nginx open source.

### Opção 4: Remover arquivos manualmente

Como o cache fica em disco, você pode remover os arquivos diretamente.

Mas isso exige saber o caminho exato, o que é complicado.

### Opção 5: TTL curto + stale

Usar TTL curto com `proxy_cache_use_stale` é uma estratégia prática para muitos casos.

---

## 9. Exemplo completo com todas as boas práticas

```
http {

    proxy_cache_path /var/cache/nginx/proxy
        levels=1:2
        keys_zone=cache_app:50m
        max_size=5g
        inactive=120m
        use_temp_path=off;

    server {
        listen 80;
        server_name meusite.com;

        # Rota pública com cache agressivo
        location /api/produtos {
            proxy_pass <http://127.0.0.1:3000>;

            proxy_cache cache_app;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 5m;
            proxy_cache_valid 404 30s;

            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;

            add_header X-Cache-Status $upstream_cache_status;
        }

        # Rota autenticada sem cache
        location /api/usuario {
            proxy_pass <http://127.0.0.1:3000>;

            proxy_cache cache_app;
            proxy_cache_bypass $http_cookie;
            proxy_no_cache $http_cookie;

            add_header X-Cache-Status $upstream_cache_status;
        }

        # Rota que nunca deve ser cacheada
        location /api/checkout {
            proxy_pass <http://127.0.0.1:3000>;
            proxy_no_cache 1;
            proxy_cache_bypass 1;
        }
    }
}
```

---

### Frase para memorizar

**Cache HTTP diz ao navegador o que guardar. Cache reverso faz o Nginx guardar o que o backend respondeu. O objetivo dos dois é o mesmo: evitar trabalho repetido.**

---

## Conexão com Sistemas Operacionais

- **Cache reverso em disco** — `proxy_cache_path` configura onde o Nginx armazena as respostas do backend em arquivos no disco. Cada entrada de cache é um arquivo cujo nome é derivado do hash da cache key; servir do cache significa `open()` + `read()` → [[Arquivos]], [[System Calls]]

- **Cache key como hash** — `proxy_cache_key "$scheme$request_method$host$request_uri"` é hasheada (MD5) para gerar o nome do arquivo no disco. Esse hash mapeia a string da requisição para um identificador de tamanho fixo usado como caminho de arquivo → [[Bits e Bytes]]

- **keys_zone: memória compartilhada entre workers** — a zona de metadados (`keys_zone=name:10m`) é um segmento de memória compartilhada mapeado via `mmap()` entre todos os worker processes. Todos os workers leem e escrevem os metadados de cache na mesma região de memória sem sistema de arquivos → [[Memória Virtual]], [[Processos]]

- **Stale-while-revalidate** — quando `proxy_cache_background_update on` está ativo, o Nginx serve a resposta expirada imediatamente e dispara uma subrequisição em background para atualizar o cache. Essa subrequisição é processada dentro do worker sem bloquear o event loop → [[Processos]]

---

## Conexão com Go

- **Go: cache em memória com sync.Map** — um servidor Go pode implementar cache reverso simples usando `sync.Map` (ou `map` + `sync.RWMutex`) para armazenar respostas em memória. Para cache distribuído ou persistente usa-se Redis via client Go → [[sync.WaitGroup e sync.Mutex]]
