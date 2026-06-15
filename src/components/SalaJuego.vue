<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import TarjetaJugador from './TarjetaJugador.vue'

const props = defineProps({
  config: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['fin-juego', 'volver'])

// Estado del juego
const fase = ref('asignacion') // 'asignacion' | 'pistas' | 'adivinanza' | 'votacion' | 'resultado_local'
const jugadorActualIdx = ref(0)
const jugadores = ref(JSON.parse(JSON.stringify(props.config.jugadores)))
const palabraAdivinada = ref('')
const impostorVotado = ref(null)
const jugadorViendo = ref(null) // para mostrar rol privado
const rolMostrado = ref(false)
const tiempoRestante = ref(90)
const timerActivo = ref(false)
let intervalo = null

// Audio refs para efectos de sonido
const audioClick = ref(null)
const audioAcierto = ref(null)
const audioError = ref(null)
const audioVotacion = ref(null)

const jugadorActual = computed(() => jugadores.value[jugadorActualIdx.value])

const impostor = computed(() => jugadores.value.find(j => j.rol === 'impostor'))

const todosDeronPista = computed(() => jugadores.value.every(j => j.pista.trim() !== ''))

function reproducir(audioRef) {
  if (audioRef?.value) {
    audioRef.value.currentTime = 0
    audioRef.value.play().catch(() => {})
  }
}

// FASE: Asignación de roles - muestra rol a cada jugador en privado
function verMiRol() {
  jugadorViendo.value = jugadorActual.value
  rolMostrado.value = true
  reproducir(audioClick)
}

function rolVisto() {
  rolMostrado.value = false
  jugadorViendo.value = null
  if (jugadorActualIdx.value < jugadores.value.length - 1) {
    jugadorActualIdx.value++
  } else {
    // Todos vieron su rol, pasar a pistas
    iniciarFasePistas()
  }
}

function iniciarFasePistas() {
  fase.value = 'pistas'
  jugadorActualIdx.value = 0
  iniciarTimer()
}

// Timer para pistas
function iniciarTimer() {
  tiempoRestante.value = 90
  timerActivo.value = true
  clearInterval(intervalo)
  intervalo = setInterval(() => {
    if (tiempoRestante.value > 0) {
      tiempoRestante.value--
    } else {
      clearInterval(intervalo)
      timerActivo.value = false
    }
  }, 1000)
}

const timerColor = computed(() => {
  if (tiempoRestante.value > 30) return '#4caf50'
  if (tiempoRestante.value > 10) return '#ff9800'
  return '#f44336'
})

const timerPorcentaje = computed(() => (tiempoRestante.value / 90) * 100)

// FASE: Pistas - cada jugador escribe su pista
function confirmarPista() {
  if (!jugadorActual.value.pista.trim()) return
  reproducir(audioClick)
  if (jugadorActualIdx.value < jugadores.value.length - 1) {
    jugadorActualIdx.value++
  } else {
    clearInterval(intervalo)
    timerActivo.value = false
    fase.value = 'adivinanza'
  }
}

// FASE: Adivinanza
function pasarAVotacion() {
  if (!palabraAdivinada.value.trim()) return
  reproducir(audioVotacion)
  fase.value = 'votacion'
}

// FASE: Votación del impostor
function votar(jugador) {
  impostorVotado.value = jugador.id
  reproducir(audioClick)
}

function confirmarVoto() {
  if (!impostorVotado.value) return
  calcularResultado()
}

function calcularResultado() {
  const palabraCorrecta = palabraAdivinada.value.trim().toLowerCase() === props.config.palabra.palabra.toLowerCase()
  const impostorIdentificado = impostorVotado.value === impostor.value.id

  if (palabraCorrecta && impostorIdentificado) {
    reproducir(audioAcierto)
  } else {
    reproducir(audioError)
  }

  const puntos = {}
  jugadores.value.forEach(j => { puntos[j.id] = 0 })

  jugadores.value.forEach(j => {
    if (j.rol === 'impostor') {
      if (!impostorIdentificado) puntos[j.id] += 3
      if (palabraCorrecta && !impostorIdentificado) puntos[j.id] += 1
    } else {
      if (palabraCorrecta) puntos[j.id] += 2
      if (impostorIdentificado) puntos[j.id] += 1
    }
  })

  emit('fin-juego', {
    puntos,
    palabraCorrecta,
    impostorIdentificado,
    impostorId: impostor.value.id,
    palabraAdivinada: palabraAdivinada.value.trim(),
    impostorVotadoId: impostorVotado.value
  })
}

onMounted(() => {
  jugadorActualIdx.value = 0
})

onUnmounted(() => {
  clearInterval(intervalo)
})
</script>

<template>
  <div class="sala-juego">
    <!-- Audios ocultos para efectos de sonido -->
    <audio ref="audioClick" src="/audio/click.mp3" preload="auto"></audio>
    <audio ref="audioAcierto" src="/audio/acierto.mp3" preload="auto"></audio>
    <audio ref="audioError" src="/audio/error.mp3" preload="auto"></audio>
    <audio ref="audioVotacion" src="/audio/votacion.mp3" preload="auto"></audio>

    <div class="sala-contenido fade-in">

      <!-- === FASE: ASIGNACIÓN DE ROLES === -->
      <div v-if="fase === 'asignacion'" class="fase-asignacion">
        <h1>🎭 Asignación de Roles</h1>
        <p class="instruccion">Cada jugador verá su rol en privado. ¡No lo reveles!</p>

        <!-- Overlay de rol privado -->
        <div v-if="rolMostrado" class="overlay-rol">
          <div class="tarjeta-rol" :class="jugadorViendo?.rol">
            <div class="rol-icono">{{ jugadorViendo?.rol === 'impostor' ? '🎭' : '🌿' }}</div>
            <h2 class="rol-nombre">{{ jugadorViendo?.nombre }}</h2>
            <div class="rol-titulo">
              {{ jugadorViendo?.rol === 'impostor' ? 'ERES EL IMPOSTOR' : 'ERES INFORMADO' }}
            </div>
            <div v-if="jugadorViendo?.rol === 'informado'" class="rol-palabra">
              <p>La palabra secreta es:</p>
              <strong>{{ config.palabra.palabra }}</strong>
              <p class="rol-categoria">{{ config.palabra.categoria }}</p>
            </div>
            <div v-else class="rol-impostor-aviso">
              <p>NO conoces la palabra secreta.</p>
              <p>Escucha las pistas y da respuestas creíbles para pasar desapercibido.</p>
            </div>
            <button class="btn-primario btn-rol-visto" @click="rolVisto">
              ✓ Entendido, pasar el dispositivo
            </button>
          </div>
        </div>

        <!-- Lista de jugadores pendientes -->
        <div v-else class="lista-asignacion">
          <div
            v-for="(j, idx) in jugadores"
            :key="j.id"
            class="jugador-asignacion"
            :class="{
              'pendiente': idx === jugadorActualIdx,
              'listo': idx < jugadorActualIdx,
              'espera': idx > jugadorActualIdx
            }"
          >
            <span class="asig-estado">
              {{ idx < jugadorActualIdx ? '✓' : idx === jugadorActualIdx ? '▶' : '○' }}
            </span>
            <span class="asig-nombre">{{ j.nombre }}</span>
            <span class="asig-etiqueta">
              {{ idx < jugadorActualIdx ? 'Listo' : idx === jugadorActualIdx ? 'Turno' : 'Espera' }}
            </span>
          </div>
        </div>

        <button
          v-if="!rolMostrado"
          class="btn-primario btn-ver-rol"
          @click="verMiRol"
        >
          👁️ {{ jugadorActual?.nombre }}, ver mi rol
        </button>
      </div>

      <!-- === FASE: PISTAS === -->
      <div v-else-if="fase === 'pistas'" class="fase-pistas">
        <div class="pistas-header">
          <h1>💬 Ronda de Pistas</h1>
          <!-- Timer visual -->
          <div class="timer-bloque">
            <svg class="timer-svg" viewBox="0 0 60 60">
              <circle cx="30" cy="30" r="26" fill="none" stroke="rgba(255,255,255,0.1)" stroke-width="4"/>
              <circle
                cx="30" cy="30" r="26"
                fill="none"
                :stroke="timerColor"
                stroke-width="4"
                stroke-linecap="round"
                :stroke-dasharray="`${timerPorcentaje * 1.634} 163.4`"
                stroke-dashoffset="40.85"
                transform="rotate(-90 30 30)"
                style="transition: stroke-dasharray 1s linear, stroke 0.5s"
              />
            </svg>
            <span class="timer-num" :style="{ color: timerColor }">{{ tiempoRestante }}</span>
          </div>
        </div>

        <p class="instruccion">
          Turno de <strong>{{ jugadorActual?.nombre }}</strong>: escribe tu pista sobre la palabra secreta.
        </p>

        <!-- Pistas ya escritas -->
        <div v-if="jugadorActualIdx > 0" class="pistas-anteriores">
          <p class="pistas-anteriores-titulo">Pistas anteriores:</p>
          <div class="pistas-lista">
            <div v-for="j in jugadores.slice(0, jugadorActualIdx)" :key="j.id" class="pista-previa">
              <span class="pista-autor">{{ j.nombre }}:</span>
              <span class="pista-texto">"{{ j.pista }}"</span>
            </div>
          </div>
        </div>

        <!-- Input pista actual -->
        <div class="input-pista-bloque">
          <div class="avatar-jugador">{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][jugadorActual?.id % 8] }}</div>
          <div class="input-pista-grupo">
            <label class="pista-label">Tu pista, {{ jugadorActual?.nombre }}:</label>
            <input
              v-model="jugadores[jugadorActualIdx].pista"
              placeholder="Escribe una pista relacionada con la palabra..."
              class="input-pista"
              @keydown.enter="confirmarPista"
              maxlength="80"
            />
          </div>
        </div>

        <div class="pistas-acciones">
          <button
            class="btn-primario"
            @click="confirmarPista"
            :disabled="!jugadores[jugadorActualIdx]?.pista?.trim()"
          >
            {{ jugadorActualIdx < jugadores.length - 1 ? '→ Siguiente jugador' : '✓ Terminar ronda' }}
          </button>
        </div>

        <!-- Progreso -->
        <div class="progreso-pistas">
          <span v-for="(j, idx) in jugadores" :key="j.id" class="punto-progreso" :class="{ activo: idx <= jugadorActualIdx, hecho: idx < jugadorActualIdx }"></span>
        </div>
      </div>

      <!-- === FASE: ADIVINANZA === -->
      <div v-else-if="fase === 'adivinanza'" class="fase-adivinanza">
        <h1>🤔 ¿Cuál es la palabra?</h1>
        <p class="instruccion">Revisen todas las pistas y lleguen a un consenso. Luego escriban la palabra.</p>

        <div class="resumen-pistas card">
          <h3>📋 Pistas dadas:</h3>
          <div class="pistas-completas">
            <div v-for="j in jugadores" :key="j.id" class="pista-completa">
              <span class="pista-avatar">{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][j.id % 8] }}</span>
              <div>
                <strong>{{ j.nombre }}:</strong>
                <span> "{{ j.pista || '(sin pista)' }}"</span>
              </div>
            </div>
          </div>
        </div>

        <div class="input-adivinanza">
          <label>La palabra secreta es:</label>
          <input
            v-model="palabraAdivinada"
            placeholder="Escribe la palabra..."
            class="input-respuesta"
            @keydown.enter="pasarAVotacion"
          />
        </div>

        <button
          class="btn-primario btn-votar"
          @click="pasarAVotacion"
          :disabled="!palabraAdivinada.trim()"
        >
          🗳️ Pasar a Votación del Impostor
        </button>
      </div>

      <!-- === FASE: VOTACIÓN === -->
      <div v-else-if="fase === 'votacion'" class="fase-votacion">
        <h1>🗳️ ¿Quién es el Impostor?</h1>
        <p class="instruccion">Discutan las pistas sospechosas. ¿Quién NO sabía la palabra?</p>

        <div class="pistas-referencia card">
          <h3>📋 Recuerden las pistas:</h3>
          <div class="pistas-mini">
            <span v-for="j in jugadores" :key="j.id" class="pista-mini-chip">
              <strong>{{ j.nombre }}:</strong> "{{ j.pista }}"
            </span>
          </div>
        </div>

        <h3 class="votar-titulo">Vota por el sospechoso:</h3>
        <div class="grid-votacion">
          <TarjetaJugador
            v-for="j in jugadores"
            :key="j.id"
            :jugador="j"
            modo="voto"
            :seleccionado="impostorVotado === j.id"
            @seleccionar="votar"
          />
        </div>

        <button
          class="btn-peligro btn-confirmar-voto"
          @click="confirmarVoto"
          :disabled="!impostorVotado"
        >
          🎯 Confirmar Voto y Ver Resultado
        </button>
      </div>

    </div>
  </div>
