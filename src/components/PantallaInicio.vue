<script setup>
import { ref, onMounted } from 'vue'

const emit = defineEmits(['jugar'])

const particulas = ref([])

onMounted(() => {
  // Genera partículas decorativas de fondo
  particulas.value = Array.from({ length: 20 }, (_, i) => ({
    id: i,
    x: Math.random() * 100,
    y: Math.random() * 100,
    size: Math.random() * 6 + 2,
    delay: Math.random() * 3,
    duracion: Math.random() * 3 + 2
  }))
})
</script>

<template>
  <div class="inicio">
    <!-- Partículas animadas de fondo -->
    <div class="particulas-fondo">
      <div
        v-for="p in particulas"
        :key="p.id"
        class="particula"
        :style="{
          left: p.x + '%',
          top: p.y + '%',
          width: p.size + 'px',
          height: p.size + 'px',
          animationDelay: p.delay + 's',
          animationDuration: p.duracion + 's'
        }"
      ></div>
    </div>

    <div class="inicio-contenido fade-in">
      <!-- Logo/Título -->
      <div class="titulo-bloque">
        <div class="logo-icono">🌍</div>
        <h1 class="titulo-principal">El Impostor Ambiental</h1>
        <p class="subtitulo">Juego de deducción con temática ecológica</p>
      </div>

      <!-- Descripción del juego -->
      <div class="descripcion-bloque card">
        <h2>¿Cómo se juega?</h2>
        <div class="pasos-juego">
          <div class="paso">
            <span class="paso-icono">🎭</span>
            <div>
              <strong>Asignación de Roles</strong>
              <p>La mayoría recibe la palabra secreta ambiental. Uno es el Impostor y no la sabe.</p>
            </div>
          </div>
          <div class="paso">
            <span class="paso-icono">💬</span>
            <div>
              <strong>Ronda de Pistas</strong>
              <p>Cada jugador da una pista sobre la palabra. El impostor debe inventar algo convincente.</p>
            </div>
          </div>
          <div class="paso">
            <span class="paso-icono">🗳️</span>
            <div>
              <strong>Votación</strong>
              <p>El grupo adivina la palabra y vota quién es el impostor. ¡La deducción decide todo!</p>
            </div>
          </div>
          <div class="paso">
            <span class="paso-icono">🏆</span>
            <div>
              <strong>Puntuación</strong>
              <p>Informados ganan si adivinan la palabra e identifican al impostor. El impostor gana si pasa inadvertido.</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Puntuación rápida -->
      <div class="puntuacion-resumen card">
        <h3>Sistema de Puntos</h3>
        <div class="puntos-grid">
          <div class="punto-item informado">
            <span class="pts">+2</span>
            <span>Adivinan la palabra</span>
          </div>
          <div class="punto-item informado">
            <span class="pts">+1</span>
            <span>Identifican al impostor</span>
          </div>
          <div class="punto-item impostor">
            <span class="pts">+3</span>
            <span>Impostor no es descubierto</span>
          </div>
          <div class="punto-item impostor">
            <span class="pts">+1</span>
            <span>Impostor adivina la palabra</span>
          </div>
        </div>
      </div>

      <!-- Botón de inicio -->
      <button class="btn-primario btn-inicio-grande" @click="emit('jugar')">
        🌱 ¡Comenzar Juego!
      </button>

      <p class="creditos">Proyecto IF7102 Multimedios · Vue 3 · UCR</p>
    </div>
  </div>
</template>

<style scoped>
.inicio {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  position: relative;
  overflow: hidden;
  background: radial-gradient(ellipse at center top, #0d2a0d 0%, var(--azul-oscuro) 60%);
}

.particulas-fondo {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.particula {
  position: absolute;
  background: var(--verde-brillante);
  border-radius: 50%;
  opacity: 0.15;
  animation: flotar linear infinite;
}

@keyframes flotar {
  0%, 100% { transform: translateY(0) scale(1); opacity: 0.1; }
  50% { transform: translateY(-20px) scale(1.2); opacity: 0.3; }
}

.inicio-contenido {
  max-width: 750px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 24px;
  position: relative;
  z-index: 1;
}

.titulo-bloque {
  text-align: center;
}

.logo-icono {
  font-size: 5rem;
  margin-bottom: 16px;
  display: block;
  filter: drop-shadow(0 0 20px rgba(105, 240, 174, 0.5));
  animation: pulso 3s ease-in-out infinite;
}

.titulo-principal {
  font-size: 2.8rem;
  font-weight: 800;
  background: linear-gradient(135deg, var(--verde-brillante), var(--verde-claro));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin-bottom: 8px;
  line-height: 1.2;
}

.subtitulo {
  color: var(--texto-gris);
  font-size: 1.1rem;
}

.descripcion-bloque h2 {
  color: var(--verde-brillante);
  margin-bottom: 20px;
  font-size: 1.3rem;
}

.pasos-juego {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.paso {
  display: flex;
  align-items: flex-start;
  gap: 14px;
}

.paso-icono {
  font-size: 1.6rem;
  flex-shrink: 0;
  margin-top: 2px;
}

.paso strong {
  display: block;
  color: var(--texto-principal);
  margin-bottom: 2px;
}

.paso p {
  color: var(--texto-gris);
  font-size: 0.9rem;
  line-height: 1.4;
}

.puntuacion-resumen h3 {
  color: var(--verde-brillante);
  margin-bottom: 16px;
  font-size: 1.1rem;
}

.puntos-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.punto-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  border-radius: 8px;
  font-size: 0.9rem;
}

.punto-item.informado {
  background: rgba(76, 175, 80, 0.1);
  border: 1px solid rgba(76, 175, 80, 0.3);
}

.punto-item.impostor {
  background: rgba(229, 57, 53, 0.1);
  border: 1px solid rgba(229, 57, 53, 0.3);
}

.pts {
  font-size: 1.3rem;
  font-weight: 800;
  min-width: 36px;
}

.punto-item.informado .pts { color: var(--verde-brillante); }
.punto-item.impostor .pts { color: var(--rojo-claro); }

.btn-inicio-grande {
  width: 100%;
  padding: 18px;
  font-size: 1.2rem;
  border-radius: 12px;
  animation: brillar 2s ease-in-out infinite;
}

.creditos {
  text-align: center;
  color: var(--texto-gris);
  font-size: 0.8rem;
}

@media (max-width: 600px) {
  .titulo-principal { font-size: 2rem; }
  .logo-icono { font-size: 3.5rem; }
  .puntos-grid { grid-template-columns: 1fr; }
}
</style>
