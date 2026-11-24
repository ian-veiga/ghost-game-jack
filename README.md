# Fantasma x Pessoa (Jack / nand2tetris)

Jogo para dois jogadores no mesmo teclado, escrito em **Jack** para o computador Hack do nand2tetris (Projeto 6 — UFRPE).

O **Fantasma** persegue a **Pessoa** num labirinto aleatório. A Pessoa tem **3 s de vantagem**, depois são **20 s de perseguição**:

- se o Fantasma pegar a Pessoa → o Jogador 2 perde;
- se o tempo acabar → o Jogador 1 perde.

Depois de ~3 s uma nova rodada começa sozinha, com outro labirinto.

## Controles

| Jogador | Personagem | Mover | Parar |
|---|---|---|---|
| Jogador 1 | Fantasma | setas ↑ ↓ ← → | `.` |
| Jogador 2 | Pessoa | `W` `A` `S` `D` | `E` |

- Basta **um toque**: o boneco continua andando na direção escolhida até bater numa parede ou você mandar parar (o Hack só lê uma tecla por vez, então segurar não funciona para dois jogadores).
- Na tela inicial, aperte qualquer tecla para começar.
- **Esc** sai do jogo a qualquer momento.

## Arquivos

| Arquivo | Conteúdo |
|---|---|
| `Main.jack` | Ponto de entrada |
| `GhostGame.jack` | Laço principal, rodadas, teclado, captura, relógios |
| `Maze.jack` | Labirinto aleatório |
| `Ghost.jack` / `Person.jack` | Os dois personagens |
| `Hud.jack` | Textos da tela (relógios, tela inicial, Game Over) |
| `Random.jack` | Gerador de números aleatórios |
| `ESTUDO.md` | Explicação detalhada do código e perguntas de estudo |

Os arquivos `.vm` **não ficam no repositório**: eles são gerados ao compilar (passo 1 abaixo).

## Como rodar no nand2tetris

> ⚠️ **Importante: rode no modo _fast_ (velocidade máxima).** Em qualquer velocidade mais baixa o jogo fica lento demais para jogar e os relógios não batem.

O jogo precisa de duas etapas: **compilar** os `.jack` (gerando `.vm`) e **executar** os `.vm` no **VM Emulator**.

### Opção A — Web IDE (no navegador)

1. Baixe este repositório (**Code → Download ZIP**) e extraia numa pasta.
2. Abra o Web IDE do nand2tetris: <https://nand2tetris.github.io/web-ide>
3. Vá para a ferramenta **Jack Compiler**, abra a pasta com os 7 arquivos `.jack` e compile. Devem sair **0 erros** e 7 arquivos `.vm`.
4. Vá para o **VM Emulator** e carregue a pasta compilada (todos os `.vm` juntos — o programa começa em `Main.main`).
5. **Coloque o emulador no modo _fast_** (velocidade máxima). Sem isso o jogo fica lento demais.
6. Aperte **Run** (▶) e clique na tela do emulador para ela receber o teclado.

### Opção B — Ferramentas de desktop (nand2tetris Software Suite)

1. Baixe o *Software Suite* em <https://www.nand2tetris.org/software> (precisa de Java instalado).
2. Compile a pasta do projeto:
   - Windows: `tools\JackCompiler.bat "caminho\da\pasta"`
   - Linux/macOS: `tools/JackCompiler.sh caminho/da/pasta`
3. Abra o VM Emulator (`tools\VMEmulator.bat` ou `tools/VMEmulator.sh`) e use **File → Load Program** para carregar a **pasta** (não um arquivo só).
4. **Coloque no modo _fast_:** em **Animate**, escolha **No animation** e arraste a velocidade para o máximo (*Fast*).
5. Aperte **Run** (▶) e clique na área da tela/teclado do emulador para os comandos chegarem ao jogo.

## Observações

- **Os segundos são aproximados.** O Jack não tem relógio; o jogo conta "ticks" com `Sys.wait`, e a velocidade varia conforme o computador e o emulador. Se o relógio estiver rápido ou lento demais, ajuste `TICK_MS` no construtor de `GhostGame.jack` e recompile.
- Outros valores de ajuste (duração da perseguição, velocidade dos bonecos, % de laços no labirinto etc.) também ficam no construtor de `GhostGame` — veja a seção "Valores de ajuste" do [ESTUDO.md](ESTUDO.md).
