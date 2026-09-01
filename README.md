# evolution-api-esteticago

Deploy da **Evolution API v2** (gateway de WhatsApp) no Render, via Docker.
Consumida pelo backend do EsteticaGo (`WhatsAppService` → `EVOLUTION_API_*`).

## Conteúdo

| Arquivo | Para quê |
|---|---|
| `Dockerfile` | Imagem: parte de `evoapicloud/evolution-api:latest`. |
| `render.yaml` | Blueprint do Render: web service + Postgres + variáveis. |
| `.env.example` | Referência das variáveis (uso local / consulta). |

## Deploy no Render (Blueprint)

1. Suba esta pasta para um repositório no GitHub (ver comandos abaixo).
2. Render → **New → Blueprint** → conecte o repo `evolution-api-esteticago` → **Apply**.
   O Render cria o Postgres `evolution-db` e o web service `evolution-api-esteticago`,
   e **gera** o valor de `AUTHENTICATION_API_KEY`.
3. Quando o serviço subir, copie a URL pública
   (`https://evolution-api-esteticago.onrender.com`), cole em **`SERVER_URL`**
   (aba *Environment* do serviço) e faça **Manual Deploy → Deploy latest commit**.
4. Em *Environment*, revele e copie o valor de **`AUTHENTICATION_API_KEY`**.

> **Plano:** o `render.yaml` já usa `plan: starter` (pago) no web service de
> propósito — no plano *free* o serviço hiberna após 15 min ocioso e a sessão do
> WhatsApp cai, exigindo reparear pelo QR toda vez. O Postgres está em `free`
> (expira em 90 dias) — troque para `basic-256mb` para produção.

## Criar a instância e parear o WhatsApp

Com `BASE=https://evolution-api-esteticago.onrender.com` e `KEY=<AUTHENTICATION_API_KEY>`:

```bash
# 1. cria a instância "esteticago"
curl -X POST "$BASE/instance/create" \
  -H "apikey: $KEY" -H "Content-Type: application/json" \
  -d '{"instanceName":"esteticago","integration":"WHATSAPP-BAILEYS","qrcode":true}'

# 2. pega o QR Code (campo "base64" = data:image/png;base64,... — abra no navegador)
curl "$BASE/instance/connect/esteticago" -H "apikey: $KEY"

# 3. confere o status (deve virar "open" após escanear)
curl "$BASE/instance/connectionState/esteticago" -H "apikey: $KEY"
```

Ou use a UI: `https://evolution-api-esteticago.onrender.com/manager` (login com a API key).

## Ligar no EsteticaGo

No serviço **principal** do EsteticaGo no Render, aba *Environment*:

| Variável | Valor |
|---|---|
| `EVOLUTION_API_URL` | `https://evolution-api-esteticago.onrender.com` |
| `EVOLUTION_API_KEY` | a `AUTHENTICATION_API_KEY` gerada aqui |
| `EVOLUTION_API_INSTANCE` | `esteticago` (o `instanceName` do passo 1) |

Redeploy do serviço principal. O `WhatsAppService` já detecta as 3 variáveis e
passa a enviar de verdade (confirmação de agendamento pós-PIX, cobrança de
mensalidade).

## Notas

- **Migrações do banco:** o `CMD` do `Dockerfile` é `npm run start:prod` (conforme
  pedido). Se o boot falhar com erro de tabela/Prisma inexistente, é porque a
  migração não rodou — troque o `CMD` por:
  ```dockerfile
  CMD ["/bin/bash", "-c", ". ./Docker/scripts/deploy_database.sh && npm run start:prod"]
  ```
  (roda `prisma migrate deploy` antes de subir a API).
- **Porta:** a Evolution escuta em `SERVER_PORT` (8080). O Render detecta a porta
  automaticamente em serviços Docker; se não detectar, adicione `PORT=8080` nas
  variáveis do serviço.
- **"Versão desatualizada" ao parear:** defina `CONFIG_SESSION_PHONE_VERSION` com
  a versão web atual do Baileys.
