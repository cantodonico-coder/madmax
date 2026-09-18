# ARENA MAD — contexto do projeto

## O que é

Protótipo de jogo arcade 2D top-down para **mobile** (portrait, controle por toque),
gênero "arena survival" tipo *bullet-heaven*: a nave do jogador atira
automaticamente com **todas as armas equipadas ao mesmo tempo**, o jogador só
controla o movimento (foco em desviar), sobrevive a ondas crescentes de
inimigos por 5 minutos até o chefe aparecer, e acumula progresso permanente
entre partidas. Nada no cenário é indestrutível — destroços/asteroides morrem
pra qualquer tiro, igual um inimigo.

Estética: sucata/pós-apocalíptico com neon (rosa `#ff2e97`, verde-limão `#c6ff1e`,
ciano `#00e5ff`), sprites reais recortados com fundo removido ("removebg"),
tema Mad Max espacial.

**Objetivo de design:** ser "viciante" — arsenal que só cresce (nunca troca,
nunca acaba munição), progresso permanente que sobrevive à morte, feedback
visual/sonoro forte a cada acerto, sensação de pilotagem/manobra real.

## Stack e arquitetura

- **Sem build step.** Um único `index.html` com HTML+CSS+JS inline (IIFE), renderizado
  em `<canvas>` 2D puro. Sem frameworks, sem dependências externas.
- Áudio e sprites/ícones grandes ficam em **arquivos externos** (não em base64
  inline) — ver [`audio/`](audio/) e [`assets/`](assets/). Isso é proposital:
  embutir binários grandes como base64 já causou travamentos de vários segundos
  no carregamento (decodificação síncrona via `atob`). Novos assets grandes
  **sempre** devem ir pra arquivo externo, nunca base64 inline.
- Os sprites dos 9 capangas/chefe *originais* (zumbi, boca, chefe, ferraoNegro,
  serraDisco, barrilTank, fortaleza, espinhoFerro, aranha, flecha) continuam
  embutidos em base64 dentro do `index.html` — não foram migrados pra arquivo
  externo (funciona, só não é consistente com o resto; baixo risco mexer nisso
  sem necessidade).
- `index.html` é o nome usado de propósito porque o deploy é via **GitHub Pages**
  (sobe a pasta `madmax/` inteira; o link final aponta pro `index.html` na raiz).
- Compartilhamento: link único enviado por WhatsApp — por isso a page precisa
  carregar rápido mesmo em conexão de dados móvel.

## Estrutura de arquivos

```
madmax/
├── index.html            # jogo inteiro (HTML+CSS+JS), ~2.4MB — SÓ ISSO PRECISA IR PRO DEPLOY
├── audio/                 # 6 trilhas .ogg (opus) EM USO, carregadas via <audio src>
│   └── title.ogg, boss.ogg, phase1.ogg..phase4.ogg
├── assets/                # imagens EM USO, todas referenciadas em index.html
│   ├── hero_ship.png            # nave do jogador, tira 8 direções (ver seção "Nave do jogador")
│   ├── space_bg.jpg              # textura de fundo repetível (tileable), pixel-art
│   ├── planet.png                 # planeta decorativo fixo no mundo (fundo removido)
│   ├── supportStation.png         # estação de apoio — reenergiza a nave (fundo removido)
│   ├── debRock1..debRock8.png     # asteroides/destroços novos (variedade, fundo removido)
│   ├── debDish.png, debScrapPile.png, debBarrel.png, debSolarPanel.png,
│   │   debCables.png              # sucata solta nova — destroços pequenos (fundo removido)
│   ├── chest.png                  # baú da cápsula de suprimento (fundo removido)
│   ├── explosion_sheet.png        # folha 6×1 — explosão padrão (chefe, fallback)
│   ├── explosion_sheet_obstacle.png # folha 3×2 — explosão de destroços/meteoros
│   ├── logo_intro.jpg, logo_gameover.jpg  # logos das telas de início/fim
│   ├── cacadora_ferrugem.png, garra_voraz.png, lanca_espectral.png,
│   │   vigia_storm.png, chamine_dupla.png, tanque_canhao.png    # 6 capangas novos
│   ├── icon_pea.png, icon_leque.png, icon_metralha.png, icon_missil.png,
│   │   icon_laser.png, icon_granada.png, icon_rkl88.png, icon_tnt.png  # ícones das 8 armas (fundo removido)
│   └── icon_shop_hp.jpg, icon_shop_speed.jpg, icon_shop_dash.jpg  # ícones da loja de upgrades permanentes
└── material_bruto/        # TUDO que não é referenciado pelo jogo — não precisa subir
    ├── (dezenas de imagens-fonte soltas, de onde os assets acima foram
    │   recortados/copiados; material disponível pra futuras iterações.
    │   Duplicatas exatas de arquivos já em assets/ são removidas quando
    │   encontradas — ver nota de limpeza abaixo)
    └── audio_extras/      # faixas extras que chegaram mas ainda não foram
        usadas nem descartadas (1.opus..4.opus, "arena 07"/"arena n9" em
        mp3 e opus) — decisão pendente do usuário
```

