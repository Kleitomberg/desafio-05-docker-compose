# 🐳 Desafio 05 — Django + PostgreSQL + pgAdmin com Docker Compose

Evolução do [Desafio 4.5](NOTAS-4.5.md): o mesmo ambiente com Django, PostgreSQL e pgAdmin, antes configurado manualmente por comandos Docker, agora descrito de forma declarativa em um único `compose.yaml`.

O objetivo é aprender a orquestrar múltiplos serviços, centralizar a configuração da infraestrutura e compreender como o Docker Compose gerencia containers, redes e volumes.

## 🎯 Objetivos

- Declarar serviços, redes e volumes em um único arquivo Compose.
- Substituir comandos manuais `docker network create`, `docker volume create`, `docker build` e `docker run`.
- Configurar a comunicação entre Django e PostgreSQL.
- Gerenciar configurações por variáveis de ambiente.
- Utilizar healthchecks e dependências entre serviços.
- Validar a persistência dos dados após recriar os containers.
- Comparar a execução manual do Docker CLI com a abordagem declarativa do Compose.

## 🏗️ Arquitetura

```text
                         Docker Host
                              │
                        desafio5_net
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
           Django          PostgreSQL        pgAdmin
        desafio5_app       desafio5_db     desafio5_pgadmin
       localhost:8010     localhost:5432    localhost:80
                              │                │
                         desafio5_data      pgadmin_data
                           (volume)           (volume)
```

### Serviços

| Serviço | Imagem | Acesso pelo host | Volume |
|---|---|---|---|
| `app` | Build do Dockerfile local | http://localhost:8010 | — |
| `postgres` | `postgres:18` | `localhost:5432` | `desafio5_data` |
| `pgadmin` | `dpage/pgadmin4` | http://localhost | `pgadmin_data` |

Os três serviços compartilham a rede `desafio5_net` e podem se comunicar pelos nomes dos serviços do Compose.

Por isso, o Django utiliza `DB_HOST=postgres`, e o pgAdmin utiliza `postgres` como hostname ao registrar o servidor de banco de dados.

## 🔄 Do Docker CLI para Docker Compose

No Desafio 4.5, o ambiente foi montado executando comandos Docker individualmente. No Desafio 5, essas configurações passaram a ser declaradas no `compose.yaml`, permitindo que o Compose gerencie a criação e a inicialização dos serviços.

### Comparativo dos comandos

| Desafio 4.5 — Comando manual | Desafio 5 — Declaração ou comando equivalente |
|---|---|
| `docker network create desafio45_net` | `networks:` com `desafio5_net` e `driver: bridge` |
| `docker volume create desafio45_dados` | `volumes:` com `desafio5_data` |
| `docker build -t desafio4-django .` | `build: ./` no serviço `app` |
| `docker run ... postgres:18` | Serviço `postgres`, com `image`, `environment`, `volumes`, `networks` e, opcionalmente, `ports` |
| `docker run ...` para iniciar o Django | Serviço `app`, com `build`, `environment`, `ports`, `networks` e `depends_on` |
| `docker run ... dpage/pgadmin4` | Serviço `pgadmin`, com `image`, `environment`, `ports`, `volumes` e `networks` |
| `docker network connect ...` | Associação dos serviços à rede por `networks` |
| `docker run -e ...` | `environment` no Compose, usando valores do `.env` |
| Inicialização manual dos containers | `docker compose up -d` |
| `docker ps` | `docker compose ps` para visualizar os serviços do projeto |
| `docker logs -f ...` | `docker compose logs -f <serviço>` |
| Remoção manual dos containers | `docker compose down` |

### Exemplo: rede

**Antes — criar a rede manualmente:**

```bash
docker network create desafio45_net
```

**Depois — declarar a rede no `compose.yaml`:**

```yaml
networks:
  desafio5_net:
    driver: bridge
```

Os serviços que precisam se comunicar referenciam essa rede em suas respectivas configurações.

### Exemplo: PostgreSQL

**Antes:** executar `docker run` com a imagem, variáveis de ambiente, portas e volumes.

**Depois:** declarar o serviço no Compose:

```yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - desafio5_data:/var/lib/postgresql
    networks:
      - desafio5_net
```

O Compose passa a gerenciar a configuração e a inicialização do serviço.

### O que continua sendo executado manualmente?

Nem todo comando precisa virar uma declaração no Compose. Operações administrativas do Django continuam sendo executadas dentro do container em funcionamento:

| Operação | Comando |
|---|---|
| Aplicar migrations | `docker compose exec app python manage.py migrate` |
| Criar superusuário | `docker compose exec app python manage.py createsuperuser` |
| Verificar migrations | `docker compose exec app python manage.py showmigrations` |
| Abrir o shell do Django | `docker compose exec app python manage.py shell` |

O Compose define como os serviços são configurados e iniciados; ele não executa automaticamente todas as tarefas administrativas da aplicação.

**Principal aprendizado:** o desafio evolui de uma sequência de comandos operacionais para uma infraestrutura descrita declarativamente em um arquivo reproduzível.

## ⚙️ Configuração

### Pré-requisitos

- Docker Desktop instalado e em execução.
- Docker Compose disponível pelo comando `docker compose`.
- Código Django e seu `Dockerfile`.

### Variáveis de ambiente

Crie seu arquivo local a partir do exemplo:

```bash
cp .env.example .env
```

Preencha as variáveis necessárias:

