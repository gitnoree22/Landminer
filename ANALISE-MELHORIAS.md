# NÚCLEO — Análise Completa e Plano de Melhorias

> Documento gerado a partir da auditoria do arquivo `Nucleo-Main.html` (1925 linhas, arquivo único sem dependências externas).
> Data: 2026-08-24

---

## Sumário

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Pontos Fortes (manter)](#2-pontos-fortes--manter)
3. [Bugs Concretos Identificados](#3-bugs-concretos-identificados)
4. [Melhorias de Performance](#4-melhorias-de-performance)
5. [Melhorias de Gameplay / UX](#5-melhorias-de-gameplay--ux)
6. [Melhorias Visuais / Polimento](#6-melhorias-visuais--polimento)
7. [Acessibilidade](#7-acessibilidade)
8. [Sugestões de Novos Recursos](#8-sugestões-de-novos-recursos)
9. [Resumo de Prioridades](#9-resumo-de-prioridades)

---

## 1. Visão Geral da Arquitetura

| Bloco | Linhas (aprox.) | Função |
|---|---|---|
| Setup + utils | 29-55 | PROBE, SEED, mulberry32, hash2, fbm |
| Materiais | 57-78 | 19 tipos, 7 tabelas (STATE, DENS, DISP, FLAM, FLIFE, HARD, ATT, EMIT, VARI, PAL, MINI) |
| Mundo | 80-239 | Grid 608×1280, chunks 32×32, geração por camadas + fbm |
| Simulação CA | 241-406 | Step chunked, sand/liquid/gas/fire steps |
| Escavação/Explosão | 408-482 | `digCircle`, `explode` com knockback |
| Partículas | 484-538 | Pool de 1500, `tryDeposit` para "pousar" |
| Entidades | 540-898 | Player, beam, bombs, worms, pickups, core, tide |
| Iluminação | 901-969 | Buffer Lw×Lh, 2 rounds de varredura chanfrada |
| Renderização | 971-1377 | `simC` → canvas, `miniC` p/ sonar |
| HUD/Telas | 1241-1546 | Painel, régua de profundidade, msgs, dead/win/title |
| Áudio | 1548-1724 | WebAudio procedural, música generativa pentatônica |
| Loop principal | 1847-1925 | RAF com acumulador de 60Hz fixo |

---

## 2. Pontos Fortes (manter)

- **Arquitetura de chunks com active-set** (`chCur`/`chNext`): simula só regiões "acordadas" — performance excelente.
- **Semente determinística** via URL (`?seed=123`) + `frnd` separado para FX — replayável.
- **Iluminação por varredura chanfrada** com LUT pré-computada: técnica barata e visualmente crível.
- **Reações químicas profundas**: lava+água=obsidiana+vapor, areia+calor=vidro, ácido corrói, gás explode em cadeia.
- **Áudio procedural sem arquivos**: propulsor/laser/rosnado/alarme/música — tudo sintetizado.
- **Polimento atmosférico**: vinheta por profundidade, screen shake, hitstop, slowmo no grab do núcleo.
- **HUD denso e informativo**: casco, bombas, cristais, régua de profundidade, sonar, mensagens rádio.
- **Auto-qualidade** da luz (`LIGHT_ROUNDS=1` se render passar de 11ms médios em 180 frames).

---

## 3. Bugs Concretos Identificados

### 3.1 — Timeouts fantasma na nova corrida (linha ~1780)
```js
setTimeout(()=>{ if(state==='play')say('BASE','Drone VL-7 online...'); },600);
setTimeout(()=>{ if(state==='play')say('DRONE','Laser...'); },4200);
setTimeout(()=>{ if(state==='play')say('DRONE','Bombas...'); },8200);
```
Os `setTimeout` não são cancelados no `restart()`. Atualmente funciona porque `restart()` faz `location.reload()`, mas qualquer refatoração para restart sem reload vai quebrar isso.

### 3.2 — `gasBurst` resetado por frame, não por step
`stepSim` roda uma vez por `stepGame`, então funciona. Mas o `sfxThump` só dispara se `>14` ignições de gás num único step — difícil de alcançar e o som de explosão de gás quase nunca toca.

### 3.3 — Sons throttled demais
- `sfxPing` (cristal) tem throttle de 70ms — ao minerar vários cristais num raio, ouve só o primeiro.
- `sfxExplosion` throttle de 60ms — encadeamento de bombas corta o som.

### 3.4 — Semente inválida vira 0 silenciosamente (linha 40)
```js
const SEED = Q.get('seed') ? (parseInt(Q.get('seed'),10)>>>0) : ((Math.random()*1e9)>>>0);
```
`parseInt('abc',10)` retorna `NaN` e `NaN>>>0===0`. Semente inválida vira 0 sem aviso.

### 3.5 — `aux` para FIRE vira lifetime, para outros materiais é descartado
`setCell` seta `aux` mas a maioria dos materiais não usa. Quando FIRE é extinto vira SMOKE com `aux=50`. Não causa bug visível mas torna `aux` um campo "frágil".

### 3.6 — Minimapa regenerado a cada 24 frames
Recria `Uint32Array(mw*mh)` (~12KB) — gera GC pressure. Não é bug, é micro-otimização.

### 3.7 — `restart()` sempre recarrega a página
Perde estado de áudio (WebAudio recriado), força `genWorld()` do zero (~50ms de travada), e descarta qualquer input rápido do jogador. Para um jogo roguelite, isso é pesado.

### 3.8 — Cursor `crosshair` no body inteiro (linha 8)
```css
html,body{...cursor:crosshair}
```
Em telas de título/pausa/morte o cursor de crosshair é estranho. Deveria ser default fora do `state==='play'`.

### 3.9 — `drawMini.done` flag global suja
Se o viewport muda de tamanho (`resize`), `drawMini.done` permanece `true` e o minimapa fica com dimensões antigas até o próximo ciclo de 24 frames.

### 3.10 — Não pausa em visibilitychange
Quando o jogador troca de aba, o RAF continua (browser throttle pra 1Hz mas o loop lógico acumula `acc` e pode disparar 4 steps em rajada ao voltar). Sem falar em WebAudio que continua tocando.

---

## 4. Melhorias de Performance

| # | Onde | Melhoria | Impacto |
|---|---|---|---|
| P1 | `parts.splice(k,1)` em hot loops | Trocar por **object pool** com índice `count` em vez de array dinâmico | Evita GC em bursts de partículas |
| P2 | `stepSim` itera todos os chunks | Pular varredura de linhas totalmente vazias | +5-10% FPS |
| P3 | Iluminação recomputada todo frame | Cache incremental: marcar células sujas e propagar só mudanças | +20-30% FPS em cenas paradas |
| P4 | `Math.random()` em hot loops | Substituir por `rnd()` (mulberry32) | Reproducibilidade |
| P5 | `lights.length=Math.min(lights.length,40)` | Pool fixo de luzes com índice `count` | Sem GC |
| P6 | `computeLight` realoca `LBuf` se viewport crescer | Pré-alocar para tamanho máximo | Sem stall em resize |
| P7 | `bodyHits` faz 9 amostras por eixo | Reduzir para 5 (cross) | -44% colisões |
| P8 | Render varre todo viewport por pixel | Manter — já é ótimo com `data32` | — |

---

## 5. Melhorias de Gameplay / UX

### 5.1 — Tutorial fracassado
As mensagens rádio (`say('DRONE', 'Laser...')`) são informativas mas **não contextualizadas**. O jogador precisa ler 3 mensagens em 8s enquanto já está jogando.

**Sugestão**: tooltips contextuais — ao segurar botão esq pela 1ª vez, mostrar "Laser ativo". Ao morrer 1ª vez, mostrar dica da causa.

### 5.2 — Sem feedback de progresso além do sonar
O jogador não sabe:
- Quantos cristais faltam para o próximo milestone (30, 60, 90...).
- Distância até o núcleo (só marca no sonar).
- Tempo médio de uma run.

**Sugestão**: barra de progresso vertical ao lado da régua, mostrando profundidade vs núcleo, com tickmarks dos milestones.

### 5.3 — Sem checkpoint
Se morrer a 800m de profundidade, perde tudo. Roguelite puro, mas para session casual pode frustrar.

**Sugestão adotada**: manter sem checkpoint, mas **adicionar modos de jogo** para variar a experiência:
- Clássico (comportamento atual).
- Fossa (endless, sem núcleo, só profundidade máxima).
- Hardcore (sem milestones, sem heals, permadeath real).
- Diário (semente baseada na data).

### 5.4 — Bombas sem feedback visual de fuse
A luz pisca mas não há contagem regressiva visível.

**Sugestão**: anel contrátil em torno da bomba, ou número de segundos flutuante.

### 5.5 — Morte súbita por magma no tide
`if(tideOn&&P.y>tideY-3) hurt(3,'derretido por magma');` — 3 dano/frame = 180 dps. Morte instantânea se encostar.

**Sugestão**: manter o dano alto mas dar 0.5s de invuln pós-contato para permitir fuga.

### 5.6 — Sem indicador de "direção do núcleo" no early game
Quando o jogador desce sem saber onde fica o núcleo, só vê o losango no sonar.

**Sugestão**: seta sutil na borda da tela apontando para o núcleo (até primeiro contato).

### 5.7 — Win screen sem "próxima semente"
Ao vencer, `[R]` reinicia com semente aleatória. Sem opção de "replay mesma semente para tentar tempo menor".

### 5.8 — Sem comparativo com a melhor run
`loadBest()` mostra string estática no title. Em jogo, nada lembra "você está 50m acima do seu recorde".

---

## 6. Melhorias Visuais / Polimento

| # | Item | Sugestão |
|---|---|---|
| V1 | Filtro CRT / scanlines | Overlay sutil opcional (`?crt=1`) |
| V2 | Trail do drone | Pequeno rastro de partículas quando velocidade > 2 |
| V3 | Partículas redondas | Trocar `fillRect` por `arc` em glow particles (mais caro, mas só em partículas grandes) |
| V4 | Parallax em estrelas | Estrelas da superfície atualmente estáticas — mover com câmera |
| V5 | Iluminação da lava mais dramática | Pulsação mais lenta + flicker esporádico forte |
| V6 | Vignette vermelha crescente com profundidade | Já existe, mas o limiar é 900m — baixar para 600m e intensificar |
| V7 | Drone com lean animation | Já tem `tilt` no eixo X — adicionar bob vertical sutil |
| V8 | Texto com glow consistente | Mensagens rádio sem glow — adicionar `shadowBlur` para legibilidade |
| V9 | Cursor de mira custom | Desenhar crosshair SVG no mouse, ocultar nativo |
| V10 | Hit-flash em vermes ao tomar dano | Vermes não mudam de cor ao receber dano — adicionar flash branco de 1 frame |

---

## 7. Acessibilidade

### Problemas atuais:
- **Sem redução de movimento** — screen shake intenso em explosões pode causar enjoo.
- **Sem opção de alto contraste**.
- **Fonte Consolas fixa** — sem fallback elegante (apenas `monospace`).
- **Mouse-only para mira** — sem aim assist nem modo keyboard-only.
- **Cores críticas codificam estado**: vermelho=dano, laranja=magma, azul=drone, ciano=cristal. Daltônicos podem confundir.
- **Sem legendas para áudio** — mas o jogo não usa áudio informativo essencial.

### Sugestões:
- `?reduceMotion=1` → shake=0, flashWhite dampened, hitstop mantido.
- Paleta alternativa daltônica (trocar vermelho por magenta em `dmgFlash`).
- Suporte básico a gamepad via Gamepad API.
- Slider de volume + toggle "reduzir flashes" acessível em pausa.

---

## 8. Sugestões de Novos Recursos

### Tier S (alto valor, baixo esforço):
1. **Modo Diário**: semente baseada na data (`?seed=YYYYMMDD`), compartilhável.
2. **Replay/fantasma**: gravar inputs e reproduzir como "ghost" do melhor run.
3. **Estatísticas persistentes**: total de cristais extraídos, runs, mortes por causa.
4. **Marcador de "melhor profundidade anterior"** na régua.

### Tier A (médio esforço):
5. **Power-ups**: baterias (recarga de laser mais rápido), blindagem temporária, sonic boom (limpa tela).
6. **Tipos de verme**: 2-3 variantes com comportamentos diferentes (rápido, blindado, cuspidor de ácido).
7. **Biomas laterais**: cavernas de gelo, bolsões radioativos, ruínas antigas.
8. **Modo "fossa"** (sem núcleo, só profundidade máxima).

### Tier B (alto esforço, alto impacto):
9. **Sistema de upgrades persistentes** entre runs (meta-progressão).
10. **Boss no núcleo**: defender o núcleo enquanto fugir.
11. **Multiplayer local split-screen** — bem complexo, talvez via SharedArrayBuffer.

---

## 9. Resumo de Prioridades

Ordem sugerida para próximas edições:

1. **Bugs concretos**: 3.1 (timeouts), 3.7 (restart sem reload), 3.8 (cursor), 3.10 (visibilitychange) — rápidos.
2. **Performance**: P1 (object pool de partículas), P3 (iluminação cacheada) — médio esforço, alto ganho.
3. **UX essencial**: 5.1 (tutorial contextual), 5.2 (barra de progresso), 5.4 (fuse visível).
4. **Polimento visual**: V10 (hit-flash vermes), V6 (vignette profunda), V4 (parallax estrelas).
5. **Acessibilidade básica**: reduceMotion + paleta daltônica + slider de volume.
6. **Novos recursos** Tier S (modo diário é trivial).

---

## 10. Correções Aplicadas Nesta Iteração

| # | Item | Status |
|---|---|---|
| C1 | Restart sem reload de página | ✅ Aplicado |
| C2 | Pausar em `visibilitychange` | ✅ Aplicado |
| C3 | Slider de volume + toggle "reduzir flashes" | ✅ Aplicado |
| C4 | Adicionar novos modos de jogo (Clássico / Fossa / Hardcore / Diário) | ✅ Aplicado |
| C5 | Win screen com "Bater Recorde" (mesma seed) e "Explorar novas áreas" (nova seed) | ✅ Aplicado |
| C6 | Comparativo com a melhor run durante gameplay (marcador na régua) | ✅ Aplicado |

Detalhes técnicos de cada correção estão nos commits correspondentes.
