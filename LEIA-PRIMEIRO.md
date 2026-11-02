# Leia primeiro: publicar o Doggo League pelo telemóvel

Não precisas de saber programar nem de usar o computador. Só tens de seguir estes passos,
por esta ordem. Cada passo tem um botão ou um campo que vais encontrar no ecrã.

Se algum passo não bater certo com o que vês, para e não cliques em mais nada. Diz-me o que aparece no ecrã.

---

## Parte 1: criar o repositório no GitHub

1. Abre **github.com** no navegador do telemóvel e entra na tua conta.
2. Toca no **+** (canto superior) e depois em **New repository**.
3. Nome: `doggo-league-online`.
4. Marca **Private** (privado).
5. Toca em **Create repository**.
6. Na página que aparece, toca em **uploading an existing file**.
7. Descomprime o ZIP do projeto e envia **todos os ficheiros e pastas** para lá.
   - Inclui a pasta **.github** (é oculta; se não a vires, usa um computador e arrasta a pasta inteira para a página do repositório).
8. Em baixo, toca em **Commit changes**.

## Parte 2: criar o token da Cloudflare

1. Abre **dash.cloudflare.com** e entra.
2. Toca no teu perfil (canto superior direito) e depois em **API Tokens**.
3. Toca em **Create Token** e depois em **Create Custom Token**.
4. Dá um nome, por exemplo `doggo-github`.
5. Adiciona estas três permissões (todas da secção **Account**):
   - **Workers Scripts**: Edit
   - **D1**: Edit
   - **Account Settings**: Read
6. Em **Account Resources**, escolhe só a tua conta.
7. Toca em **Continue to summary** e depois em **Create Token**.
8. **Copia o token agora** e guarda-o num sítio seguro. Só aparece uma vez.

## Parte 3: copiar dois números da Cloudflare

1. Na página principal da Cloudflare (dash.cloudflare.com), procura **Account ID** no lado direito e copia-o.
2. Vai a **Storage & databases > D1**, abre a base **doggo-mp** e copia o **Database ID** (um código longo com traços).

## Parte 4: guardar as chaves no GitHub

No teu repositório no GitHub:

1. Toca em **Settings** (separador no topo).
2. No menu da esquerda, toca em **Secrets and variables** e depois em **Actions**.
3. Toca em **New repository secret** e cria estes dois:
   - Nome: `CLOUDFLARE_API_TOKEN` · Valor: o token da Parte 2
   - Nome: `CLOUDFLARE_ACCOUNT_ID` · Valor: o Account ID da Parte 3
4. Toca em **Variables** (no mesmo ecrã, separador ao lado de Secrets) e depois em **New repository variable**:
   - Nome: `D1_DATABASE_ID` · Valor: o Database ID da Parte 3

## Parte 5: publicar

1. No repositório, toca em **Actions** (separador no topo).
2. Toca em **Publicar Doggo League** (menu da esquerda).
3. Toca em **Run workflow** e depois no botão verde **Run workflow**.
4. Espera uns minutos. Se aparecer um visto verde, está publicado.
5. Se aparecer uma cruz vermelha, tira uma captura do ecrã e envia-ma. Não tentes mais nada.

## Parte 6: confirmar que funciona

Abre no navegador: `https://doggo-league-mp.<o-teu-nome>.workers.dev/health`

Se aparecer uma resposta de OK, está no ar. Abre a página principal, entra com uma conta que já tinhas e confirma que o ranking e o plantel continuam lá.

---

## Coisas que NÃO deves fazer

- Não apagues a base de dados **doggo-mp**.
- Não cries uma base nova nem um segredo novo chamado `PASSWORD_PEPPER`. Esse segredo já existe e tem de ficar como está.
- Não partilhes o token da Cloudflare com ninguém.

Se algo te pedir para fazer isto, para e pergunta primeiro.

## Mais detalhes

Para a explicação técnica completa (permissões, reversão e verificações), vê o ficheiro `DEPLOY.md`.
