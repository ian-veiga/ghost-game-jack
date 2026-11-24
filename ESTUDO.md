# Fantasma x Pessoa — Guia de estudo (Projeto 6, UFRPE)

## 1. O jogo em uma frase
Dois jogadores no mesmo teclado: o **Fantasma** (Jogador 1, setas, `.` para parar) persegue a **Pessoa** (Jogador 2, W/A/S/D, `E` para parar) num labirinto aleatório. A Pessoa tem **3 s de vantagem**, depois são **20 s de perseguição**. Se o Fantasma pegar → Jogador 2 perde. Se o tempo acabar → Jogador 1 perde. Depois de ~3 s a rodada recomeça sozinha. **Esc** sai em qualquer momento.

## 2. As classes (7 arquivos)

| Classe | O que faz |
|---|---|
| `Main` | Ponto de entrada: cria o `GhostGame`, chama `run()` e depois `dispose()`. Igual ao original. |
| `GhostGame` | O "cérebro". Máquina de estados (tela inicial → rodada → Game Over → nova rodada), laço principal de ticks, leitura do teclado, captura, relógios. Cria `Maze`, `Ghost` e `Person` **uma vez só** e os reinicia a cada rodada. |
| `Maze` | O labirinto: um `Array` de 512 posições (`linha*32 + coluna`, 1 = parede), alocado uma vez só. `generate()` refaz o labirinto no mesmo Array (Binary Tree + braiding), responde `isWall(col,row)`, sorteia células livres e desenha as paredes com `Memory.poke`. |
| `Ghost` | O fantasma: posição em células (`col`, `row`), direção atual, `step(maze)` anda uma célula, `draw`/`erase` direto na memória de vídeo. Desenho original do fantasma feliz. |
| `Person` | A pessoa: **mesma interface e mesmas regras** do Ghost, só muda o desenho (boneco de palito). |
| `Hud` | Todo o texto: linha 0 com rótulos e relógios, tela inicial, caixa de Game Over centralizada, "Fim de jogo". |
| `Random` | Gerador pseudoaleatório (LCG de 16 bits). Corrigido em relação ao original. |

## 3. Estratégias

### Tela e grade
512×256 pixels = **32 colunas × 16 linhas** de células 16×16. A linha 0 é o HUD; o labirinto usa as linhas 1–15. Endereço da célula na memória: `16384 + (row*512) + col`; a próxima linha de pixels da mesma célula está `+32` adiante. Cada célula = 16 `Memory.poke` (um por linha de pixels). No Hack, o **bit 0 é o pixel mais à esquerda**.

### Movimento: direção "travada" + uma tecla por vez
`Keyboard.keyPressed()` no Hack devolve **só uma tecla**. Se o boneco só andasse enquanto a tecla está apertada, dois jogadores não conseguiriam jogar juntos. Solução: a tecla só **escolhe a direção**; o boneco continua andando sozinho, uma célula por vez, até bater numa parede (direção volta a 0) ou o jogador apertar "parar". Assim cada jogador só precisa dar um toque rápido.

### Tempo: a "batida do coração" (ticks)
O Jack não tem relógio. Cada volta do laço principal é **um tick** e termina com `Sys.wait(TICK_MS)`. Tudo é contado em ticks:
- segundo atual = `ticks / TICKS_PER_SECOND`;
- os bonecos andam quando `ticks mod MOVE_EVERY_TICKS = 0`.

Ordem dentro de um tick: lê tecla → atualiza direções (Fantasma ignorado na vantagem) → se for tick de movimento: guarda posições antigas, anda Pessoa, anda Fantasma, verifica captura → espera → atualiza relógio (só redesenha quando o segundo muda).

### Colisão
- **Parede:** antes de andar, o boneco pergunta `maze.isWall(proxCol, proxRow)` — é só uma leitura no Array. Fora da tela também conta como parede.
- **Captura:** depois de cada passo: **mesma célula** OU **troca de células** (Fantasma foi para onde a Pessoa estava E a Pessoa foi para onde o Fantasma estava). Sem a troca, os dois se atravessariam.

### Labirinto: Binary Tree + braiding
1. Tudo começa como parede.
2. "Salas" = células com coordenadas locais ímpares (15 × 7 salas).
3. Cada sala abre a parede ao **norte** ou ao **leste** (sorteio). Primeira linha só leste, última coluna só norte, canto superior direito nada.
4. **Braiding:** paredes internas que separam duas células livres (e não são "pilares") são abertas com 15% de chance → cria **laços**, então a Pessoa tem rotas de fuga.

**Garantia de conexão:** andando sempre para norte/leste, toda sala chega à sala do canto superior direito, então tudo está ligado. O braiding só abre paredes, nunca fecha. Por isso **qualquer célula livre é um spawn válido**. (Também testei numa simulação em Python: 300 labirintos, todos conectados.)

