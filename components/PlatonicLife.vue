<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'
// 1. Import the class directly for static access (Quadrille.cellLength, Quadrille.or, etc.)
import Quadrille from 'p5.quadrille'
// 2. Import p5.platonic for side-effects (adds p.platonicGeometry)
import 'p5.platonic'

const sketchContainer = ref(null)
let p5Instance = null

const sketch = (p) => {
  // --- Configuration ---
  // Configure static property directly on the class
  Quadrille.cellLength = 40
  
  const ROWS = 20
  const COLS = 20
  
  let game, pattern
  let buffer
  let mode = 4 // Default shape (Box)
  let lifeColor
  
  // We will store the generated platonic geometry here
  let platonicModel 

  // Shape selector for the main 3D object
  const shapes = {
    1: () => p.plane(p.width, p.height),
    2: () => p.sphere(150),
    3: () => p.torus(150, 50),
    4: () => p.box(300),
    5: () => p.cylinder(100, 300, 24, 1, false, false),
    6: () => p.cone(100, 300, 24, 1, false),
    7: () => p.ellipsoid(100, 180)
  }

  p.setup = function () {
    // Canvas size matches the quadrille resolution
    const canvas = p.createCanvas(ROWS * Quadrille.cellLength, COLS * Quadrille.cellLength, p.WEBGL)
    canvas.parent(sketchContainer.value)

    // --- Geometry Setup ---
    // Generate the 3D model for the cells ONCE.
    // args: radius, detail, type
    platonicModel = p.platonicGeometry(Quadrille.cellLength / 2.5, 1, 'Icosahedron')
    
    // --- Quadrille Setup ---
    lifeColor = p.color('lime')
    
    // Create the main game grid
    game = p.createQuadrille(COLS, ROWS)
    
    // Create a pattern (BigInt represents a glider/spaceship structure)
    pattern = p.createQuadrille(3, 16252911n) 
    
    // Convert the pattern's "1s" into our rendering function
    pattern.visit(({row, col, value}) => {
      if(value) pattern.fill(row, col, displayNode)
    })

    // Apply the pattern to the game using Static method Quadrille.or
    game = Quadrille.or(game, pattern, 6, 8)

    // --- Framebuffer Setup ---
    buffer = p.createFramebuffer({ format: p.FLOAT })
    
    // Initial update
    update()
  }

  p.draw = function () {
    // 1. Update Game Logic (Game of Life)
    if (p.frameCount % 20 === 0) update()

    // 2. Render the Quadrille to the Framebuffer (Texture)
    buffer.begin()
    p.background('indigo') 
    
    // drawQuadrille will call 'displayNode' for every filled cell
    p.drawQuadrille(game, { 
      outline: 'cyan',
      weight: 2,
      origin: 'corner' 
    })
    buffer.end()

    // 3. Render the Main Scene
    p.background(20)
    p.orbitControl()
    
    p.rotateX(p.frameCount * 0.005)
    p.rotateY(p.frameCount * 0.005)

    p.noStroke()
    
    // Apply the framebuffer as a texture on the 3D shape
    p.texture(buffer)
    
    // Draw the selected shape
    if (shapes[mode]) {
      shapes[mode]()
    } else {
      shapes[1]()
    }
  }

  // --- Game of Life Logic ---
  function update() {
    const next = game.clone()
    
    game.visit(({ row, col, value }) => {
      const order = game.ring(row, col).order
      
      // Use Static method Quadrille.isFilled
      const isAlive = Quadrille.isFilled(value)

      if (isAlive) {
        // Die (Under/Over population)
        if (order < 2 || order > 3) {
          next.clear(row, col)
        }
      } else {
        // Born (Reproduction)
        if (order === 3) {
          // Fill with the rendering function
          next.fill(row, col, displayNode) 
        }
      }
    })
    
    game = next
  }

  // --- Rendering Function for Cells ---
  // This is passed to .fill() and executed by drawQuadrille
  function displayNode({ row, col }) {
    p.push()
    // Local rotation for the individual cell
    const t = p.millis() * 0.002
    p.rotateX(t + row)
    p.rotateY(t + col)
    
    p.fill('lime')
    p.stroke('black')
    p.strokeWeight(1)
    
    // Render the platonic solid
    p.model(platonicModel)
    p.pop()
  }

  p.keyPressed = function () {
    // Change Shape
    if (p.key >= '1' && p.key <= '7') {
      mode = +p.key
    }
    
    // Reset
    if (p.key === 'r' || p.key === 'R') {
      game.clear()
      game = Quadrille.or(game, pattern, p.int(p.random(COLS)), p.int(p.random(ROWS)))
    }
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
  <div class="flex flex-col items-center justify-center space-y-4">
    <div ref="sketchContainer" class="rounded-lg overflow-hidden border border-white/10 shadow-2xl"></div>
    <div class="text-sm text-gray-400 font-mono">
      <p>Keys [1-7]: Change Shape | [R]: Reset</p>
    </div>
  </div>
</template>
