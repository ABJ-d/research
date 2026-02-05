<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import p5 from 'p5'

const sketchContainer = ref(null)
let p5Instance = null

const sketch = (p) => {
  let edge = 80
  let teapot, bunny, texShader
  let teapotTex, bunnyTex
  
  // Define cameras explicitly to manage references manually
  // 'mainCam' is controlled by the user (orbitControl)
  // 'fboCam' is a "slave" camera used inside Framebuffers to bypass v2 beta bugs
  let mainCam, fboCam

  // --- Shader Definitions ---

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
      // Calculate normalized coordinates
      outColor = texture(model, gl_FragCoord.xy / u_resolution);
    }`

  p.setup = async function () {
    const canvas = p.createCanvas(400, 400, p.WEBGL)
    canvas.parent(sketchContainer.value)
    
    // Load 3D assets asynchronously
    teapot = await p.loadModel('teapot.obj', true)
    bunny = await p.loadModel('bunny.obj', true)
    
    // Initialize cameras
    mainCam = p.createCamera()
    fboCam = p.createCamera()

    // Create Framebuffers (FBOs) to hold the textures
    teapotTex = p.createFramebuffer({ format: p.FLOAT })
    bunnyTex = p.createFramebuffer({ format: p.FLOAT })

    // Initialize Shader
    texShader = p.createShader(vertSrc, fragSrc)
    p.shader(texShader)
    texShader.setUniform('u_resolution', p.resolution())
    
    console.log("Using p5 version:", p.VERSION)
  }

  p.draw = function () {
    p.background(20)
    
    // 1. Update Main Camera
    // We only apply orbitControl to the main camera
    p.setCamera(mainCam)
    p.orbitControl()

    // 2. Camera Synchronization (The Fix)
    // Manually link the slave camera's matrices to the main camera.
    // This bypasses p5.js v2 beta bugs regarding p.camera() getters in FBOs.
    fboCam.cameraMatrix = mainCam.cameraMatrix
    fboCam.projMatrix = mainCam.projMatrix

    p.resetShader()

    // 3. Render Pass: Front Face (Bunny) 
    bunnyTex.begin()
    p.clear()
    p.setCamera(fboCam) // Activate the synchronized slave camera
    p.push()
    p.noStroke()
    p.fill('red')
    p.scale(1, -1) // Correction for model coordinate system
    p.scale(0.8)
    p.model(bunny)
    p.pop()
    bunnyTex.end()

    // 4. Render Pass: Right Face (Teapot) 
    teapotTex.begin()
    p.clear()
    p.setCamera(fboCam) // Re-use the synchronized camera
    p.push()
    p.noStroke()
    p.fill('blue')
    p.scale(1, -1)
    p.model(teapot)
    p.pop()
    teapotTex.end()

    // 5. Final Scene Composition
    // Switch back to the main camera to draw the cube faces
    p.setCamera(mainCam)
    
    p.shader(texShader)
    p.push()
    p.stroke('purple')
    p.strokeWeight(2)

    // Apply the Bunny FBO texture to the FRONT face
    texShader.setUniform('model', bunnyTex.color)
    cubeFace('FRONT', edge)

    // Apply the Teapot FBO texture to the RIGHT face
    texShader.setUniform('model', teapotTex.color)
    cubeFace('RIGHT', edge)
    
    p.pop()
  }

  // --- Custom Geometry Helper ---
  // Draws specific cube faces without relying on external libraries like p5.tree
  function cubeFace(faceName, r) {
    // r = radius (half of the edge length)
    p.push()
    p.beginShape()
    
    // Manual vertex definition for a cube centered at (0,0,0).
    // The vertex order is crucial for correct texture mapping 
    // (The shader relies on implicit UV generation based on screen coords).
    const faces = {
      'FRONT':  [[-r, -r, +r], [+r, -r, +r], [+r, +r, +r], [-r, +r, +r]], // Z+
      'BACK':   [[-r, -r, -r], [+r, -r, -r], [+r, +r, -r], [-r, +r, -r]], // Z-
      'LEFT':   [[-r, -r, -r], [-r, -r, +r], [-r, +r, +r], [-r, +r, -r]], // X-
      'RIGHT':  [[+r, -r, +r], [+r, -r, -r], [+r, +r, -r], [+r, +r, +r]], // X+
      'BOTTOM': [[-r, +r, +r], [+r, +r, +r], [+r, +r, -r], [-r, +r, -r]], // Y+
      'TOP':    [[-r, -r, +r], [+r, -r, +r], [+r, -r, -r], [-r, -r, -r]], // Y-
    }
    
    if (faces[faceName]) {
      for (const [vx, vy, vz] of faces[faceName]) {
        p.vertex(vx, vy, vz)
      }
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
