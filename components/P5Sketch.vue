<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'

const sketchContainer = ref(null)
let p5Instance = null

// Aquí defines tu sketch
const sketch = (p) => {
  let particles = []
  let mX, mY

  p.setup = () => {
    // Ajustamos al ancho del contenedor de la slide
    const canvas = p.createCanvas(600, 300)
    canvas.parent(sketchContainer.value)
  }

  p.draw = () => {
    p.background(30, 30, 35)
    p.noStroke()
    
    // Un círculo que sigue al mouse
    p.fill(100, 255, 200, 150)
    //p.ellipse(mouseX, mouseY, 50, 50);
    p.ellipse(mX, mY, 50, 50);

    // Dibujar texto dinámico
    p.fill(255)
    p.textSize(16)
    p.text("Mueve el mouse sobre el canvas", 20, 30)
  }
  
  p.mouseMoved = () => {
    const r = p.canvas.getBoundingClientRect();
    mX = (p.winMouseX - r.left) * (p.width / r.width)
    mY = (p.winMouseY - r.top) * (p.height / r.height)
  }
}

onMounted(() => {
  // Inicializamos p5 al montar el componente
  p5Instance = new p5(sketch, sketchContainer.value)
})

onUnmounted(() => {
  // ¡IMPORTANTE! Limpiamos el proceso al cambiar de slide
  if (p5Instance) p5Instance.remove()
})
</script>

<template>
  <div class="flex flex-col items-center">
    <div ref="sketchContainer" class="border border-gray-500 rounded-lg overflow-hidden shadow-xl"></div>
  </div>
</template>
