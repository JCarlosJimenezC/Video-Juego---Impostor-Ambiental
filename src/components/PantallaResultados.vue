<script setup>
import { computed, onMounted, ref } from 'vue'
import TarjetaJugador from './TarjetaJugador.vue'

const props = defineProps({
  datos: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['jugar-otra-vez', 'volver-inicio'])

const mostrarConfeti = ref(false)

onMounted(() => {
  setTimeout(() => { mostrarConfeti.value = true }, 300)
})

const jugadoresOrdenados = computed(() => {
  return [...props.datos.jugadores]
    .map(j => ({ ...j, puntosGanados: props.datos.puntos[j.id] || 0 }))
    .sort((a, b) => b.puntosGanados - a.puntosGanados)
})

const ganadores = computed(() => {
  const maxPts = Math.max(...jugadoresOrdenados.value.map(j => j.puntosGanados))
  return jugadoresOrdenados.value.filter(j => j.puntosGanados === maxPts && maxPts > 0)
})

const impostorReal = computed(() =>
  props.datos.jugadores.find(j => j.id === props.datos.impostorId)
)

const impostorVotado = computed(() =>
  props.datos.jugadores.find(j => j.id === props.datos.impostorVotadoId)
)

const resultadoTitulo = computed(() => {
  if (props.datos.palabraCorrecta && props.datos.impostorIdentificado) return '¡Victoria Total de los Informados!'
  if (props.datos.palabraCorrecta && !props.datos.impostorIdentificado) return '¡El Impostor Pasó Desapercibido!'
  if (!props.datos.palabraCorrecta && props.datos.impostorIdentificado) return '¡Impostor Descubierto pero Confundieron!'
  return '¡El Impostor Ganó la Ronda!'
})

const resultadoEmoji = computed(() => {
  if (props.datos.palabraCorrecta && props.datos.impostorIdentificado) return '🏆'
  if (!props.datos.palabraCorrecta || !props.datos.impostorIdentificado) return '🎭'
  return '⚖️'
})

const medallones = ['🥇', '🥈', '🥉']
</script>

<template>
  <div class="resultados-pantalla">
    <!-- Confeti animado -->
    <div v-if="mostrarConfeti" class="confeti-contenedor">
      <div v-for="i in 30" :key="i" class="confeti-pieza" :style="{
        left: (Math.random() * 100) + '%',
        animationDelay: (Math.random() * 2) + 's',
        animationDuration: (2 + Math.random() * 2) + 's',
        background: ['#69f0ae','#4caf50','#ffd700','#ff5252','#69b2f0'][i % 5]
      }"></div>
    </div>

    <div class="resultados-contenido fade-in">

      <!-- Banner de resultado principal -->
      <div class="banner-resultado" :class="{ victoria: datos.palabraCorrecta && datos.impostorIdentificado, derrota: !datos.palabraCorrecta || !datos.impostorIdentificado }">
        <div class="resultado-emoji">{{ resultadoEmoji }}</div>
        <h1>{{ resultadoTitulo }}</h1>
      </div>

      <!-- Detalle de la ronda -->
      <div class="card detalle-ronda">
        <h2>📊 Resumen de la Ronda</h2>
        <div class="detalle-grid">
          <div class="detalle-item">
            <span class="detalle-label">Palabra Secreta</span>
            <span class="detalle-valor palabra-secreta">{{ datos.palabra.palabra }}</span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">Grupo adivinó</span>
            <span class="detalle-valor" :class="datos.palabraCorrecta ? 'correcto' : 'incorrecto'">
              {{ datos.palabraAdivinada || '(nada)' }}
              {{ datos.palabraCorrecta ? '✓' : '✗' }}
            </span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">Impostor Real</span>
            <span class="detalle-valor impostor-nombre">🎭 {{ impostorReal?.nombre }}</span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">Votaron por</span>
            <span class="detalle-valor" :class="datos.impostorIdentificado ? 'correcto' : 'incorrecto'">
              {{ impostorVotado?.nombre }}
              {{ datos.impostorIdentificado ? '✓ ¡Correcto!' : '✗ Incorrecto' }}
            </span>
          </div>
        </div>
      </div>

      <!-- Tabla de puntuación -->
      <div class="card tabla-puntuacion">
        <h2>🏅 Puntuación Final</h2>
        <div class="lista-puntuacion">
          <div
            v-for="(j, idx) in jugadoresOrdenados"
            :key="j.id"
            class="fila-puntuacion"
            :class="{
              'primer-lugar': idx === 0 && j.puntosGanados > 0,
              'es-impostor': j.id === datos.impostorId
            }"
          >
            <span class="posicion">{{ medallones[idx] || (idx + 1) + '°' }}</span>
            <span class="jugador-avatar-pts">{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][j.id % 8] }}</span>
            <span class="jugador-nombre-pts">
              {{ j.nombre }}
              <span v-if="j.id === datos.impostorId" class="tag-impostor">IMPOSTOR</span>
            </span>
            <span class="puntos-ganados">
              +{{ j.puntosGanados }} pts
            </span>
          </div>
        </div>
      </div>

      <!-- Análisis de pistas -->
      <div class="card analisis-pistas">
        <h2>🔍 Análisis de Pistas</h2>
        <p class="analisis-subtitulo">¿Notaron algo sospechoso?</p>
        <div class="pistas-analisis">
          <div
            v-for="j in datos.jugadores"
            :key="j.id"
            class="pista-analisis-item"
            :class="{ impostor: j.id === datos.impostorId }"
          >
            <div class="pista-analisis-header">
              <span class="pista-analisis-nombre">
                {{ j.nombre }}
                <span v-if="j.id === datos.impostorId" class="badge-impostor">🎭 IMPOSTOR</span>
              </span>
            </div>
            <p class="pista-analisis-texto">"{{ j.pista || '(sin pista)' }}"</p>
          </div>
        </div>
      </div>

      <!-- Ganadores destacados -->
      <div v-if="ganadores.length > 0" class="card ganadores-bloque">
        <h2>🎉 {{ ganadores.length === 1 ? 'Ganador' : 'Empate' }} de la Ronda</h2>
        <div class="ganadores-lista">
          <div v-for="g in ganadores" :key="g.id" class="ganador-chip">
            <span>{{ ['🧑','👩','👨','🧒','👧','🧔','👱','🧕'][g.id % 8] }}</span>
            <strong>{{ g.nombre }}</strong>
            <span class="ganador-pts">{{ g.puntosGanados }} pts</span>
          </div>
        </div>
      </div>

      <!-- Botones de acción -->
      <div class="acciones-finales">
        <button class="btn-primario btn-accion" @click="emit('jugar-otra-vez')">
          🔄 Jugar Otra Ronda
        </button>
        <button class="btn-secundario btn-accion" @click="emit('volver-inicio')">
          🏠 Menú Principal
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.resultados-pantalla {
  min-height: 100vh;
  padding: 20px;
  background: radial-gradient(ellipse at top center, #0d2a0d 0%, var(--azul-oscuro) 60%);
  display: flex;
  justify-content: center;
  position: relative;
  overflow: hidden;
}

/* Confeti */
.confeti-contenedor {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
}

.confeti-pieza {
  position: absolute;
  top: -10px;
  width: 8px;
  height: 8px;
  border-radius: 2px;
  animation: caer linear forwards;
}

@keyframes caer {
  0% { transform: translateY(0) rotate(0deg); opacity: 1; }
  100% { transform: translateY(110vh) rotate(720deg); opacity: 0; }
}

.resultados-contenido {
  max-width: 750px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding-bottom: 40px;
  position: relative;
  z-index: 1;
}

/* Banner */
.banner-resultado {
  text-align: center;
  padding: 28px 20px;
  border-radius: 16px;
  border: 2px solid;
}

.banner-resultado.victoria {
  background: rgba(76,175,80,0.1);
  border-color: var(--verde-claro);
}

.banner-resultado.derrota {
  background: rgba(229,57,53,0.1);
  border-color: var(--rojo-claro);
}

.resultado-emoji {
  font-size: 3.5rem;
  margin-bottom: 10px;
  animation: pulso 1.5s ease-in-out infinite;
}

.banner-resultado h1 {
  font-size: 1.6rem;
  -webkit-text-fill-color: initial;
  background: none;
  color: var(--texto-principal);
}

/* Detalle */
.card h2 {
  color: var(--verde-brillante);
  margin-bottom: 16px;
  font-size: 1.1rem;
}

.detalle-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.detalle-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 10px;
  background: rgba(255,255,255,0.03);
  border-radius: 8px;
}

