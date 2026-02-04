<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'
import Quadrille from 'p5.quadrille'

const sketchContainer = ref(null)
let p5Instance = null

const sketch = (p) => {
  // Configuración de la rejilla
  Quadrille.cellLength = 40
  const COLS = 8
  const ROWS = 8

  let board, patternEditor, patternHint, fenDisplay, hint
  const defaultFen = 'r3kb1r/ppN1pppp/5n2/8/8/8/PPP3PP/R3KB1R'
  let fenInput, pieces, hits

  const r = Quadrille.chessSymbols.get('r')
  const k = Quadrille.chessSymbols.get('k')
  const N = Quadrille.chessSymbols.get('N')

  p.setup = () => {
    const canvas = p.createCanvas(
      (2 * COLS + 1) * Quadrille.cellLength,
      (ROWS + 1) * Quadrille.cellLength,
    )
    canvas.parent(sketchContainer.value)

    // FEN input field
    fenInput = p.createInput()
    fenInput.parent(sketchContainer.value)
    fenInput.size(8 * Quadrille.cellLength - 5)
    // Posicionamiento absoluto relativo al contenedor
    fenInput.style('position', 'absolute')
    fenInput.style('left', `${9 * Quadrille.cellLength + 5}px`)
    fenInput.style('top', `${8 * Quadrille.cellLength + 15}px`)
    fenInput.changed(update)

    // Selector de piezas
    pieces = p.createSelect()
    pieces.parent(sketchContainer.value)
    pieces.option('clear')
    for (const symbol of Quadrille.chessSymbols.values()) {
      pieces.option(symbol)
    }
    pieces.selected('clear')
    pieces.style('position', 'absolute')
    pieces.style('left', '10px')
    pieces.style('top', `${8 * Quadrille.cellLength + 15}px`)

    board = p.createQuadrille()
    reset()
  }

  p.draw = () => {
    p.background('darkkhaki')
    
    // Tablero izquierda
    p.drawQuadrille(board, { tileDisplay: 0 })
    p.drawQuadrille(patternEditor, { textColor: 'black', tileDisplay: 0 })
    if (patternHint) p.drawQuadrille(patternHint, { tileDisplay: 0 })

    // Tablero derecha (FEN)
    p.drawQuadrille(board, { col: 9, tileDisplay: 0 })
    p.drawQuadrille(fenDisplay, {
      col: 9,
      textColor: 'black',
      tileDisplay: 0,
    })

    // Highlight matches
    if (hits && hits.length > 0) {
      const hit = hits[0]
      p.drawQuadrille(hint, {
        row: hit.row,
        col: 9 + hit.col,
        outline: 'magenta',
        outlineWeight: 2,
      })
    }
  }

  const reset = () => {
    fenInput.value(defaultFen)
    fenDisplay = p.createQuadrille(fenInput.value())
    patternEditor = p.createQuadrille(8, 8)
    patternEditor.fill(3, 1, r)
    patternEditor.fill(3, 5, k)
    patternEditor.fill(4, 3, N)
    update()
  }

  const update = () => {
    fenDisplay = p.createQuadrille(fenInput.value())
    hits = []
    const pattern = patternEditor.trim()
    patternHint = undefined
    if (pattern) {
      hits = fenDisplay.search(pattern, true)
      hint = Quadrille.not(pattern, p.color(0, 180))
      const patternMask = pattern.fill(1)
      const span = patternEditor.span
      patternHint = Quadrille.or(
        p.createQuadrille(8, 8),
        patternMask,
        span.row,
        span.col,
      )
      patternHint.not(p.color(0, 180))
    }
  }

  p.keyPressed = () => {
    if (p.key.toLowerCase() === 'r') reset()
  }

  p.mouseClicked = () => {
    const row = patternEditor.mouseRow
    const col = patternEditor.mouseCol
    if (!patternEditor.isValid(row, col)) return
    
    if (pieces.value() === 'clear') {
      patternEditor.clear(row, col)
    } else {
      patternEditor.fill(row, col, pieces.value())
    }
    update()
  }
}

onMounted(() => {
  p5Instance = new p5(sketch, sketchContainer.value)
})

onUnmounted(() => {
  if (p5Instance) p5Instance.remove()
})
</script>

<template>
  <div class="flex flex-col items-center">
    <div ref="sketchContainer" class="relative border-4 border-stone-800 shadow-xl overflow-hidden rounded-lg"></div>
    <p class="mt-12 text-sm opacity-60 italic">
      Dibuja un patrón a la izquierda para buscarlo en el tablero FEN de la derecha.
    </p>
  </div>
</template>
