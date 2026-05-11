<template>
  <canvas ref="bgCanvas" class="background-canvas"></canvas>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const bgCanvas = ref(null)
let ctx, particles, animationId
const mouse = { x: null, y: null, radius: 150 } // Radio de interacción

class Particle {
  constructor(w, h) {
    this.w = w
    this.h = h
    this.x = Math.random() * w
    this.y = Math.random() * h
    this.size = Math.random() * 1.5 + 0.5
    this.speedX = Math.random() * 0.4 - 0.2
    this.speedY = Math.random() * 0.4 - 0.2
  }

  update() {
    // Movimiento básico
    this.x += this.speedX
    this.y += this.speedY

    // Rebote en bordes
    if (this.x > this.w || this.x < 0) this.speedX *= -1
    if (this.y > this.h || this.y < 0) this.speedY *= -1

    // Interacción con el mouse (se alejan sutilmente)
    const dx = mouse.x - this.x
    const dy = mouse.y - this.y
    const distance = Math.sqrt(dx * dx + dy * dy)
    if (distance < mouse.radius) {
      if (mouse.x < this.x && this.x < this.w - 10) this.x += 2
      if (mouse.x > this.x && this.x > 10) this.x -= 2
      if (mouse.y < this.y && this.y < this.h - 10) this.y += 2
      if (mouse.y > this.y && this.y > 10) this.y -= 2
    }
  }

  draw() {
    ctx.fillStyle = 'rgba(255, 0, 52, 1)' // Color verde neón
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2)
    ctx.fill()
  }
}

// Función para dibujar líneas entre partículas cercanas
const drawLines = () => {
  for (let a = 0; a < particles.length; a++) {
    for (let b = a; b < particles.length; b++) {
      const dx = particles[a].x - particles[b].x
      const dy = particles[a].y - particles[b].y
      const distance = Math.sqrt(dx * dx + dy * dy)

      if (distance < 100) {
        ctx.strokeStyle = `rgba(0, 0, 0, 1 ${1 - distance / 100})`
        ctx.lineWidth = 0.5
        ctx.beginPath()
        ctx.moveTo(particles[a].x, particles[a].y)
        ctx.lineTo(particles[b].x, particles[b].y)
        ctx.stroke()
      }
    }
  }
}

const init = () => {
  const w = bgCanvas.value.width = window.innerWidth
  const h = bgCanvas.value.height = window.innerHeight
  // Aumentamos a 100 partículas para que la red se vea poblada
  particles = Array.from({ length: 100 }, () => new Particle(w, h))
}

const animate = () => {
  ctx.clearRect(0, 0, bgCanvas.value.width, bgCanvas.value.height)
  
  particles.forEach(p => {
    p.update()
    p.draw()
  })
  drawLines()
  
  animationId = requestAnimationFrame(animate)
}

const handleMouseMove = (e) => {
  mouse.x = e.clientX
  mouse.y = e.clientY
}

const handleMouseLeave = () => {
  mouse.x = null
  mouse.y = null
}

onMounted(() => {
  ctx = bgCanvas.value.getContext('2d')
  init()
  animate()
  window.addEventListener('resize', init)
  window.addEventListener('mousemove', handleMouseMove)
  window.addEventListener('mouseleave', handleMouseLeave)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
  window.removeEventListener('resize', init)
  window.removeEventListener('mousemove', handleMouseMove)
  window.removeEventListener('mouseleave', handleMouseLeave)
})
</script>

<style scoped>
.background-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: -2;
  /* Fondo con profundidad */
  background: radial-gradient(circle at center, #000000 0%, #ffffff 100%);
}
</style>