**Ao adicionar um asset novo:** copiar pra dentro de `assets/` ou `audio/` (nunca
deixar solto na raiz). Ao remover o uso de um asset, mover ele pra
`material_bruto/` (ou a subpasta correspondente) em vez de apagar — o
`index.html` e a `CLAUDE.md` são a fonte da verdade de "usado vs não usado";
pra conferir, `grep -oE "(assets|audio)/[A-Za-z0-9_. -]+\.(png|jpg|jpeg|ogg|webp|gif)" index.html`
lista tudo que está realmente referenciado.

**Limpeza de duplicatas em `material_bruto/`:** o usuário pediu explicitamente
pra apagar (não só mover) arquivos que sejam cópia exata de algo já em
`assets/` — comparar por **hash de conteúdo**, não por nome (arquivos-fonte
viram assets com nome diferente). Script de referência:
```js
const crypto=require('crypto'), fs=require('fs'), path=require('path');
const hash = p => crypto.createHash('md5').update(fs.readFileSync(p)).digest('hex');
const assetsHashes = new Map(fs.readdirSync('assets').map(f=>[hash('assets/'+f),f]));
// percorrer material_bruto/ recursivamente, comparar hash(arquivo) contra assetsHashes
```
Arquivos que NÃO têm hash correspondente em `assets/` são material genuinamente
não usado (não duplicado) — não apagar sem perguntar, só o usuário decide
descartar aquele conteúdo de vez (ex: `audio_extras/` ainda pendente).

## Sistemas de jogo (estado atual, com as melhorias já incorporadas)

### Loop principal
`ONDA` cronometrada (`waveTime`) → 4 fases de dificuldade (0–75s, 75–150s,
150–225s, 225–300s) → aos 5:00 o chefe ("MECHA DE SUCATA") aparece → ao
derrotá-lo, o ciclo recomeça com um respiro. Morrer termina a partida
(`endGame()`), mas **sucata total, armas desbloqueadas, loadout e melhorias
permanentes sobrevivem à morte** (salvas em `localStorage`, chave
`arenamad_save_v1`). O HUD mostra "⏱ CHEFE EM MM:SS" contando regressivo
(esconde quando ele aparece) — existe pra deixar claro que o chefe é por
**tempo** (5min), não por número de onda.

**`waveTime` vs `realElapsedTime`:** são duas variáveis DIFERENTES de propósito.
`waveTime` é o cronômetro interno usado pra fase/chefe — ele é **rebobinado**
de propósito ao derrotar o chefe (`waveTime = CFG.bossAt-40`, dá um respiro de
40s antes do próximo ciclo). `realElapsedTime` só cresce, nunca rebobina, e é
usado exclusivamente pro número de "ONDA" exibido e pro banner de troca de
onda (`#waveRevealBanner`/`showWaveReveal()`, dispara a cada 25s de
`realElapsedTime`) — **nunca** usar `waveTime` pra isso, ou o número da onda
"anda pra trás" visualmente depois de vencer o chefe (bug já corrigido uma vez).

**Duração mínima do combate do chefe:** como as armas agora acumulam e sobem
de nível sem limite, um build forte podia matar o chefe (2200 HP fixos) quase
instantaneamente. `CFG.bossMinFightTime` (30s) garante um piso de HP que
decai linearmente até zero ao longo desse tempo (`boss.hp = Math.max(boss.hp,
hpFloor)` em `update(dt)`, bloco `// chefe`) — a barra ainda drena visualmente
mesmo se o dano real já teria matado, só não deixa morrer antes do prazo.
Builds fracos não são afetados (o piso só importa quando o dano real já
derrubaria o chefe mais rápido que isso).

**Pulso de choque de curto alcance** (`bossPulseRadius`=150, `bossPulseDmg`=14,
`bossPulseNearThreshold`=1.3s): antes só existia a barragem à distância —
ficar colado no chefe pra bater era seguro, sem nenhum risco. A primeira
versão do pulso usava um timer cego (disparava a cada 2.4s se a nave
estivesse no raio naquele instante) — **isso fechou a brecha por completo**:
até quem tava desviando corretamente da barragem à distância também tomava
pulso sem ter feito nada de errado, o chefe virou impossível de "respirar".
Trocado por **tempo de proximidade sustentada**: `boss.nearT` só acumula
enquanto a nave está dentro do raio (zera assim que sai); o pulso só dispara
se `nearT` passar de `bossPulseNearThreshold` (1.3s), com `pulseLockT` (0.5s)
travando reativação imediata. Isso deixa hit-and-run (entrar, bater, sair)
seguro — só pune ficar **parado/camping** colado no chefe. Telegraph (anel
branco tracejado) só aparece quando a nave está de fato dentro do raio e
`nearT` já passou de `bossPulseNearThreshold - CFG.telegraphTime` — ou seja,
só avisa quem já está sob risco real, não todo mundo o tempo todo.

