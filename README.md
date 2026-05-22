# phaser3-do-zero


> Material didático de introdução à Programação de Jogos Digitais com Phaser 3 e JavaScript — do Game Loop ao primeiro personagem animado.

---

## O que é este repositório

Material completo desenvolvido para o curso de ** Programação de Jogos Digitais **. O projeto-base é um jogo top-down com movimento livre, animações de spritesheet, física arcade e leitura de teclado — simples o suficiente para uma primeira aula, completo o suficiente para introduzir conceitos reais de game dev.


## O jogo

O personagem se move livremente pela tela usando **setas direcionais** ou **WASD**, exibe animações diferentes para cada direção e não atravessa as bordas da tela. Toda a lógica cabe em um único arquivo HTML — sem build, sem bundler, sem instalação.

```html
<!-- Única dependência: Phaser via CDN -->
<script src="https://cdn.jsdelivr.net/npm/phaser@3.80.1/dist/phaser.min.js"></script>
```

Para rodar: abra `jogo/index.html` em qualquer navegador moderno.

---

## Conceitos cobertos

### O Game Loop

Toda a lógica do jogo vive em três funções que o Phaser chama na ordem certa:

```javascript
class JogoPrincipal extends Phaser.Scene {
    preload() { /* 1. carrega assets    — roda 1 vez     */ }
    create()  { /* 2. monta o mundo    — roda 1 vez     */ }
    update()  { /* 3. lógica em tempo real — ~60x/s     */ }
}
```

### Carregando um spritesheet

```javascript
preload() {
    this.load.spritesheet('dude',
        'https://labs.phaser.io/assets/sprites/dude.png',
        { frameWidth: 32, frameHeight: 48 }
    );
}
```

O Phaser fatia automaticamente a imagem em 9 frames de 32×48 px:

```
[ 0 ][ 1 ][ 2 ][ 3 ]   [ 4 ]   [ 5 ][ 6 ][ 7 ][ 8 ]
 ←  walk left      →   idle    ←   walk right      →
```

### Sprite com física vs. sem física

```javascript
// Só uma imagem — sem colisões, sem velocidade
this.add.sprite(400, 300, 'dude');

// Imagem + corpo físico — colisões, setVelocity, bordas
this.player = this.physics.add.sprite(400, 300, 'dude');
this.player.setCollideWorldBounds(true);
```

### Definindo animações

```javascript
create() {
    this.anims.create({
        key: 'left',
        frames: this.anims.generateFrameNumbers('dude', { start: 0, end: 3 }),
        frameRate: 10,
        repeat: -1      // -1 = loop infinito
    });

    this.anims.create({
        key: 'right',
        frames: this.anims.generateFrameNumbers('dude', { start: 5, end: 8 }),
        frameRate: 10,
        repeat: -1
    });
}
```

### Leitura de teclado — polling vs. eventos

O Phaser usa **polling**: a cada frame, o código consulta o estado atual das teclas — em vez de esperar um evento chegar.

```javascript
// Polling — consultado 60x/s no update()
if (this.cursors.left.isDown) moveX = -1;

// isDown     → verdadeiro em TODOS os frames pressionados  → movimento
// justDown   → verdadeiro em APENAS 1 frame (o do aperto) → pulo, disparo
if (Phaser.Input.Keyboard.JustDown(this.cursors.up)) {
    this.player.setVelocityY(-400); // pula uma vez só
}
```

### Normalização do vetor diagonal

Sem normalização, mover na diagonal é 41% mais rápido — porque o Teorema de Pitágoras não perdoa:

```
√(200² + 200²) ≈ 283 px/s   ← errado
```

A correção usa o fator `1 / √2 ≈ 0.7071`:

```javascript
if (moveX !== 0 && moveY !== 0) {
    const fator = 0.7071;
    velX *= fator;  // 200 * 0.7071 = 141.4
    velY *= fator;  // √(141.4² + 141.4²) = 200 px/s ✓
}
```

### O update() completo

```javascript
update() {
    // 1. Ler teclado
    let moveX = 0, moveY = 0;
    if (this.cursors.left.isDown  || this.wasd.left.isDown)  moveX = -1;
    if (this.cursors.right.isDown || this.wasd.right.isDown) moveX =  1;
    if (this.cursors.up.isDown    || this.wasd.up.isDown)    moveY = -1;
    if (this.cursors.down.isDown  || this.wasd.down.isDown)  moveY =  1;

    // 2. Velocidade + normalização diagonal
    let velX = moveX * 200;
    let velY = moveY * 200;
    if (moveX !== 0 && moveY !== 0) {
        velX *= 0.7071;
        velY *= 0.7071;
    }

    // 3. Mover
    this.player.setVelocity(velX, velY);

    // 4. Animar
    if (velX < 0)      this.player.anims.play('left', true);
    else if (velX > 0) this.player.anims.play('right', true);
    else {
        if (this.player.anims.isPlaying) this.player.anims.stop();
        this.player.setFrame(4); // idle
    }
}
```

---

## Erros comuns e como evitar

**Pulo infinito**
```javascript
// ✗ isDown dispara 60x/s enquanto a tecla estiver pressionada
if (this.cursors.up.isDown) this.player.setVelocityY(-400);

// ✓ justDown dispara exatamente 1 vez
if (Phaser.Input.Keyboard.JustDown(this.cursors.up)) this.player.setVelocityY(-400);
```

**Frame idle piscando**
```javascript
// ✗ A animação sobrescreve setFrame() no próximo ciclo
this.player.setFrame(4);

// ✓ Para primeiro, depois fixa o frame
if (this.player.anims.isPlaying) this.player.anims.stop();
this.player.setFrame(4);
```

**Variável fora do escopo**
```javascript
// ✗ var player dentro do create() — o update() não enxerga
create() { var player = this.physics.add.sprite(...); }

// ✓ this.player — propriedade da cena, acessível em qualquer método
create() { this.player = this.physics.add.sprite(...); }
```

---

## Desafios para os alunos

```javascript
// Nível 1 — Sprint com Shift
let velocidadeBase = 200;
if (this.cursors.shift.isDown) velocidadeBase = 450;

// Nível 2 — frameRate assimétrico
// Mude 'right' para frameRate: 20 e 'left' para frameRate: 5
// Observe como a sensação muda sem alterar a velocidade física

// Nível 3 — HUD de zona de perigo
if (posX < 100 || posX > 700 || posY < 100 || posY > 500) {
    this.posText.setColor('#ff4444');
} else {
    this.posText.setColor('#ccffcc');
}
```


## Referências

- [Documentação Phaser 3](https://phaser.io/docs)
- [Exemplos interativos](https://labs.phaser.io)
- [CDN Phaser 3.80.1](https://cdn.jsdelivr.net/npm/phaser@3.80.1/dist/phaser.min.js)
