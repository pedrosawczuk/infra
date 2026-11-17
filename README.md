# infra

Kit de deploy da minha VPS. Com ele, colocar um projeto no ar com CI/CD vira um comando, rodado de dentro do repositório no meu PC. Sem argumentos, ele pergunta tudo (nome, domínio, serviços Postgres/Redis/S3...). Com argumentos, funciona só por flags:

```bash
init-deploy
init-deploy meu-app api.pedrosawczuk.dev --path /meu-app --redis --env-file .env.production
```

Passo a passo prático em [`GUIA-DE-USO.md`](./GUIA-DE-USO.md).

Esse comando:

1. Roda o `new-app` na VPS por SSH, que cria a pasta com `compose.yaml` e `.env`, o usuário e o banco no Postgres (e, se pedido, o usuário no Redis e o bucket no Garage) e a rota no Caddy, com HTTPS automático.
2. Envia as variáveis do `.env.production` local para o `.env` do app na VPS.
3. Cadastra os secrets do deploy no repositório.
4. Gera o `.github/workflows/deploy.yml` com os checks que o projeto tiver (lint, typecheck, test, build).

Daí para frente, cada push na `main` roda os checks e, se passarem, builda a imagem, publica no GHCR, roda a migration e troca a versão sem downtime.

Este repositório não guarda nenhum segredo e pode ser público. Isso deixa os outros repositórios chamarem o workflow sem configuração extra e permite clonar na VPS sem autenticação.

## Como a VPS está organizada

```
~/komodo/
├── kit.env              # configuração do new-app (tem a senha do banco, chmod 600)
├── infra/               # clone deste repositório
├── postgres/            # Postgres 18, na rede db
├── redis/               # Redis compartilhado, na rede cache (opcional)
├── garage/              # Garage (S3) e painel, na rede storage (opcional)
├── caddy/
│   ├── Caddyfile        # só importa sites/*.caddy
│   ├── sites/           # um arquivo por domínio
│   └── routes/          # uma pasta por domínio, um arquivo por app
└── apps/
    └── <nome>/          # compose.yaml + .env de cada app
```

Redes Docker: `proxy` (Caddy e apps), `db` (Postgres e apps), `cache` (Redis e apps) e `storage` (Garage e apps). O app só entra nas redes dos serviços que pediu, então um projeto só com Postgres não enxerga o Redis nem o Garage. Nenhum app publica porta no host.

Cada app recebe credenciais próprias: um usuário e um banco no Postgres, um usuário no Redis restrito às chaves com prefixo `<app>:` e uma chave no Garage com acesso só ao bucket `<app>`, com limite de tamanho. O Caddy encontra as instâncias de cada app pelo nome do serviço, o que permite ter duas instâncias durante o deploy.

## Instalação (uma vez só)

### 1. Clonar e instalar o comando

```bash
git clone https://github.com/pedrosawczuk/infra.git ~/komodo/infra
mkdir -p ~/.local/bin
ln -s ~/komodo/infra/vps/new-app ~/.local/bin/new-app
```

O Ubuntu coloca `~/.local/bin` no PATH no próximo login. Saia e entre de novo na VPS e confira com `new-app --help`.

### 2. Configuração

```bash
cp ~/komodo/infra/vps/kit.env.example ~/komodo/kit.env
chmod 600 ~/komodo/kit.env
nano ~/komodo/kit.env
```

### 3. Redis e Garage (só se for usar)

Preencha `REDIS_ADMIN_PASSWORD` no `kit.env` e suba o Redis:

```bash
docker network create --internal cache
mkdir -p ~/komodo/redis && cd ~/komodo/redis
cp ~/komodo/infra/vps/redis/compose.yaml .
grep '^REDIS_' ~/komodo/kit.env > .env
docker compose up -d

```

O Garage (S3) se instala com um comando. Ele gera os segredos, sobe o Garage e o painel, aplica o layout e cria a chave de backup. Pode rodar de novo sem medo:

```bash
~/komodo/infra/vps/garage/garage-setup
```