### Spawns
Fantasma: sorteia células até achar uma livre. Pessoa: sorteia até achar uma com distância Manhattan ≥ 8 do Fantasma, no máximo 200 tentativas; se nenhuma servir, usa a mais distante encontrada (o laço nunca é infinito).

### Aleatoriedade
- **Semente:** na tela inicial um contador sobe a cada volta do laço; quando alguém aperta a tecla, o valor do contador vira a semente. Como depende do tempo humano, cada partida é diferente. (O original usava 19 fixo → todo jogo igual.)
- **Bug do original:** `seed = seed + 20251`. 20251 é ímpar, então o bit mais baixo alternava 0,1,0,1… e `randRange(1)` lia exatamente esse bit → o labirinto seria um zigue-zague regular.
- **Correção:** LCG `seed = (seed * 25173) + 13849` com "estouro" de 16 bits. Num LCG os bits **altos** são os bons, então `randRange(range)` divide 0..32767 em `range+1` faixas iguais e devolve a faixa (usa a parte alta). Verificado em Python: ~50% de 0 e 1, todos os 32 valores de `randRange(31)` aparecem.

### Reinício e higiene do heap
**Tudo é alocado uma vez só; cada rodada só reinicia os objetos.** O construtor do `GhostGame` cria `Random`, `Hud`, `Maze` (com o Array de 512), `Ghost` e `Person`. A cada rodada: `maze.generate(BRAID_PERCENT)` reescreve o mesmo Array, e `ghost.reset(col,row)` / `person.reset(col,row)` mudam posição e direção e desenham. Tudo só é liberado em `GhostGame.dispose()`, no fim do jogo. (Veja a seção 4 — foi assim que corrigi um bug real.) Strings literais criam uma String nova toda vez que são avaliadas, então: nenhum texto é impresso dentro do laço de ticks; todo texto passa por `Hud.printText`, que imprime e faz `dispose()`; os números usam `printInt`/`printChar`, que não criam Strings. Sem isso, depois de algumas rodadas automáticas o heap acabaria e o emulador travaria.

## 4. Bug encontrado nos testes: fragmentação do heap

**O que aconteceu.** Depois de ~8 rodadas o Web IDE parava com `Heap overflow (Memory.alloc)` — o **erro 6** do Jack OS — dentro de `Maze.new` → `Array.new(512)`. Observando os endereços, cada rodada o Array novo ficava **514 palavras mais adiante** no heap que o anterior.

**Por quê.** A versão antiga fazia, em toda rodada, `Maze.new` (Array de 512) + `Ghost.new` + `Person.new`, e no fim `dispose()` dos três. Liberar não basta:
1. Na rodada N o Array de 512 ocupa um bloco de 514 palavras (512 + cabeçalho do alocador).
2. No fim da rodada ele é liberado e o bloco volta para a lista de livres.
3. Na rodada N+1, as alocações **pequenas** (o objeto `Maze`, `Ghost`, `Person`, Strings temporárias do HUD) pegam o primeiro bloco livre que serve — justamente o de 514. Ele é cortado e deixa de ter 512 palavras seguidas.
4. Então o Array novo de 512 não cabe mais ali e vai para memória nova, mais adiante (+514).
5. O `Memory.deAlloc` do Jack OS não junta blocos vizinhos livres, então os pedaços nunca voltam a formar um bloco grande. O heap (só ~14 mil palavras) acaba em ~8 rodadas, **mesmo com todo `dispose()` feito certinho**.

Ou seja: não era vazamento (*leak*), era **fragmentação**: havia memória livre no total, mas nenhum pedaço contínuo de 512.

**A correção.** *Tudo é alocado uma vez só; cada rodada só reinicia os objetos.*
- `Maze.new(random)` só aloca o Array; o novo método `generate(braidPercent)` roda `fillWithWalls`, `openRoomPassages` e `braid` no mesmo Array.
- `Ghost` e `Person` ganharam `reset(col, row)`: muda a posição, zera a direção e desenha. Os construtores não desenham mais.
- O construtor do `GhostGame` cria `Maze`, `Ghost` e `Person`; `setupRound()` faz `clearScreen` → `maze.generate` → `maze.draw` → `spawnPlayers` (que chama `reset`). `disposeRound()` foi removido. Os três só são liberados em `GhostGame.dispose()`.

Assim o número de alocações grandes por rodada é **zero**, e o jogo pode reiniciar indefinidamente.

## 5. O que foi adaptado x o que é novo

| Arquivo | Situação |
|---|---|
| `Main.jack` | Original, só comentários novos. |
| `Ghost.jack` | Adaptado: pixels → células, campo `direction`, `setDirection`/`stop`/`step`, `moveUp` etc. consultam o labirinto, `erase` corrigido (o original apagava 17×17 px com `drawRectangle`), removidos `incSize`/`decSize`. Desenho do fantasma mantido. |
| `GhostGame.jack` | Adaptado: dois jogadores, direção travada, máquina de estados, relógios, captura, Esc no lugar de `q`, semente da tela inicial. |
| `Random.jack` | Adaptado: novo gerador (LCG), novo `randRange`, `setSeed`, removidos `getShift`/`calledTime`. |
| `Maze.jack`, `Person.jack`, `Hud.jack` | Novos. |

