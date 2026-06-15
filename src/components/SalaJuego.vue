<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import TarjetaJugador from './TarjetaJugador.vue'

const props = defineProps({
  config: { type: Object, required: true }
})

const emit = defineEmits(['fin-juego', 'volver'])

// Fases: 'asignacion' | 'pistas' | 'resumen_ronda' | 'votacion'
const fase = ref('asignacion')
const rondaActual = ref(1)
const RONDAS_MINIMAS = 3

const jugadorActualIdx = ref(0)
const jugadores = ref(JSON.parse(JSON.stringify(props.config.jugadores)))

// Almacena pistas por ronda: { 1: { jugadorId: 'texto' }, 2: { ... } }
const pistasPorRonda = ref({})
const pistaActual = ref('')

const impostorVotado = ref(null)
const jugadorViendo = ref(null)
const rolMostrado = ref(false)

const tiempoRestante = ref(90)
let intervalo = null

const audioClick = ref(null)
const audioAcierto = ref(null)
const audioError = ref(null)
const audioVotacion = ref(null)

const jugadorActual = computed(() => jugadores.value[jugadorActualIdx.value])
const impostor = computed(() => jugadores.value.find(j => j.rol === 'impostor'))

function initRonda(n) {
  if (!pistasPorRonda.value[n]) {
    const mapa = {}
    jugadores.value.forEach(j => { mapa[j.id] = '' })
    pistasPorRonda.value[n] = mapa
  }
}

// Pistas ya dadas en la ronda actual (jugadores anteriores al actual)
const pistasRondaActual = computed(() => {
  const rPistas = pistasPorRonda.value[rondaActual.value] || {}
  return jugadores.value
    .slice(0, jugadorActualIdx.value)
    .map(j => ({ jugador: j, pista: rPistas[j.id] }))
    .filter(item => item.pista)
})

// Todas las pistas de todas las rondas, agrupadas por jugador
const pistasPorJugador = computed(() => {
  return jugadores.value.map(j => ({
    jugador: j,
    historial: Object.entries(pistasPorRonda.value)
      .sort(([a], [b]) => Number(a) - Number(b))
      .map(([ronda, pistas]) => ({ ronda: Number(ronda), pista: pistas[j.id] }))
      .filter(item => item.pista)
  }))
})

function reproducir(audioRef) {
  if (audioRef?.value) {
    audioRef.value.currentTime = 0
    audioRef.value.play().catch(() => {})
  }
}

// ─── FASE: ASIGNACIÓN ─────────────────────────────────────────────────────────
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
    iniciarRonda(1)
  }
}

// ─── FASE: PISTAS ─────────────────────────────────────────────────────────────
function iniciarRonda(n) {
  rondaActual.value = n
  initRonda(n)
  jugadorActualIdx.value = 0
  pistaActual.value = ''
  fase.value = 'pistas'
  iniciarTimer()
}

function iniciarTimer() {
  tiempoRestante.value = 90
  clearInterval(intervalo)
  intervalo = setInterval(() => {
    if (tiempoRestante.value > 0) {
      tiempoRestante.value--
    } else {
      clearInterval(intervalo)
    }
  }, 1000)
}

const timerColor = computed(() => {
  if (tiempoRestante.value > 30) return '#4caf50'
  if (tiempoRestante.value > 10) return '#ff9800'
  return '#f44336'
})

const timerPorcentaje = computed(() => (tiempoRestante.value / 90) * 100)

function confirmarPista() {
  const texto = pistaActual.value.trim()
  if (!texto) return
  reproducir(audioClick)

  pistasPorRonda.value[rondaActual.value][jugadorActual.value.id] = texto
  pistaActual.value = ''

  if (jugadorActualIdx.value < jugadores.value.length - 1) {
    jugadorActualIdx.value++
  } else {
    clearInterval(intervalo)
    fase.value = 'resumen_ronda'
  }
}

// ─── FASE: RESUMEN DE RONDA ───────────────────────────────────────────────────
function siguienteRonda() {
  iniciarRonda(rondaActual.value + 1)
}

function irAVotar() {
  reproducir(audioVotacion)
  fase.value = 'votacion'
}

// ─── FASE: VOTACIÓN ───────────────────────────────────────────────────────────
function votar(jugador) {
  impostorVotado.value = jugador.id
  reproducir(audioClick)
}

function confirmarVoto() {
  if (!impostorVotado.value) return
  calcularResultado()
}

