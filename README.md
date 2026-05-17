# ilovepauldomar POS

WebApp de Controlo de Stocks e Ponto de Venda para o projeto comunitário **@ilovepauldomar**.

## Stack

- **Backend**: PocketBase `0.34.2` (Go + SQLite WAL) em Docker
- **Frontend**: React + Vite (a adicionar na Fase 4)
- **Infra**: Synology DS220+ via Container Manager
- **Domínio**: `pos.ilovepauldomar.pt` (Cloudflare → DSM Reverse Proxy → container)

---

## 🚀 Fase 1 — Bootstrap do Backend

### Pré-requisitos no NAS

1. **Container Manager** instalado (DSM Package Center).
2. **SSH** ativo (Control Panel → Terminal & SNMP → Enable SSH).
3. Pasta `/volume1/docker/` criada e acessível à conta admin.

### Setup local (no teu Mac/PC com VS Code)

```bash
# 1. Clonar o repo
git clone git@github.com:<teu-user>/ilpdm-pos.git
cd ilpdm-pos/backend

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
# Esperado: {"code":200,"message":"API is healthy.","data":{...}}

# 7. Aceder ao admin UI
# http://127.0.0.1:8090/_/
# (primeiro acesso pede criar conta de superuser)
```

### Deploy no Synology

```bash
# Via SSH no NAS
cd /volume1/docker
git clone git@github.com:<teu-user>/ilpdm-pos.git
cd ilpdm-pos/backend

# Repetir passos 2 e 3 (gerar chave + .env) DIRETAMENTE no NAS.
# Não copiar o .env do teu portátil — cada ambiente tem a sua chave.

sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f pocketbase
```

Caminho final no NAS: `/volume1/docker/ilpdm-pos/backend/`

---

## 🌐 Exposição Externa: Cloudflare + DSM Reverse Proxy

### 1. Cloudflare (DNS)

No dashboard do `ilovepauldomar.pt`:

- **DNS → Add record**
  - Type: `A` (ou `CNAME` se apontares para outro hostname já existente)
  - Name: `pos`
  - Target: IP público do NAS (ou hostname dinâmico do Synology)
  - Proxy status: **DNS only (cinzento)** — recomendado para este caso

> **Porquê DNS only e não Proxy laranja?**
> O PocketBase usa **WebSockets/SSE** para o realtime (subscrições live a coleções).
> Funciona com o proxy da Cloudflare ativo, mas obriga a:
> - SSL/TLS em **Full (strict)** (não "Flexible")
> - WebSockets ligados em *Network → WebSockets*
> - Timeouts da Cloudflare (100s no plano Free) podem cortar sessões longas de admin ou uploads grandes
>
> Se já tens o padrão DNS-only a funcionar noutros projetos teus, mantém a coerência. Caso prefiras Proxy laranja, confirma os 3 pontos acima.

### 2. DSM Reverse Proxy

**Control Panel → Login Portal → Advanced → Reverse Proxy → Create**

| Campo | Valor |
|---|---|
| Description | `ilpdm-pos` |
| **Source Protocol** | HTTPS |
| **Source Hostname** | `pos.ilovepauldomar.pt` |
| **Source Port** | `443` |
| **Destination Protocol** | HTTP |
| **Destination Hostname** | `localhost` |
| **Destination Port** | `8090` |

Depois, separador **Custom Header → WebSocket → Create**: adiciona automaticamente os headers `Upgrade` e `Connection`.

> ⚠️ **Sem este passo o realtime do PocketBase falha silenciosamente** — a app parece funcionar mas nunca atualiza em direto.

### 3. Certificado SSL

**Control Panel → Security → Certificate → Add → Add a new certificate → Get from Let's Encrypt**

- Domain name: `pos.ilovepauldomar.pt`
- Após emitir, em **Settings → Configure**, garante que o cert está atribuído ao Reverse Proxy de `pos.ilovepauldomar.pt`.

---

## 🗂️ Estrutura do Monorepo

```
ilpdm-pos/
├── backend/         # PocketBase + Docker (Fase 1, ativa)
├── frontend/        # SPA React (Fase 4)
├── scripts/         # Backup, restore, utilitários (Fase 2)
└── docs/            # Arquitetura e decisões
```

---

## ⚙️ Atualizar a Versão do PocketBase

A versão está fixada em `backend/docker-compose.yml` (`image: ghcr.io/muchobien/pocketbase:X.Y.Z`).

Para atualizar:

1. Ler o changelog: <https://github.com/pocketbase/pocketbase/blob/master/CHANGELOG.md>
2. Testar **sempre primeiro em local**:
   ```bash
   cd backend
   # Backup defensivo da BD local antes de mexer
   docker compose exec pocketbase wget -O /pb_data/pre-upgrade-$(date +%F).zip \
       http://127.0.0.1:8090/api/backups -q --post-data=''
   # Editar docker-compose.yml com a nova tag
   docker compose pull
   docker compose up -d
   docker compose logs -f pocketbase
   # Validar admin UI e endpoints essenciais
   ```
3. Commit + push do `docker-compose.yml` ao GitHub.
4. No NAS: `git pull && sudo docker compose pull && sudo docker compose up -d`.

> ⚠️ Antes de qualquer upgrade em produção, correr `scripts/backup.sh` (Fase 2).

---

## ⚠️ Avisos Críticos

- **Backups**: o `pb_data/` contém SQLite com WAL. **Nunca** copiar diretamente com o container ativo. Usar o script `scripts/backup.sh` (Fase 2) que invoca a API de snapshot do PocketBase.
- **Chave de encriptação**: guardar em gestor de passwords externo. Perdê-la = perder settings encriptados (SMTP, OAuth, S3, etc).
- **Versão do PocketBase**: fixada no `docker-compose.yml`. Atualizar manualmente e testar em local antes de subir ao NAS — ver secção acima.
- **WebSockets**: confirmar sempre que o Reverse Proxy do DSM tem o header WebSocket após qualquer alteração à regra.
