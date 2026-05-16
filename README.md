# ilovepauldomar POS

WebApp de Controlo de Stocks e Ponto de Venda para o projeto comunitário **@ilovepauldomar**.

## Stack

- **Backend**: PocketBase (Go + SQLite WAL) em Docker
- **Frontend**: React + Vite (a adicionar na Fase 4)
- **Infra**: Synology DS220+ via Container Manager

---

## 🚀 Fase 1 — Bootstrap do Backend

### Pré-requisitos no NAS

1. **Container Manager** instalado (DSM Package Center).
2. **SSH** ativo (Control Panel → Terminal & SNMP → Enable SSH).
3. Pasta `/volume1/docker/` criada e acessível à conta admin.

### Setup local (no teu Mac/PC com VS Code)

```bash
# 1. Clonar o repo
git clone git@github.com:<teu-user>/ilovepauldomar-pos.git
cd ilovepauldomar-pos/backend

# 2. Gerar chave de encriptação
openssl rand -hex 32
# Copiar o output

# 3. Preparar .env
cp .env.example .env
# Editar .env e colar a chave em PB_ENCRYPTION_KEY

# 4. Criar pastas de persistência (vazias, com .gitkeep)
mkdir -p pb_data pb_migrations pb_hooks pb_public

# 5. Arranque local para testar
docker compose up -d

# 6. Verificar saúde
curl http://127.0.0.1:8090/api/health
# Esperado: {"code":200,"message":"API is healthy.","data":{}}

# 7. Aceder ao admin UI
# http://127.0.0.1:8090/_/
# (primeiro acesso pede criar conta de superuser)
```

### Deploy no Synology

```bash
# Via SSH no NAS
cd /volume1/docker
git clone git@github.com:<teu-user>/ilovepauldomar-pos.git
cd ilovepauldomar-pos/backend

# Repetir passos 2 e 3 (gerar chave + .env) DIRETAMENTE no NAS
# Não copiar o .env do teu portátil — cada ambiente tem a sua chave.

sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f pocketbase
```

### Expor via DSM Reverse Proxy

1. **Control Panel → Login Portal → Advanced → Reverse Proxy → Create**
2. Configurar:
   - **Source**: `https://pos.teudominio.com` (porta 443, HTTPS, com cert Let's Encrypt do DSM)
   - **Destination**: `http://localhost:8090`
3. **Custom Header → WebSocket** (adicionar `Upgrade` e `Connection`) — necessário para realtime do PocketBase.

---

## 🗂️ Estrutura

```
ilovepauldomar-pos/
├── backend/         # PocketBase + Docker
├── frontend/        # SPA React (Fase 4)
├── scripts/         # Backup, restore, utilitários (Fase 2)
└── docs/            # Arquitetura e decisões
```

---

## ⚠️ Avisos Críticos

- **Backups**: o `pb_data/` contém SQLite com WAL. **Nunca** copiar diretamente com o container ativo. Usar o script `scripts/backup.sh` (Fase 2) que invoca a API de snapshot do PocketBase.
- **Chave de encriptação**: guardar em gestor de passwords externo. Perdê-la = perder settings encriptados.
- **Versão do PocketBase**: fixada no `docker-compose.yml`. Atualizar manualmente e testar em local antes de subir ao NAS.
