<script setup>
import { computed, onMounted, ref } from 'vue'

const props = defineProps({
  datos: { type: Object, required: true },
  puntosAcumulados: { type: Object, default: () => ({}) }
})

const emit = defineEmits(['jugar-otra-vez', 'volver-inicio', 'bonus-impostor'])

const mostrarConfeti = ref(false)
// null = pendiente de decisión | true = adivinó | false = no adivinó
const bonusDecision = ref(null)

onMounted(() => {
  setTimeout(() => { mostrarConfeti.value = true }, 300)
})

// Lee props.datos.puntos reactivamente → se actualiza cuando App muta el objeto
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

const resultadoTitulo = computed(() =>
  props.datos.impostorIdentificado
    ? '¡Los Informados Descubrieron al Impostor!'
    : '¡El Impostor Pasó Desapercibido!'
)

const resultadoEmoji = computed(() =>
  props.datos.impostorIdentificado ? '🏆' : '🎭'
)

// Leaderboard acumulado de la sesión, ordenado de mayor a menor
const leaderboardSesion = computed(() => {
  return Object.entries(props.puntosAcumulados)
    .map(([nombre, total]) => ({ nombre, total }))
    .sort((a, b) => b.total - a.total)
})

const haySesionAcumulada = computed(() =>
  leaderboardSesion.value.some(e => e.total > 0)
)

function confirmarBonus(adivinó) {
  bonusDecision.value = adivinó
  if (adivinó) emit('bonus-impostor')
}

const avatares = ['🧑','👩','👨','🧒','👧','🧔','👱','🧕']
const medallones = ['🥇', '🥈', '🥉']
</script>

