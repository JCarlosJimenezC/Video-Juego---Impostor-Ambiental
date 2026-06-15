<script setup>
import { ref, computed, onMounted } from 'vue'

const emit = defineEmits(['comenzar', 'volver'])

const palabras = ref([])
const cargando = ref(true)
const error = ref(null)

const nombreJugador = ref('')
const jugadores = ref([])
const categoriaFiltro = ref('Todos')
const categorias = ref([])
const palabraSeleccionada = ref(null)
const modoAleatorio = ref(true)

onMounted(async () => {
  try {
    const res = await fetch('/juego-data.json')
    if (!res.ok) throw new Error('No se pudo cargar el archivo de datos')
    const data = await res.json()
    palabras.value = data.palabras_ambientales
    categorias.value = data.categorias
    cargando.value = false
  } catch (e) {
    error.value = e.message
    cargando.value = false
  }
})

const palabrasFiltradas = computed(() => {
  if (categoriaFiltro.value === 'Todos') return palabras.value
  return palabras.value.filter(p => p.categoria === categoriaFiltro.value)
})

function agregarJugador() {
  const nombre = nombreJugador.value.trim()
  if (!nombre) return
  if (jugadores.value.length >= 8) return
  if (jugadores.value.some(j => j.nombre.toLowerCase() === nombre.toLowerCase())) return
  jugadores.value.push({ id: Date.now(), nombre })
  nombreJugador.value = ''
}

function eliminarJugador(id) {
  jugadores.value = jugadores.value.filter(j => j.id !== id)
}

function seleccionarPalabra(palabra) {
  if (!modoAleatorio.value) {
    palabraSeleccionada.value = palabra
  }
}

const puedeComenzar = computed(() => {
  return jugadores.value.length >= 3 && (modoAleatorio.value || palabraSeleccionada.value)
})

function comenzarJuego() {
  if (!puedeComenzar.value) return

  let palabraElegida
  if (modoAleatorio.value) {
    const lista = palabrasFiltradas.value
    palabraElegida = lista[Math.floor(Math.random() * lista.length)]
  } else {
    palabraElegida = palabraSeleccionada.value
  }

  // Asignar roles: un impostor aleatorio, resto informados
  const shuffled = [...jugadores.value].sort(() => Math.random() - 0.5)
  const impostorIdx = Math.floor(Math.random() * shuffled.length)
  const jugadoresConRol = shuffled.map((j, idx) => ({
    ...j,
    rol: idx === impostorIdx ? 'impostor' : 'informado',
    pista: '',
    votos: 0
  }))

  emit('comenzar', {
    palabra: palabraElegida,
    jugadores: jugadoresConRol,
    impostorId: jugadoresConRol[impostorIdx].id
  })
}

function teclaEnter(e) {
  if (e.key === 'Enter') agregarJugador()
}
</script>

