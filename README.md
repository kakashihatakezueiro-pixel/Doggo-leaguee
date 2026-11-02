# Doggo League — multiplayer online (1 contra 1)

Servidor multiplayer do Doggo League para a tua conta Cloudflare, no plano gratuito.
O jogo (HTML) é servido pelo próprio Worker em HTTPS, por isso não precisas de outro alojamento.
O modo single-player não foi alterado. O servidor Node antigo continua a ser o fallback, fora deste pacote.

## O que está no pacote

| Pasta / ficheiro | Para quê |
|---|---|
| `Doggo_League.html` | O jogo (cópia de trabalho). |
| `public/index.html` | Cópia que o Worker serve em `/` (gerada por `scripts/sync-public.mjs`). |
| `src/` | Worker (`index.js`), Durable Objects (`do-hub.js`, `do-room.js`), contas, salas, combate. |
| `migrations/` | Esquema D1 (contas, ranking, plantéis, inventário). |
| `wrangler.toml` | Configuração do Worker, Durable Objects, D1 e do jogo estático. |
| `scripts/` | `install.sh` (testes locais), `publish.sh` (publicação), `check-config.mjs`. |
| `test/` | Testes automáticos (protocolo, contas, salas, combate, cache do combate, Worker). |
| `local/` | Servidor local que executa o código do Worker (para testar sem Cloudflare). |
| `e2e/` | Testes com dois navegadores (Chromium) e do single-player. |
| `bench/` | Medição de CPU do combate. |

## Segurança das palavras-passe (leia uma vez)

As palavras-passe **nunca** saem do computador em texto simples:

1. O jogo pede ao servidor o *sal* da conta (`prelogin`).
2. O navegador calcula `PBKDF2-SHA256(palavra-passe, sal, 100 000 iterações)` e envia só esse resultado.
3. O servidor guarda `HMAC-SHA256(PASSWORD_PEPPER, resultado)`.

**`PASSWORD_PEPPER`** é um segredo do Worker (mínimo 32 caracteres). Sem ele ninguém consegue verificar uma palavra-passe,
mesmo que copie a base de dados. Não o ponhas no código nem em `wrangler.toml`. Define-o assim:

```
npx wrangler secret put PASSWORD_PEPPER
```

Guarda uma cópia do valor num gestor de palavras-passe: se o perderes, as contas existentes deixam de entrar.
Contrapartida conhecida: as palavras-passe só são validadas no navegador (o servidor não as vê).

## Requisitos

- Node.js 22 ou superior.
- Conta Cloudflare (plano gratuito basta para o teste). Subdomínio `workers.dev` ativo.
- Chromium/Chrome só para os testes de navegador (opcional).

## Passo 1 — instalar e testar localmente (sem conta)

```
bash scripts/install.sh
```

Corre as dependências e a suíte de testes. Deve terminar com "Instalação e testes locais concluídos".

## Passo 2 — entrar na Cloudflare

```
npx wrangler login
```

## Passo 3 — publicar

```
bash scripts/publish.sh
```

O script:
1. copia o jogo para `public/`;
2. cria a base D1 `doggo-mp` se ainda não existir e grava o `database_id` em `wrangler.toml`;
3. pergunta se queres definir o `PASSWORD_PEPPER` (responde `s` na primeira vez; gera um valor longo e guarda-o);
4. aplica as migrações D1 na base remota;
5. publica o Worker.

No fim mostra o endereço, por exemplo `https://doggo-league-mp.<conta>.workers.dev`.

Se preferires fazer passo a passo: `npx wrangler d1 create doggo-mp`, cola o `database_id` em `wrangler.toml`,
`npx wrangler secret put PASSWORD_PEPPER`, `npx wrangler d1 migrations apply doggo-mp --remote`, `npx wrangler deploy`.

## Passo 4 — verificar

- `https://<o-teu-worker>.workers.dev/health` deve responder `{"ok":true,...}`.
- `https://<o-teu-worker>.workers.dev/` abre o jogo.

## Partida de teste (dois computadores)

1. Computador A: separador **Multiplayer** → **Criar conta** → **Criar sala** (pública, ou privada com palavra-passe).
2. Computador B: **Criar conta** → **Atualizar lista** e **Entrar** na sala (ou entra na sala privada com a palavra-passe).
3. Ambos: **Usar a minha equipa**, depois **Estou pronto**.
4. O anfitrião (quem criou a sala, ou quem ficou com a sala se o outro saiu) carrega em **Começar combate**.
5. Jogar. O resultado é igual nos dois ecrãs.

Regras que vais notar:
- Uma conta só está numa sala de cada vez. Sair de um combate conta como **desistência** (o adversário vence).
- Se o navegador fechar, tens **45 segundos** para voltar: o combate fica em pausa e retoma ao entrar de novo.
- Se a ligação falhar fora desse prazo, perdes o combate por desconexão.
- Só o anfitrião pode iniciar o combate.

## Endereço do servidor noutro domínio

Por omissão, o jogo liga-se ao mesmo endereço onde está a página (`wss://<mesmo-endereço>/ws`).
Se alojares o HTML noutro sítio, aponta-o para o Worker:

```
node scripts/set-game-server.mjs wss://doggo-league-mp.<conta>.workers.dev/ws
```

## Testar localmente (sem Cloudflare)

```
PASSWORD_PEPPER=um-valor-com-pelo-menos-32-caracteres npm run serve:local
```

