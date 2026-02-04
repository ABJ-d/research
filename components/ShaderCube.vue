<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'
import 'p5.tree' // Esto registra p5.Tree globalmente

const sketchContainer = ref(null)
let p5Instance = null

const sketch = (p) => {
  let edge = 80
  let teapot, bunny, texShader
  let teapotTex, bunnyTex

  let vertSrc = `#version 300 es
    precision highp float;
    in vec4 aPosition;
    uniform mat4 uModelViewProjectionMatrix;
    void main() {
      gl_Position = uModelViewProjectionMatrix * aPosition;
    }`

  let fragSrc = `#version 300 es
    precision highp float;
    uniform sampler2D model;
    uniform vec2 u_resolution;
    out vec4 outColor;
    void main() {
      vec2 st = gl_FragCoord.xy / u_resolution;
      outColor = texture(model, vec2(st.s, 1.0 - st.t));
    }`

  p.setup = async function () {
    const canvas = p.createCanvas(400, 400, p.WEBGL)
    canvas.parent(sketchContainer.value)

    texShader = p.createShader(vertSrc, fragSrc)
    
    // Acceso correcto a las constantes de p5.Tree
    console.log("Usando p5.tree versión:", p5.Tree.VERSION)

    teapot = await p.loadModel('/teapot.obj', true)
    bunny = await p.loadModel('/bunny.obj', true)

    p.shader(texShader)
    texShader.setUniform('u_resolution', p.resolution())

    teapotTex = p.createFramebuffer({ format: p.FLOAT })
    bunnyTex = p.createFramebuffer({ format: p.FLOAT })
  }

  p.draw = function () {
    p.background(20)
    p.orbitControl()

    // 1) Sincronización de cámaras usando p5.Tree
    const position = p.mapLocation({ from: p5.Tree.EYE, to: p5.Tree.WORLD })
    const direction = p.mapDirection({ from: p5.Tree.EYE, to: p5.Tree.WORLD })
    const center = p5.Vector.add(position, direction)
    const up = p.mapDirection(p5.Tree.j, { from: p5.Tree.EYE, to: p5.Tree.WORLD })

    // 2) Renderizado a FBOs
    p.resetShader()

    // --- Bunny ---
    bunnyTex.begin()
    p.clear()
    p.camera(position.x, position.y, position.z, center.x, center.y, center.z, up.x, up.y, up.z)
    p.push()
    p.noStroke()
    p.fill('red')
    p.scale(1, -1)
    p.scale(0.8)
    p.model(bunny)
    p.pop()
    bunnyTex.end()

    // --- Teapot ---
    teapotTex.begin()
    p.clear()
    p.camera(position.x, position.y, position.z, center.x, center.y, center.z, up.x, up.y, up.z)
    p.push()
    p.noStroke()
    p.fill('blue')
    p.scale(1, -1)
    p.model(teapot)
    p.pop()
    teapotTex.end()

    // 3) Dibujo final con Shader
    p.shader(texShader)
    p.push()
    p.stroke('purple')
    p.strokeWeight(2)

    texShader.setUniform('model', bunnyTex.color)
    cubeFace(p5.Tree.NEAR, edge)

    texShader.setUniform('model', teapotTex.color)
    cubeFace(p5.Tree.RIGHT, edge)
    p.pop()
  }

  function cubeFace(face, edgeSize) {
    p.push()
    p.beginShape()
    const faces = {
      [p5.Tree.NEAR]:   [[-edgeSize, -edgeSize, +edgeSize], [+edgeSize, -edgeSize, +edgeSize], [+edgeSize, +edgeSize, +edgeSize], [-edgeSize, +edgeSize, +edgeSize]],
      [p5.Tree.FAR]:    [[-edgeSize, -edgeSize, -edgeSize], [+edgeSize, -edgeSize, -edgeSize], [+edgeSize, +edgeSize, -edgeSize], [-edgeSize, +edgeSize, -edgeSize]],
      [p5.Tree.LEFT]:   [[-edgeSize, -edgeSize, -edgeSize], [-edgeSize, -edgeSize, +edgeSize], [-edgeSize, +edgeSize, +edgeSize], [-edgeSize, +edgeSize, -edgeSize]],
      [p5.Tree.RIGHT]:  [[+edgeSize, -edgeSize, +edgeSize], [+edgeSize, -edgeSize, -edgeSize], [+edgeSize, +edgeSize, -edgeSize], [+edgeSize, +edgeSize, +edgeSize]],
      [p5.Tree.BOTTOM]: [[-edgeSize, +edgeSize, +edgeSize], [+edgeSize, +edgeSize, +edgeSize], [+edgeSize, +edgeSize, -edgeSize], [-edgeSize, +edgeSize, -edgeSize]],
      [p5.Tree.TOP]:    [[-edgeSize, -edgeSize, +edgeSize], [+edgeSize, -edgeSize, +edgeSize], [+edgeSize, -edgeSize, -edgeSize], [-edgeSize, -edgeSize, -edgeSize]],
    }
    
    if (faces[face]) {
      for (const [vx, vy, vz] of faces[face]) p.vertex(vx, vy, vz)
    }
    p.endShape(p.CLOSE)
    p.pop()
  }

  p.mouseWheel = () => false
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
    <p class="text-xs text-gray-400 italic">Non-euclidean rendering</p>
  </div>
</template>
