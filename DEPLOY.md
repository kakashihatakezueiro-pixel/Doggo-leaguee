# Publicar o Doggo League Online (GitHub + Cloudflare)

Publicação automática: cada `push` para `main` executa `.github/workflows/deploy.yml`, que testa, aplica
as migrações pendentes do D1 e publica o Worker. Não é preciso CMD nem comandos no computador.

## O que o fluxo NÃO faz (de propósito)

- Não cria bases D1 (`wrangler d1 create`). A base `doggo-mp` existente é reutilizada.
- Não executa `wrangler secret put` e não gera `PASSWORD_PEPPER`. O segredo que já existe no Worker fica como está.
- Não apaga nem recria tabelas. As migrações `0001` e `0002` usam `IF NOT EXISTS`, e o Wrangler só aplica as que faltam.
- Não guarda credenciais no repositório. O token e o ID da conta ficam nos Secrets do GitHub.

## 1. Confirmar na Cloudflare (painel, sem comandos)

1. **Workers & Pages**: confirma que existe um Worker chamado exatamente `doggo-league-mp`.
   Se tiver outro nome, a publicação cria um Worker novo sem o `PASSWORD_PEPPER` e ninguém consegue entrar.
   Nesse caso, muda `name` em `wrangler.toml` para o nome que já existe.
2. Nesse Worker, em **Settings > Variables and Secrets**, confirma que `PASSWORD_PEPPER` está definido (tipo Secret). Não o copies nem o alteres.
3. **Storage & databases > D1**: abre a base `doggo-mp` e copia o **Database ID** (UUID, 36 caracteres). Este ID não é segredo, mas guarda-o.
4. Em **Settings > Bindings** do Worker deves ver: `DB` (D1 `doggo-mp`), `HUB` e `ROOM` (Durable Objects, classes `Hub` e `Room`) e `ASSETS`.
   Estes bindings vêm do `wrangler.toml`, por isso não precisas de os criar à mão.

## 2. Criar o token da API (Cloudflare)

1. Cloudflare > **My Profile > API Tokens > Create Token > Create Custom Token**.
2. Permissões (todas em **Account**):
   - Workers Scripts: **Edit**
   - D1: **Edit**
   - Account Settings: **Read**
3. Em **Account Resources**, escolhe apenas a tua conta.
4. Copia o token. Só aparece uma vez.
5. Copia também o **Account ID** (está no painel da Cloudflare, na coluna da direita da página principal).

## 3. Configurar o GitHub

No repositório: **Settings > Secrets and variables > Actions**.

| Tipo | Nome | Valor |
|---|---|---|
| Secret | `CLOUDFLARE_API_TOKEN` | o token do passo 2 |
| Secret | `CLOUDFLARE_ACCOUNT_ID` | o Account ID do passo 2 |
| Variable | `D1_DATABASE_ID` | o UUID da base `doggo-mp` do passo 1 |

Opcional, mas recomendado: **Settings > Environments > production > Required reviewers**. Assim, cada publicação
pede a tua confirmação antes de correr.

Não existe nenhum outro segredo para configurar. `PASSWORD_PEPPER` continua só na Cloudflare.

## 4. Primeira execução

Depois de enviar os ficheiros para o GitHub (ver "Passos no telemóvel"):

1. Abre a aba **Actions** do repositório e confirma que aparece "Publicar Doggo League".
2. Se ainda não quiseres publicar, não faças push para `main`. Quando quiseres, usa **Run workflow**.
3. Acompanha os passos. Se falhar, a mensagem indica o passo: `D1_DATABASE_ID` em falta, configuração,
   testes, migrações ou publicação.

## 5. Verificar depois de publicar

- `https://<o-teu-worker>.workers.dev/health` deve responder OK.
- Abre `https://<o-teu-worker>.workers.dev/` e entra com uma conta que já existia (prova de que o `PASSWORD_PEPPER` e a base são os mesmos).
- Confirma que o ranking e o plantel antigos continuam lá.

## Se algo correr mal

- **Reverter o código**: Cloudflare > Workers & Pages > `doggo-league-mp` > **Deployments**, escolhe a versão
  anterior e clica em **Rollback**. Não toca na base de dados.
- **Migrações não se desfazem automaticamente.** Se uma migração falhar, a publicação para antes do deploy.
  Não apagues a base nem recries o Worker para "resolver": pede ajuda antes.

## Passos no telemóvel (resumo)

1. No GitHub (navegador do telemóvel), cria um repositório privado chamado `doggo-league-online`.
2. Envia os ficheiros do projeto (a pasta inteira, com `.github`, `src`, `public`, `scripts`, `migrations`, `test`...).
   - Pelo telemóvel, o envio de ficheiros funciona, mas o envio de pastas inteiras é difícil. Se não conseguires,
     abre a página do repositório num computador e arrasta a pasta para o navegador. Não é preciso CMD.
3. Confirma que o `.github/workflows/deploy.yml` existe no repositório (é uma pasta oculta; se não aparecer, o envio falhou).
4. Faz os passos 1 a 3 acima (Cloudflare e GitHub). Depois, em **Actions**, executa **Run workflow**.
5. Verifica os pontos da secção 5.

## Limitações conhecidas

- Não existe `package-lock.json` no projeto. Por isso, cada execução instala as versões mais recentes permitidas por
  `package.json` (`wrangler ^4`). Quando tiveres um computador, gera o lockfile uma vez e comita-o para tornar as publicações repetíveis.
- Os testes de navegador (`e2e/`) não correm no GitHub. Precisam de Chromium e continuam a ser executados localmente.
- Publicação local (alternativa): `bash scripts/publish.sh` ou o `publicar-windows.bat`. Ambos procuram a base `doggo-mp`
  pelo nome e não a recriam.
