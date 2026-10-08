# 🐳 Desafio 4.5 — Django + PostgreSQL + Docker

## 🔥 Desafio de Consolidação — Sem Cola

Este desafio foi criado como uma repetição intencional do **Desafio 04**, porém com um objetivo diferente:

> **Reconstruir toda a infraestrutura manualmente, sem consultar os comandos utilizados anteriormente.**

Depois de reproduzir o ambiente, o desafio foi além, adicionando:

- persistência com volumes;
- testes de remoção e recriação de containers;
- cenários de falha;
- terceiro container;
- comunicação entre múltiplos containers;
- inspeção da Docker Network.

O objetivo principal foi entender, na prática, **por que o Docker Compose existe**.

---

# 🎯 Objetivo

Construir manualmente uma infraestrutura composta por:

```text
Django
   │
   │
   ▼
PostgreSQL

pgAdmin
   │
   └──────► PostgreSQL
```

Todos os containers utilizando uma mesma Docker Network.

A infraestrutura final:

```text
                         Docker Host
                              │
                       desafio45_net
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
          Django          PostgreSQL        pgAdmin
       desafio45_app    desafio45_db    desafio45_pgadmin
          :8000             :5432             :80
                              │
                              ▼
                     desafio45_dados
                          Volume
```

---

# 🧠 O que foi praticado

Durante o desafio foram utilizados:

- Docker Image;
- Docker Container;
- Docker Network;
- Docker Volume;
- Docker DNS;
- variáveis de ambiente;
- `.env`;
- `docker exec`;
- `docker logs`;
- `docker network inspect`;
- PostgreSQL;
- Django;
- Gunicorn;
- WhiteNoise;
- pgAdmin;
- persistência de dados;
- comunicação entre containers.

---

# 📌 Ponto de partida

O projeto Django já possuía:

```text
config/
app/
requirements.txt
.env.example
.dockerignore
```

O `settings.py` já estava preparado para receber as configurações através de variáveis de ambiente:

```env
SECRET_KEY=
DEBUG=
DB_NAME=
DB_USER=
DB_PASSWORD=
DB_HOST=
DB_PORT=
```

O projeto também utilizava PostgreSQL como banco de dados.

O Dockerfile, entretanto, deveria ser criado durante o desafio.

---

# 🏗️ Parte 1 — Infraestrutura manual

## 1. Criando a Docker Network

Primeiro foi criada uma rede exclusiva para os containers do desafio:

```bash
docker network create desafio45_net
```

Essa rede seria responsável pela comunicação entre:

```text
desafio45_app
desafio45_db
desafio45_pgadmin
```

---

# 💾 2. Criando o Volume

Foi criado um volume nomeado para armazenar os dados do PostgreSQL:

```bash
docker volume create desafio45_dados
```

O objetivo é separar:

```text
Container
```

de:

```text
Dados persistentes
```

Dessa forma, remover o container PostgreSQL não significa necessariamente perder os dados.

---

# 🐘 3. Criando o PostgreSQL

O PostgreSQL foi executado utilizando a imagem oficial:

```text
postgres
```

Comando utilizado:

```bash
docker run \
  --network desafio45_net \
  --name desafio45_db \
  -e POSTGRES_DB=desafio45_db \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v desafio45_dados:/var/lib/postgresql \
  -d \
  postgres
```

### Configurações

| Configuração | Valor |
|---|---|
| Container | `desafio45_db` |
| Database | `desafio45_db` |
| User | `admin` |
| Network | `desafio45_net` |
| Volume | `desafio45_dados` |
| Porta interna | `5432` |

A porta `5432` **não foi publicada no host**.

O PostgreSQL é acessado pelos outros containers através da Docker Network.

---

# 🐳 4. Criando o Dockerfile

O Dockerfile foi criado manualmente, sem consultar o utilizado no desafio anterior.

O objetivo era preparar a aplicação Django para:

- instalar as dependências;
- executar `collectstatic`;
- executar com Gunicorn;
- funcionar dentro de um container.

Dockerfile utilizado:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY ./requirements.txt /app/

RUN pip install -r ./requirements.txt

COPY . /app/

RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD [ "gunicorn", "-b", "0.0.0.0:8000", "config.wsgi:application" ]
```

---

# 🏗️ 5. Criando a imagem Django

A imagem foi criada com:

```bash
docker build -t desafio45_app .
```

---

# 🚀 6. Criando o container Django

O container Django foi iniciado utilizando o `.env`:

```bash
docker run -d \
  --name desafio45_app \
  -p 8009:8000 \
  --env-file .env \
  --network desafio45_net \
  desafio45_app
