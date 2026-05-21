### Задача:

- получить в результате клиентского запроса всю цепочку адресов;
- не допустить обработку поддельного запроса от клиента;

### Решение:
---
#### Необходимо разделить прокси на два контура: EDGE и Internal.

__EDGE (обработка внешних запросов)__
1. Обрабатывает только внешние запросы.
2. Сбрасывает любой X-Forwarded-For, присланный клиентом, и создаёт новый с $remote_addr.

__Internal (обработка запросов после EDGE)__
1. Проксирует трафик до нужного приложения.
2. Принимает запрос только от доверенных nginx EDGE
3. Добавляет свой $remote_addr к уже очищенной цепочке через $proxy_add_x_forwarded_for, чтобы не потерять IP клиента.

#### Преимущества схемы - аналог эшелонированной защиты. В качестве примера приведена структура Ansible роли. В задаче нет требований по автоматизации, поэтому приведен только концептуальный подход для развертывания и масштабирования.
---

### Файл: `edge-proxy.conf`

```bash
server {
    listen 443;
    server_name _;
    # Удобный формат лога для диагностики
    log_format proxy_chain '$remote_addr -> xff="$http_x_forwarded_for" '
                           'x_real_ip="$http_x_real_ip" '
                           'host="$host" '
                           'proto="$scheme" '
                           'request="$request" '
                           'status=$status';

    access_log /var/log/nginx/access.log proxy_chain;

    location / {
        # Оригинальный Host от клиента
        proxy_set_header Host $host; 

        # Сброс X-Forwarded-For, подставляем реальный IP TCP соединения
        proxy_set_header X-Forwarded-For $remote_addr;

        # Сохраняем исходный IP клиента
        proxy_set_header X-Real-IP $remote_addr;

        # Схема запроса на этом инстансе nginx
        proxy_set_header X-Forwarded-Proto $scheme;

        # Тероминируем прокси на INTERNAL nginx
        proxy_pass http://internal:8080;
    }
}
```
---

### Файл: `int-proxy.conf`

```bash
server {
    listen 8080;

    # Разрешаем только доверенные EDGE
    allow 10.10.90.10;  # EDGE1
    allow 10.10.90.11;  # EDGE2
    deny all;

    # Логирование для диагностики
    log_format proxy_chain '$remote_addr -> xff="$http_x_forwarded_for" '
                           'x_real_ip="$http_x_real_ip" '
                           'host="$host" '
                           'proto="$http_x_forwarded_proto" '
                           'request="$request" '
                           'status=$status';

    access_log /var/log/nginx/access.log proxy_chain;

    location / {
        proxy_set_header Host $host;

        # Добавить IP предыдущего nginx к цепочке
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Сохранить исходного клиента
        proxy_set_header X-Real-IP $http_x_real_ip;

        # Схема запроса от EDGE
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;

        # Прокси на тестовое приложение
        proxy_pass http://app:5678;
    }
}
```
---

### Файл: docker-compose.yml

---

```bash
services:
  app:
    image: hashicorp/http-echo:0.2.3
    container_name: app
    command: ["-text", "Hello from app", "-headers"]
    ports:
      - "5678:5678"
    networks:
      - proxy_chain

  internal:
    image: nginx:1.25
    container_name: internal
    volumes:
      - ./int-proxy.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - proxy_chain

  edge1:
    image: nginx:1.25
    container_name: edge1
    ports:
      - "80:80"
    volumes:
      - ./edge-proxy.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - internal
    networks:
      - proxy_chain

  edge2:
    image: nginx:1.25
    container_name: edge2
    ports:
      - "81:80"
    volumes:
      - ./edge-proxy.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - internal
    networks:
      - proxy_chain

networks:
  proxy_chain:

```
---

## Структура Ansible роли

```text
ansible/roles/proxy-chain/
├── defaults
│   └── main.yml        # переменные: пути, IP EDGE, порты
├── meta
│   └── main.yml
├── playbooks
│   ├── docker.yml
│   └── proxy-chain.yml
├── README.md
├── tasks
│   ├── docker
│   │   └── main.yml
│   ├── main.yml
│   └── proxy-chain
│       └── main.yml
└── templates
    ├── .env.j2
    ├── docker-compose.yml.j2
    ├── edge-proxy.conf.j2
    └── int-proxy.conf.j2
```