### Nave do jogador
- Movimento: joystick virtual/teclado, velocidade `CFG.shipSpeed` (modificada
  por `ship.upgrades.speedMul`).
- **Giro por taxa limitada, quase instantâneo**: o ângulo da nave persegue o
  ângulo do movimento a `CFG.shipTurnRate` (40 rad/s, ~0,08s pra virar 90°) em
  vez de saltar direto pro ângulo alvo — dá sensação de manobra/curva visível
  sem parecer que ela anda de lado/de costas. Já foi 13 rad/s, subiu porque
  ficava perceptível demais em jogo real com mudanças de direção frequentes.
  Ver bloco `// gira até o ângulo alvo` em `update(dt)`.
- **Hitbox justa**: o sprite visual usa `CFG.shipRadius` (32), mas o raio que
  realmente conta pra levar dano de capanga/bala é `CFG.shipHitRadius` (10,
  ~30% do visual) — dá a sensação de "escapei por pouco" mesmo quando parece
  que encostou. Usado nos dois checks de dano por contato com inimigo e por
  bala inimiga/chefe; a física de empurrão contra destroços continua usando
  o raio visual (evita atravessar cenário visualmente). `shipRadius` já foi
  41 — reduzido porque em tela de celular (mais estreita) a nave ficava
  desproporcionalmente grande.
- **Cura por moeda**: cada sucata/moeda coletada cura `CFG.coinHeal` (2 hp) na
  hora — não existe regeneração passiva por tempo nem compra de vida na loja.
  Barra de vida pisca verde no momento da cura (`ship.regenFlashT`).
- Sprite real: nave-disco (`assets/hero_ship.png`), tira horizontal **8
  direções** (`SPRITES.heroShip = {frames:8}`) — usa o `drawDirSprite()`
  genérico, igual todos os outros sprites (capangas/chefe), sem nenhum hack
  especial. **Isso substitui a "Junkyard Titan" antiga** (folha de 50 quadros
  cobrindo só 180°, que renderizava como um blob irreconhecível — o
  `pickHeroFrame`/`HERO_NOSE_ANGLES`/espelhamento que existiam só por causa
  disso foram removidos). Gerada a partir de `material_bruto/
  ref_hero_ship_rotation_8dir.jpg` (8 poses em fundo preto puro): recorte por
  frame + chroma-key (qualquer pixel com `max(r,g,b)<6` vira transparente,
  com rampa suave até 30 pra não deixar franja) + crop vertical uniforme
  (mesmo recorte em todos os 8 frames, pra não desalinhar o pivô de rotação
  entre eles) — script de referência em `material_bruto/` não guardado, mas
  reproduzível: `PIL.ImageChops.lighter(lighter(r,g),b)` pra achar o canal
  máximo, `.point()` pra rampa de alpha. Sem sprite carregado, cai num
  fallback pixel-art gerado proceduralmente.

### Habilidade: dash com i-frames
Botão de toque no canto inferior direito (espelha o joystick), também aciona
com barra de espaço. Desloca a nave ~150px na direção do movimento atual,
0,3s de invulnerabilidade (nave pisca), cooldown de 5s (reduzível pela loja,
`dashCooldown()`) com indicador visual de recarga no próprio botão.

### Armas — sistema **acumulativo**, não mais de troca (`WEAPONS`)
Reformulado: antes, pegar uma cápsula TROCAVA a arma ativa com munição finita.
Agora **todas as armas equipadas disparam ao mesmo tempo, pra sempre, sem
acabar munição**:
- 8 armas no total: 3 básicas sempre equipadas (pea "CANHÃO SUCATA", leque
  "LEQUE TRIPLO", metralha "METRALHA") + 5 compráveis com sucata (missil,
  laser, granada, rkl88 "RKL-88 PESADO", tnt "TNT DEMOLIDOR").
- **Loadout**: na aba ARSENAL da loja, cada arma comprada tem um botão
  EQUIPAR/✔ EQUIPADA que alterna se ela entra na partida (`meta.loadout`,
  persistido). Comprar uma arma já equipa ela automaticamente.
