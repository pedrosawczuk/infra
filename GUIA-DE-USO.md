# Guia de uso na prática

Este guia mostra o dia a dia do kit. A instalação inicial (uma vez só) está no [README](./README.md).

## Resumo

Para colocar um projeto no ar:

1. Deixe o projeto pronto para o kit (Dockerfile, `/health`, graceful shutdown).
2. Rode `init-deploy` dentro do repositório e responda às perguntas.
3. Faça `git push` na `main`.

## 1. Preparar o projeto

O repositório precisa estar no GitHub e ter um Dockerfile. Abra o projeto no Claude Code e cole o prompt da seção "O que cada app precisa ter" do README. Ele cria o Dockerfile, o `.dockerignore`, a rota `GET /health` e o desligamento gracioso, sem mexer na regra de negócio.

Se o projeto usa Redis ou S3, veja a seção 5 antes de continuar. O código precisa ler as variáveis certas.

## 2. Rodar o init-deploy

No Git Bash, dentro da pasta do repositório:

```bash
init-deploy
```

Ele pergunta, nesta ordem:

```
Nome do app [meu-projeto]:
Domínio [api.pedrosawczuk.dev]:
Caminho no domínio (vazio = raiz):
Porta do app no container [3000]:
Serviços:
  [x] Postgres
  [ ] Redis
  [ ] S3
Dockerfile [Dockerfile]:
Arquivo de variáveis para enviar à VPS [.env.production]:
Comando de migration (vazio = nenhum):
Gerar o job de checks (lint, typecheck, test, build)? [S/n]:
Fazer commit e push do workflow agora? [s/N]:
```

Como responder:

| Pergunta | Dica |
|---|---|
| Nome | Vira a pasta na VPS, o serviço, a imagem no GHCR e o nome do banco. O padrão vem do nome do repositório |
| Domínio | O padrão é o último que você usou |
| Caminho | Deixe vazio para o app responder na raiz do domínio. Se o domínio já tem outros apps, informe um caminho como `/meu-app` |
| Serviços | Espaço marca e Enter confirma, se o `gum` estiver instalado. Sem ele, o comando pergunta `[S/n]` para cada um |
| Dockerfile | Detecta `Dockerfile` na raiz ou `apps/*/Dockerfile` |
| Variáveis | Detecta `.env.production`. Digite `-` para não enviar nada |
| Migration | Comando rodado dentro da imagem antes de cada troca de versão. Ex.: `node dist/migrate.js` |
| Push | Se disser que sim, o primeiro deploy começa na hora. Se disser que não, você revisa o `deploy.yml` antes |

Digitar `-` em qualquer pergunta deixa o campo vazio, mesmo quando há um padrão.

No fim ele mostra um resumo e pede confirmação. Só depois disso mexe na VPS e nos secrets.

## 3. Primeiro deploy

Se você não deixou o `init-deploy` fazer o push:

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: deploy automático na VPS"
git push
gh run watch
```

Quando a execução terminar, teste:

```bash
curl https://api.pedrosawczuk.dev/meu-app/health
```

Se o DNS ainda não apontava para a VPS, o `init-deploy` avisa. Crie o registro A para o IP da VPS. O certificado HTTPS é emitido sozinho quando o DNS propagar.

## 4. Exemplos de cenários

**Projeto só com Postgres.** Aceite os padrões. Só o Postgres vem marcado.

```bash
init-deploy meu-api api.pedrosawczuk.dev --path /meu-api
```

**Projeto com Postgres, Redis e S3.**

```bash
init-deploy meu-sistema api.meusistema.com.br --redis --s3 --s3-quota 50GiB --dockerfile apps/api/Dockerfile --migrate "node dist/migrate.js"
```

**Projeto sem banco (por exemplo, um site estático ou um worker).**

```bash
init-deploy meu-site site.pedrosawczuk.dev --no-db
```

**Só regerar o workflow, sem mexer na VPS** (o app já existe lá):

```bash
init-deploy meu-app --skip-vps --force
```

## 5. Usando Redis e S3 no código

O `init-deploy` grava as credenciais no `.env` do app na VPS. O código só precisa ler as variáveis.

**Postgres:** `DATABASE_URL`. Cada app tem usuário e banco próprios, sem acesso aos outros bancos.

**Redis:** `REDIS_URL`. O usuário do app só acessa chaves que começam com `<nome-do-app>:`. Configure o prefixo no cliente. Exemplos:

```typescript
new Redis(env.REDIS_URL, { keyPrefix: 'meu-app:' })
new Queue('emails', { connection, prefix: 'meu-app:bull' })
```

Sem o prefixo, o Redis responde `NOPERM`.

**S3 (Garage):** as variáveis são `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` e `S3_FORCE_PATH_STYLE`. Com o SDK da AWS:

```typescript
new S3Client({
  endpoint: env.S3_ENDPOINT,
  region: env.S3_REGION,
  forcePathStyle: env.S3_FORCE_PATH_STYLE === 'true',
  credentials: {
    accessKeyId: env.S3_ACCESS_KEY,
    secretAccessKey: env.S3_SECRET_KEY,
  },
})
```

O endpoint `http://garage:3900` só funciona de dentro da rede Docker. Para links públicos de arquivos, gere URLs assinadas ou sirva os arquivos pelo próprio app.

