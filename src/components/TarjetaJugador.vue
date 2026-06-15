<script setup>
defineProps({
  jugador: {
    type: Object,
    required: true
  },
  mostrarRol: {
    type: Boolean,
    default: false
  },
  seleccionado: {
    type: Boolean,
    default: false
  },
  desactivado: {
    type: Boolean,
    default: false
  },
  modo: {
    type: String,
    default: 'pista' // 'pista' | 'voto' | 'resultado'
  }
})

defineEmits(['seleccionar'])

const avatares = ['🧑', '👩', '👨', '🧒', '👧', '🧔', '👱', '🧕']

function getAvatar(id) {
  return avatares[id % avatares.length]
}
</script>

<template>
  <div
    class="tarjeta-jugador"
    :class="{
      'seleccionada': seleccionado,
      'desactivada': desactivado,
      'es-impostor': mostrarRol && jugador.rol === 'impostor',
      'es-informado': mostrarRol && jugador.rol === 'informado',
      'modo-voto': modo === 'voto'
    }"
    @click="!desactivado && $emit('seleccionar', jugador)"
  >
    <div class="jugador-avatar">{{ getAvatar(jugador.id) }}</div>
    <div class="jugador-info">
      <span class="jugador-nombre">{{ jugador.nombre }}</span>
      <span v-if="mostrarRol" class="jugador-rol" :class="jugador.rol">
        {{ jugador.rol === 'impostor' ? '🎭 Impostor' : '🌿 Informado' }}
      </span>
    </div>
    <div v-if="jugador.pista && modo === 'resultado'" class="jugador-pista-texto">
      "{{ jugador.pista }}"
    </div>
    <div v-if="seleccionado && modo === 'voto'" class="check-seleccion">✓</div>
  </div>
</template>

<style scoped>
.tarjeta-jugador {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: rgba(255, 255, 255, 0.04);
  border: 2px solid transparent;
  border-radius: 10px;
  transition: all 0.25s ease;
  position: relative;
}

.modo-voto {
  cursor: pointer;
  border-color: var(--borde);
}

.modo-voto:hover:not(.desactivada) {
  border-color: var(--rojo-claro);
  background: rgba(229, 57, 53, 0.08);
  transform: translateY(-1px);
}

.seleccionada {
  border-color: var(--rojo-claro) !important;
  background: rgba(229, 57, 53, 0.15) !important;
  box-shadow: 0 0 15px rgba(229, 57, 53, 0.3);
}

.desactivada {
  opacity: 0.5;
  cursor: not-allowed;
}

.es-impostor {
  border-color: var(--rojo-claro);
  background: rgba(229, 57, 53, 0.1);
}

.es-informado {
  border-color: var(--verde-claro);
  background: rgba(76, 175, 80, 0.1);
}

.jugador-avatar {
  font-size: 2rem;
  flex-shrink: 0;
}

.jugador-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex: 1;
}

.jugador-nombre {
  font-weight: 600;
  color: var(--texto-principal);
  font-size: 1rem;
}

.jugador-rol {
  font-size: 0.8rem;
  font-weight: 500;
}

.jugador-rol.impostor { color: var(--rojo-claro); }
.jugador-rol.informado { color: var(--verde-brillante); }

.jugador-pista-texto {
  font-size: 0.85rem;
  color: var(--texto-gris);
  font-style: italic;
  margin-top: 4px;
  padding-top: 8px;
  border-top: 1px solid var(--borde);
  width: 100%;
}

.check-seleccion {
  position: absolute;
  right: 14px;
  top: 50%;
  transform: translateY(-50%);
  width: 28px;
  height: 28px;
  background: var(--rojo-impostor);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.9rem;
  color: white;
}
</style>
