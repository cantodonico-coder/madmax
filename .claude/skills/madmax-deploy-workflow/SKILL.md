---
name: madmax-deploy-workflow
description: Fluxo completo de edição, teste e deploy do jogo HTML5 "ARENA MAD" / "ARENA MADMAX" (arquivo único index.html em C:\Users\user\Desktop\CURSO CLAUDE AI\madmax, publicado em https://cantodonico-coder.github.io/madmax/ via GitHub Pages). USE SEMPRE que o pedido envolver mexer nesse jogo — layout de HUD, armas, inimigos, música, áudio, bugs, deploy, "testar no celular", "publicar", "subir pro github" — mesmo que o usuário não peça explicitamente pra seguir um processo. Também use quando o usuário pedir só pra "ver se funcionou" ou "testar o site" depois de uma mudança nesse projeto.
---

# Fluxo de trabalho — ARENA MAD / ARENA MADMAX

Jogo arcade top-down de sobrevivência em canvas 2D, **arquivo único** (`index.html`,
CSS + JS tudo inline, sem build step). Deploy é só `git push` pro branch `main` —
o GitHub Pages serve o repo direto. O usuário (apelido Nico) testa no celular via
link do WhatsApp, então "funcionar" pra ele quer dizer "funcionar no site publicado",
não só localmente.

## Por que seguir este processo

Sem essas verificações, é fácil quebrar o jogo de um jeito que só aparece no celular
do usuário — e como ele não manda vídeo/console log facilmente, cada bug não pego
antes do deploy custa uma rodada inteira de ida-e-volta. As etapas abaixo existem
porque cada uma já pegou um problema real nesta base de código.

## O loop, passo a passo

1. **Editar `index.html`** diretamente (CSS num `<style>`, JS num `<script>` só,
   tudo dentro de uma IIFE `(function(){ 'use strict'; ... })();`). Não existe
   bundler — o que está no arquivo é o que vai pro navegador.

2. **Checar sintaxe do JS extraído** antes de considerar qualquer mudança pronta:
   ```bash
   node -e "
   const fs = require('fs');
   const html = fs.readFileSync('index.html','utf8');
   const matches = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)];
   fs.writeFileSync('<scratchpad>/extracted_script_check.js', matches.map(m=>m[1]).join('\n;\n'));
   "
   node --check "<scratchpad>/extracted_script_check.js" && echo SYNTAX_OK
   ```
   Troque `<scratchpad>` pelo diretório de scratchpad da sessão atual — nunca
   `/tmp` (no Windows isso falha com EPERM; o scratchpad é o lugar certo pra
   arquivos temporários desta sessão).

3. **Testar com o skill `browser-automation`**, primeiro local
   (`file:///.../madmax/index.html`), depois — depois do deploy — no site ao
   vivo. Salve os scripts de teste (`.mjs`) no diretório de scratchpad, nunca
   em `/tmp`. Pontos a verificar:
   - Zero erros de console / requisições falhas.
   - Em mudança de **layout de HUD** (minimapa, botão de dash, placar, aba de
     áudio etc.): pegue `getBoundingClientRect()` de cada elemento envolvido
     e confira que não há sobreposição entre eles — não confie só em olhar a
     screenshot, calcule.
   - Tire um screenshot mesmo assim pra conferência visual — ajuda a pegar
     coisas que a checagem de retângulos não cobre (cor, legibilidade, clipping).
   - O hook de debug `window.__mmTest = {...}` (setado como última linha da
     IIFE) é **conhecido por ser instável** neste projeto específico — às
     vezes fica `undefined` mesmo logo depois de setado, por razão nunca
     totalmente diagnosticada. Se ele falhar, não insista tentando de novo:
     prefira reimplementar a lógica pura em Node (quando é uma função
     determinística, tipo rotação de música ou cálculo de ângulo) ou apoie-se
     em `waitForSelector`/screenshot. Sempre remova o hook do arquivo antes
     de commitar — ele é só pra teste.

4. **Atualizar `CLAUDE.md`** (documentação viva do projeto, na raiz de
   `madmax/`) descrevendo a mudança e o porquê — não só o quê. Esse arquivo é
   o que dá contexto pra próxima sessão sem precisar reler todo o histórico.

5. **Commit e push pro `main`**:
   ```bash
   git add <arquivos>
   git commit -m "..."
   git push origin main
   ```
   Sem `--no-verify`, sem `--force` a menos que o usuário peça explicitamente.
   Mantenha `assets/` e `audio/` só com arquivos de fato referenciados pelo
   `index.html` — binário grande (imagem, áudio) **nunca** deve virar base64
   inline no HTML (já causou um arquivo de 8MB+ com 20s+ de carregamento);
   sempre arquivo externo em `assets/`/`audio/`, referenciado por caminho
   relativo. Se sobrar arquivo não usado, mova pra `material_bruto/`
   (gitignored) em vez de apagar — comparando por hash de conteúdo (MD5), não
   por nome de arquivo, já que arquivos de origem costumam ser renomeados ao
   entrar em `assets/`.

6. **Reverificar o site ao vivo com cache-busting**
   (`https://cantodonico-coder.github.io/madmax/?v=<timestamp>`). O GitHub
   Pages/CDN normalmente demora **1–2 minutos** pra propagar depois do push —
   se a primeira checagem trouxer o HTML/layout antigo, isso não é falha do
   deploy, é só cache. Agende um retry (`ScheduleWakeup` ou similar) em vez de
   concluir a tarefa como feita; só reporte sucesso pro usuário depois de
   confirmar que o site ao vivo reflete a mudança.

## Comunicação com o usuário

Nico escreve em português brasileiro, às vezes com mensagens densas cobrindo
vários pedidos diferentes de uma vez (e com typos ocasionais). É importante
não deixar nenhum item passar batido — ao processar uma mensagem longa, vale
listar mentalmente cada pedido separado antes de começar a implementar, e
confirmar no final que todos foram endereçados. Se um pedido for ambíguo,
prefira perguntar (`AskUserQuestion`) a assumir — ele já pediu desculpas uma
vez por mandar prompts longos demais, mas o problema real era eu perder itens,
não ele escrever demais.
