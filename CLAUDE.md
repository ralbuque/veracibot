# VeraciBot — Contexto do Projeto

## O que é

VeraciBot (**@veracibot** no X, site **https://veraci.bot**) é o "primeiro tribunal
de microcausas da internet": um agente de IA que monitora menções no X, interpreta
threads e emite julgamentos públicos. Tese: causas pequenas demais para a justiça
real (prato quebrado, aposta de R$ 50, fake news) ficam sem juiz — o VeraciBot as
julga em minutos, de graça, em público. Não compete com o judiciário; casos graves
são encaminhados a advogados parceiros. Dono: Ricardo (@peter_ancapsu / ralbuque@gmail.com).

## Arquitetura

- **Bot** (`src/veracibot/`): loop de polling (120s) na X API v2 via tweepy.
  - `main.py` — roteamento de menções e tarefas do ciclo. Ordem no `process_mention`:
    inscrição promoção → abertura formal de processo → menção em processo aberto →
    convites → caso existente (provas/followup) → gate de convite → limites → caso novo.
  - `judge.py` — o juiz (Claude via API Anthropic, modelo em `ANTHROPIC_MODEL`).
    Prompt com classificação de tipo, regras por tipo, JSON estruturado. Web search
    da Anthropic habilitada p/ fact-check (`JUDGE_WEB_SEARCH`). Lê imagens da thread
    (visão, máx 4). Sanitiza tags `<cite>` do web search (quebravam o JSON).
  - `x_client.py` — tweepy: menções, reconstrução de thread (com quotes 1 nível,
    `note_tweet` para notas longas, mídia), replies com throttle de 6s, outbox.
  - `scoring.py` — matriz de pontos; `reply.py` — formatação (contagem ponderada do X);
    `store.py` — SQLite (`veracibot.db`); `geo.py` — UF por `location` de perfil;
    `invites.py` — sistema de convites.
- **Site** (`src/veracibot/web/`): FastAPI + Jinja2 + Bootstrap 5.3 (CDN), lê o
  mesmo SQLite (leitura p/ páginas públicas; `webdb.py` escreve users/firms/feedback).
  Bilíngue pt/en (`i18n.py`). Identidade: V-balão teal (#2a7086), creme (#f7f1e5),
  navy (#22304f), fonte Nunito; mascote robô (`static/mascote.png`).
- **Banco**: SQLite único. Tabelas: cases, scores, ledger (auditoria de todo ponto),
  members, rejected_notices, compositions, appeals, appeal_votes (legado),
  promo_participants, processes, outbox, state (chaves genéricas + avisos únicos),
  users/firms/feedback (site).

## Regras de negócio

### Tipos de caso (o juiz classifica; JSON com tipo_caso, vencedor, afirmacoes etc.)
1. **fact_check** — decompõe em até 4 afirmações atômicas independentes; cada uma é
   `verdadeiro | falso | indeterminado` (NÃO existe "parcialmente" — leitura literal:
   exagero/distorção = falso). Multi-afirmações → replies encadeados (intro + 1 por
   afirmação). Escopo: só o FIO PRINCIPAL (raiz → menção + quotes dela).
2. **disputa** — quem tem razão numa discussão (condutas, interpretações).
3. **debate** — posições opostas sobre tema de opinião; julga MÉRITO ARGUMENTATIVO,
   quase sempre com vencedor; nunca declara a tese "correta". Header 🎤.

### Pontuação (tudo no ledger, saldo em scores; início: 1000 pts)
- Chamar custa 1 pt (saldo ≥ 1; estornado se erro, arquivado ou limite).
- **Fact-check**: partes = autor da afirmação × chamador (quem desmentiu antes na
  thread NÃO pontua). Falsa: chamador +11 (líq. +10), autor −11. Verdadeira:
  chamador −10 (total −11), autor +10. Self-check: ±11/−10. Cada afirmação pontua
  independente. Indeterminado: só o custo.
- **Disputa/debate**: vencedor/perdedor pelo juiz. Chamador-parte: +11/−10; outro
  lado +10/−11. Chamador neutro: só custo; lados ±10. Empate: só custo.
- **Composição**: SUSPENSA (`COMPOSITION_ENABLED=false`) — código intacto p/ religar.
- **Recurso**: perdedor, 1×/caso, 5 pts (voltam se reformar). Enquete pública nativa
  do X de 24h (@vencedor × @perdedor); quorum `APPEAL_QUORUM` (padrão 5); reforma
  inverte a pontuação. Acórdão como reply da enquete.
- **Fase de provas**: contradição factual decisiva sem prova nos autos → pede link/
  print em 48h (ônus de quem alega; prova negativa não se exige). Prazo vencido:
  alegante perde. Prints = indício não autenticável; links públicos pesam mais.

### Processos com partes registradas
- Formal: "@veracibot vamos iniciar um debate|disputa entre @a e @b sobre X" →
  registra partes/tema (tabela processes), instruções, encerramento por parte/abridor
  ("encerrado", "quem venceu?"). Expira em 7 dias com estorno. Tipo fixado.
- Informal: "quem venceu (o debate) entre @a e @b?" → filtro de partes na hora.
- Com partes: tweets delas nunca caem no corte de thread e entram no fio julgável.

### Convites e acesso
- `INVITE_ONLY` — atualmente **false** (aberto ao público; 1000 pts no 1º uso).
- Membros convidam 5 ("@veracibot convido @fulano"); dono convida ilimitado
  tweetando "Convido @x" da conta do bot. Não-convidado (se gate ativo): aviso 1×.

### Anti-farming / anti-spam
- 5 casos/hora por conta (aviso 1×/dia); 5 checagens/semana contra o mesmo autor
  (estorno + aviso 1× por par); autor com 5+ falsos = fonte de baixa credibilidade
  (julga, mas ninguém pontua); menção fora de contexto = silêncio total + estorno;
  recusa de pedido genuíno = reply explicando + estorno (campo recusa_silenciosa).

### Promoções
- 1ª: "Em Busca da Verdade" (16–22/08/2026, prêmios em μBTC, pagos 31/08). Encerrada;
  vencedores na landing (placar congelado via ledger). Infra reutilizável:
  inscrição "@veracibot quero participar" (exige selo azul + seguir), reset p/ 1000
  no início, apuração/anúncio automáticos (`PROMO_*` no .env).
- 2ª planejada: **Ciclo de Debates** — usar abertura formal como padrão.

## Site (rotas principais)

`/` landing (pt; `/en`), `/ranking` (abas geral/promoção quando ativa), `/casos`,
`/caso/{id}` (autos completos), `/advogados` (diretório por UF), `/registro`,
`/login`, `/painel` (escritório), `/admin` (aprovar escritórios), `/admin/stats`
(gráficos Chart.js: casos/dia, chamadas, adesões), `/problemas` (report → issue no
GitHub via `GITHUB_TOKEN`). Admin criado no boot via `ADMIN_EMAIL/ADMIN_PASSWORD`.

## Infra e deploy

- **Produção**: Windows Server, repo em `C:\Git\veracibot`. Três serviços NSSM:
  `veracibot-bot`, `veracibot-web` (uvicorn :8000), `veracibot-caddy` (HTTPS
  veraci.bot). Guia: `deploy/windows/DEPLOY_WINDOWS.md`.
- **Fluxo**: desenvolver no Mac/PC → commit/push (GitHub ralbuque/veracibot) →
  no servidor `git pull` + `nssm restart veracibot-bot` (e/ou `-web`). NUNCA editar
  no servidor (se travar merge: `git merge --abort && git reset --hard origin/main`).
- Logs: `C:\Git\veracibot\logs\{bot,web,caddy}.log`
  (`Get-Content ... -Tail 50 -Wait`). Mac dev: `./bin/veracibotctl start|stop|logs [bot|web]`.
- **Alertas por e-mail** (SMTP_*, 1×/dia por tipo): créditos Anthropic esgotados
  (pausa e reprocessa menções após recarga), conta do X bloqueada (replies vão pro
  outbox), cota X API 429.
- **Outbox**: escritas que falham (conta bloqueada/429) ficam na fila e reenviam
  (5/ciclo, 6s entre elas). `scripts/requeue_replies.py` reenfileira vereditos perdidos.

## Variáveis do .env (segredos NUNCA no git; .env.example é o modelo)

X_BEARER_TOKEN, X_API_KEY/SECRET, X_ACCESS_TOKEN/SECRET (OAuth1, app Read+Write —
regenerar tokens APÓS mudar permissão) · ANTHROPIC_API_KEY, ANTHROPIC_MODEL ·
BOT_HANDLE=veracibot · POLL_INTERVAL_SECONDS=120 · DB_PATH · POST_REPLIES ·
MAX_THREAD_TWEETS=50 · JUDGE_WEB_SEARCH · INVITE_ONLY · MAX_REPLY_LEN=4000 (conta
Premium posta longo) · APPEAL_QUORUM · COMPOSITION_ENABLED=false · SITE_URL ·
WEB_PORT=8000 · WEB_SECRET · ADMIN_EMAIL/PASSWORD · GITHUB_TOKEN/REPO ·
SMTP_HOST/PORT/USER/PASSWORD (senha de app do Gmail), ALERT_EMAIL ·
PROMO_ENABLED/START/END.

## Lições aprendidas (X API e operação — não repetir!)

1. **Notas longas**: `text` vem truncado; pedir `note_tweet` (bug que inverteu
   argumentos num caso real — o juiz leu "Não existe [cortado]").
2. **280 ponderado**: emoji vale 2, URL 23 — usar `reply.x_len`, nunca `len()`
   (403 "not permitted" enganoso quando estoura).
3. **since_id > 7 dias** na busca recente → 400 (auto-reset implementado).
4. **Conta bloqueada** ("temporarily locked") por rajada de replies → throttle 6s
   + outbox; desbloqueio manual em x.com.
5. **Threads virais**: priorizar tweets das partes no corte de 50; fio principal ≠
   ramos; posições em vídeo/imagem inacessível NÃO podem ser reconstruídas por
   inferência (juiz declara limitação/recusa).
6. Enquetes nativas não restringem votante; PowerShell come aspas em `-c` (usar
   here-string `@'...'@ | Set-Content x.py`); `.ps1` com acento precisa ser ASCII;
   API v2 não posta >280 sem conta Premium (fallback curto automático);
   quote-tweets exigem seguir `referenced_tweets` (a afirmação mora fora da thread).

## Pendências / próximos passos

- **Mudança 3 (pendente)**: validade de pontos ~1 ano — saldo derivado do ledger em
  janela móvel (Ricardo ainda define a regra).
- **Mudança 4 (pendente)**: ranking de fontes confiáveis (só checagens RECEBIDAS;
  métrica provável: taxa de acerto com volume mínimo; exigirá etiquetar papel
  autor×chamador no ledger). Ricardo estuda mecanismo com dados da conta.
- Promoção Ciclo de Debates (regras/prêmios a definir).
- Ideias faladas: `scripts/replay.py` (rejulgar threads arquivadas p/ testar prompt),
  formalizar `scripts/dump_caso.py` (perícia de casos).

## Convenções de trabalho

- Testes: `python3 -m py_compile src/veracibot/*.py src/veracibot/web/*.py` +
  testes inline com `Store(':memory:')` e stubs de tweepy/anthropic quando preciso.
- Toda mudança de regra atualiza: código + README.md + site (i18n pt E en).
- Contas de teste: @testonildo01/02/03 (testar com elas antes de caso real).
- Rejulgar um caso: reverter deltas do ledger da conversa, apagar cases/
  compositions/ledger da conversa, apagar o reply do bot no X, nova menção na thread.
- Estilo: módulos pequenos, pt-BR nos comentários/mensagens, defesas em camadas
  (código > prompt), flags no .env em vez de deletar features.