function calcularResultado() {
  const impostorIdentificado = impostorVotado.value === impostor.value.id

  if (impostorIdentificado) {
    reproducir(audioAcierto)
  } else {
    reproducir(audioError)
  }

  const puntos = {}
  jugadores.value.forEach(j => { puntos[j.id] = 0 })
  jugadores.value.forEach(j => {
    if (j.rol === 'impostor') {
      if (!impostorIdentificado) puntos[j.id] += 3
    } else {
      if (impostorIdentificado) puntos[j.id] += 3
    }
  })

  // Construir jugadores con historial de pistas para los resultados
  const jugadoresConHistorial = jugadores.value.map(j => ({
    ...j,
    historialPistas: Object.entries(pistasPorRonda.value)
      .sort(([a], [b]) => Number(a) - Number(b))
      .map(([ronda, pistas]) => ({ ronda: Number(ronda), pista: pistas[j.id] || '' }))
      .filter(item => item.pista)
  }))

  emit('fin-juego', {
    puntos,
    impostorIdentificado,
    impostorId: impostor.value.id,
    impostorVotadoId: impostorVotado.value,
    jugadores: jugadoresConHistorial,
    totalRondas: rondaActual.value
  })
}

onMounted(() => { jugadorActualIdx.value = 0 })
onUnmounted(() => { clearInterval(intervalo) })
</script>

