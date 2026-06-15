<script setup>
import { ref } from 'vue'
import PantallaInicio from './components/PantallaInicio.vue'
import ConfiguracionJuego from './components/ConfiguracionJuego.vue'
import SalaJuego from './components/SalaJuego.vue'
import PantallaResultados from './components/PantallaResultados.vue'

const pantalla = ref('inicio') // 'inicio' | 'configuracion' | 'juego' | 'resultados'
const estadoJuego = ref(null)

function iniciarConfiguracion() {
  pantalla.value = 'configuracion'
}

function comenzarJuego(config) {
  estadoJuego.value = config
  pantalla.value = 'juego'
}

function mostrarResultados(resultado) {
  estadoJuego.value = { ...estadoJuego.value, ...resultado }
  pantalla.value = 'resultados'
}

function volverInicio() {
  pantalla.value = 'inicio'
  estadoJuego.value = null
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
      @jugar-otra-vez="pantalla = 'configuracion'"
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

button {
  cursor: pointer;
  font-family: inherit;
}

input {
  font-family: inherit;
}
</style>
