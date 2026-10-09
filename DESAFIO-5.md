# Desafio 5 — Django + PostgreSQL + pgAdmin com Docker Compose

Branch: `feat/desafio-05-docker-compose`
Objetivo: trocar "qual comando executo?" por "como descrevo esse ambiente?".

O `compose.yml` **não existe de propósito**. Você escreve.

## Passo 0 — Inventário (antes do YAML)

Releia `NOTAS-4.5.md` e preencha: cada coisa que você fez à mão → onde ela vira declaração?

| Fiz à mão no 4.5 | Vira o quê no Compose? |
|---|---|
| `docker network create ...` | ? |
| `docker volume create ...` | ? |
| `docker run postgres -e ... -v ... --network ...` | ? |
| `docker build -t ...` | ? |
| `docker run django -e ... -p ...` | ? |
| `--env-file .env` | ? |
| `python manage.py migrate` | ? (quem roda? quando?) |
| `docker run dpage/pgadmin4 ...` | ? |

## Pontos para pensar (sem resposta aqui)

- O `.env` herdado tem `DB_HOST=desafio45_db`. Isso continua certo? Qual é o nome do host agora?
- `EXPOSE` no Dockerfile publica a porta no host?
- O Django sobe antes do Postgres estar pronto para aceitar conexões. Isso importa? O que existe no Compose para lidar com isso?
- Onde as migrations rodam agora? E o `createsuperuser`?
- `ALLOWED_HOSTS` no settings aceita o que você vai acessar?
- O `.dockerignore` e o `.gitignore` ignoram `.env`. Por que, e o que isso muda para quem clona o repo? (veja `.env.example`)

## Roteiro de validação

1. `docker compose up` (em primeiro plano; leia os logs)
2. Acesse o Django, crie um superusuário, entre no Admin
3. `docker compose down` → `docker compose up` → o usuário continua?
4. `docker compose down -v` → `docker compose up` → e agora?
5. Diagnóstico: `docker compose ps`, `docker ps`, `docker network ls`, `docker volume ls`, `docker network inspect <rede>`
6. Adicione o pgAdmin e registre o servidor usando o **nome do serviço**
7. `docker compose up -d` e `docker compose down` limpos

## Critério de conclusão — explique sem consultar

- [ ] 1. O que é um service
- [ ] 2. Como o Django encontra o Postgres
- [ ] 3. Por que `DB_HOST=localhost` está errado
- [ ] 4. Como o volume mantém os dados
- [ ] 5. `down` vs `down -v`
- [ ] 6. Como o Compose cria a network
- [ ] 7. Como o pgAdmin encontra o Postgres
- [ ] 8. Imagem vs container vs service vs network vs volume
- [ ] 9. O que `build:` significa
- [ ] 10. Por que Compose > vários `docker run`

## Minhas respostas / anotações

(escreva com as suas palavras aqui conforme for testando)

## Quando terminar

Mande o `compose.yml` para revisão.