<template>
  <div class="config-pantalla">
    <div class="config-contenido fade-in">
      <div class="config-header">
        <button class="btn-volver" @click="emit('volver')">← Volver</button>
        <h1>⚙️ Configurar Partida</h1>
      </div>

      <!-- Estado de carga -->
      <div v-if="cargando" class="cargando">
        <div class="spinner"></div>
        <p>Cargando palabras ambientales...</p>
      </div>

      <div v-else-if="error" class="error-msg">
        <p>⚠️ Error: {{ error }}</p>
        <button class="btn-secundario" @click="emit('volver')">Volver</button>
      </div>

      <template v-else>
        <!-- Sección jugadores -->
        <section class="card seccion-jugadores">
          <h2>🎮 Jugadores ({{ jugadores.length }}/8)</h2>
          <p class="hint">Mínimo 3 jugadores para comenzar</p>

          <div class="input-grupo">
            <input
              v-model="nombreJugador"
              @keydown="teclaEnter"
              placeholder="Nombre del jugador..."
              maxlength="20"
              class="input-nombre"
            />
            <button
              class="btn-primario btn-agregar"
              @click="agregarJugador"
              :disabled="!nombreJugador.trim() || jugadores.length >= 8"
            >
              + Agregar
            </button>
          </div>

          <div class="lista-jugadores">
            <TransitionGroup name="jugador">
              <div v-for="(j, idx) in jugadores" :key="j.id" class="jugador-chip">
                <span class="jugador-numero">{{ idx + 1 }}</span>
                <span class="jugador-nombre">{{ j.nombre }}</span>
                <button class="btn-eliminar" @click="eliminarJugador(j.id)" title="Eliminar">✕</button>
              </div>
            </TransitionGroup>
          </div>
        </section>

        <!-- Sección palabra -->
        <section class="card seccion-palabra">
          <h2>🌿 Palabra Secreta</h2>

          <div class="modo-seleccion">
            <label class="modo-btn" :class="{ activo: modoAleatorio }">
              <input type="radio" v-model="modoAleatorio" :value="true" /> 🎲 Aleatoria
            </label>
            <label class="modo-btn" :class="{ activo: !modoAleatorio }">
              <input type="radio" v-model="modoAleatorio" :value="false" /> 🔍 Elegir yo
            </label>
          </div>

          <template v-if="modoAleatorio">
            <p class="hint">Se elegirá una palabra al azar de la categoría seleccionada.</p>
            <div class="filtros-categoria">
              <button
                v-for="cat in categorias"
                :key="cat"
                class="chip-categoria"
                :class="{ activo: categoriaFiltro === cat }"
                @click="categoriaFiltro = cat"
              >
                {{ cat }}
              </button>
            </div>
            <p class="palabras-disponibles">{{ palabrasFiltradas.length }} palabras disponibles</p>
          </template>

          <template v-else>
            <div class="filtros-categoria">
              <button
                v-for="cat in categorias"
                :key="cat"
                class="chip-categoria"
                :class="{ activo: categoriaFiltro === cat }"
                @click="categoriaFiltro = cat"
              >
                {{ cat }}
              </button>
            </div>
            <div class="grid-palabras">
              <button
                v-for="p in palabrasFiltradas"
                :key="p.id"
                class="palabra-opcion"
                :class="{ seleccionada: palabraSeleccionada?.id === p.id }"
                @click="seleccionarPalabra(p)"
              >
                <span class="palabra-texto">{{ p.palabra }}</span>
                <span class="palabra-cat">{{ p.categoria }}</span>
              </button>
            </div>
          </template>
        </section>

        <!-- Botón comenzar -->
        <button
          class="btn-primario btn-comenzar"
          :disabled="!puedeComenzar"
          @click="comenzarJuego"
        >
          🌍 ¡Empezar Partida!
        </button>

        <p v-if="!puedeComenzar" class="aviso-requisito">
          {{ jugadores.length < 3 ? 'Necesitas al menos 3 jugadores' : 'Selecciona una palabra para continuar' }}
        </p>
      </template>
    </div>
  </div>
</template>