<template>
  <div class="sala-juego">
    <audio ref="audioClick" src="/audio/click.mp3" preload="auto"></audio>
    <audio ref="audioAcierto" src="/audio/acierto.mp3" preload="auto"></audio>
    <audio ref="audioError" src="/audio/error.mp3" preload="auto"></audio>
    <audio ref="audioVotacion" src="/audio/votacion.mp3" preload="auto"></audio>

    <!-- ═══ OVERLAY ROL (Teleport para centrado perfecto) ═══ -->
    <Teleport to="body">
      <Transition name="overlay">
        <div v-if="rolMostrado" class="overlay-rol" @click.self="null">
          <div class="tarjeta-rol" :class="jugadorViendo?.rol">
            <div class="rol-icono">{{ jugadorViendo?.rol === 'impostor' ? '🎭' : '🌿' }}</div>
            <h2 class="rol-nombre">{{ jugadorViendo?.nombre }}</h2>
            <div class="rol-titulo">
              {{ jugadorViendo?.rol === 'impostor' ? 'ERES EL IMPOSTOR' : 'ERES INFORMADO' }}
            </div>
            <div v-if="jugadorViendo?.rol === 'informado'" class="rol-palabra">
              <p class="rol-subtexto">La palabra secreta es:</p>
              <strong class="rol-palabra-texto">{{ config.palabra.palabra }}</strong>
              <p class="rol-categoria">{{ config.palabra.categoria }}</p>
            </div>
            <div v-else class="rol-impostor-aviso">
              <p>NO conoces la palabra secreta.</p>
              <p>Escucha las pistas de los demás y da respuestas creíbles para pasar desapercibido. ¡Buena suerte!</p>
            </div>
            <button class="btn-primario btn-rol-visto" @click="rolVisto">
              ✓ Entendido — pasar el dispositivo
            </button>
          </div>
        </div>
      </Transition>
    </Teleport>

    <div class="sala-contenido fade-in">

      <!-- ═══ ASIGNACIÓN DE ROLES ═══ -->
      <div v-if="fase === 'asignacion'" class="fase-asignacion">
        <h1>🎭 Asignación de Roles</h1>
        <p class="instruccion">Cada jugador verá su rol en privado. ¡No lo reveles a nadie!</p>

        <div class="lista-asignacion">
          <div
            v-for="(j, idx) in jugadores"
            :key="j.id"
            class="jugador-asignacion"
            :class="{
              pendiente: idx === jugadorActualIdx,
              listo: idx < jugadorActualIdx,
              espera: idx > jugadorActualIdx
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

        <button class="btn-primario btn-ver-rol" @click="verMiRol">
          👁️ {{ jugadorActual?.nombre }}, ver mi rol
        </button>
      </div>

      <!-- ═══ RONDA DE PISTAS ═══ -->
      <div v-else-if="fase === 'pistas'" class="fase-pistas">
        <div class="pistas-header">
          <div>
            <div class="ronda-badge">Ronda {{ rondaActual }} de {{ Math.max(rondaActual, RONDAS_MINIMAS) }}+</div>
            <h1>💬 Pistas</h1>
          </div>
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
          Turno de <strong>{{ jugadorActual?.nombre }}</strong>: escribe una pista sobre la palabra secreta.
        </p>

        <!-- Pistas de rondas anteriores -->
        <div v-if="rondaActual > 1" class="pistas-anteriores historial-rondas">
          <p class="pistas-anteriores-titulo">Pistas de rondas anteriores:</p>
          <div v-for="r in rondaActual - 1" :key="r" class="ronda-historial">
            <span class="ronda-label-mini">Ronda {{ r }}</span>
            <div class="pistas-lista">
              <div v-for="j in jugadores" :key="j.id" class="pista-previa">
                <span class="pista-autor">{{ j.nombre }}:</span>
                <span class="pista-texto">"{{ pistasPorRonda[r]?.[j.id] || '—' }}"</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Pistas ya dadas en esta ronda -->
        <div v-if="pistasRondaActual.length > 0" class="pistas-anteriores">
          <p class="pistas-anteriores-titulo">Esta ronda — pistas anteriores:</p>
          <div class="pistas-lista">
            <div v-for="item in pistasRondaActual" :key="item.jugador.id" class="pista-previa">
              <span class="pista-autor">{{ item.jugador.nombre }}:</span>
              <span class="pista-texto">"{{ item.pista }}"</span>
            </div>
          </div>
        </div>

        <!-- Input pista actual -->
        <div class="input-pista-bloque">
          <div class="avatar-jugador">{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][jugadorActual?.id % 8] }}</div>
          <div class="input-pista-grupo">
            <label class="pista-label">Tu pista, {{ jugadorActual?.nombre }}:</label>
            <input
              v-model="pistaActual"
              placeholder="Escribe una pista sobre la palabra secreta..."
              class="input-pista"
              @keydown.enter="confirmarPista"
              maxlength="80"
              autofocus
            />
          </div>
        </div>

        <div class="pistas-acciones">
          <button class="btn-primario" @click="confirmarPista" :disabled="!pistaActual.trim()">
            {{ jugadorActualIdx < jugadores.length - 1 ? '→ Siguiente jugador' : '✓ Terminar ronda' }}
          </button>
        </div>

        <!-- Progreso jugadores en esta ronda -->
        <div class="progreso-pistas">
          <span
            v-for="(j, idx) in jugadores"
            :key="j.id"
            class="punto-progreso"
            :class="{ activo: idx <= jugadorActualIdx, hecho: idx < jugadorActualIdx }"
            :title="j.nombre"
          ></span>
        </div>
      </div>

      <!-- ═══ RESUMEN DE RONDA ═══ -->
      <div v-else-if="fase === 'resumen_ronda'" class="fase-resumen">
        <div class="resumen-badge">✓ Ronda {{ rondaActual }} completada</div>
        <h1>📋 Pistas de la Ronda {{ rondaActual }}</h1>

        <!-- Pistas de esta ronda -->
        <div class="card resumen-pistas-ronda">
          <div class="pistas-completas">
            <div v-for="j in jugadores" :key="j.id" class="pista-completa">
              <span class="pista-avatar">{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][j.id % 8] }}</span>
              <div>
                <strong>{{ j.nombre }}:</strong>
                <span> "{{ pistasPorRonda[rondaActual]?.[j.id] || '(sin pista)' }}"</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Historial de rondas anteriores (colapsado) -->
        <div v-if="rondaActual > 1" class="card historial-compacto">
          <p class="historial-titulo">Rondas anteriores:</p>
          <div v-for="r in rondaActual - 1" :key="r" class="ronda-historial">
            <span class="ronda-label-mini">Ronda {{ r }}</span>
            <div class="pistas-lista pistas-lista-compacta">
              <span v-for="j in jugadores" :key="j.id" class="pista-compacta">
                <strong>{{ j.nombre }}:</strong> "{{ pistasPorRonda[r]?.[j.id] || '—' }}"
              </span>
            </div>
          </div>
        </div>

        <!-- Acciones según ronda -->
        <div class="resumen-acciones">
          <!-- Rondas obligatorias (< 3): solo continuar -->
          <template v-if="rondaActual < RONDAS_MINIMAS">
            <p class="resumen-info">
              Quedan <strong>{{ RONDAS_MINIMAS - rondaActual }}</strong> ronda(s) obligatoria(s) más.
            </p>
            <button class="btn-primario btn-siguiente-ronda" @click="siguienteRonda">
              🔄 Iniciar Ronda {{ rondaActual + 1 }}
            </button>
          </template>

          <!-- Rondas opcionales (>= 3): elegir -->
          <template v-else>
            <p class="resumen-info decision-texto">
              Ya completaron {{ rondaActual }} rondas. ¿Quieren dar más pistas o pasar a votar?
            </p>
            <div class="decision-botones">
              <button class="btn-secundario" @click="siguienteRonda">
                🔄 Una ronda más (Ronda {{ rondaActual + 1 }})
              </button>
              <button class="btn-peligro" @click="irAVotar">
                🗳️ ¡Pasar a votar al Impostor!
              </button>
            </div>
          </template>
        </div>
      </div>

      <!-- ═══ VOTACIÓN DEL IMPOSTOR ═══ -->
      <div v-else-if="fase === 'votacion'" class="fase-votacion">
        <h1>🗳️ ¿Quién es el Impostor?</h1>
        <p class="instruccion">
          Analizaron <strong>{{ rondaActual }} ronda(s)</strong> de pistas. ¿Quién creen que NO sabía la palabra?
        </p>

        <!-- Resumen de todas las pistas por jugador -->
        <div class="card pistas-referencia">
          <h3>📋 Todas las pistas dadas:</h3>
          <div class="pistas-por-jugador">
            <div v-for="item in pistasPorJugador" :key="item.jugador.id" class="jugador-pistas-bloque">
              <div class="jugador-pistas-nombre">
                <span>{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][item.jugador.id % 8] }}</span>
                <strong>{{ item.jugador.nombre }}</strong>
              </div>
              <div class="jugador-pistas-lista">
                <span v-for="hp in item.historial" :key="hp.ronda" class="pista-por-ronda">
                  <span class="ronda-mini-label">R{{ hp.ronda }}:</span> "{{ hp.pista }}"
                </span>
              </div>
            </div>
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