- Nada fica exposto na internet. O S3 escuta só em `127.0.0.1:3900` na VPS e, para os apps, em `http://garage:3900` na rede `storage`. A API de administração não é publicada nem no host.
- Painel (garage-webui): `ssh -L 3909:localhost:3909 usuario@IP-da-VPS` e acesse `http://localhost:3909`. A senha é gerada no setup em `~/komodo/garage/webui-credentials.txt`. Guarde no gerenciador de senhas e apague o arquivo.
- Para navegar nos arquivos como pastas, use o Cyberduck (Amazon S3, `localhost`, porta `3900`, sem HTTPS) com `ssh -L 3900:localhost:3900`.
- O CLI fica no container: `docker exec garage /garage status`, `bucket list`, `key list`.

### 4. docker-rollout

```bash
mkdir -p ~/.docker/cli-plugins
curl https://raw.githubusercontent.com/wowu/docker-rollout/main/docker-rollout -o ~/.docker/cli-plugins/docker-rollout
chmod +x ~/.docker/cli-plugins/docker-rollout
```

### 5. Passar o Caddy para o formato do kit

O Caddyfile único vira um arquivo por domínio e um por app. Os certificados ficam no volume `caddy_data` e não se perdem na troca.

```bash
cd ~/komodo/caddy
cp Caddyfile Caddyfile.antigo
cp ~/komodo/infra/vps/caddy/compose.yaml ~/komodo/infra/vps/caddy/Caddyfile .
mkdir -p sites routes/api.pedrosawczuk.dev
```

Recrie a rota do Estudazz no formato novo. Ele continua com o compose antigo (`container_name: estudazz-api`), então a rota aponta direto para o container:

`sites/api.pedrosawczuk.dev.caddy`

```
api.pedrosawczuk.dev {
	import /etc/caddy/routes/api.pedrosawczuk.dev/*.caddy

	handle {
		respond "Not found" 404
	}
}
```

`routes/api.pedrosawczuk.dev/estudazz.caddy`

```
# new-app: estudazz path=/estudazz
handle_path /estudazz/* {
	reverse_proxy estudazz-api:3000
}
```

A primeira linha de comentário não é enfeite: o `new-app` lê ela para saber que `/estudazz` já está ocupado nesse domínio.

```bash
docker compose up -d
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
curl https://api.pedrosawczuk.dev/estudazz/health
```

Se algo der errado, o `Caddyfile.antigo` está ali para voltar.

### 6. No PC (Windows, pelo Git Bash)

Instale e faça login no GitHub CLI:

```powershell
winget install GitHub.cli
gh auth login
```