- `ship.weapons = { [armaId]: {level, cd} }` — montado em `resetGame()` a
  partir de `['pea','leque','metralha', ...meta.loadout]`, todas começando
  no nível 1.
- **Nível por arma** (`ship.weapons[id].level`, até `CFG.maxWeaponLevel`=10):
  cápsulas de suprimento no mapa não trocam mais a arma — elas SOBEM DE NÍVEL
  a arma correspondente (`weaponLevelBonus()`), que aumenta dano (+15%/nível)
  e, pra armas de múltiplos projéteis (straight/spread/homing), soma mais
  tiros simultâneos por disparo (até +9). Mostra um texto flutuante
  "NOME DA ARMA NV.N" ao subir de nível.
- Disparo: em `update(dt)`, cada arma equipada tem seu próprio cooldown
  (`wState.cd`) e dispara independente das outras quando o alvo está no
  alcance — não existe mais um `ship.fireCd`/`ship.weaponId` único.
- **Destroços do cenário são destrutíveis por qualquer arma** — mesma lógica
  de dano de inimigo, incluindo o feixe do laser (pierce) e o raio de explosão
  de granada/rkl88/tnt. "Nada fica na frente."
- HUD (`#weaponHud`) mostra um "chip" pequeno (ícone + nível) pra cada arma
  equipada com nível>0, gerado por `updateWeaponHud()`.
- Todas as 8 armas têm ícone (`WEAPON_ICON_BY_ID` → `assets/icon_*.png`, fundo
  removido, usados também na cápsula e no chip do HUD).
- **Mira própria por arma** (`WEAPONS[id].aimMode`, resolvido em
  `computeWeaponAim()`) — cada arma tem uma "assinatura visual" diferente em
  vez de todas mirarem no inimigo mais próximo:
  - `rank` — pea (aimRank 0, o mais próximo), leque (1, 2º mais próximo),
    metralha (2), laser (3), rkl88 (4) — `nthNearestEnemyOrBoss(x,y,rank)`
    ordena por distância e pega a posição N da fila. **Espalha o dano entre
    vários capangas** em vez de todas as armas baterem no mesmo alvo — isso
    substituiu uma versão anterior onde metralha/leque miravam por direção de
    movimento (`moving`/`retreat`), trocado a pedido do usuário porque não
    lia como "focar em inimigos". Se sobrar menos capangas na tela que ranks
    pedidos, cai pro último disponível (não erra/trava).
  - `random` — granada, tnt: atira num ponto aleatório dentro do alcance
    (esses dois mantêm a regra de área, não fazem parte do "rank").
  - `heavy` — míssil: persegue o capanga vivo de maior HP do Grupo C
    (`heaviestGroupCEnemy()`), cai pro inimigo mais próximo se não houver
    nenhum Grupo C na tela.
  - `moving`/`retreat` (atira na direção que anda / oposta) ainda existem no
    código (`computeWeaponAim()`, `dirAimPoint()`) mas nenhuma arma usa mais —
    ficaram disponíveis caso queira uma arma nova com esse comportamento.
  - Modos `random` sempre tem mira válida (não depende de ter inimigo por
    perto); `rank`/`heavy` só disparam com alvo no alcance.
- **Rajada (metralha)**: em vez de cadência constante, acumula por
  `WEAPONS.metralha.burst.chargeTime` (1,5s de silêncio) e solta
  `burst.shots` tiros (10) com `burst.shotInterval` (0,045s) entre eles —
  contraste "silêncio→caos". Estado por arma em `wState.burstShotsLeft`/
  `burstCd` (inicializado junto com `level`/`cd` sempre que uma arma é
  criada — em `resetGame()` e na coleta de cápsula). Generalizável pra
  outras armas só adicionando `burst:{...}` na config dela.
- **Evolução no nível máximo — Laser Orbital**: ao chegar em
  `CFG.maxWeaponLevel` (10), o laser para de disparar em linha reta com
  cooldown e vira um anel de dano contínuo 360° ao redor da nave (raio 95,
  gira via `ship.laserOrbitAngle`), aplicado direto no loop de `update(dt)`
  antes do disparo normal (`if(id==='laser' && wState.level>=CFG.maxWeaponLevel)`).
  É só a implementação de referência dessa mecânica — as outras 7 armas ainda
  não têm uma evolução própria no Nv.10, só ficam mais fortes numericamente.
  Se for generalizar, o padrão é: checar `wState.level>=CFG.maxWeaponLevel`
  no topo do loop de disparo, tratar o caso especial, e `continue` pra pular
  o disparo padrão daquela arma no frame.

### Explosões animadas
Duas folhas registradas em `EXPLOSION_SHEETS` (cada uma com seu próprio
layout de grade — `cols`/`rows` — porque não são todas tiras horizontais):
- `default` → `assets/explosion_sheet.png`, tira 6×1. Usada na morte do chefe
  (tamanho maior) e como fallback.
