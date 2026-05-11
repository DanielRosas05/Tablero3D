<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { Chess } from 'chess.js'

const contenedor3D = ref(null)

let escena, camara, renderizador, controles, animacionId
let raycaster, raton
let tableroMeshes = []
let piezas = []
let piezaSeleccionada = null
let highlightedSquares = []   // ← Nuevo: meshes de resaltado

const chess = new Chess()

const tamañoCasilla = 4
const offset = -14

// ===================== UTILIDADES =====================
function squareToPosition(square) {
  const col = square.charCodeAt(0) - 97
  const fila = 8 - parseInt(square[1])
  return {
    x: col * tamañoCasilla + offset,
    z: fila * tamañoCasilla + offset
  }
}

// ===================== INICIALIZACIÓN =====================
onMounted(() => {
  initEscena()
  crearTablero()
  actualizarPiezas()

  controles = new OrbitControls(camara, renderizador.domElement)
  controles.enableDamping = true
  controles.dampingFactor = 0.08
  controles.enablePan = false

  raycaster = new THREE.Raycaster()
  raton = new THREE.Vector2()

  renderizador.domElement.addEventListener('mousedown', onClick)
  animate()
})

function initEscena() {
  escena = new THREE.Scene()
  escena.background = new THREE.Color(0x1a1a1a)

  escena.add(new THREE.AmbientLight(0xffffff, 0.75))

  const dirLight = new THREE.DirectionalLight(0xffffff, 1.4)
  dirLight.position.set(20, 50, 30)
  dirLight.castShadow = true
  escena.add(dirLight)

  camara = new THREE.PerspectiveCamera(45, 900 / 600, 0.1, 200)
  camara.position.set(25, 45, 55)
  camara.lookAt(0, 0, 0)

  renderizador = new THREE.WebGLRenderer({ antialias: true })
  renderizador.setSize(900, 600)
  renderizador.shadowMap.enabled = true
  contenedor3D.value.appendChild(renderizador.domElement)
}

function crearTablero() {
  tableroMeshes = []
  const geo = new THREE.BoxGeometry(tamañoCasilla, 0.8, tamañoCasilla)

  for (let fila = 0; fila < 8; fila++) {
    for (let col = 0; col < 8; col++) {
      const esBlanca = (fila + col) % 2 === 0
      const material = new THREE.MeshStandardMaterial({
        color: esBlanca ? 0xede9d5 : 0x5c4033,
        roughness: 0.85,
      })

      const casilla = new THREE.Mesh(geo, material)
      casilla.position.set(col * tamañoCasilla + offset, 0, fila * tamañoCasilla + offset)
      casilla.userData.square = String.fromCharCode(97 + col) + (8 - fila)
      casilla.receiveShadow = true
      escena.add(casilla)
      tableroMeshes.push(casilla)
    }
  }
}

// ===================== PIEZAS =====================
function crearGeometriaPieza(tipo) { /* ... (se mantiene igual) */ 
  // (Mantengo tu función original para no alargar mucho)
  let puntos = []
  const crearBase = (ancho) => {
    puntos.push(new THREE.Vector2(0, 0))
    puntos.push(new THREE.Vector2(ancho, 0))
    puntos.push(new THREE.Vector2(ancho, 0.4))
    puntos.push(new THREE.Vector2(ancho * 0.8, 0.6))
    puntos.push(new THREE.Vector2(ancho * 0.7, 1.2))
  }

  switch (tipo) {
    case 'p': crearBase(1.2); puntos.push(new THREE.Vector2(0.6, 2)); puntos.push(new THREE.Vector2(0.9, 2.8)); puntos.push(new THREE.Vector2(0, 3.2)); break;
    case 'r': crearBase(1.4); puntos.push(new THREE.Vector2(1, 3.5)); puntos.push(new THREE.Vector2(1.3, 3.8)); break;
    case 'b': crearBase(1.3); puntos.push(new THREE.Vector2(0.6, 3)); puntos.push(new THREE.Vector2(0.8, 4)); puntos.push(new THREE.Vector2(0.2, 4.8)); puntos.push(new THREE.Vector2(0, 5)); break;
    case 'k': crearBase(1.5); puntos.push(new THREE.Vector2(0.8, 4)); puntos.push(new THREE.Vector2(1.2, 5)); puntos.push(new THREE.Vector2(0, 5.2)); break;
    case 'q': crearBase(1.5); puntos.push(new THREE.Vector2(0.7, 3.5)); puntos.push(new THREE.Vector2(1.3, 4.8)); puntos.push(new THREE.Vector2(0, 5.1)); break;
    case 'n': return new THREE.BoxGeometry(2, 3.8, 2.4);
    default: return new THREE.BoxGeometry(1.8, 3, 1.8)
  }
  return new THREE.LatheGeometry(puntos, 24)
}