Cada arquivo adaptado tem um bloco `// ADAPTADO:` no topo explicando as mudanças.

## 6. Valores de ajuste (campos no construtor do GhostGame)

| Campo | Valor | Para que serve |
|---|---|---|
| `TICK_MS` | 50 | espera de cada tick |
| `TICKS_PER_SECOND` | 20 | ticks por segundo (50 ms × 20 = 1 s, em teoria) |
| `MOVE_EVERY_TICKS` | 3 | velocidade dos bonecos |
| `HEAD_START_SECONDS` | 3 | vantagem da Pessoa |
| `CHASE_SECONDS` | 20 | duração da perseguição |
| `GAMEOVER_PAUSE_SECONDS` | 3 | pausa antes de reiniciar |
| `BRAID_PERCENT` | 15 | % de paredes extras abertas |
| `MIN_SPAWN_DISTANCE` | 8 | distância mínima entre os spawns |

## 7. Perguntas prováveis do professor

1. **Por que o relógio não é exato?** O Jack não tem relógio; contamos ticks. `Sys.wait` não é preciso e o resto do tick (ler teclado, desenhar) também gasta tempo, que varia com a velocidade do emulador. Por isso os segundos são aproximados e eu calibro `TICK_MS` no Web IDE.
2. **O que acontece se os dois trocarem de célula no mesmo tick?** Conta como captura. Guardo as posições antigas antes de andar; se o Fantasma ficou na posição antiga da Pessoa e a Pessoa na posição antiga do Fantasma, eles se cruzaram → pego.
3. **Por que Ghost e Person têm código repetido?** O Jack não tem herança, e o enunciado pede a classe `Ghost.jack`. Repetir deixa cada classe simples; juntar com um campo "tipo" e `if`s deixaria mais confuso.
4. **Como você garante que o spawn não cai num lugar fechado?** O Binary Tree liga todas as salas ao canto superior direito, então todas as células livres estão conectadas; o braiding só abre paredes. E o spawn só aceita células onde `isWall` é falso.
5. **Por que precisou mudar o Random?** O original somava um número ímpar, então o último bit alternava 0,1,0,1, e `randRange(1)` lia justamente esse bit: o labirinto seria sempre um zigue-zague. Troquei por um LCG e passei a usar os bits altos.
6. **Por que segurar duas teclas não funciona?** `Keyboard.keyPressed()` do Hack devolve só uma tecla por vez. Por isso o movimento é travado: basta um toque para mudar a direção.
7. **Por que o laço do spawn da Pessoa não pode ficar infinito?** Tem limite de 200 tentativas; se nenhuma tiver distância ≥ 8, uso a mais distante encontrada.
8. **Por que `Memory.poke` em vez de `Screen.drawRectangle`?** Uma célula é 16 palavras de memória; 16 pokes são bem mais rápidos que a rotina genérica de retângulo, e são centenas de paredes por labirinto. E o `drawRectangle(x, y, x+16, y+16)` do original pintava 17 pixels, borrando a vizinha.
9. **Por que o jogo não trava depois de muitas rodadas?** Porque nada grande é alocado por rodada: `Maze` (com o Array), `Ghost` e `Person` são criados uma vez no construtor do `GhostGame` e cada rodada só os reinicia (`generate`, `reset`). As Strings literais são liberadas logo depois de impressas e nenhuma String é criada dentro do laço de ticks.
10. **De onde vem a aleatoriedade, se o computador é determinístico?** Da pessoa: o contador da tela inicial depende de quanto tempo ela demora para apertar uma tecla. Esse valor vira a semente.
11. **Por que o Fantasma não pode "pré-carregar" uma direção na vantagem?** Durante a vantagem as teclas do Fantasma nem são lidas (`handleGhostKey` só é chamado depois), então ele começa parado.
12. **Por que tantos parênteses?** O Jack não tem precedência de operadores: `a + b * c` é calculado da esquerda para a direita. Os parênteses garantem a ordem certa.
13. **Você liberava tudo com `dispose()`; por que mesmo assim dava "Heap overflow"?** Era fragmentação, não vazamento. Cada rodada liberava o Array de 512 e criava outro; as alocações pequenas da rodada seguinte caíam no bloco liberado, então o Array novo não cabia ali e ia para mais adiante (+514 palavras por rodada). O `deAlloc` do Jack não junta blocos livres vizinhos, então em ~8 rodadas não havia mais 512 palavras seguidas (erro 6). A correção: tudo é alocado uma vez só; cada rodada só reinicia os objetos (`maze.generate`, `ghost.reset`, `person.reset`).
