# 🌍 El Impostor Ambiental

Juego educativo de deducción con temática ecológica. Desarrollado con **Vue 3 + Vite** para el curso IF7102 Multimedios, UCR.

## Descripción

"El Impostor Ambiental" es un juego multijugador local (en un mismo dispositivo) donde los jugadores reciben roles secretos. La mayoría conoce una **palabra ambiental secreta** y debe dar pistas. Un jugador — el **Impostor** — no sabe la palabra y debe fingir que sí la conoce. El grupo debe adivinar la palabra Y descubrir al Impostor.

### Palabras temáticas incluidas

Deforestación · Reciclaje · Cambio Climático · Energía Solar · Biodiversidad · Efecto Invernadero · Contaminación Plástica · Fotosíntesis · Sequía · Energía Eólica · Huella de Carbono · Manglar · Compostaje · Capa de Ozono · Calentamiento Global

## Tecnologías

- **Framework**: Vue 3 (Composition API con `<script setup>`)
- **Build tool**: Vite
- **Estilos**: CSS personalizado con variables CSS (sin librerías de UI externas)
- **Datos**: `fetch()` cargando `juego-data.json` en `onMounted`

## Requisitos del IF7102 cubiertos

| Requisito | Detalle |
|-----------|---------|
| ✅ Framework Vue 3 | Composition API, `ref`, `computed`, `onMounted`, `props`, `emit` |
| ✅ 4+ componentes | `PantallaInicio`, `ConfiguracionJuego`, `SalaJuego`, `PantallaResultados`, `TarjetaJugador` |
| ✅ JSON + fetch() | `juego-data.json` cargado dinámicamente al montar el componente |
| ✅ Responsividad | CSS Grid + media queries para móvil y escritorio |
| ✅ Multimedia | Efectos de sonido (4 clips), animaciones CSS |
| ✅ 3 pantallas | Inicio → Configuración → Juego → Resultados |
| ✅ Estado reactivo | `ref` para fases, puntos, pistas, votos |
| ✅ Eventos | `@click`, `@keydown`, `@keydown.enter`, `v-model` |

## Estructura del Proyecto

```
src/
├── components/
│   ├── PantallaInicio.vue      ← Pantalla de bienvenida con instrucciones
│   ├── ConfiguracionJuego.vue  ← Configurar jugadores y palabra
│   ├── SalaJuego.vue           ← Asignación de roles, pistas y votación
│   ├── PantallaResultados.vue  ← Resultados y puntuación final
│   └── TarjetaJugador.vue      ← Componente reutilizable de jugador
├── data/
│   └── juego-data.json         ← Palabras ambientales y configuración
├── App.vue                     ← Orquestador de pantallas
├── main.js
└── style.css

public/
├── juego-data.json             ← Copia accesible por fetch()
├── audio/
│   ├── click.mp3
│   ├── acierto.mp3
│   ├── error.mp3
│   └── votacion.mp3
└── images/
```

## Cómo ejecutar

```bash
npm install
npm run dev
```

Abrir en el navegador: `http://localhost:5173`

## Cómo construir para producción

```bash
npm run build
npm run preview
```

## Cómo jugar

1. **Inicio**: Lee las instrucciones y pulsa "Comenzar Juego"
2. **Configurar**: Agrega los nombres de todos los jugadores (mín. 3) y elige si la palabra es aleatoria o manual
3. **Asignación de Roles**: Cada jugador pasa el dispositivo para ver su rol en privado (el Impostor no verá la palabra)
4. **Pistas**: Cada jugador escribe una pista. El Impostor debe inventar algo plausible
5. **Adivinanza**: El grupo discute y escribe su mejor conjetura de la palabra
6. **Votación**: El grupo vota quién creen que es el Impostor
7. **Resultados**: Se revelan roles, pistas y puntuación

---

**IF7102 Multimedios | I Ciclo 2026 | UCR — Sede Regional**
