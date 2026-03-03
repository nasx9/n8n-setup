# n8n-dev

Base de ambiente local para desenvolvimento e testes de automações complexas no n8n, com arquitetura robusta:

* n8n em **queue mode**
* **Postgres** como banco
* **Redis** para fila
* **Worker** separado para execuções em background
* Timezone consistente **America/Sao_Paulo**
* Pasta compartilhada para código JS e scripts Python
* Volume `/files` para arquivos e artefatos entre containers

---

## Estrutura de pastas

```text
n8n-dev/
  infra/
    docker-compose.yml
  workflows/
  shared/
    js/
    snippets/
    python/
      scripts/
      requirements.txt
  docs/
    NOTES.md
  Makefile
  .gitignore
  README.md
```

---

## Requisitos

* Docker Desktop instalado e rodando
* Docker Compose v2 (`docker compose`)
* Git

---

## Comandos rápidos (Makefile)

### Subir o ambiente

```bash
make up
```

Acesso:

* [http://localhost:5678](http://localhost:5678)
* Usuário/senha: `admin/admin` (ajuste no compose se desejar)

### Descer o ambiente

```bash
make down
```

### Status

```bash
make ps
```

### Logs

```bash
make logs
```

Worker:

```bash
make logs-worker
```

### Reiniciar

* Apenas n8n (main):

```bash
make restart
```

* Apenas worker:

```bash
make restart-worker
```

* Tudo:

```bash
make restart-all
```

### Reset completo (apaga volumes e banco)

```bash
make reset
```

### Backup do diretório de dados do n8n

O diretório `workflows/` é montado como `/home/node/.n8n` dentro do container.

```bash
make backup
```

### Entrar no Python Runner

```bash
make python
```

---

## Docker Compose (infra/docker-compose.yml)

> Ajuste usuário/senha e demais variáveis conforme sua política.

```yaml
services:
  postgres:
    image: postgres:15
    container_name: n8n_postgres
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: n8n_pass
      TZ: America/Sao_Paulo
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7
    container_name: n8n_redis
    environment:
      TZ: America/Sao_Paulo
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n_main
    ports:
      - "5678:5678"
    environment:
      # Auth
      N8N_BASIC_AUTH_ACTIVE: "true"
      N8N_BASIC_AUTH_USER: "admin"
      N8N_BASIC_AUTH_PASSWORD: "admin"

      # Timezone
      TZ: America/Sao_Paulo
      N8N_DEFAULT_TIMEZONE: America/Sao_Paulo

      # Database (Postgres)
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: n8n_pass

      # Queue mode (Redis)
      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
      QUEUE_BULL_REDIS_PORT: 6379

      # Logs e execução
      N8N_LOG_LEVEL: info
      N8N_EXECUTIONS_DATA_SAVE_ON_ERROR: all
      N8N_EXECUTIONS_DATA_SAVE_ON_SUCCESS: none
      N8N_DIAGNOSTICS_ENABLED: "false"

      # Code node permissões (dev)
      NODE_FUNCTION_ALLOW_BUILTIN: "*"
      NODE_FUNCTION_ALLOW_EXTERNAL: "*"

    volumes:
      - ../workflows:/home/node/.n8n
      - ../shared/js:/data/shared/js
      - n8n_files:/files
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  worker:
    image: n8nio/n8n:latest
    container_name: n8n_worker
    environment:
      TZ: America/Sao_Paulo
      N8N_DEFAULT_TIMEZONE: America/Sao_Paulo

      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: n8n_pass

      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
      QUEUE_BULL_REDIS_PORT: 6379

      N8N_LOG_LEVEL: info
      N8N_DIAGNOSTICS_ENABLED: "false"

      NODE_FUNCTION_ALLOW_BUILTIN: "*"
      NODE_FUNCTION_ALLOW_EXTERNAL: "*"

    command: worker
    volumes:
      - ../workflows:/home/node/.n8n
      - ../shared/js:/data/shared/js
      - n8n_files:/files
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  python_runner:
    image: python:3.11-slim
    container_name: n8n_python_runner
    environment:
      TZ: America/Sao_Paulo
    working_dir: /work
    volumes:
      - ../shared/python:/work
      - n8n_files:/files
    command: ["sleep", "infinity"]
    restart: unless-stopped

volumes:
  pgdata:
  n8n_files:
```

---

## Makefile (raiz do projeto)

Crie um arquivo `Makefile` na raiz com o conteúdo abaixo:

```makefile
.PHONY: up down ps logs logs-worker restart restart-worker restart-all reset backup python

COMPOSE_DIR=infra

up:
	cd $(COMPOSE_DIR) && docker compose up -d

down:
	cd $(COMPOSE_DIR) && docker compose down

ps:
	cd $(COMPOSE_DIR) && docker compose ps

logs:
	cd $(COMPOSE_DIR) && docker compose logs -f n8n

logs-worker:
	cd $(COMPOSE_DIR) && docker compose logs -f worker

restart:
	cd $(COMPOSE_DIR) && docker compose restart n8n

restart-worker:
	cd $(COMPOSE_DIR) && docker compose restart worker

restart-all:
	cd $(COMPOSE_DIR) && docker compose restart

reset:
	cd $(COMPOSE_DIR) && docker compose down -v

backup:
	tar -czf n8n_workflows_backup_`date +%Y%m%d_%H%M%S`.tar.gz workflows

python:
	docker exec -it n8n_python_runner bash
```

---

## .gitignore (raiz do projeto)

Crie um arquivo `.gitignore` na raiz com o conteúdo abaixo:

```gitignore
# n8n runtime data (se você não quiser versionar tudo)
workflows/*
!workflows/exports/
!workflows/exports/**

# Env
.env
*.env

# OS/Editor
.DS_Store
.vscode/

# Logs
*.log

# Python
__pycache__/
*.pyc
.venv/

# Node
node_modules/
```

---

## Boas práticas para projetos complexos

### Timezone

* `TZ=America/Sao_Paulo` define o timezone no Linux do container.
* `N8N_DEFAULT_TIMEZONE=America/Sao_Paulo` garante que o n8n use o timezone para cron e datas.

Teste:

```bash
docker exec -it n8n_main date
```

### Código compartilhado (JS)

Use `shared/js/` como fonte de verdade para funções reutilizáveis.

Estratégias:

1. simples: refatorar no VS Code e copiar/colar no Code node.
2. avançada: usar `require('/data/shared/js/arquivo')` dentro do Code node.

### Python Runner

O `python_runner` existe para scripts auxiliares e relatórios.

Entrar:

```bash
make python
```

Instalar dependências:

```bash
pip install -r requirements.txt
```

### Segurança

Este template libera `NODE_FUNCTION_ALLOW_EXTERNAL="*"` e `NODE_FUNCTION_ALLOW_BUILTIN="*"` por conveniência em dev. Para ambientes mais restritos, substitua por allowlist.