.detalle-label { font-size: 0.78rem; color: var(--texto-gris); text-transform: uppercase; letter-spacing: 0.5px; }

.detalle-valor { font-weight: 600; font-size: 0.95rem; }

.palabra-secreta { color: var(--verde-brillante); font-size: 1.1rem; }
.impostor-nombre { color: var(--rojo-claro); }
.correcto { color: var(--verde-brillante); }
.incorrecto { color: var(--rojo-claro); }

/* Puntuación */
.lista-puntuacion { display: flex; flex-direction: column; gap: 8px; }

.fila-puntuacion {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: rgba(255,255,255,0.03);
  border-radius: 10px;
  border: 1px solid transparent;
  transition: all 0.3s;
}

.fila-puntuacion.primer-lugar {
  background: rgba(255,215,0,0.08);
  border-color: rgba(255,215,0,0.3);
}

.fila-puntuacion.es-impostor {
  background: rgba(229,57,53,0.08);
  border-color: rgba(229,57,53,0.2);
}

.posicion { font-size: 1.2rem; width: 28px; }
.jugador-avatar-pts { font-size: 1.4rem; }
.jugador-nombre-pts { flex: 1; font-weight: 600; display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.tag-impostor { font-size: 0.7rem; background: rgba(229,57,53,0.2); color: var(--rojo-claro); padding: 2px 8px; border-radius: 10px; font-weight: 700; }
.puntos-ganados { color: var(--verde-brillante); font-weight: 800; font-size: 1.05rem; }

/* Análisis */
.analisis-subtitulo { color: var(--texto-gris); font-size: 0.85rem; margin-bottom: 14px; }

.pistas-analisis { display: flex; flex-direction: column; gap: 10px; }

.pista-analisis-item {
  padding: 12px;
  background: rgba(255,255,255,0.03);
  border-radius: 8px;
  border-left: 3px solid var(--borde);
}

.pista-analisis-item.impostor {
  border-left-color: var(--rojo-claro);
  background: rgba(229,57,53,0.06);
}

.pista-analisis-header { margin-bottom: 6px; }
.pista-analisis-nombre { font-weight: 600; display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.badge-impostor { font-size: 0.75rem; background: rgba(229,57,53,0.2); color: var(--rojo-claro); padding: 2px 8px; border-radius: 10px; }
.pista-analisis-texto { color: var(--texto-gris); font-style: italic; font-size: 0.9rem; }

/* Ganadores */
.ganadores-lista { display: flex; flex-wrap: wrap; gap: 12px; }

.ganador-chip {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 18px;
  background: rgba(255,215,0,0.1);
  border: 2px solid rgba(255,215,0,0.4);
  border-radius: 30px;
  font-size: 1.1rem;
  animation: brillar 2s ease-in-out infinite;
}

.ganador-pts { color: var(--dorado); font-weight: 700; }

/* Acciones */
.acciones-finales { display: flex; gap: 12px; }

.btn-accion { flex: 1; padding: 14px; font-size: 1rem; }

@media (max-width: 600px) {
  .detalle-grid { grid-template-columns: 1fr; }
  .acciones-finales { flex-direction: column; }
  .banner-resultado h1 { font-size: 1.3rem; }
}
</style>
