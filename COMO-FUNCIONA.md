# FARM — Sistema completo (painel web + agente local no Mac)

## 1. O que é cada parte

| Parte | Onde roda | Função |
| --- | --- | --- |
| Painel web (`src/`) | Nuvem — https://macbook-farm-app.lovable.app | Cadastro de instâncias (perfis), CNPJs próprios, sites institucionais, status/progresso em tempo real |
| Banco de dados | Nuvem (Lovable Cloud) | Tabelas `instances`, `own_companies`, `company_sites`, `profiles` |
| Agente local (`agent/`) | Seu MacBook (Node + Playwright) | Abre o Chromium real com o perfil/proxy de cada linha e executa a rotina do BM |
| Instalador | `/api/public/install-agent` | Script bash que grava a pasta `~/farm-agent`, instala tudo e liga o agente |

O navegador do painel **não pode** abrir perfis nem digitar logins — por isso o agente é local. O painel é o cérebro (dados e comandos), o agente é as mãos.

## 2. Instalar no MacBook (uma vez)

1. Instale o Node.js 20+ (https://nodejs.org/pt-br/download).
2. Abra o Terminal e rode:

```bash
rm -rf ~/farm-agent && curl -fsSL https://macbook-farm-app.lovable.app/api/public/install-agent | bash
```

3. Digite o e-mail e a senha da sua conta do painel quando pedir.
4. Deixe essa janela do Terminal aberta — ela fica em modo `watch`.

Para religar depois:

```bash
cd ~/farm-agent && node farm-agent.mjs watch
```

Outros comandos:

```bash
node farm-agent.mjs list                     # lista as instâncias
node farm-agent.mjs run --instance ab12cd     # roda uma instância
node farm-agent.mjs run --instance all        # roda todas
```

## 3. Fluxo de uso

1. No painel, cadastre a instância (nome, e-mail, senha, 2FA, proxy, tema do site) — ou cole a linha `id|senha|2fa|email`.
2. Em **Empresas**, ficam os seus CNPJs (já salvos: 58.683.704, 31.807.515, 63.072.303). O agente escolhe um por round-robin.
3. Clique em **Iniciar**. O status vira `rodando` e o agente do Mac pega a instância em até 10s.
4. O Chromium abre com o perfil persistente (`~/.farmapp/profiles/<short_id>`) e o proxy da linha.
5. O progresso (passo x/12) aparece ao vivo no painel.
6. No fim, o agente **pausa em `analise_pendente`** aguardando o SMS. Quando o código chegar, clique em **Continuar após SMS** no painel e ele retoma.

## 4. Rotina automatizada (ordem real)

1. Abrir perfil do Facebook
2. Login com e-mail/senha da linha
3. 2FA automático (TOTP gerado do segredo da linha)
4. Pegar dados do CNPJ (Receita Federal visível → fallback cnpj.biz/API)
5. Publicar o site institucional da empresa em `/s/<slug>` (tema escolhido)
6. Criar a BM em business.facebook.com
7. Preencher **Business details** com os dados do CNPJ (país sempre Brasil)
8. Adicionar o domínio raiz `macbook-farm-app.lovable.app` (Add → Create a domain)
9. Copiar a meta tag gerada e publicá-la no site/raiz do painel
10. Verificar o domínio
11. Criar a WABA do WhatsApp (nome do perfil, categoria Outro, sem número)
12. Telefone/SMS (SMS 24h) → **pausa para você confirmar no painel** → envia para verificação

Cada etapa concluída é gravada em `tags` da instância (`feito:dominio`, `feito:waba`, …). Se der erro e você clicar em Iniciar de novo, ele **pula o que já foi feito** e continua de onde parou.

## 5. Recursos de apoio

- **CAPTCHA**: `agent/lib/captcha.mjs` resolve hCaptcha, reCAPTCHA e FunCaptcha (Arkose/Facebook) via 2Captcha (chave no `.env` como `FARM_CAPTCHA_API_KEY`).
- **SMS**: `agent/lib/sms.mjs` usa `api.sms24h.org` com a sua API key.
- **Digitação humana**: `agent/lib/human.mjs` (delays, foco, sem duplicar texto).
- **Erro**: a janela do Chromium fica aberta para correção manual (`FARM_KEEP_OPEN=true`).

## 6. Estrutura do zip

```
src/                    painel web (TanStack Start + React)
  routes/               páginas: /, /auth, /empresas, /sites, /s/$slug
  components/farm/      tabela, dialogs, stats
  lib/                  farm.ts, siteThemes.ts, verification.functions.ts
  integrations/supabase/ conexão com o banco
agent/                  agente local (Node + Playwright)
  farm-agent.mjs        CLI: login | list | run | watch
  lib/                  api, browser, human, cnpj, sms, captcha, totp, site
  tasks/default.mjs     as 12 etapas da rotina
supabase/               config
```

## 7. Ajustes no `.env` do agente (`~/farm-agent/.env`)

| Variável | Padrão | Função |
| --- | --- | --- |
| `FARM_CONCURRENCY` | 2 | perfis simultâneos |
| `FARM_POLL_MS` | 10000 | intervalo do watch |
| `FARM_HEADLESS` | false | `true` roda sem janela |
| `FARM_SPEED` | 0.6 | velocidade da digitação |
| `FARM_KEEP_OPEN` | true | mantém janela aberta em erro |