```

### Configurações

```text
Container: desafio45_app
Host: 8009
Container: 8000
Network: desafio45_net
```

A aplicação ficou disponível em:

```text
http://localhost:8009
```

---

# 🔗 7. Comunicação Django → PostgreSQL

O Django não utilizou:

```env
DB_HOST=localhost
```

O PostgreSQL foi acessado pelo nome do container:

```env
DB_HOST=desafio45_db
```

Isso foi possível graças ao DNS interno da Docker Network.

A comunicação ficou:

```text
desafio45_app
       │
       │ desafio45_db
       ▼
desafio45_db
```

---

# 🗄️ 8. Executando as migrations

As migrations foram executadas dentro do container Django:

```bash
docker exec -it desafio45_app python manage.py migrate
```

Isso criou as tabelas do Django no PostgreSQL.

Entre elas:

```text
auth_group
auth_permission
auth_user
django_admin_log
django_content_type
django_migrations
django_session
```

---

# 👤 9. Criando o superusuário

O superusuário foi criado diretamente dentro do container:

```bash
docker exec -it desafio45_app python manage.py createsuperuser
```

Depois foi possível acessar:

```text
http://localhost:8009/admin/
```

O Django Admin carregou corretamente, incluindo seus arquivos estáticos.

---

# 🖥️ Parte 3 — Adicionando o pgAdmin

Depois de configurar Django e PostgreSQL, foi adicionado um terceiro container para administrar o banco visualmente.

Imagem utilizada:

```text
dpage/pgadmin4
```

O container foi criado com:

```bash
docker run \
  -p 80:80 \
  --network desafio45_net \
  --name desafio45_pgadmin \
  -e 'PGADMIN_DEFAULT_EMAIL=admin@admin.com' \
  -e 'PGADMIN_DEFAULT_PASSWORD=SuperSecret' \
  -d \
  dpage/pgadmin4
```

### Configurações

| Configuração | Valor |
|---|---|
| Container | `desafio45_pgadmin` |
| Imagem | `dpage/pgadmin4` |
| Host | `localhost` |
| Porta | `80` |
| Network | `desafio45_net` |

O pgAdmin ficou disponível em:

```text
http://localhost
```

---

# 🔌 Conectando o pgAdmin ao PostgreSQL

Dentro do pgAdmin foi criado um novo servidor PostgreSQL.

A conexão utilizou:

```text
Host: desafio45_db
Port: 5432
Username: admin
Password: mysecretpassword
Database: desafio45_db
```

O ponto mais importante foi o `Host`.

Não foi utilizado:

```text
localhost
```

porque, dentro do container do pgAdmin:

```text
localhost
    ↓
desafio45_pgadmin
```

O PostgreSQL estava em outro container.

A comunicação correta foi:

```text
desafio45_pgadmin
        │
        │ desafio45_db
        ▼
desafio45_db
```

---

# 🔎 Visualizando o banco pelo pgAdmin

Depois da conexão, o pgAdmin conseguiu visualizar:

- database `desafio45_db`;
- tabelas criadas pelo Django;
- usuários cadastrados;
- demais estruturas do banco.

Isso confirmou que:

```text
Django
   ↓
PostgreSQL
   ↑
pgAdmin
```

estavam utilizando o mesmo banco.

---

# 🌐 Inspecionando a Docker Network

A rede foi inspecionada utilizando:

```bash
docker network inspect desafio45_net
```

Foram encontrados os três containers:

```text
desafio45_db
desafio45_app
desafio45_pgadmin
```

A Docker Network atribuiu os seguintes IPs internos:

```text
desafio45_db
172.20.0.2

desafio45_app
172.20.0.3

desafio45_pgadmin
172.20.0.4
```

Arquitetura observada:

```text
                 desafio45_net
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
  desafio45_db   desafio45_app   desafio45_pgadmin
   172.20.0.2      172.20.0.3       172.20.0.4
```

### Por que utilizar nomes em vez dos IPs?

Apesar de os containers possuírem IPs internos, a aplicação utiliza:

```text
desafio45_db
```

em vez de:

```text
172.20.0.2
```

Isso ocorre porque o Docker fornece resolução de nomes dentro da Network.

Se o container PostgreSQL for recriado, seu IP pode mudar.

O nome continua sendo a referência utilizada pelos outros containers.

---

# 💾 Parte 2 — Persistência

Um dos objetivos do desafio foi compreender a diferença entre:

```text
Container
```

e:

```text
Volume
```

O PostgreSQL utilizou:

```text
desafio45_dados
```

como volume persistente.

A arquitetura ficou:

```text
desafio45_db
      │
      ▼