<template>
  <div class="resultados-pantalla">
    <!-- Confeti -->
    <div v-if="mostrarConfeti" class="confeti-contenedor">
      <div v-for="i in 30" :key="i" class="confeti-pieza" :style="{
        left: (Math.random() * 100) + '%',
        animationDelay: (Math.random() * 2) + 's',
        animationDuration: (2 + Math.random() * 2) + 's',
        background: ['#69f0ae','#4caf50','#ffd700','#ff5252','#69b2f0'][i % 5]
      }"></div>
    </div>

    <div class="resultados-contenido fade-in">

      <!-- Banner principal -->
      <div
        class="banner-resultado"
        :class="{ victoria: datos.impostorIdentificado, derrota: !datos.impostorIdentificado }"
      >
        <div class="resultado-emoji">{{ resultadoEmoji }}</div>
        <h1>{{ resultadoTitulo }}</h1>
        <p class="rondas-jugadas">{{ datos.totalRondas }} ronda(s) de pistas jugadas</p>
      </div>

      <!-- Resumen -->
      <div class="card detalle-ronda">
        <h2>📊 Resumen</h2>
        <div class="detalle-grid">
          <div class="detalle-item">
            <span class="detalle-label">Palabra Secreta</span>
            <span class="detalle-valor palabra-secreta">{{ datos.palabra.palabra }}</span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">Categoría</span>
            <span class="detalle-valor">{{ datos.palabra.categoria }}</span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">Impostor Real</span>
            <span class="detalle-valor impostor-nombre">🎭 {{ impostorReal?.nombre }}</span>
          </div>
          <div class="detalle-item">
            <span class="detalle-label">El grupo votó por</span>
            <span class="detalle-valor" :class="datos.impostorIdentificado ? 'correcto' : 'incorrecto'">
              {{ impostorVotado?.nombre }}
              {{ datos.impostorIdentificado ? '✓ ¡Correcto!' : '✗ Incorrecto' }}
            </span>
          </div>
        </div>
      </div>

      <!-- Análisis de pistas por ronda -->
      <div class="card analisis-pistas">
        <h2>🔍 Análisis de Pistas</h2>
        <p class="analisis-subtitulo">Las pistas del Impostor eran las más vagas o incoherentes.</p>

        <div class="jugadores-analisis">
          <div
            v-for="j in datos.jugadores"
            :key="j.id"
            class="jugador-analisis-bloque"
            :class="{ impostor: j.id === datos.impostorId }"
          >
            <div class="jugador-analisis-header">
              <span class="jugador-analisis-avatar">{{ avatares[j.id % 8] }}</span>
              <span class="jugador-analisis-nombre">
                {{ j.nombre }}
                <span v-if="j.id === datos.impostorId" class="badge-impostor">🎭 IMPOSTOR</span>
              </span>
            </div>
            <div class="historial-pistas">
              <div
                v-for="hp in j.historialPistas"
                :key="hp.ronda"
                class="pista-historial-item"
              >
                <span class="pista-ronda-label">Ronda {{ hp.ronda }}</span>
                <span class="pista-historial-texto">"{{ hp.pista }}"</span>
              </div>
              <p v-if="!j.historialPistas?.length" class="sin-pistas">(sin pistas registradas)</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Bonus del Impostor -->
      <div class="card bonus-impostor-card">
        <h2>🎯 Bonus del Impostor</h2>
        <p class="bonus-descripcion">
          Ahora que se reveló la palabra, ¿logró
          <strong>{{ impostorReal?.nombre }}</strong> adivinarla sin saberla?
        </p>

        <!-- Pendiente de decisión -->
        <div v-if="bonusDecision === null" class="bonus-pregunta">
          <p class="bonus-palabra-ref">
            La palabra era: <strong class="bonus-palabra-txt">{{ datos.palabra.palabra }}</strong>
          </p>
          <div class="bonus-botones">
            <button class="btn-bonus-si" @click="confirmarBonus(true)">
              ✓ Sí adivinó — sumar +1 punto
            </button>
            <button class="btn-bonus-no" @click="confirmarBonus(false)">
              ✗ No adivinó
            </button>
          </div>
        </div>

        <!-- Decisión tomada -->
        <div v-else class="bonus-resultado" :class="bonusDecision ? 'bonus-si' : 'bonus-no'">
          <span v-if="bonusDecision">
            🏅 +1 punto otorgado a <strong>{{ impostorReal?.nombre }}</strong> — ¡lo adivinó!
          </span>
          <span v-else>
            El impostor no adivinó la palabra.
          </span>
        </div>
      </div>

      <!-- Tabla de puntos actualizada (se re-renderiza reactivamente) -->
      <div class="card tabla-puntuacion-final">
        <h2>📊 Puntos de esta Ronda (final)</h2>
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
            <span class="jugador-avatar-pts">{{ avatares[j.id % 8] }}</span>
            <span class="jugador-nombre-pts">
              {{ j.nombre }}
              <span v-if="j.id === datos.impostorId" class="tag-impostor">IMPOSTOR</span>
            </span>
            <span class="puntos-ganados">+{{ j.puntosGanados }} pts</span>
          </div>
        </div>
      </div>

      <!-- Marcador Acumulado de la Sesión -->
      <div v-if="haySesionAcumulada" class="card marcador-sesion">
        <h2>🏆 Marcador Total de la Sesión</h2>
        <div class="lista-puntuacion">
          <div
            v-for="(entry, idx) in leaderboardSesion"
            :key="entry.nombre"
            class="fila-puntuacion fila-acumulada"
            :class="{ 'primer-lugar': idx === 0 }"
          >
            <span class="posicion">{{ medallones[idx] || (idx + 1) + '°' }}</span>
            <span class="jugador-nombre-pts">{{ entry.nombre }}</span>
            <span class="puntos-acumulados-total">{{ entry.total }} pts</span>
          </div>
        </div>
      </div>

      <!-- Ganadores de la ronda -->
      <div v-if="ganadores.length > 0" class="card ganadores-bloque">
        <h2>🎉 {{ ganadores.length === 1 ? 'Ganador' : 'Empate' }} de la Ronda</h2>
        <div class="ganadores-lista">
          <div v-for="g in ganadores" :key="g.id" class="ganador-chip">
            <span>{{ avatares[g.id % 8] }}</span>
            <strong>{{ g.nombre }}</strong>
            <span class="ganador-pts">{{ g.puntosGanados }} pts</span>
          </div>
        </div>
      </div>

      <!-- Acciones -->
      <div class="acciones-finales">
        <button class="btn-primario btn-accion" @click="emit('jugar-otra-vez')">
          🔄 Jugar Otra Vez
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
  margin-bottom: 6px;
}

.rondas-jugadas {
  color: var(--texto-gris);
  font-size: 0.88rem;
}

/* Detalle */
.card h2 {
  color: var(--verde-brillante);
  margin-bottom: 14px;
  font-size: 1.1rem;
}

.detalle-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.detalle-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 10px;
  background: rgba(255,255,255,0.03);
  border-radius: 8px;
}

.detalle-label { font-size: 0.75rem; color: var(--texto-gris); text-transform: uppercase; letter-spacing: 0.5px; }
.detalle-valor { font-weight: 600; font-size: 0.95rem; }
.palabra-secreta { color: var(--verde-brillante); font-size: 1.1rem; }
.impostor-nombre { color: var(--rojo-claro); }
.correcto { color: var(--verde-brillante); }
.incorrecto { color: var(--rojo-claro); }

/* Puntuación */
.puntuacion-leyenda {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 14px;
}

.leyenda-item {
  font-size: 0.78rem;
  padding: 3px 10px;
  border-radius: 20px;
}

.leyenda-item.informado {
  background: rgba(76,175,80,0.1);
  color: var(--verde-brillante);
  border: 1px solid rgba(76,175,80,0.25);
}

.leyenda-item.impostor-leg {
  background: rgba(229,57,53,0.1);
  color: var(--rojo-claro);
  border: 1px solid rgba(229,57,53,0.25);
}