function crearPiezaMesh(piezaInfo, square) {
  const group = new THREE.Group()
  const material = new THREE.MeshStandardMaterial({
    color: piezaInfo.color === 'w' ? 0xfafafa : 0x222222,
    metalness: 0.35,
    roughness: 0.25,
  })

  const geo = crearGeometriaPieza(piezaInfo.type)
  const cuerpo = new THREE.Mesh(geo, material)
  cuerpo.castShadow = true
  cuerpo.receiveShadow = true
  group.add(cuerpo)

  if (piezaInfo.type === 'k') {
    const cruzH = new THREE.Mesh(new THREE.BoxGeometry(1.4, 0.3, 0.3), material)
    const cruzV = new THREE.Mesh(new THREE.BoxGeometry(0.3, 1.4, 0.3), material)
    cruzH.position.y = cruzV.position.y = 5.7
    group.add(cruzH, cruzV)
  }

  if (piezaInfo.type === 'r') {
    const corona = new THREE.Mesh(new THREE.CylinderGeometry(1.45, 1.35, 0.7, 8), material)
    corona.position.y = 4.1
    group.add(corona)
  }

  const { x, z } = squareToPosition(square)
  group.position.set(x, 0.4, z)
  group.userData = { ...piezaInfo, square }

  return group
}

function actualizarPiezas() {
  piezas.forEach(p => p.parent?.remove(p))
  piezas = []

  const board = chess.board()
  for (let fila = 0; fila < 8; fila++) {
    for (let col = 0; col < 8; col++) {
      const pieza = board[fila][col]
      if (pieza) {
        const square = String.fromCharCode(97 + col) + (8 - fila)
        const mesh = crearPiezaMesh(pieza, square)
        escena.add(mesh)
        piezas.push(mesh)
      }
    }
  }
}

// ===================== RESALTADO DE MOVIMIENTOS =====================
function mostrarMovimientosLegales() {
  limpiarResaltados()

  if (!piezaSeleccionada) return

  const from = piezaSeleccionada.userData.square
  const movimientos = chess.moves({ square: from, verbose: true })

  movimientos.forEach(move => {
    const casilla = tableroMeshes.find(c => c.userData.square === move.to)
    if (!casilla) return

    // Crear resaltado
    const highlightGeo = new THREE.PlaneGeometry(tamañoCasilla * 0.9, tamañoCasilla * 0.9)
    const highlightMat = new THREE.MeshBasicMaterial({
      color: 0x00ff88,
      transparent: true,
      opacity: 0.35,
      side: THREE.DoubleSide
    })

    const highlight = new THREE.Mesh(highlightGeo, highlightMat)
    highlight.rotation.x = -Math.PI / 2
    highlight.position.copy(casilla.position)
    highlight.position.y = 0.45  // justo encima del tablero

    escena.add(highlight)
    highlightedSquares.push(highlight)
  })
}

function limpiarResaltados() {
  highlightedSquares.forEach(h => {
    if (h.parent) h.parent.remove(h)
  })
  highlightedSquares = []
}

// ===================== INTERACCIÓN =====================
function onClick(event) {
  const rect = contenedor3D.value.getBoundingClientRect()
  raton.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  raton.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(raton, camara)

  const intersectsPiezas = raycaster.intersectObjects(piezas, true)
  if (intersectsPiezas.length > 0) {
    let obj = intersectsPiezas[0].object
    while (obj && !obj.userData?.square) obj = obj.parent

    if (obj?.userData?.color === chess.turn()) {
      seleccionarPieza(obj)
      return
    }
  }

  if (piezaSeleccionada) {
    const intersectsTablero = raycaster.intersectObjects(tableroMeshes)
    if (intersectsTablero.length > 0) {
      intentarMover(intersectsTablero[0].object.userData.square)
    }
  }
}

function seleccionarPieza(pieza) {
  // Quitar highlight anterior
  if (piezaSeleccionada) {
    piezaSeleccionada.traverse(c => {
      if (c.material?.emissive) c.material.emissive.setHex(0x000000)
    })
  }

  piezaSeleccionada = pieza
  pieza.traverse(c => {
    if (c.material?.emissive) c.material.emissive.setHex(0x00ff88)
  })

  mostrarMovimientosLegales()   // ← Aquí se muestran los movimientos
}

function intentarMover(toSquare) {
  try {
    const move = chess.move({
      from: piezaSeleccionada.userData.square,
      to: toSquare,
      promotion: 'q'
    })

    if (move) {
      limpiarResaltados()
      actualizarPiezas()
      piezaSeleccionada = null
    }
  } catch (e) {
    console.log("Movimiento inválido")
  }
}

function animate() {
  animacionId = requestAnimationFrame(animate)
  controles?.update()
  renderizador.render(escena, camara)
}

onBeforeUnmount(() => {
  cancelAnimationFrame(animacionId)
  if (renderizador) renderizador.dispose()
})
</script>


<template>
  <div class="ajedrez-container">
    <div ref="contenedor3D" class="canvas-container"></div>
  </div>
</template>

<style scoped>
.ajedrez-container {
  display: flex ;
  justify-content: center;
  
  padding: 20px;
}

.canvas-container {
  width: 900px;
  height: 600px;
  border: 3px solid #0f0000;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgb(255, 255, 255);
}
</style>