Gere a chave de deploy, que vai para os secrets de todos os repositórios. Ela fica sem passphrase porque o GitHub Actions precisa usar sozinho, então trate esse arquivo com o mesmo cuidado da sua chave pessoal:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_vps -N "" -C "github-actions-deploy"
cat ~/.ssh/deploy_vps.pub
```

Adicione a chave pública numa linha nova do `~/.ssh/authorized_keys` da VPS, sem apagar as outras.

Instale o comando e a configuração. O Git Bash já coloca `~/bin` no PATH:

```bash
mkdir -p ~/bin ~/.config
cp infra/local/init-deploy ~/bin/
cp infra/local/infra-kit.env.example ~/.config/infra-kit.env
nano ~/.config/infra-kit.env
```

Feche e abra o Git Bash e confira com `init-deploy --help`.

Opcional: instale o [gum](https://github.com/charmbracelet/gum) (`winget install charmbracelet.gum`) para escolher Postgres, Redis e S3 com caixas de seleção no modo interativo. Sem ele, o `init-deploy` pergunta `[S/n]` para cada serviço.

## Subir um projeto novo

### Com um comando (recomendado)

De dentro do repositório, já publicado no GitHub e com Dockerfile. Rode `init-deploy` sem argumentos para o modo interativo, ou use as flags:

```bash
init-deploy meu-app api.pedrosawczuk.dev --path /meu-app --redis --env-file .env.production
```

| Opção | Efeito |
|---|---|
| `--path`, `--port`, `--no-db`, `--redis`, `--s3`, `--s3-quota` | Repassadas ao `new-app` (veja abaixo) |
| `--dockerfile apps/api/Dockerfile` | Dockerfile fora da raiz, para monorepo |
| `--migrate "node dist/migrate.js"` | Migration rodada antes de cada troca de versão |
| `--env-file .env.production` | Envia as variáveis para a VPS. `NODE_ENV`, `PORT` e as variáveis dos serviços escolhidos (`DATABASE_URL`, `REDIS_URL`, `S3_*`) são ignoradas, porque a VPS já define |
| `--no-check` | Gera o workflow sem o job de checks |
| `--skip-vps` | Não mexe na VPS. Útil para regerar só o workflow (com `--force`) |
| `--push` | Faz commit do workflow e push, disparando o primeiro deploy |

O job de checks é montado a partir do projeto: detecta pnpm, yarn ou npm pelo lockfile, usa a versão do `.nvmrc` se existir (senão Node 24) e roda só os scripts `lint`, `typecheck`, `test` e `build` que existirem no `package.json`. Em monorepo com Turborepo, esses scripts da raiz rodam em todos os pacotes. Se quiser filtrar ou limitar o deploy por `paths`, edite o workflow gerado como em [`examples/deploy-monorepo.yml`](./examples/deploy-monorepo.yml).

O `--env-file` acrescenta no fim do `.env` da VPS. Rodar de novo com o mesmo arquivo duplica as linhas, então, para trocar uma variável depois, edite direto na VPS.

O workflow gerado também aceita disparo manual (`workflow_dispatch`), útil para refazer um deploy sem commit: `gh workflow run Deploy`.

### Passo a passo manual

#### Na VPS

```bash
new-app meu-app api.pedrosawczuk.dev --path /meu-app
# ou, com domínio próprio:
new-app meu-app meuapp.pedrosawczuk.dev
```

| Opção | Efeito |
|---|---|
| `--path /prefixo` | O app responde em `dominio/prefixo`. O prefixo é removido antes de chegar no app, então as rotas no código não mudam |
| `--port 8080` | Porta em que o app escuta no container. Padrão: 3000 |
| `--no-db` | Não cria banco nem `DATABASE_URL` |
| `--redis` | Cria um usuário no Redis e grava `REDIS_URL`. O app só acessa chaves que começam com `<app>:` |
| `--s3` | Cria no Garage o bucket `<app>` e a chave `<app>-app`, com acesso só a ele. Grava `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` e `S3_FORCE_PATH_STYLE` |
| `--s3-quota 50GiB` | Limite de tamanho do bucket. Padrão: `S3_DEFAULT_QUOTA` do `kit.env` (20GiB). Protege o disco que o Postgres também usa |

O comando valida o Caddy antes e depois. Se a configuração ficar inválida, ele desfaz a rota e a pasta. No fim, ele avisa se o DNS ainda não aponta para a VPS e imprime os comandos dos secrets e o `deploy.yml` prontos, com o nome do app preenchido.

Depois, complete o `.env` com as variáveis do app.

#### No repositório

1. Secrets (`VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`): o `new-app` imprime os comandos `gh secret set`. A chave de deploy pode ser a mesma para todos os projetos.
2. `.github/workflows/deploy.yml`: copie de [`examples/deploy-simples.yml`](./examples/deploy-simples.yml) ou, para monorepo, de [`examples/deploy-monorepo.yml`](./examples/deploy-monorepo.yml). O `app:` precisa ser igual ao nome usado no `new-app`.
3. Push na `main`.

Entradas do workflow reutilizável:

| Entrada | Padrão | Uso |
|---|---|---|
| `app` | obrigatório | Nome do app: pasta, serviço e imagem `ghcr.io/<dono>/<app>` |
| `dockerfile` | `Dockerfile` | Caminho do Dockerfile. Em monorepo: `apps/api/Dockerfile` |
| `context` | `.` | Contexto do build |
| `migrate` | vazio | Comando rodado na imagem nova antes da troca. Ex.: `node dist/migrate.js` |

O primeiro deploy sobe o serviço com `docker compose up -d --wait`. Do segundo em diante, usa `docker rollout`: sobe a versão nova ao lado da antiga, espera o healthcheck passar e só então remove a antiga.

## O que cada app precisa ter

O kit não funciona com qualquer imagem. Antes do primeiro deploy, confira:

- **`HEALTHCHECK` no Dockerfile.** Sem ele, o rollout não sabe quando a versão nova está pronta.
- **Escutar em `0.0.0.0`** na porta da variável `PORT`.
- **Desligar com calma no SIGTERM:** parar de aceitar conexões, terminar as requisições em andamento, fechar o banco e sair em até 25 segundos. O compose dá 30.
- **Node como processo principal** (`CMD ["node", ...]`), sem `pnpm` ou `npm` no meio, senão o sinal não chega.
- **Migrations compatíveis com a versão anterior**, porque as duas versões rodam juntas por alguns segundos. Adicionar coluna é tranquilo. Renomear ou remover se faz em dois deploys.
- **Se usar `migrate` no workflow, a migration sai do start do container**, para não rodar duas vezes.
- **Cron dentro do processo** pode disparar duas vezes se o deploy cair no horário exato.

Prompt para adaptar um projeto (cole no Claude Code com o repositório aberto):

```
Prepare este projeto para o kit de deploy da minha VPS. Leia o README, o package.json e o ponto de entrada do servidor antes de mudar algo. Não altere regra de negócio.
1. Dockerfile multi-stage com node:24-alpine, rodando como usuário node, NODE_ENV=production, CMD na forma exec com node (sem pnpm/npm como processo principal), HEALTHCHECK em GET http://localhost:$PORT/health usando node e fetch. Se for monorepo, o contexto é a raiz e o Dockerfile fica na pasta do app.
2. .dockerignore sem node_modules, .env*, .git e testes.
3. Rota GET /health respondendo 200, servidor em 0.0.0.0 na porta da variável PORT (padrão 3000) e trustProxy ligado.
4. Graceful shutdown em SIGTERM e SIGINT: fechar o servidor esperando as requisições em andamento, parar jobs agendados, fechar o pool do banco e sair com 0. Timeout de segurança de 25 segundos.
5. Se houver migrations, um comando separado para rodá-las dentro da imagem, fora do start do servidor. Me diga qual é.
Ao terminar, rode lint, typecheck e docker build e me mostre o resultado.
```

## Operação

**Logs:** `cd ~/komodo/apps/<nome> && docker compose logs -f`

**Voltar para uma versão anterior:** pegue o SHA do commit na aba Actions e rode na pasta do app:

```bash
IMAGE_TAG=<sha> docker rollout <nome>
```

O próximo push na `main` volta para a versão mais nova.

**Remover um app:**

```bash
cd ~/komodo/apps/<nome> && docker compose down
rm -rf ~/komodo/apps/<nome>
rm ~/komodo/caddy/routes/<dominio>/<nome>.caddy
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Se era o último app do domínio, apague também `sites/<dominio>.caddy` e a pasta vazia em `routes/`. O banco não é apagado automaticamente: rode `DROP DATABASE` pelo TablePlus se quiser. O bucket também fica: `docker exec garage /garage bucket delete --yes <nome>` (precisa estar vazio) e `docker exec garage /garage key delete --yes <nome>-app`.