- `obstacle` → `assets/explosion_sheet_obstacle.png`, grade 3×2 (nuvem de
  fumaça com estilhaços de chip). Usada especificamente quando um destroço/
  meteoro do cenário é destruído (qualquer arma ou capanga quebrador).
`spawnExplosionFx(x,y,size,sheetKey)` cria uma entrada em `explosionFx[]`
(default se `sheetKey` for omitido), desenhada em `draw()` avançando de
quadro conforme o tempo. Não é chamado pra cada inimigo comum morrendo
(ficaria poluído visualmente com o volume de inimigos).

### Inimigos — 15 tipos em 3 grupos (`ENEMY_TYPES`, `t1`–`t15`)
- **Grupo A** (rápidos, corpo a corpo): t1 zumbi, t2 flecha, t3 aranha, t10
  cacadoraFerrugem, t11 garraVoraz.
- **Grupo B** (à distância, hp médio): t4 boca, t5 ferraoNegro, t6 serraDisco,
  t12 lancaEspectral, t13 vigiaStorm, t14 chamineDupla.
- **Grupo C** (pesados, quebram destroços): t7 barrilTank, t8 fortaleza, t9
  espinhoFerro, t15 tanqueCanhao.
- Cada fase de onda (`WAVE_PHASES`) libera gradualmente mais tipos no pool de
  spawn — os 6 tipos novos (t10–t15) entram progressivamente a partir da fase 2.
- Inimigos à distância e o chefe mostram um **telegraph** (anel pulsante na
  cor do ataque, ~0,35s antes de atirar, `CFG.telegraphTime`) — dá tempo de
  reagir em vez do tiro parecer instantâneo/injusto.
- Cor de cada tipo é escolhida pra não colidir com a cor das moedas
  (`#ffd23f`) — capangas à distância nunca devem usar essa cor pro próprio
  tiro, senão fica indistinguível de moeda no chão (t4 já foi corrigido pra
  `#ff5fa3`; ao adicionar um tipo `ranged` novo, checar isso).

### Planeta — marco fixo no mundo
`PLANET_POS = {x:2800, y:-2000}`, `PLANET_DIAM = 640` — imagem única
(`assets/planet.png`, fundo removido), desenhada em `draw()` via
`worldToScreen()` como qualquer objeto de mundo (não é tileable como o
fundo, é um ponto fixo). Sem colisão, só decorativo — jogador se afasta,
volta e pode circular ao redor. Aparece no minimapa (`drawMiniMap()`) como
um ponto ciano-claro quando dentro do raio visível. Distância do spawn
(~3440px, ship a 190px/s) dá uns 15-18s de voo pra alcançar.

### Fundo espacial — textura pixel-art
`assets/space_bg.jpg` já passou por duas trocas: primeiro uma textura
"pintada"/fotorrealista (nebulosa roxa/azul), depois substituída de novo pela
atual, **pixel-art** (mesma linguagem visual dos sprites do jogo, ladrilha
sem emenda — testado em mosaico 2×2 antes de integrar). A primeira troca
também teve um problema à parte: era tão parecida em brilho/contraste com a
textura anterior que **no jogo real parecia que nada tinha mudado** — se for
trocar de novo, comparar lado a lado em brilho/contraste real, não só na
paleta de cor. Versões anteriores preservadas em `material_bruto/
space_bg_old.jpg` (a original) e `material_bruto/space_bg_painterly.jpg` (a
intermediária).

### Estações de apoio — pontos de respiro
Diferente do planeta (marco único fixo), as estações (`supportStations[]`)
**vão surgindo periodicamente pelo mundo**, uma de cada vez, a uma distância
aleatória da nave (`CFG.stationSpawnMin/MaxDist`, timer
`CFG.stationSpawnMin/Max`=35–55s) — no máximo `CFG.stationMaxCount` (4)
simultâneas, a mais antiga some quando esse limite é passado. Sprite:
`assets/supportStation.png` (base azul/tech, contraste proposital com a
estética enferrujada do resto — sinaliza "isso é diferente, isso é seguro").
Dentro de `CFG.stationHealRadius` (110), cura `CFG.stationHealRate` (14
hp/s) enquanto a nave não estiver com vida cheia; ao chegar no máximo, a
estação **entra em cooldown** (`CFG.stationCooldown`=40s, anel/sprite ficam
foscos) — é um respiro tático, não um esconderijo permanente. Anel pulsante
verde (`#7cff00`) ao redor mostra o raio quando pronta; texto flutuante "⚡
REENERGIZANDO" aparece enquanto cura. Ponto verde no minimapa (cinza quando
em cooldown), mesmo padrão do planeta (ciano).