Abre `http://localhost:8080/`. Para testar com dois navegadores: `npm run e2e:browsers` (precisa de Chrome;
por omissão usa `CHROME_PATH` ou o caminho definido em `e2e/two-browsers.e2e.mjs`).

**Importante:** o servidor local executa o código real do Worker, mas **não é a Cloudflare**. Os resultados locais
não provam o comportamento em produção.

## Limite de CPU do combate (plano gratuito)

O plano gratuito limita cada pedido a 10 ms de CPU. O combate é determinístico: é reconstruído a partir da semente
e das ações guardadas. Para evitar reconstruir o combate a cada alarme (500 ms), a sala mantém o combate em memória e só o
reconstrói quando a instância é reiniciada. Medições locais em `bench/cpu-results.json` (Node, não Cloudflare):
por alarme, cerca de 2 a 3,5 ms com reconstrução completa e cerca de 0,03 ms com a cache. Em combates muito longos e
após reinícios frequentes da instância, a reconstrução a frio pode aproximar-se do limite de 10 ms. Isto **não foi
medido na Cloudflare**; o teste está descrito abaixo.

## Testes na Cloudflare (a fazer pelo dono da conta)

Não foram feitos por este pacote. Para os fazer:
1. Publicar (passo 3) e abrir o jogo em dois computadores.
2. Fazer duas partidas completas e uma com um dos computadores a fechar o separador e a voltar em menos de 45 s.
3. No painel Cloudflare (Workers & Pages → doggo-league-mp → Logs), confirmar que não aparecem erros de
   "exceeded CPU" nessas partidas.

## Resolução de problemas

- **"PASSWORD_PEPPER em falta"**: não definiste o segredo. Corre `npx wrangler secret put PASSWORD_PEPPER` e volta a publicar.
- **"Conta antiga"**: contas do servidor Node antigo não são migradas por este servidor. Cria uma conta nova.
- **"Este navegador não suporta a cifra necessária"**: abre o jogo pelo endereço `https://` do Worker (ou `localhost`).
- **Não consigo ligar ao servidor**: confirma `/health` no endereço publicado.

## Cães novos: Pinscher, Shih Tzu e Maltês

Três cães jogáveis que se obtêm **apenas pela caixa de recrutamento** (não são vendidos na Loja e não aparecem como inimigos da campanha).

| Cão | Classe / raridade | HP / ATK / DEF / VEL | Especial | Efeito |
|---|---|---|---|---|
| Pinscher Ágil | Assassino / Raro | 95 / 16 / 5 / 1.4 | Investida Relâmpago (recarga 14 s) | 3 golpes do próprio cão ao alvo inimigo |
| Shih Tzu Calmo | Suporte / Raro | 115 / 10 / 9 / 1.0 | Brisa Purificadora (recarga 16 s) | cura os aliados (ATK × 2,5) e remove lentidão e atordoamento |
| Maltês Real | Tanque / Épico | 150 / 10 / 11 / 0.85 | Pelo Dourado (recarga 18 s) | golpe no alvo, atordoa-o 1,5 s e dá escudo de 30% do HP máximo ao próprio cão |

Os valores foram afinados por medição: contra os oito cães existentes (nível 19, ambos os lados a usar o especial) cada um vence entre 53% e 56% dos combates.

**Impacto na caixa.** O sorteio é por raridade, por isso os três entram nos pools de Raro (5 cães) e de Épico (3 cães). Probabilidades por cão numa caixa (sem contar a garantia):

| Cão | Antes | Depois |
|---|---|---|
| Comum (corgi, bulldog, collie) | 20% | 20% (sem alteração) |
| Raro (shiba, husky, bernard) | 9,3% | 5,6% |
| Épico (golden, doberman) | 6,0% | 4,0% |
| Pinscher e Shih Tzu (novos) | — | 5,6% |
| Maltês (novo) | — | 4,0% |

**Como se desbloqueia**
- Single-player: caixa de recrutamento. Fica guardado no progresso local.
- Multiplayer: o cliente declara o cão no plantel (nível 1, sem nós de árvore). O servidor concede-o uma vez, como `dog:<id>` no inventário, com origem `claimed_unverified`. Um cão da caixa nunca entra no plantel sem concessão.

**Limitações**
- O servidor não consegue verificar se a conta obteve o cão na caixa no single-player. A concessão no multiplayer tem a mesma confiança inicial que a importação do plantel.
- O ranking é por conta, não por cão.
- Os testes de multiplayer usam o Durable Object simulado e um servidor local; não correm no runtime Cloudflare nem em redes diferentes.

**Testes**
- `npm test`: `test/new-dogs.test.js` (dados, pool, efeitos dos especiais, posse no servidor, regressão) e `test/new-dogs-mp.test.js` (partida com os cães novos e validação do plantel).
- `test/fixtures/existing-dogs-digests.json`: impressões digitais de 336 combates entre cães existentes. Qualquer alteração que mude um combate antigo falha o teste.
- `npm run e2e:newdogs`: single-player no Chromium (caixa, equipa, desenho, combate, campanha). Requer `CHROME_PATH`.
- `npm run e2e:two-newdogs`: multiplayer com dois navegadores, cada um com cães novos. Requer `CHROME_PATH`.
- `npm run e2e:browsers`: teste original de dois navegadores, sem alterações.