**Atualizar o kit na VPS:** `cd ~/komodo/infra && git pull`

## Backups

Todo dia às 03h (Brasília), o `backup-run` salva cada banco do Postgres (mais as roles), o snapshot do Redis, as configs de `~/komodo`, o backup mais recente do Komodo e a lista de buckets, chaves e permissões do Garage. Cada arquivo é conferido (`pg_restore --list` e `sha256`) antes de o backup valer. Depois ele faz a rotação e envia tudo criptografado para o Google Drive.

| Nível | Guarda | Quando entra |
|---|---|---|
| `daily` | 7 | todo dia |
| `weekly` | 4 | primeiro backup de cada semana |
| `monthly` | 6 | primeiro backup de cada mês |

- A rotação é por quantidade: se o backup falhar por dias seguidos, nenhum backup bom é apagado. Os níveis semanal e mensal são hardlinks e não ocupam espaço extra.
- Se um dump encolher mais de 50% em relação ao dia anterior, a rotação diária é pausada e chega um alerta.
- Aos domingos, cada banco é restaurado num banco temporário, comparado com o original e apagado.
- No Drive, o que a rotação remove vai para `trash/` e fica 30 dias. Uma execução que tentar apagar mais de 40 arquivos lá é abortada.
- O [healthchecks.io](https://healthchecks.io) manda e-mail se o backup falhar ou não rodar.

### Arquivos do Garage (S3)

Os arquivos dos buckets não seguem a rotação diária/semanal/mensal: 17 cópias completas de cada bucket encheriam o Drive. Depois do envio normal, o `backup-run` copia cada bucket para o Drive assim:

```
gdrive-crypt:s3/
├── current/<bucket>/          estado atual (uma cópia só, incremental)
└── history/<data>/<bucket>/   o que foi apagado ou alterado nesse dia
```

- Nada é perdido no ato: arquivo apagado ou sobrescrito no bucket vai para `history/<data>` e fica lá por `S3_HISTORY_DAYS` (60 dias).
- A chave `backup` só lê. O `backup-run` libera a leitura bucket a bucket e lê a chave do próprio Garage, então nenhum segredo do S3 fica no `backup.env`.
- Se um bucket perder mais de 50% dos objetos de um dia para o outro, a cópia dele é pulada e chega um alerta. Uma execução que tentar apagar mais de 500 arquivos no Drive é abortada.
- Aos domingos, cada bucket é conferido contra a cópia do Drive.
- Os arquivos não ficam na VPS, só no Drive. Se o bucket for apagado no Garage, a cópia do Drive é mantida.
- Alerta de disco: chega um aviso quando o `/` passa de `DISK_ALERT_PERCENT` (80%).
- Uma falha no Garage nunca derruba o backup do Postgres: vira aviso.

### Instalação

```bash
cp ~/komodo/infra/vps/backup/{backup-run,backup-restore,drive-setup} ~/.local/bin/
chmod 700 ~/.local/bin/backup-*  ~/.local/bin/drive-setup
cp ~/komodo/infra/vps/backup/backup.env.example ~/komodo/backup.env
chmod 600 ~/komodo/backup.env && nano ~/komodo/backup.env
```

**rclone e Drive:** instale o rclone em `~/.local/bin` e crie um client OAuth próprio no Google Cloud (app "Externo", só o escopo `drive.file`, publicado **em produção** para o token não expirar em 7 dias, client do tipo "App para computador"). No PC, gere o token:

```powershell
$id = Read-Host "Client ID"
$secret = Read-Host "Client secret"
$json = @{ client_id = $id; client_secret = $secret; scope = "drive.file" } | ConvertTo-Json -Compress
$blob = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($json)).TrimEnd('=')
rclone authorize "drive" $blob
```

Na VPS, rode `drive-setup` e cole o Client ID, o secret e o token. Depois crie o remote `gdrive-crypt` (tipo `crypt`, `remote = gdrive:backups-vps`, `filename_encryption = off`, `directory_name_encryption = false`) e **guarde a senha e o salt no gerenciador de senhas**. Sem eles, os backups do Drive não podem ser abertos.

**Agendamento** (02h no fuso da VPS, `America/Cuiaba`, que equivale a 03h em Brasília):

```bash
echo '0 2 * * * $HOME/.local/bin/backup-run >> $HOME/backups/logs/cron.log 2>&1' | crontab -
```

### Restaurar

```bash
backup-restore list                               # backups na VPS e no Drive
backup-restore verify 2026-11-14                  # confere os checksums
backup-restore db estudazz_api                    # cria estudazz_api_restored_<data> ao lado do original
backup-restore db estudazz_api 2026-11-14 --replace   # apaga o atual e restaura no lugar (pede confirmação)
backup-restore fetch 2026-10-01                   # baixa um backup que só existe no Drive
backup-restore files 2026-11-14                   # extrai as configs para consulta, sem alterar nada
backup-restore s3 list                            # buckets no Drive e dias com histórico
backup-restore s3 find meu-app contrato           # procura um arquivo na cópia atual e no histórico
backup-restore s3 fetch meu-app uploads/a.pdf 2026-11-14   # baixa para ~/backups/restore/s3
backup-restore s3 restore meu-app uploads/a.pdf 2026-11-14 # devolve ao bucket (sem data: da cópia atual)
backup-restore s3 restore meu-app uploads/ --replace       # pasta inteira, sobrescrevendo o que existe
```

**Se a VPS morrer:** no PC, configure o rclone com o mesmo client (`rclone config`, remote `gdrive` do tipo drive com escopo `drive.file`) e um remote `gdrive-crypt` com a senha e o salt do gerenciador. Depois, `rclone copy gdrive-crypt:data/daily/<data> ./restore` traz os arquivos já decifrados. O `configs.tar.gz` tem os compose, `.env` e rotas do Caddy para reconstruir a VPS nova. Para o Garage: rode o `garage-setup`, recrie os buckets e as chaves (a lista está em `garage/buckets-info.txt` do backup) e suba os arquivos de volta com `rclone copy gdrive-crypt:s3/current/<bucket> ...`.

Logs de cada execução: `~/backups/logs/<data>.log`. Para rodar na hora: `backup-run` (ou `backup-run --restore-test`).
