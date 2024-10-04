# Jogo de Adivinhação com Flask e React

Este é um simples jogo de adivinhação desenvolvido utilizando o framework Flask. O jogador deve adivinhar uma senha criada aleatoriamente, e o sistema fornecerá feedback sobre o número de letras corretas e suas respectivas posições.

## Funcionalidades

- Criação de um novo jogo com uma senha fornecida pelo usuário.
- Adivinhe a senha e receba feedback se as letras estão corretas e/ou em posições corretas.
- As senhas são armazenadas utilizando base64.
- As adivinhações incorretas retornam uma mensagem com dicas.

## Requisitos

- Docker 20.10.0+
- Docker Compose 3.9

## Execução

1. Clone o repositório:

   ```bash
   git clone https://github.com/fams/guess_game.git
   cd guess-game
   ```

2. Execute o projeto em containers com o docker compose:

   ```bash
   docker compose up -d
   ```

3. Acesse o jogo em http://localhost:80 pelo seu navegador e comece a jogar 😉

## Como Jogar

### 1. Criar um novo jogo

Acesse a url do frontend http://localhost:80/maker

Digite uma frase secreta

Envie

Salve o game-id

### 2. Adivinhar a senha

Acesse a url do frontend http://localhost:80/breaker

entre com o game_id que foi gerado pelo Creator

Tente adivinhar

## Contenirização com Docker e Docker Compose

### Design e Estrutura

Este projeto utiliza Docker e Docker Compose para facilitar a contenirização e gerenciamento dos serviços. As opções de design adotadas incluem:

#### Serviços: O projeto é dividido em três serviços principais:

```yaml
services:
  db:
  backend:
  frontend:
```

**db**: Um banco de dados PostgreSQL (imagem postgres:14.13) que armazena os dados do jogo. A saúde do serviço é monitorada por meio de um healthcheck. Esse serviço tambem conta com um volume para persistir os dados do banco de dados e esta associado a rede `backnet`.

```yml
db:
  image: postgres:14.13
  restart: always
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
    interval: 3s
    retries: 5
    start_period: 30s
  secrets:
    - db-password
  volumes:
    - db-data:/var/lib/postgresql/data
  networks:
    - backnet
  environment:
    - POSTGRES_PASSWORD=/run/secrets/db-password
  expose:
    - 5432
```

**backend**: O servidor Flask que manipula a lógica do jogo e interage com o banco de dados. A imagem desse serviço é construída a partir do diretório backend e a sua execução define as variaveis de ambiente para a conexão com o banco de dados. O serviço é conectado às redes `backnet` e `frontnet`, e sua inicialização depende da condição de que o serviço `db` esteja saudável antes de ser iniciado.

```yml
backend:
  build:
    context: backend
  restart: always
  secrets:
    - db-password
  environment:
    - FLASK_DB_PASSWORD=/run/secrets/db-password
    - FLASK_DB_HOST=db
  expose:
    - 5000
  networks:
    - backnet
    - frontnet
  depends_on:
    db:
      condition: service_healthy
```

**frontend**: Servidor Nginx que serve o frontend em React na porta 80, que é exposta para o host. O Nginx, configurado como servidor web, não apenas fornece a interface do usuário, mas também atua como um balanceador de carga, gerenciando as requisições entre o cliente e o backend. Este serviço está conectado à rede `frontnet` e possui uma configuração de restart automático, garantindo que ele seja reiniciado em caso de falhas.

```yml
frontend:
  build:
    context: frontend
  restart: always
  ports:
    - 80:80
  depends_on:
    - backend
  networks:
    - frontnet
```

#### Volumes: O volume db-data é utilizado para persistir os dados do banco de dados, garantindo que os dados não sejam perdidos quando o container for reiniciado.

```yml
volumes:
  db-data:
```

#### Redes: Duas redes são definidas: backnet e frontnet. A rede backnet conecta o banco de dados e o backend, enquanto a rede frontnet conecta o frontend ao backend, garantindo que cada componente se comunique de forma segura e eficiente.

```yml
networks:
  backnet:
  frontnet:
```

#### Balanceamento de Carga: O Nginx, utilizado no frontend, além de servir a aplicação frontend, atua como um proxy reverso, distribuindo as requisições para o backend. Ele também permite a configuração de cabeçalhos HTTP, essenciais para a comunicação correta entre os serviços.

```
upstream backend {
  server backend:5000;
}

server {
  listen 80;

  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
  }


  location /api {
    proxy_pass http://backend/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```
