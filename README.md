# n8n-dev (Base de desenvolvimento robusta)
Base de ambiente local para desenvolvimento e testes de automações complexas no n8n, com arquitetura robusta:

- n8n em **queue mode**
- **Postgres** como banco
- **Redis** para fila
- **Worker** separado (execuções em background)
- Timezone consistente (America/Sao_Paulo)
- Pasta compartilhada para código JS e scripts Python
- Volume `/files` para arquivos e artefatos entre containers

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
Requisitos

Docker Desktop instalado e rodando

Docker Compose v2 (docker compose)

Git

Subir o ambiente
make up

Acesso:

http://localhost:5678

Usuário/senha: admin/admin (ajuste no compose se desejar)

Descer o ambiente
make down
Ver logs

Logs do n8n (main):

make logs

Logs do worker:

make logs-worker
Reiniciar serviços

Reiniciar apenas n8n (main):

make restart

Reiniciar worker:

make restart-worker

Reiniciar tudo:

make restart-all
Reset completo (APAGA banco e dados)

Use com cuidado. Remove volumes e dados persistidos.

make reset
Backup do diretório de dados do n8n

O diretório workflows/ é montado como /home/node/.n8n dentro do container.
Este comando gera um .tar.gz local.

make backup

Para restaurar manualmente, você pode descompactar e substituir o conteúdo de workflows/ (com o ambiente parado).

Timezone

Este template define:

TZ=America/Sao_Paulo

N8N_DEFAULT_TIMEZONE=America/Sao_Paulo

Isso evita problemas com agendamentos (cron) e datas divergentes.

Código compartilhado (JS)

Use shared/js/ como fonte de verdade para funções reutilizáveis em Code nodes.

Estratégias:

Simples e robusto:

Desenvolver/refatorar em shared/js

Copiar/colar no Code node do n8n

Avançado:

Usar require('/data/shared/js/arquivo') dentro do Code node

Necessita manter o volume montado (já está no compose)

Python Runner

O n8n não é um container “canivete suíço” para Python.
Este template inclui um container python_runner para scripts auxiliares.

Entrar no container:

make python

Rodar script:

python scripts/meu_script.py

Se quiser instalar dependências:

Coloque em shared/python/requirements.txt

Dentro do container:

pip install -r requirements.txt

Dica: para persistir libs, use venv no diretório montado ou crie uma imagem Python customizada (futuro).

Segurança

Este template libera:

NODE_FUNCTION_ALLOW_EXTERNAL="*"

NODE_FUNCTION_ALLOW_BUILTIN="*"

Isso é conveniente para dev, mas é permissivo.
Para ambientes mais restritos, use allowlist e evite *.

Convenções recomendadas

Tudo que for código reutilizável deve existir em shared/js ou shared/python.

O que estiver em shared/snippets é legado/rascunho e deve ser refatorado.

Workflows exportados devem ser versionados quando fizer sentido (ex: workflows/exports/).

Troubleshooting rápido
n8n não executa tarefas (queue)

Verifique se worker está rodando:

docker ps
make logs-worker
Cron executando fora do horário

Confirme timezone no container:

docker exec -it n8n_main date
Conflito de porta

Se 5678 estiver ocupada, altere a porta no docker-compose.yml:

ports:
  - "5678:5678"
Licença

Defina a licença conforme sua necessidade (MIT, privada, etc.).


---

## Makefile

Crie `Makefile` na raiz do projeto:

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
.gitignore (recomendado)

Crie .gitignore na raiz:

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

Se você quiser versionar alguns exports, crie workflows/exports/ e exporte seus fluxos para lá.
