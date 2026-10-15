# Moodle 5.3 LTS — Docker Compose (Coolify + Cloudflare Tunnel)

Stack pronta para subir o **Moodle 5.3 LTS** (suporte de segurança até outubro de 2029) no Coolify, com acesso via Cloudflare Tunnel.

| Serviço | Imagem | Função |
|---------|--------|--------|
| `moodle` | `oksigenia/moodle:5.3` | LMS (Apache + PHP 8.4) |
| `db` | `mariadb:11.4` | Banco (requisito do Moodle 5.3) |
| `redis` | `redis:7-alpine` | Sessões e cache de aplicação |

> **Por que não Bitnami?** As imagens gratuitas `bitnami/moodle` pararam de receber updates em agosto/2025. `oksigenia/moodle` é o substituto drop-in mantido (código oficial Moodle, rebuilds semanais).

## Início rápido (local)

```bash
cp .env.example .env
# edite MOODLE_URL e todas as senhas
docker compose up -d
docker compose logs -f moodle
```

Primeira instalação leva alguns minutos. Login: usuário `admin` + `MOODLE_ADMIN_PASSWORD`.

## Deploy

Guia completo: **[DEPLOY.md](DEPLOY.md)** (Coolify + Cloudflare Tunnel).

## Variáveis principais

Veja [`.env.example`](.env.example). Obrigatórias:

- `MOODLE_URL` — URL pública HTTPS (ex.: `https://moodle.seudominio.com`)
- `MOODLE_ADMIN_PASSWORD`, `MOODLE_DB_PASSWORD`, `MARIADB_ROOT_PASSWORD`, `REDIS_PASSWORD`

## Licença

Configuração deste repositório: uso livre. Moodle™ e logos são marcas da Moodle Pty Ltd. A imagem `oksigenia/moodle` é GPL-3.0 (não afiliada à Moodle HQ).
