# Especificações de assets — ARENA MAD (espacial / Mad Max no espaço)

Guia pra buscar/gerar imagens novas. Baseado no que já funciona hoje no jogo
(os capangas novos ficaram bons — a nave do jogador não, e o motivo está na
seção 1).

## Estilo geral (vale pra tudo)

- Tema: pós-apocalíptico Mad Max, mas no espaço — sucata, ferrugem, chapa
  remendada, solda, cabos expostos.
- Paleta de destaque (luzes, motores, brilhos — não pintar o objeto inteiro
  nessas cores): rosa `#ff2e97`, verde-limão `#c6ff1e`, ciano `#00e5ff`. Asset
  novo tem que "conversar" com essas 3 cores, não introduzir uma cor de
  destaque nova.
- Fundo **sempre removido** (transparente), exceto fundo de cenário/telas de
  UI (esses ficam sólidos de propósito).
- Formato: PNG com alpha (quando precisa transparência) ou JPEG qualidade
  ~82 (fundo full-bleed sem transparência).
- Peso: manter cada arquivo abaixo de ~200KB depois de comprimido.

## 1. Nave do jogador — PRIORIDADE (é o que está quebrado hoje)

O processo atual (folha de rotação contínua gerada por IA, tipo turntable)
já foi tentado duas vezes e saiu borrado/irreconhecível nas duas. Trocar pra
o mesmo formato que já funciona bem nos capangas:

- **12 quadros**, tira horizontal única, cada quadro **quadrado**, mínimo
  **200×200px** por quadro (capangas usam 144×144 e funcionam; a nave é o
  elemento mais importante da tela, pode ser maior).
- Fundo transparente.
- **Frame 0 = nariz apontando pra CIMA (norte)**, quadros seguintes giram em
  **sentido horário**, 30° entre cada um (frame 1 = 30°, frame 2 = 60° ...
  frame 11 = 330°) — 360° completos, sem precisar espelhar nada.
- Vista **top-down** (reto de cima, tipo mapa), não a vista heroica de baixo
  que `hero_ship_pose.png` tem hoje — essa mudança de ângulo de câmera é
  provavelmente o principal motivo de nenhuma tentativa ter ficado legível
  em jogo.
- Manter a identidade "Junkyard Titan" (chapas de sucata, motores brilhando
  verde-limão atrás) — pode usar `hero_ship_pose.png` como referência de
  estilo/detalhe, só mudando câmera pra top-down e girando em 12 posições.
- Em qualquer um dos 12 quadros, tem que dar pra saber na hora qual lado é a
  proa (frente) só de olhar aquele quadro sozinho — se não der, não serve.

## 2. Capangas / inimigos novos

Mesmo padrão que já funciona (`cacadora_ferrugem`, `garra_voraz`, etc.):

- 12 quadros, tira horizontal, quadro quadrado ~144×144px (pode ser maior,
  mantendo proporção).
- Frame 0 = norte, sentido horário, top-down, fundo transparente.
- Silhueta clara e diferenciável dos outros inimigos em tamanho pequeno de
  tela (eles aparecem minúsculos).
- Cor do tiro/destaque do inimigo **não pode ser `#ffd23f`** (cor das
  moedas) — senão confunde com moeda no chão.

## 3. Chefes (bosses)

- Mesmo padrão dos capangas (12 direções, top-down, fundo removido), só
  maior escala e mais detalhe.
- Seguir o padrão de nome/formato já usado em `material_bruto/` (ex:
  `boss master CHEFE_MAXIMO_12_LIMPO`, `boss BARRIL_TANK_12_DIRECOES`).

## 4. Cenário — planeta + fundo espacial

Dois assets **diferentes**, propósitos diferentes:

- **Textura de fundo (tileable/repetível)**: cobre o campo de jogo inteiro,
  repete infinitamente — precisa ser **sem emenda visível** ao repetir lado
  a lado. Tema: nebulosa/estrelas escura (como hoje), mas pode ganhar mais
  "cara" de campo de detritos espaciais em vez de só nebulosa lisa.
- **Planeta**: imagem única, **não repete**, grande, vista de longe (com
  curvatura/atmosfera visível), fundo transparente. Fica num ponto **fixo**
  do mapa (não acompanha a câmera) — o jogador se afasta, volta e pode dar a
  volta ao redor dele. Estilo: planeta arrasado/pós-apocalíptico (crateras,
  rachaduras vulcânicas, ou anel de destroços) combinando com o tema sucata.

## 5. Destroços/obstáculos do cenário

- Já existem: `debCon`, `debPole`, `debFridge`, `debBus`, `debContainer`,
  `debBillboard`, `debCar` — imagem única estática, fundo removido.
- Pra tipos novos: mesma lógica, mas tema sucata **espacial** (destroços de
  nave, contêineres flutuando, asteroide com pedaço de metal cravado) já
  que agora é ambientado no espaço, não mais estrada.

## 6. Ícones de armas

- Ícone único quadrado, fundo removido, ~256×256px, silhueta simples e
  reconhecível pequena (aparece minúsculo no HUD).
- Já existem 8: pea, leque, metralha, missil, laser, granada, rkl88, tnt —
  qualquer arma nova segue o mesmo padrão.

## 7. Explosões (spritesheet)

- Já tem 2 prontas (`explosion_sheet` 6×1, `explosion_sheet_obstacle` 3×2).
  Só precisa de uma nova se quiser um efeito específico (ex: explosão só da
  nave do jogador morrendo, diferente da explosão padrão).

## 8. Telas de UI (logo título / game over)

- Pode manter fundo sólido tipo cartão/pôster, como já está — não precisa
  remover fundo dessas duas.