/* ── Overlay de rol (Teleport) ────────────────────────────────────────── */
.overlay-rol {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.92);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  padding: 20px;
  overflow-y: auto;
}

.tarjeta-rol {
  max-width: 440px;
  width: 100%;
  background: var(--azul-card);
  border-radius: 16px;
  padding: 32px 28px;
  text-align: center;
  border: 3px solid;
  /* Margen vertical para cuando el contenido es alto */
  margin: auto;
}

.tarjeta-rol.informado {
  border-color: var(--verde-claro);
  box-shadow: 0 0 40px rgba(76, 175, 80, 0.4);
}

.tarjeta-rol.impostor {
  border-color: var(--rojo-claro);
  box-shadow: 0 0 40px rgba(229, 57, 53, 0.4);
}

.rol-icono { font-size: 4rem; margin-bottom: 12px; }

.rol-nombre {
  font-size: 1.5rem;
  color: var(--texto-principal);
  margin-bottom: 8px;
  -webkit-text-fill-color: initial;
  background: none;
}

.rol-titulo {
  font-size: 1.25rem;
  font-weight: 800;
  letter-spacing: 2px;
  margin-bottom: 20px;
}

.tarjeta-rol.informado .rol-titulo { color: var(--verde-brillante); }
.tarjeta-rol.impostor .rol-titulo { color: var(--rojo-claro); }

.rol-subtexto { color: var(--texto-gris); margin-bottom: 6px; }

.rol-palabra-texto {
  font-size: 1.9rem;
  color: var(--verde-brillante);
  display: block;
  margin: 8px 0 4px;
  line-height: 1.2;
}

.rol-categoria { color: var(--texto-gris); font-size: 0.85rem; margin-bottom: 20px; }

.rol-impostor-aviso {
  color: var(--texto-gris);
  margin-bottom: 20px;
  line-height: 1.7;
}

.btn-rol-visto { width: 100%; margin-top: 8px; }

/* Transición overlay */
.overlay-enter-active, .overlay-leave-active { transition: opacity 0.25s ease; }
.overlay-enter-from, .overlay-leave-to { opacity: 0; }

/* ── Asignación ─────────────────────────────────────────────────────────── */
.fase-asignacion {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  padding-top: 20px;
  text-align: center;
}

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

.jugador-asignacion.pendiente { border-color: var(--verde-claro); background: rgba(76,175,80,0.1); }
.jugador-asignacion.listo { border-color: rgba(76,175,80,0.2); opacity: 0.6; }
.jugador-asignacion.espera { border-color: var(--borde); opacity: 0.4; }

.asig-estado { font-weight: 700; color: var(--verde-brillante); width: 20px; }
.asig-nombre { flex: 1; font-weight: 600; }
.asig-etiqueta { font-size: 0.8rem; color: var(--texto-gris); }
.btn-ver-rol { padding: 16px 40px; font-size: 1.1rem; }