### Destroços do cenário
Campo infinito procedural determinístico (`genChunk`, seed por chunk),
`DEBRIS_KEYS` inclui os 7 destroços originais (carro, ônibus, container...)
mais 13 novos (`debRock1..8` asteroides, `debDish`/`debScrapPile`/`debBarrel`/
`debSolarPanel`/`debCables` sucata solta) — todos com fundo removido, escala
individual em `DEBRIS_SCALE`. São destrutíveis por **qualquer arma do
jogador** (não só capangas do grupo C), mas **bloqueiam tiro inimigo** (bala
de capanga à distância ou barragem do chefe soma no destroço sem quebrá-lo —
cobertura tática real, ver bloco `// balas de inimigo pesado / chefe` em
`update(dt)`). O estado de destruído **persiste** mesmo saindo e voltando
pra área (`destroyedObstacleKeys`, resetado só em nova partida).

### Progresso permanente — três camadas, todas com sucata (`meta`, localStorage)
1. **Arsenal** (`meta.unlocked`) — compra armas.
2. **Loadout** (`meta.loadout`) — escolhe quais das compradas entram equipadas
   na próxima partida (ver seção Armas acima).
3. **Loja de upgrades permanentes** (`meta.metaLevels`, `META_UPGRADES`, cada
   item com ícone próprio) — bônus fixos que valem pra sempre, aplicados no
   início de toda corrida (`applyMetaUpgrades()`):
   - CASCO REFORÇADO: +12 vida máx/nível, até 5 níveis.
   - MOTOR TURBO: +4% velocidade/nível, até 5 níveis.
   - RECARGA RÁPIDA: −0,6s no cooldown do dash/nível, até 3 níveis (mín. 3,2s).

### Upgrades **dentro da corrida** (somem só se a nave explodir)
Orbes flutuantes com 7 tipos em `UPGRADE_TYPES` (dmg, cadência, velocidade,
casco máx + 3 novos: ímã de sucata `magnetMul`, blindagem `dmgTakenMul`,
propulsor auxiliar `dashCdMul` — todos multiplicadores em `ship.upgrades`,
sistema separado do nível das armas). Pool de 7 existe de propósito: com só 4
tipos e 3 escolhas por vez, praticamente sempre apareciam as mesmas opções
(sensação de repetição); 7 dá variedade real. Ao coletar, o jogo **pausa e
mostra 3 opções aleatórias pra escolher** (`openUpgradeChoice`, estilo
roguelite, `pickUpgradeChoices(3)`).

### Feedback / juice
Flash branco no inimigo/chefe ao ser atingido, números de dano flutuantes,
som de acerto com variação de tom (`playHitTick`), variação de pitch em
explosões, explosões animadas (ver acima).

**Drop de sucata com valor variável** (`dropCoin()`): em vez de toda moeda
valer sempre 1, agora sorteia raridade a cada drop — 88% normal (valor 1,
visual/som padrão), 10% "média" (`CFG.coinMedChance`/`coinMedValue`, um
pouco maior), 2% "grande" (`CFG.coinBigChance`/`coinBigValue`=8, cor ciano
`#00e5ff` em vez do amarelo padrão, raio maior, `playCoinBig()` — arpejo de
3 notas em vez do de 2 — e texto flutuante "+N SUCATA!"). Isso é reforço de
razão variável (Skinner): recompensa de valor imprevisível engaja mais do
que a mesma recompensa fixa repetida. Tier fica salvo no próprio objeto da
moeda (`coin.tier`), lido tanto no desenho (`draw()`) quanto na coleta.

**Impacto de colisão** (`damageShip()`, gatilho compartilhado por bater em
destroço OU em capanga): agora sempre soca som (`playImpact()`, mais grave se
`amount>=18`), tremor (`shake()`) e hit-stop (`hitStop()`, 3-5ms) — antes só
tinha vibração do celular e tremor, sem som nem hit-stop nessa gatilho
específico. Tudo isso é gated por `ship.vibCd` (0,16s) pra não virar um
zumbido constante durante contato contínuo.

**Screen shake diferenciado por evento** (todos passam por `shake(mag,t)`):
dash = leve/rápido (`shake(3,0.1)` em `tryDash()`), morte de capanga comum =
médio (`shake(3,0.15)` via `enemyDeathFx()`, chamado nos 3 pontos onde um
inimigo morre), chefe aparecendo = forte (`shake(10,0.5)`). `enemyDeathFx()`
também dá hit-stop extra (0,04s) quando quem morreu é do Grupo C (pesado).