desafio45_dados
```

O container pode ser removido e recriado sem que os dados necessariamente sejam perdidos, desde que o mesmo volume seja utilizado.

---

# 💥 Testes de falha

Também foram explorados cenários em que a infraestrutura não funciona corretamente.

## Django sem PostgreSQL

Foi analisado o comportamento da aplicação quando o PostgreSQL não está disponível.

Os logs podem ser consultados com:

```bash
docker logs desafio45_app
```

Esse teste mostrou que:

> Iniciar os containers não significa que todos os serviços estejam automaticamente prontos para receber conexões.

---

# 💥 `DB_HOST=localhost`

Também foi analisado o problema de utilizar:

```env
DB_HOST=localhost
```

dentro do container Django.

Dentro de um container:

```text
localhost
    ↓
o próprio container
```

Portanto:

```text
desafio45_app
       │
       X
       │
   localhost
```

não aponta para o PostgreSQL.

A configuração correta é:

```env
DB_HOST=desafio45_db
```

---

# 🧰 Comandos utilizados

Durante o desafio foram utilizados principalmente:

### Containers

```bash
docker run
docker ps
docker rm
docker exec
docker logs
```

### Imagens

```bash
docker build
docker images
```

### Network

```bash
docker network create
docker network inspect
docker network ls
```

### Volumes

```bash
docker volume create
docker volume ls
docker volume rm
```

---

# 🧠 Principais aprendizados

## 1. Container não é armazenamento

O container PostgreSQL pode ser removido sem que os dados precisem ser perdidos, desde que estejam em um volume persistente.

```text
Container
    ≠
Dados
```

---

## 2. Containers possuem rede própria

O Django e o PostgreSQL não precisam estar instalados diretamente no host.

Eles podem se comunicar através de uma Docker Network:

```text
Django ─────► PostgreSQL
       Network
```

---

## 3. `localhost` depende do contexto

No host:

```text
localhost
```

representa o computador local.

Dentro do Django:

```text
localhost
```

representa o próprio container Django.

Dentro do pgAdmin:

```text
localhost
```

representa o próprio container pgAdmin.

---

## 4. Docker fornece DNS interno

Na mesma Network, os containers podem se encontrar pelo nome:

```text
desafio45_db
```

sem precisar conhecer o IP.

---

## 5. Portas publicadas e portas internas são coisas diferentes

Django:

```text
localhost:8009
        ↓
container:8000
```

pgAdmin:

```text
localhost:80
        ↓
container:80
```

PostgreSQL:

```text
container:5432
```

sem necessidade de publicar a porta no host.

---

# 😅 A "dor" do Docker manual

Uma das principais conclusões do desafio foi perceber a quantidade de comandos e configurações repetidas.

Foi necessário declarar manualmente informações como:

```text
--name
--network
-e
-v
-p
--env-file
```

Além disso, foi necessário criar separadamente:

```bash
docker network create
docker volume create
docker build
docker run
docker exec
```

E repetir diversas configurações entre os containers.

Isso tornou evidente um dos problemas que o Docker Compose resolve.

---

# 🔄 Do Docker manual para o Compose

A infraestrutura construída manualmente ficou aproximadamente assim:

```text
Network
   │
   ├── Django
   │
   ├── PostgreSQL
   │      │
   │      └── Volume
   │
   └── pgAdmin
```

Para reproduzir tudo novamente, seria necessário executar diversos comandos.

No próximo desafio, essa infraestrutura será declarada em um arquivo:

```text
compose.yaml
```

E a inicialização poderá ser feita através de:

```bash
docker compose up
```

O objetivo não é simplesmente aprender mais um comando.

É entender **por que** o Docker Compose existe.

---

# 🏁 Resultado final

Ao final do Desafio 4.5, foi construída manualmente uma infraestrutura composta por:

```text
                         Docker Host
                              │
                       desafio45_net
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
          Django          PostgreSQL        pgAdmin
       desafio45_app    desafio45_db    desafio45_pgadmin
          :8000             :5432             :80
                              │
                              ▼
                     desafio45_dados
                          Volume
```

O Django conseguiu:

- iniciar dentro de um container;
- conectar ao PostgreSQL;
- executar migrations;
- criar usuários;
- disponibilizar o Admin;
- servir arquivos estáticos.

O pgAdmin conseguiu:

- conectar ao PostgreSQL;
- visualizar o banco;
- visualizar as tabelas;
- visualizar os usuários criados pelo Django.

Os três containers foram conectados através da mesma Docker Network.

---

# 🚀 Próximo desafio

## Desafio 05 — Docker Compose

Depois de construir toda essa infraestrutura manualmente, o próximo passo será transformar essa configuração em uma infraestrutura declarativa utilizando:

```text
compose.yaml
```

A ideia será substituir vários comandos manuais por:

```bash
docker compose up
```

**Desafio 4.5: primeiro sentir a dor.  
Desafio 05: descobrir como o Compose resolve essa dor.** 🐳