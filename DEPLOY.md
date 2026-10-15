# Deploy — Moodle 5.3 no Coolify + Cloudflare Tunnel

Este guia sobe o Compose deste repositório no [Coolify](https://coolify.io) e expõe o site pela internet com [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) (sem abrir portas 80/443 no servidor).

## Arquitetura

```
Usuário → Cloudflare (HTTPS) → cloudflared (Tunnel)
                                      ↓
                              Coolify host → container moodle:8080
                                      ↓
                           MariaDB 11.4 + Redis 7
```

- TLS termina no Cloudflare.
- O container Moodle escuta HTTP na porta **8080**.
- `MOODLE_SSL_PROXY=auto` faz o Moodle gerar links `https://` quando `MOODLE_URL` é HTTPS.

## Pré-requisitos

1. Servidor com Coolify instalado (Docker).
2. Conta Cloudflare com um domínio na zona.
3. Repositório Git deste projeto (já no GitHub) **ou** upload do Compose.
4. Senhas fortes para admin, DB e Redis.

## 1. Preparar o `.env`

No Coolify você define as variáveis na UI. Localmente:

```bash
cp .env.example .env
```

| Variável | Exemplo | Notas |
|----------|---------|--------|
| `MOODLE_URL` | `https://moodle.seudominio.com` | **Sem** barra no final. Deve ser a URL pública final. |
| `MOODLE_ADMIN_PASSWORD` | … | Só usada na **primeira** instalação. |
| `MOODLE_DB_PASSWORD` | … | Mesma senha do usuário MariaDB. |
| `MARIADB_ROOT_PASSWORD` | … | Root do MariaDB. |
| `REDIS_PASSWORD` | … | Redis com autenticação. |
| `MOODLE_LANG` | `pt_br` | Idioma padrão na instalação. |
| `MOODLE_TIMEZONE` | `America/Sao_Paulo` | Fuso e PHP. |
| `MOODLE_HOST_PORT` | `8080` | Porta no host; use uma livre no Coolify. |

## 2. Criar o Tunnel no Cloudflare

1. Cloudflare Zero Trust → **Networks** → **Tunnels** → **Create a tunnel**.
2. Escolha **Cloudflared**, dê um nome (ex.: `coolify-moodle`).
3. Copie o **token** do conector (você vai usá-lo no Coolify ou no host).
4. Em **Public Hostname**:
   - **Subdomain:** `moodle` (ou o que preferir)
   - **Domain:** `seudominio.com`
   - **Type:** `HTTP`
   - **URL:** endereço interno do Moodle (veja opções abaixo)

### Opção A — Tunnel aponta para o host Coolify

Se o Compose publica `MOODLE_HOST_PORT` no host:

```
http://127.0.0.1:8080
```

(ou a porta que você definiu em `MOODLE_HOST_PORT`)

O `cloudflared` deve rodar **no mesmo servidor** que o Coolify (serviço do sistema, container Coolify “Cloudflare Tunnel”, ou app separado).

### Opção B — Tunnel na rede Docker do Compose

Se o conector enxerga a rede do stack:

```
http://moodle:8080
```

(`moodle` = nome do serviço no `docker-compose.yml`)

### Opção C — Domínio via proxy do Coolify + DNS Cloudflare

Alternativa sem Tunnel: no Coolify, anexe o domínio ao serviço `moodle`, porta **8080**, e no Cloudflare crie um registro DNS **Proxied** (nuvem laranja) apontando para o IP do servidor. Nesse caso você **abre** 80/443 no firewall para o Coolify. O Tunnel (opções A/B) evita isso.

## 3. Deploy no Coolify

### Aplicação Docker Compose a partir do Git

1. Coolify → **New Resource** → **Docker Compose**.
2. Conecte o repositório GitHub deste projeto.
3. Confirme o arquivo: `docker-compose.yml`.
4. Em **Environment Variables**, cole todas as variáveis do `.env.example` (valores reais).
5. Garanta volume persistente (Coolify costuma persistir os named volumes do Compose).
6. **Port** do serviço web: **8080** (serviço `moodle`).
7. Deploy.

### Primeira subida

Acompanhe os logs do serviço `moodle`:

```text
Installing Moodle...
...
```

A instalação pode levar **2–5 minutos**. Quando o healthcheck passar, abra `MOODLE_URL`.

Login inicial:

- Usuário: valor de `MOODLE_ADMIN_USER` (padrão `admin`)
- Senha: `MOODLE_ADMIN_PASSWORD`

## 4. Checklist pós-deploy

1. Acesse `https://moodle.seudominio.com` — cadeado Cloudflare OK.
2. **Site administration → Notifications** — sem erros críticos de ambiente.
3. Confirme fuso, idioma e e-mail do admin.
4. (Opcional) Configure SMTP real via variáveis `MOODLE_SMTP_*` (veja README da imagem) ou em *Server → Email*.
5. Faça backup dos volumes antes de qualquer upgrade de tag.

## 5. Persistência (não perca dados)

| Volume | Conteúdo |
|--------|----------|
| `moodle_db` | Banco MariaDB |
| `moodle_code` | Código Moodle + plugins instalados pela UI |
| `moodle_data` | Arquivos de usuários, cache, backups da imagem |
| `moodle_redis` | Dados Redis (AOF) |

**Não remova** esses volumes ao redesploy. No Coolify, evite “Remove volumes” ao deletar o recurso.

## 6. Atualizar o Moodle

A tag `5.3` recebe rebuilds com patches. Para atualizar:

1. Backup (export DB + snapshot dos volumes, se possível).
2. No Coolify, faça **Redeploy** (pull da imagem `oksigenia/moodle:5.3`).
3. Na subida, a imagem faz backup interno e roda o upgrade do Moodle se a versão mudou.

Para pin imutável em produção: use tag tipo `5.3.0-r14` (veja [Docker Hub](https://hub.docker.com/r/oksigenia/moodle/tags)).

## 7. Troubleshooting

| Sintoma | Causa comum | Ação |
|---------|-------------|------|
| Redirect loop / HTTP misto | `MOODLE_URL` errada ou sem HTTPS | Ajuste `MOODLE_URL` para a URL pública `https://…` e redesploy |
| “Incorrect wwwroot” | URL do túnel ≠ `MOODLE_URL` | Hostname do Tunnel deve bater com `MOODLE_URL` |
| DB connection refused | Moodle subiu antes do MariaDB | Aguarde healthcheck; `depends_on` já exige DB healthy |
| Site lento / sessões perdidas | Redis fora | Veja logs do `redis`; confira `REDIS_PASSWORD` |
| 502 no Cloudflare | Tunnel URL errada ou porta | Teste `curl http://127.0.0.1:8080` no host; confira Public Hostname |
| Upload grande falha | Limite PHP / Tunnel | Aumente `PHP_UPLOAD_MAX_FILESIZE`; no Cloudflare, plano Free limita upload (~100 MB) |

Logs úteis:

```bash
docker compose logs -f moodle
docker compose exec moodle moodle-cli admin/cli/checks.php
```

## 8. Segurança recomendada

- Não committe o arquivo `.env`.
- Use senhas longas e únicas (admin, DB, Redis, root MariaDB).
- Mantenha o Tunnel com **Zero Trust** / Access se quiser restringir quem acessa o Moodle.
- Não publique a porta `8080` na internet; só o Cloudflare deve alcançar o host (firewall).
- Atualize a imagem `5.3` periodicamente.

## Referências

- [Moodle 5.3 — requisitos](https://moodledev.io/general/releases/5.3)
- [oksigenia/moodle](https://github.com/OksigeniaSL/docker-moodle)
- [Coolify — Docker Compose](https://coolify.io/docs/knowledge-base/docker/compose)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