**Flash branco de "level up"**: ao abrir a tela de escolha de upgrade
(`openUpgradeChoice()`), dispara `levelUpFlashT` (flash branco full-screen,
decai em `frame()` — não em `update(dt)`, porque `update()` fica pausado
durante a escolha) + explosão de partículas brancas na posição da nave. Os
inimigos já ficam "congelados" de graça nesse momento, porque `update()`
inteiro é pulado enquanto `choosingUpgrade===true` (só `draw()` continua).

**Baú em vez de losango**: os orbes de upgrade permanente (dentro da corrida)
agora desenham `WEAPON_ICON_IMG.chest` (mesma imagem da cápsula de arma) em
vez do losango vetorial antigo, com fallback pro losango se a imagem não
carregar.

### Telas de UI
- **Título**: logo real (`assets/logo_intro.jpg`, `#titleLogo`) num cartão com
  borda neon-pink brilhante — o JPG tem fundo branco de propósito, tratado como
  um "pôster" em vez de tentar remover o fundo. Texto reduzido a uma linha +
  botão "❓ COMO JOGAR" que expande os detalhes (evita parede de texto). Loja e
  Arsenal são **abas** (`#shopTabs`, `.tabBtn`) em vez de empilhados — cabe na
  tela sem rolagem na maioria dos aparelhos.
- **Fim de jogo**: logo real (`assets/logo_gameover.jpg`, `.goLogo`) no mesmo
  estilo de cartão (fundo magenta do próprio JPG, combina com o pink neon do
  jogo). Estatísticas em cartões (`.goStatsGrid`/`.goStatRow`) em vez de bloco
  de texto corrido; banner "🏆 NOVO RECORDE" separado (`#goRecordBanner`), só
  aparece quando bate o recorde. Barra de progresso "SOBREVIVÊNCIA ATÉ O CHEFE"
  (`#goProgressBar`, `waveTime/CFG.bossAt`) — efeito "quase consegui"; se o
  jogador morreu sem o chefe aparecer (`!bossSpawned`) depois dos 4min
  (`waveTime>=240`), mostra uma mensagem motivacional extra (`#goMotivation`)
  incentivando tentar de novo.
- **Minimapa**: mostra coordenadas da nave (`#miniMapCoords`, "X:.. Y:..")
  no canto, atualizado em `drawMiniMap()`. Fica do lado **esquerdo**
  (`#miniMapWrap{left:10px;top:96px}`), entre a barra de CASCO/chips de arma
  (`#hpWrap`) em cima e o joystick MOVER (`#joyMock`) embaixo — posição
  escolhida pra não sobrepor nenhum dos dois.
- **Botão de DASH**: círculo pequeno (`#dashBtnWrap`, 39px = 50% do tamanho
  original) no canto superior direito, abaixo do placar (`#scoreWrap`) e da
  aba de áudio/pausa, em vez de grande e mais para baixo como antes.
- **Trilha sonora de gameplay em rotação sequencial**: `GAMEPLAY_TRACKS =
  ['phase1','phase2','phase3','phase4','phase5']` (5 faixas, a 5ª é
  `audio/phase5.opus`). `nextGameplayTrack()` avança um índice persistente
  (`gameplayTrackIdx`) em ciclo fechado (`(idx+1) % length`) — chamada toda
  vez que a fase muda, após derrotar o chefe, e no início da corrida. Isso
  garante que a mesma faixa nunca toca duas vezes seguidas nem repete a
  anterior, resolvendo o problema de repetição que o mapeamento fixo
  fase→faixa causava (principalmente depois de bosses, já que `waveTime`
  volta pra trás e caía sempre na mesma fase tardia). Substituiu a função
  antiga `musicKeyForPhase(idx)`.
- **Aba retrátil de áudio/pausa** (`#settingsWrap`): tab fixa no canto
  **superior** direito (acima do placar e do dash, que desceram pra abrir
  espaço), mostrando só o ícone de alto-falante quando fechada
  (`#settingsTabBtn`, 30×30px). Clicar abre/fecha (`.open`) um painel
  (`#settingsPanel`, transição de `width`, expande pra esquerda por cima do
  placar) com slider de volume mestre (`#volumeSlider`), botão MUDO
  (`#muteBtn`) e botão PAUSA (`#pauseBtn`). Volume e mudo persistem entre
  sessões em `localStorage` (`arenamad_audio_v1`, separado do save de
  progresso). `effectiveVolume()` = `muted ? 0 : masterVolume`, aplicado a
  dois sistemas de áudio diferentes: os SFX sintetizados via Web Audio API
  (todos ligados a um único `sfxMasterGain`, criado em `ensureAudio()`, em
  vez de cada oscilador conectar direto em `ac.destination`) e a música
  (`MUSIC_TRACKS`, volume do `<audio>` atual recalculado em
  `applyAudioSettings()`). Pausa (`setPaused()`) congela o loop inteiro —
  `update(dt)`, decaimento de `shakeT`/`hitStopT`/`levelUpFlashT` — e pausa
  o `<audio>` da faixa atual, mostrando overlay `#pauseOverlay` com
  "⏸ PAUSADO". `resetGame()` sempre reseta `paused=false` no início de cada
  corrida pra não começar travado.