```env
SECRET_KEY=uma-chave-local-de-estudo
DEBUG=0

DB_NAME=desafio5_db
DB_USER=admin
DB_PASSWORD=uma-senha-local
DB_HOST=postgres
DB_PORT=5432

PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=uma-senha-local
```

Os valores acima são ilustrativos. Utilize credenciais locais próprias.

O `.env` deve permanecer no `.gitignore`, pois contém configurações e credenciais. O `.env.example` deve conter apenas valores de exemplo, nunca segredos reais.

O Compose injeta em cada serviço somente as variáveis declaradas para ele, evitando carregar configurações exclusivas do pgAdmin no Django.

## 🚀 Como executar

Construa a imagem da aplicação e inicie os serviços:

```bash
docker compose up --build
```

Para executar em segundo plano:

```bash
docker compose up -d --build
```

Verifique o estado dos serviços:

```bash
docker compose ps
```

Valide a configuração resolvida pelo Compose:

```bash
docker compose config
```

Consulte os logs:

```bash
docker compose logs -f app
docker compose logs -f postgres
docker compose logs -f pgadmin
```

## 🗄️ Migrations e superusuário

Com os serviços iniciados e o PostgreSQL pronto para receber conexões, aplique as migrations:

```bash
docker compose exec app python manage.py migrate
```

Crie o usuário administrador:

```bash
docker compose exec app python manage.py createsuperuser
```

O comando `exec` permite executar comandos dentro de um container em funcionamento, sem precisar criar outro container manualmente.

Confira as migrations aplicadas:

```bash
docker compose exec app python manage.py showmigrations
```

As migrations aplicadas aparecem marcadas com `[X]`. Aplicações sem modelos próprios que necessitem de migrations podem exibir `(no migrations)`.

## 🌐 Acessos

- Aplicação Django: http://localhost:8010
- Django Admin: http://localhost:8010/admin/
- pgAdmin: http://localhost

### Conexão pelo pgAdmin

Ao registrar um servidor no pgAdmin, utilize os dados configurados no ambiente:

| Campo | Valor |
|---|---|
| Host | `postgres` |
| Port | `5432` |
| Maintenance database | Valor de `DB_NAME` |
| Username | Valor de `DB_USER` |
| Password | Valor de `DB_PASSWORD` |

O hostname `postgres` é resolvido dentro da rede Docker compartilhada. Ele não é um endereço para acessar diretamente pelo navegador do host.

## 🩺 Healthcheck e dependências

O PostgreSQL utiliza `pg_isready` para verificar se está aceitando conexões.

Os serviços `app` e `pgadmin` utilizam `depends_on` com `condition: service_healthy`, condicionando sua inicialização à saúde do PostgreSQL.

Isso é mais confiável do que depender apenas da ordem de inicialização dos containers.

O healthcheck indica que o banco está pronto para aceitar conexões, mas não garante que todas as operações da aplicação funcionarão corretamente.

## 💾 Persistência dos dados

O PostgreSQL utiliza o volume nomeado `desafio5_data`. O pgAdmin utiliza `pgadmin_data` para persistir suas configurações.

Os volumes permitem que os dados sobrevivam à remoção e recriação dos containers.

### Remover containers e rede

```bash
docker compose down
```

Esse comando remove os containers e a rede criada pelo Compose, mas mantém os volumes nomeados.

### Remover também os volumes

```bash
docker compose down -v
```

**Atenção:** esse comando remove os volumes gerenciados pelo Compose e pode apagar os dados do banco e as configurações persistidas do pgAdmin.

### Teste de persistência realizado

1. Os serviços foram iniciados pelo Compose.
2. O superusuário foi criado e as migrations foram aplicadas.
3. O ambiente foi encerrado com `docker compose down`.
4. Os serviços foram iniciados novamente.
5. Os dados permaneceram disponíveis após a recriação dos containers.

## 🔑 Conceitos praticados

- **Docker Compose:** orquestração declarativa de serviços.
- **Services:** definição dos componentes da aplicação.
- **Networks:** comunicação entre containers por DNS interno.
- **Volumes nomeados:** persistência independente do ciclo de vida dos containers.
- **Environment:** configuração dos serviços por variáveis de ambiente.
- **Healthcheck:** verificação da disponibilidade do banco.
- **Depends on:** definição de dependências entre serviços.
- **Ports:** publicação de portas do container no host.
- **Exec:** execução de comandos em containers existentes.
- **Migrations:** criação e atualização da estrutura do banco pelo Django.

### `EXPOSE` versus `ports`

`EXPOSE` documenta a porta utilizada pela aplicação dentro da imagem. Já `ports` publica uma porta do container no host.

### `localhost` versus nome do serviço

Dentro do container Django, `localhost` aponta para o próprio container. Para acessar o PostgreSQL, utiliza-se o nome do serviço `postgres`.

## 🧪 Validações realizadas

- [x] Serviços iniciados pelo Docker Compose.
- [x] Comunicação do Django com o PostgreSQL.
- [x] Conexão do pgAdmin com o PostgreSQL usando o hostname `postgres`.
- [x] Verificação de `DEBUG=False` nas configurações efetivas do Django.
- [x] Verificação das migrations padrão do Django aplicadas.
- [x] Persistência dos dados após `docker compose down` e nova inicialização.

## 📚 Próximos passos

- Automatizar migrations como uma etapa controlada de deploy.
- Estudar gerenciamento de segredos e configurações para produção.
- Avançar para novos desafios de DevOps.