.lista-puntuacion { display: flex; flex-direction: column; gap: 8px; }

.fila-puntuacion {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: rgba(255,255,255,0.03);
  border-radius: 10px;
  border: 1px solid transparent;
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
.analisis-subtitulo { color: var(--texto-gris); font-size: 0.85rem; margin-bottom: 16px; }

.jugadores-analisis { display: flex; flex-direction: column; gap: 14px; }

.jugador-analisis-bloque {
  border-radius: 10px;
  padding: 14px;
  background: rgba(255,255,255,0.03);
  border-left: 3px solid var(--borde);
}

.jugador-analisis-bloque.impostor {
  border-left-color: var(--rojo-claro);
  background: rgba(229,57,53,0.05);
}

.jugador-analisis-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}

.jugador-analisis-avatar { font-size: 1.4rem; }

.jugador-analisis-nombre {
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.badge-impostor {
  font-size: 0.72rem;
  background: rgba(229,57,53,0.2);
  color: var(--rojo-claro);
  padding: 2px 8px;
  border-radius: 10px;
}

.historial-pistas { display: flex; flex-direction: column; gap: 6px; }

.pista-historial-item {
  display: flex;
  align-items: baseline;
  gap: 8px;
  font-size: 0.88rem;
}

.pista-ronda-label {
  flex-shrink: 0;
  background: rgba(105,240,174,0.1);
  color: var(--verde-brillante);
  font-size: 0.72rem;
  font-weight: 700;
  padding: 1px 7px;
  border-radius: 8px;
}

.pista-historial-texto { color: var(--texto-gris); font-style: italic; }

.sin-pistas { color: var(--texto-gris); font-size: 0.82rem; font-style: italic; }

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

/* ── Bonus del Impostor ────────────────────────────────────────────────── */
.bonus-impostor-card {
  border-color: rgba(255, 215, 0, 0.25) !important;
  background: rgba(255, 215, 0, 0.04) !important;
}

.bonus-impostor-card h2 { color: var(--dorado) !important; }

.bonus-descripcion {
  color: var(--texto-gris);
  font-size: 0.95rem;
  margin-bottom: 16px;
  line-height: 1.5;
}

.bonus-descripcion strong { color: var(--texto-principal); }

.bonus-palabra-ref {
  color: var(--texto-gris);
  font-size: 0.9rem;
  margin-bottom: 14px;
}

.bonus-palabra-txt {
  color: var(--verde-brillante);
  font-size: 1.05rem;
}

.bonus-botones {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

.btn-bonus-si {
  flex: 1;
  min-width: 160px;
  padding: 13px 20px;
  background: linear-gradient(135deg, #2e7d32, var(--verde-claro));
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.3s;
  box-shadow: 0 4px 14px rgba(76,175,80,0.3);
}

.btn-bonus-si:hover { transform: translateY(-2px); box-shadow: 0 6px 18px rgba(76,175,80,0.5); }

.btn-bonus-no {
  flex: 1;
  min-width: 120px;
  padding: 13px 20px;
  background: transparent;
  color: var(--texto-gris);
  border: 2px solid var(--borde);
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.2s;
}

.btn-bonus-no:hover { border-color: var(--texto-gris); color: var(--texto-principal); }

.bonus-resultado {
  padding: 14px 18px;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
}

.bonus-resultado.bonus-si {
  background: rgba(76,175,80,0.12);
  border: 1px solid rgba(76,175,80,0.3);
  color: var(--verde-brillante);
}

.bonus-resultado.bonus-no {
  background: rgba(255,255,255,0.03);
  border: 1px solid var(--borde);
  color: var(--texto-gris);
}

/* ── Tabla final (post-bonus) ──────────────────────────────────────────── */
.tabla-puntuacion-final h2 { color: var(--verde-brillante); margin-bottom: 14px; font-size: 1.1rem; }

/* ── Marcador Acumulado de la Sesión ───────────────────────────────────── */
.marcador-sesion {
  border-color: rgba(255,215,0,0.2) !important;
  background: rgba(255,215,0,0.03) !important;
}

.marcador-sesion h2 { color: var(--dorado) !important; margin-bottom: 14px; font-size: 1.1rem; }

.fila-acumulada { background: rgba(255,215,0,0.04) !important; }

.puntos-acumulados-total {
  color: var(--dorado);
  font-weight: 800;
  font-size: 1.1rem;
}

/* ── Acciones ──────────────────────────────────────────────────────────── */
.acciones-finales { display: flex; gap: 12px; }
.btn-accion { flex: 1; padding: 14px; font-size: 1rem; }

@media (max-width: 600px) {
  .detalle-grid { grid-template-columns: 1fr; }
  .acciones-finales { flex-direction: column; }
  .banner-resultado h1 { font-size: 1.3rem; }
  .bonus-botones { flex-direction: column; }
}
</style>