## Convenções / decisões de projeto

- **Português** em toda UI, comentários de código e textos do jogo.
- Sprites de personagens/capangas usam `dirFrame()`/`drawDirSprite()` com
  frame 0 = norte, sentido horário — 12 direções pros capangas/chefe, 8 pra
  nave do jogador (nenhuma exceção mais precisa de hack especial, ver seção
  "Nave do jogador").
- **Barra de endereço do navegador mobile**: como a página trava scroll
  (`overflow:hidden`, `touch-action:none`) o navegador nunca recebe o gesto
  que normalmente esconde a barra sozinho. `nudgeAddressBar()` força um
  `window.scrollTo(0,1)` no load/orientationchange/primeiro toque como
  fallback leve. A solução principal é `requestGameFullscreen()`, chamada no
  clique do botão "ENTRAR NA ARENA" (gesto do usuário, exigido pela
  Fullscreen API) — pede `documentElement.requestFullscreen()` (com
  fallbacks `webkit`/`moz`/`ms`) o que remove a barra de verdade em
  navegadores que suportam a API (Chrome/Android, a maioria dos Android
  em geral). Em navegadores sem suporte total (ex.: Safari iOS fora de
  PWA instalado) cai de volta pro nudge de scroll — não tem solução 100%
  garantida multiplataforma sem virar PWA instalado
  (`display:standalone` no manifest elimina a barra de vez).
- Todo timer de gameplay (`ship.hitFlashT`, `dashCd`, `invulT`, etc.) é
  decrementado em `update(dt)` e ignorado quando `gameOver || choosingUpgrade`
  — pausar a escolha de upgrade pausa o jogo inteiro de propósito.
- Evitar reintroduzir binários grandes (>50KB) como base64 inline — usar
  arquivo externo em `assets/` ou `audio/` e referenciar por caminho relativo.
- Ao adicionar arma/inimigo/asset novo: checar contraste de cor contra
  elementos já existentes (moedas, outros tiros) antes de finalizar.
- Teste manual: `node --check` num JS extraído do `<script>` pra sintaxe, e o
  skill `browser-automation` (headless) pra carregar/jogar e conferir console
  sem erros antes de considerar uma mudança pronta. Esse arquivo é grande
  (2,4MB+) e a automação headless às vezes fica instável tentando expor
  variáveis de debug via `window.__algo` logo após um clique — quando isso
  acontecer, prefira aguardar um seletor/condição real (`waitForSelector`,
  `waitForFunction`) em vez de inserir um hook de debug temporário; se precisar
  do hook, teste, confirme e **remova antes de finalizar**. Pra testar coisas
  que dependem de sucata/loadout, é mais confiável semear `localStorage`
  (`arenamad_save_v1`) antes de `page.reload()` do que esperar o jogo gerar
  sucata organicamente.
- Assets de origem chegam na pasta com sufixo `-removebg-preview` quando têm
  fundo removido (melhor qualidade pra ícones/sprites) — ao integrar, copiar
  pra um nome limpo dentro de `assets/` em vez de referenciar o nome longo.

## Pendências conhecidas (não resolvidas ainda)

- **Naves extras não integradas**: o usuário mandou mais 3 folhas de
  referência com ~15 designs de naves no total (`material_bruto/
  ref_ship_variants_a.jpg`, `_b.jpg`, `_c.jpg` — battleship, buggy speeder,
  flying fortress, stealth raider, junk crab, e mais) — ainda não decidido
  se viram novos tipos de inimigo/chefe, ou ficam só como referência futura.
  Perguntar ao usuário antes de integrar (precisa de stats/balanceamento
  novos, não é só trocar sprite).
- `material_bruto/ref_earth_cracked_lore.jpg` (Terra rachada/explodindo,
  estilo Mad Max) — parece arte de lore/tela de título, não sprite de jogo;
  sem uso definido ainda.
- Existem dezenas de outras imagens em `material_bruto/` não usadas (telas de
  UI estilizadas, bosses alternativos, sprite de explosão nuclear
  alternativo, variantes de laser/minigun/bazuca, etc.) — disponíveis pra
  iterações futuras, sem uso definido ainda.
- "Informações de localização" no minimapa foi interpretado como coordenadas
  X/Y da nave — se o usuário queria outra coisa (ponto cardeal até o chefe,
  nomes de região, etc.), ajustar.