<style scoped>
.config-pantalla {
  min-height: 100vh;
  padding: 20px;
  background: radial-gradient(ellipse at top left, #0d2a0d 0%, var(--azul-oscuro) 50%);
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.config-contenido {
  max-width: 800px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding-bottom: 40px;
}

.config-header {
  display: flex;
  align-items: center;
  gap: 16px;
  padding-top: 10px;
}

.config-header h1 {
  font-size: 1.8rem;
  background: linear-gradient(135deg, var(--verde-brillante), var(--verde-claro));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.btn-volver {
  background: transparent;
  color: var(--texto-gris);
  border: 1px solid var(--borde);
  padding: 8px 14px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
  font-family: inherit;
  transition: all 0.2s;
}

.btn-volver:hover {
  color: var(--texto-principal);
  border-color: var(--verde-claro);
}

.cargando {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 60px 20px;
  color: var(--texto-gris);
}

.spinner {
  width: 40px;
  height: 40px;
  border: 3px solid var(--borde);
  border-top-color: var(--verde-claro);
  border-radius: 50%;
  animation: girar 0.8s linear infinite;
}

.error-msg {
  text-align: center;
  padding: 40px;
  color: var(--rojo-claro);
}

.seccion-jugadores h2,
.seccion-palabra h2 {
  color: var(--verde-brillante);
  margin-bottom: 6px;
  font-size: 1.2rem;
}

.hint {
  color: var(--texto-gris);
  font-size: 0.85rem;
  margin-bottom: 14px;
}

.input-grupo {
  display: flex;
  gap: 10px;
  margin-bottom: 16px;
}

.input-nombre {
  flex: 1;
  background: rgba(255,255,255,0.05);
  border: 1px solid var(--borde);
  border-radius: 8px;
  padding: 10px 14px;
  color: var(--texto-principal);
  font-size: 1rem;
  font-family: inherit;
  transition: border-color 0.2s;
}

.input-nombre:focus {
  outline: none;
  border-color: var(--verde-claro);
}

.input-nombre::placeholder { color: var(--texto-gris); }

.btn-agregar { padding: 10px 20px; white-space: nowrap; }

.lista-jugadores {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  min-height: 44px;
}

.jugador-chip {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(76, 175, 80, 0.1);
  border: 1px solid rgba(76, 175, 80, 0.3);
  border-radius: 20px;
  padding: 6px 12px;
  font-size: 0.95rem;
}

.jugador-numero {
  color: var(--verde-brillante);
  font-weight: 700;
  font-size: 0.85rem;
}

.jugador-nombre { color: var(--texto-principal); }

.btn-eliminar {
  background: none;
  border: none;
  color: var(--rojo-claro);
  cursor: pointer;
  font-size: 0.85rem;
  padding: 0 2px;
  line-height: 1;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.btn-eliminar:hover { opacity: 1; }

/* Transición lista jugadores */
.jugador-enter-active, .jugador-leave-active { transition: all 0.3s ease; }
.jugador-enter-from { opacity: 0; transform: scale(0.8); }
.jugador-leave-to { opacity: 0; transform: scale(0.8); }

.modo-seleccion {
  display: flex;
  gap: 10px;
  margin-bottom: 14px;
}

.modo-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 18px;
  border: 2px solid var(--borde);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  color: var(--texto-gris);
  font-size: 0.95rem;
}

.modo-btn input { display: none; }

.modo-btn.activo {
  border-color: var(--verde-claro);
  background: rgba(76, 175, 80, 0.1);
  color: var(--verde-claro);
}

.filtros-categoria {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 14px;
}

.chip-categoria {
  padding: 5px 12px;
  border-radius: 20px;
  border: 1px solid var(--borde);
  background: transparent;
  color: var(--texto-gris);
  font-size: 0.82rem;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.2s;
}

.chip-categoria.activo, .chip-categoria:hover {
  border-color: var(--verde-claro);
  color: var(--verde-claro);
  background: rgba(76, 175, 80, 0.1);
}

.palabras-disponibles {
  color: var(--texto-gris);
  font-size: 0.85rem;
  margin-top: 4px;
}

.grid-palabras {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 8px;
  max-height: 260px;
  overflow-y: auto;
  padding-right: 4px;
}

.palabra-opcion {
  display: flex;
  flex-direction: column;
  padding: 10px 14px;
  background: rgba(255,255,255,0.03);
  border: 1px solid var(--borde);
  border-radius: 8px;
  cursor: pointer;
  font-family: inherit;
  text-align: left;
  transition: all 0.2s;
}

.palabra-opcion:hover {
  border-color: var(--verde-claro);
  background: rgba(76, 175, 80, 0.08);
}

.palabra-opcion.seleccionada {
  border-color: var(--verde-brillante);
  background: rgba(105, 240, 174, 0.1);
}

.palabra-texto {
  color: var(--texto-principal);
  font-weight: 600;
  font-size: 0.95rem;
}

.palabra-cat {
  color: var(--texto-gris);
  font-size: 0.75rem;
  margin-top: 3px;
}

.btn-comenzar {
  width: 100%;
  padding: 18px;
  font-size: 1.15rem;
  border-radius: 12px;
}

.aviso-requisito {
  text-align: center;
  color: var(--texto-gris);
  font-size: 0.88rem;
}

@media (max-width: 600px) {
  .config-header h1 { font-size: 1.4rem; }
  .grid-palabras { grid-template-columns: 1fr 1fr; }
}
</style>