/* ── Pistas ─────────────────────────────────────────────────────────────── */
.fase-pistas { display: flex; flex-direction: column; gap: 16px; padding-top: 10px; }

.pistas-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.ronda-badge {
  display: inline-block;
  background: rgba(105, 240, 174, 0.12);
  border: 1px solid rgba(105, 240, 174, 0.3);
  color: var(--verde-brillante);
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 1px;
  padding: 3px 10px;
  border-radius: 20px;
  margin-bottom: 6px;
}

.pistas-header h1 { margin-bottom: 0; }

.timer-bloque { position: relative; width: 60px; height: 60px; flex-shrink: 0; }
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

.historial-rondas { border-color: rgba(105, 240, 174, 0.15); }

.pistas-anteriores-titulo {
  color: var(--texto-gris);
  font-size: 0.82rem;
  margin-bottom: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.ronda-historial { margin-bottom: 10px; }
.ronda-historial:last-child { margin-bottom: 0; }

.ronda-label-mini {
  display: inline-block;
  background: rgba(105, 240, 174, 0.1);
  color: var(--verde-brillante);
  font-size: 0.7rem;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 10px;
  margin-bottom: 6px;
  letter-spacing: 0.5px;
}

.pistas-lista { display: flex; flex-direction: column; gap: 5px; }

.pista-previa { display: flex; gap: 8px; font-size: 0.88rem; align-items: baseline; }
.pista-autor { color: var(--verde-brillante); font-weight: 600; flex-shrink: 0; }
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

.input-pista:focus { outline: none; border-color: var(--verde-claro); }
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
  cursor: default;
}

.punto-progreso.activo { background: var(--verde-claro); }
.punto-progreso.hecho { background: var(--verde-medio); }

/* ── Resumen de ronda ───────────────────────────────────────────────────── */
.fase-resumen { display: flex; flex-direction: column; gap: 18px; padding-top: 10px; }

.resumen-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: rgba(76,175,80,0.12);
  border: 1px solid rgba(76,175,80,0.3);
  color: var(--verde-brillante);
  font-size: 0.85rem;
  font-weight: 700;
  padding: 5px 14px;
  border-radius: 20px;
  width: fit-content;
}

.resumen-pistas-ronda { padding: 20px; }

.pistas-completas { display: flex; flex-direction: column; gap: 10px; }

.pista-completa {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  font-size: 0.95rem;
}

.pista-avatar { font-size: 1.4rem; flex-shrink: 0; }

.historial-compacto { padding: 16px; }

.historial-titulo {
  color: var(--texto-gris);
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 10px;
}

.pistas-lista-compacta { flex-direction: row; flex-wrap: wrap; gap: 6px 16px; }

.pista-compacta {
  font-size: 0.82rem;
  color: var(--texto-gris);
}

.pista-compacta strong { color: var(--texto-secundario); }

.resumen-acciones { display: flex; flex-direction: column; gap: 12px; }

.resumen-info {
  color: var(--texto-gris);
  font-size: 0.9rem;
  text-align: center;
}

.decision-texto { color: var(--texto-principal); font-size: 0.95rem; }

.decision-botones {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

.decision-botones > button { flex: 1; min-width: 180px; padding: 14px; font-size: 0.95rem; }

.btn-siguiente-ronda { padding: 16px; font-size: 1.05rem; }

/* ── Votación ───────────────────────────────────────────────────────────── */
.fase-votacion { display: flex; flex-direction: column; gap: 20px; padding-top: 10px; }

.pistas-referencia { padding: 18px; }
.pistas-referencia h3 { color: var(--verde-brillante); margin-bottom: 14px; font-size: 1rem; }

.pistas-por-jugador { display: flex; flex-direction: column; gap: 12px; }

.jugador-pistas-bloque {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

.jugador-pistas-nombre {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 110px;
  font-size: 0.9rem;
  flex-shrink: 0;
}

.jugador-pistas-lista {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1;
}

.pista-por-ronda { font-size: 0.85rem; color: var(--texto-gris); }

.ronda-mini-label {
  color: var(--verde-brillante);
  font-weight: 700;
  font-size: 0.75rem;
  margin-right: 4px;
}

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
  .pistas-header { flex-direction: column; align-items: flex-start; }
  .decision-botones { flex-direction: column; }
  .jugador-pistas-bloque { flex-direction: column; gap: 4px; }
  .jugador-pistas-nombre { min-width: unset; }
}
</style>
