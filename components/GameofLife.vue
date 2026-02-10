<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'
import Quadrille from 'p5.quadrille'

const sketchContainer = ref(null)
let p5Instance = null

const sketch = (p) => {
  'use strict'

  Quadrille.cellLength = 20

  let game, pattern
  let life
  let buffer
  let mode = 1

  const shapes = {
    1: () => p.plane(p.width, p.height),
    2: () => p.sphere(100),
    3: () => p.torus(100, 50),
    4: () => p.box(200),
    5: () => p.cylinder(100, 200, 24, 1, false, false),
    6: () => p.cone(100, 200, 24, 1, false),
    7: () => p.ellipsoid(100, 180)
  }

  p.setup = function () {
    game = p.createQuadrille(20, 20)
    life = p.color('lime')
    pattern = p.createQuadrille(3, 16252911n, life)
    game = Quadrille.or(game, pattern, 6, 8)

    const canvas = p.createCanvas(
      game.width * Quadrille.cellLength,
      game.height * Quadrille.cellLength,
      p.WEBGL
    )
    canvas.parent(sketchContainer.value)

    buffer = p.createFramebuffer({ format: p.FLOAT })

    update()
  }

  p.draw = function () {
    p.background('yellow')
    p.orbitControl()
    p.frameCount % 10 === 0 && update()
    p.noStroke()
    shapes[mode] ? shapes[mode]() : shapes[1]()
  }

  function update () {
    const next = game.clone()

    game.visit(({ row, col, value }) => {
      const order = game.ring(row, col).order
      Quadrille.isFilled(value)
        ? (order < 3 || order > 4) && next.clear(row, col)
        : order === 3 && next.fill(row, col, life)
    })

    game = next

    buffer.begin()
    p.background('blue')
    p.drawQuadrille(game, { outline: 'magenta', origin: 'corner', outlineWeight:0.5 })
    buffer.end()

    p.texture(buffer)
  }

  p.keyPressed = function () {
    mode = +p.key
    game.clear()
    game = Quadrille.or(game, pattern, 6, 8)
  }
}

onMounted(() => {
  p5Instance = new p5(sketch, sketchContainer.value)
})

onUnmounted(() => {
  p5Instance?.remove()
})
</script>

<template>
  <div class="flex flex-col items-center justify-center space-y-4">
    <div ref="sketchContainer" class="rounded-lg overflow-hidden border border-white/10 shadow-2xl"></div>
    <div class="text-sm text-gray-400 font-mono">
      <p>Keys [1-7]: Change Shape | [R]: Reset</p>
    </div>
  </div>
</template>