O bucket tem limite de tamanho (`--s3-quota`). Quando enche, o upload falha com `Forbidden: quota is reached`. Para aumentar: `docker exec garage /garage bucket set-quotas --max-size 50GiB <app>`.

Lembre de declarar essas variáveis no schema de ambiente do projeto, para o boot falhar se alguma faltar.

## 6. No dia a dia

**Novo deploy:** `git push` na `main`. Os checks rodam, a imagem é publicada e a versão nova sobe ao lado da antiga. A antiga só sai depois que o healthcheck da nova passa.

**Refazer o deploy sem commit:** `gh workflow run Deploy`.

**Ver logs:**

```bash
ssh usuario@IP-da-VPS
cd ~/komodo/apps/meu-app && docker compose logs -f
```

**Trocar uma variável:** edite `~/komodo/apps/meu-app/.env` na VPS e rode `docker compose up -d --force-recreate meu-app`. Não rode o `init-deploy --env-file` de novo, porque ele acrescenta as linhas em vez de substituir.

**Voltar para uma versão anterior:** pegue o SHA do commit na aba Actions e rode na pasta do app:

```bash
IMAGE_TAG=<sha> docker rollout meu-app
```

**Abrir o painel do Garage:**

```bash
ssh -L 3909:localhost:3909 -L 3900:localhost:3900 usuario@IP-da-VPS
```

Painel em `http://localhost:3909`. Com o túnel da 3900 aberto, o Cyberduck também conecta em `localhost:3900`.

**Recuperar um arquivo apagado do S3:** o backup guarda o que foi apagado ou alterado por 60 dias.

```bash
backup-restore s3 find meu-app contrato                             # em que dia ele aparece
backup-restore s3 restore meu-app uploads/contrato.pdf 2026-11-15   # devolve ao bucket
```

**Remover um app:** veja a seção "Operação" do README. Ela não apaga o banco, o usuário do Redis nem o bucket. Para apagar, rode `DROP DATABASE` e `DROP ROLE` no Postgres, `ACL DELUSER <app>` no Redis e, no Garage, `bucket delete --yes <app>` (vazio) e `key delete --yes <app>-app`.

## 7. Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `não achei o repositório no GitHub` | O repositório ainda não foi publicado. Rode `gh repo create --source . --private --push` |
| `$DOMAIN já tem apps com --path` | Já existe um app nesse domínio. Informe um caminho ou use outro subdomínio |
| `container redis não está rodando` | O Redis não foi instalado na VPS. Veja o passo 3 da instalação no README |
| Deploy trava esperando o healthcheck | O Dockerfile não tem `HEALTHCHECK`, ou o app não responde em `/health` na porta de `PORT` |
| `NOPERM` no Redis | O código está usando chaves sem o prefixo `<app>:` |
| O deploy falha em `Rodando migration` | A migration quebrou. Veja o log da Action e confirme que o comando roda dentro da imagem |
| `new-app` falhou no meio | O banco, o usuário do Redis e o bucket, se foram criados, continuam lá. Rodar de novo com `--skip-vps` regera só o workflow. Para refazer a VPS, apague a pasta do app em `~/komodo/apps` e rode o comando outra vez |