</template>

<style scoped>
.sala-juego {
  min-height: 100vh;
  padding: 20px;
  background: radial-gradient(ellipse at bottom right, #1a0a2e 0%, var(--azul-oscuro) 60%);
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.sala-contenido {
  max-width: 800px;
  width: 100%;
  padding-bottom: 40px;
}

h1 {
  font-size: 1.8rem;
  background: linear-gradient(135deg, var(--verde-brillante), #69b2f0);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin-bottom: 10px;
}

.instruccion {
  color: var(--texto-gris);
  margin-bottom: 24px;
  font-size: 1rem;
  line-height: 1.5;
}

/* Asignación */
.fase-asignacion {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  padding-top: 20px;
  text-align: center;
}

.overlay-rol {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.85);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  padding: 20px;
}

.tarjeta-rol {
  max-width: 420px;
  width: 100%;
  background: var(--azul-card);
  border-radius: 16px;
  padding: 32px 28px;
  text-align: center;
  border: 3px solid;
  animation: fadeIn 0.4s ease;
}

.tarjeta-rol.informado {
  border-color: var(--verde-claro);
  box-shadow: 0 0 40px rgba(76,175,80,0.3);
}

.tarjeta-rol.impostor {
  border-color: var(--rojo-claro);
  box-shadow: 0 0 40px rgba(229,57,53,0.3);
}

.rol-icono { font-size: 4rem; margin-bottom: 12px; }

.rol-nombre { font-size: 1.4rem; color: var(--texto-principal); margin-bottom: 8px; }

.rol-titulo {
  font-size: 1.3rem;
  font-weight: 800;
  letter-spacing: 2px;
  margin-bottom: 20px;
}

.tarjeta-rol.informado .rol-titulo { color: var(--verde-brillante); }
.tarjeta-rol.impostor .rol-titulo { color: var(--rojo-claro); }

.rol-palabra { margin-bottom: 20px; }
.rol-palabra p { color: var(--texto-gris); margin-bottom: 6px; }
.rol-palabra strong {
  font-size: 1.8rem;
  color: var(--verde-brillante);
  display: block;
  margin-bottom: 4px;
}
.rol-categoria { color: var(--texto-gris); font-size: 0.85rem; }

.rol-impostor-aviso {
  color: var(--texto-gris);
  margin-bottom: 20px;
  line-height: 1.6;
}

.btn-rol-visto { width: 100%; margin-top: 8px; }

.lista-asignacion {
  width: 100%;
  max-width: 400px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.jugador-asignacion {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-radius: 10px;
  border: 2px solid transparent;
  transition: all 0.3s;
}

.jugador-asignacion.pendiente {
  border-color: var(--verde-claro);
  background: rgba(76,175,80,0.1);
}

.jugador-asignacion.listo {
  border-color: rgba(76,175,80,0.2);
  opacity: 0.6;
}

.jugador-asignacion.espera {
  border-color: var(--borde);
  opacity: 0.4;
}

.asig-estado { font-weight: 700; color: var(--verde-brillante); width: 20px; }
.asig-nombre { flex: 1; font-weight: 600; }
.asig-etiqueta { font-size: 0.8rem; color: var(--texto-gris); }

.btn-ver-rol { padding: 16px 40px; font-size: 1.1rem; }

/* Pistas */
.fase-pistas { display: flex; flex-direction: column; gap: 16px; padding-top: 10px; }

.pistas-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}

.pistas-header h1 { margin-bottom: 0; }

.timer-bloque {
  position: relative;
  width: 60px;
  height: 60px;
  flex-shrink: 0;
}

.timer-svg { width: 100%; height: 100%; }

.timer-num {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  font-weight: 700;
}

.pistas-anteriores {
  background: rgba(255,255,255,0.03);
  border: 1px solid var(--borde);
  border-radius: 10px;
  padding: 14px;
}

.pistas-anteriores-titulo {
  color: var(--texto-gris);
  font-size: 0.85rem;
  margin-bottom: 8px;
}

.pistas-lista { display: flex; flex-direction: column; gap: 6px; }

.pista-previa {
  display: flex;
  gap: 8px;
  font-size: 0.9rem;
}

.pista-autor { color: var(--verde-brillante); font-weight: 600; }
.pista-texto { color: var(--texto-principal); }

.input-pista-bloque {
  display: flex;
  gap: 14px;
  align-items: flex-start;
  background: var(--azul-card);
  border: 1px solid var(--borde);
  border-radius: 12px;
  padding: 16px;
}

.avatar-jugador { font-size: 2.5rem; flex-shrink: 0; }

.input-pista-grupo { flex: 1; }

.pista-label {
  display: block;
  color: var(--verde-brillante);
  font-weight: 600;
  margin-bottom: 8px;
  font-size: 0.95rem;
}

.input-pista {
  width: 100%;
  background: rgba(255,255,255,0.05);
  border: 1px solid var(--borde);
  border-radius: 8px;
  padding: 10px 14px;
  color: var(--texto-principal);
  font-size: 0.95rem;
  font-family: inherit;
  transition: border-color 0.2s;
}

.input-pista:focus {
  outline: none;
  border-color: var(--verde-claro);
}

.input-pista::placeholder { color: var(--texto-gris); }

.pistas-acciones { display: flex; justify-content: flex-end; }

.progreso-pistas {
  display: flex;
  gap: 6px;
  justify-content: center;
  margin-top: 4px;
}

.punto-progreso {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: var(--borde);
  transition: background 0.3s;
}

.punto-progreso.activo { background: var(--verde-claro); }
.punto-progreso.hecho { background: var(--verde-medio); }

/* Adivinanza */
.fase-adivinanza { display: flex; flex-direction: column; gap: 20px; padding-top: 10px; }

.resumen-pistas h3 { color: var(--verde-brillante); margin-bottom: 14px; }

.pistas-completas { display: flex; flex-direction: column; gap: 10px; }

.pista-completa {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  font-size: 0.95rem;
}

.pista-avatar { font-size: 1.4rem; flex-shrink: 0; }

.input-adivinanza { display: flex; flex-direction: column; gap: 8px; }

.input-adivinanza label {
  color: var(--verde-brillante);
  font-weight: 600;
  font-size: 1rem;
}

.input-respuesta {
  width: 100%;
  background: rgba(255,255,255,0.05);
  border: 2px solid var(--verde-claro);
  border-radius: 10px;
  padding: 14px 18px;
  color: var(--texto-principal);
  font-size: 1.1rem;
  font-family: inherit;
}

.input-respuesta:focus { outline: none; border-color: var(--verde-brillante); }
.input-respuesta::placeholder { color: var(--texto-gris); }

.btn-votar { width: 100%; padding: 16px; font-size: 1.05rem; }

/* Votación */
.fase-votacion { display: flex; flex-direction: column; gap: 20px; padding-top: 10px; }

.pistas-referencia h3 { color: var(--verde-brillante); margin-bottom: 12px; font-size: 1rem; }

.pistas-mini { display: flex; flex-direction: column; gap: 6px; }

.pista-mini-chip { font-size: 0.85rem; color: var(--texto-gris); display: block; }
.pista-mini-chip strong { color: var(--texto-secundario); }

.votar-titulo { color: var(--texto-principal); font-size: 1rem; }

.grid-votacion {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 10px;
}

.btn-confirmar-voto { width: 100%; padding: 16px; font-size: 1.05rem; }

@media (max-width: 600px) {
  h1 { font-size: 1.4rem; }
  .grid-votacion { grid-template-columns: 1fr; }
  .pistas-header { flex-direction: column; align-items: flex-start; gap: 12px; }
}
</style>
