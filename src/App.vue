<script setup>
import { ref } from 'vue'
import PantallaInicio from './components/PantallaInicio.vue'
import ConfiguracionJuego from './components/ConfiguracionJuego.vue'
import SalaJuego from './components/SalaJuego.vue'
import PantallaResultados from './components/PantallaResultados.vue'

const pantalla = ref('inicio')
const estadoJuego = ref(null)

// Persiste entre rondas, se resetea solo al volver al menú principal
const puntosAcumulados = ref({}) // { 'NombreJugador': totalPuntos }
const jugadoresSesion = ref([])  // nombres de la última partida para pre-llenar

function iniciarConfiguracion() {
  pantalla.value = 'configuracion'
}

function comenzarJuego(config) {
  estadoJuego.value = config
  jugadoresSesion.value = config.jugadores.map(j => j.nombre)
  pantalla.value = 'juego'
}

function mostrarResultados(resultado) {
  const datosCompletos = { ...estadoJuego.value, ...resultado }

  // Sumar puntos de esta ronda al acumulado por nombre de jugador
  datosCompletos.jugadores.forEach(j => {
    const ptsRonda = resultado.puntos[j.id] || 0
    puntosAcumulados.value[j.nombre] = (puntosAcumulados.value[j.nombre] || 0) + ptsRonda
  })

  estadoJuego.value = datosCompletos
  pantalla.value = 'resultados'
}

function bonusImpostor() {
  const impostor = estadoJuego.value.jugadores.find(j => j.id === estadoJuego.value.impostorId)
  if (!impostor) return
  estadoJuego.value.puntos[impostor.id] = (estadoJuego.value.puntos[impostor.id] || 0) + 1
  puntosAcumulados.value[impostor.nombre] = (puntosAcumulados.value[impostor.nombre] || 0) + 1
}

function jugarOtraVez() {
  pantalla.value = 'configuracion'
}

function volverInicio() {
  pantalla.value = 'inicio'
  estadoJuego.value = null
  puntosAcumulados.value = {}
  jugadoresSesion.value = []
}
</script>

<template>
  <div class="app-contenedor">
    <PantallaInicio
      v-if="pantalla === 'inicio'"
      @jugar="iniciarConfiguracion"
    />
    <ConfiguracionJuego
      v-else-if="pantalla === 'configuracion'"
      :jugadores-previos="jugadoresSesion"
      :puntos-acumulados="puntosAcumulados"
      @comenzar="comenzarJuego"
      @volver="pantalla = 'inicio'"
    />
    <SalaJuego
      v-else-if="pantalla === 'juego'"
      :config="estadoJuego"
      @fin-juego="mostrarResultados"
      @volver="pantalla = 'configuracion'"
    />
    <PantallaResultados
      v-else-if="pantalla === 'resultados'"
      :datos="estadoJuego"
      :puntos-acumulados="puntosAcumulados"
      @jugar-otra-vez="jugarOtraVez"
      @volver-inicio="volverInicio"
    />
  </div>
</template>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #0a1628;
  color: #e8f5e9;
  min-height: 100vh;
}

.app-contenedor {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

button { cursor: pointer; font-family: inherit; }
input { font-family: inherit; }
</style>
