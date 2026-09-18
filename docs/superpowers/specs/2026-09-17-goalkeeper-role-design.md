# Rol de arquero

## Resumen

Hoy todos los jugadores tienen una única calificación (1.0-10.0) que
determina su poder para balancear equipos. Esta feature agrega un rol
opcional de "arquero": un jugador puede marcarse como arquero y tener una
calificación de arquero separada de su calificación como jugador de campo
("doble función"). Al armar los equipos, si hay arqueros disponibles ese
día, se reparte uno por equipo automáticamente.

## Objetivos

- Marcar un jugador como arquero (al cargarlo, o después) con una
  calificación de arquero propia, independiente de su calificación de
  jugador de campo.
- Editar jugadores ya cargados — hoy no existe ninguna forma de editar,
  solo agregar y quitar. Esta feature introduce la primera edición
  (acotada a los campos de arquero).
- Al generar los equipos, si hay arqueros disponibles ese día, asignar
  exactamente uno a cada lado.

## No-objetivos

- La calificación de arquero **no** se vota entre amigos (decisión
  explícita) — es siempre manual, la carga el admin, igual que se cargaban
  todas las calificaciones antes de que existiera la votación.
- No se agrega edición del nombre ni de la calificación de jugador de
  campo para jugadores existentes — la edición nueva se limita a los
  campos de arquero (`isGoalkeeper`, `gkRating`).
- El Elo/OVR del jugador no se separa por rol. Sigue habiendo un solo Elo
  por jugador, basado en partidos ganados/perdidos, sin importar si jugó
  de arquero o de campo ese día.
- No se fuerza a que haya arqueros — con 0 o 1 arquero disponible ese día,
  los equipos se arman igual, sin avisos ni bloqueos.
- La selección de "qué arqueros juegan de arquero hoy" (cuando hay 3 o
  más disponibles) no se persiste entre sesiones — es una elección que se
  hace cada vez, en el momento de armar el partido, igual que el tildado
  de "quién juega hoy" ya funciona hoy (`p.playing`).

## Modelo de datos

Cada jugador (`players[i]`) suma dos campos:

```js
{
  ...campos existentes (id, name, rating, elo, photo, playing)...
  isGoalkeeper: false,  // boolean, default false
  gkRating: null        // 1.0-10.0, solo relevante si isGoalkeeper === true
}
```

- `rating` sigue significando exactamente lo mismo que hoy (calificación
  de jugador de campo, con votación si existe).
- `gkRating` es independiente: manual siempre, sin fallback a `rating`
  cuando falta — si `isGoalkeeper` es `true` pero `gkRating` es `null`
  (nunca se cargó), se usa un valor por defecto razonable (6.0, el mismo
  default que ya usa el slider de calificación al agregar un jugador) en
  vez de fallar o forzar al admin a cargarlo antes de poder tildar el
  checkbox.

## Interfaz

### Alta de jugador ("Tu plantel")

Se agrega un checkbox "¿Es arquero?" al formulario existente (nombre +
calificación + foto). Al tildarlo aparece un segundo slider "Calificación
de arquero", mismo estilo 1.0-10.0 que el slider de calificación
existente, con el mismo valor default (6.0).

### Edición de jugador existente ("Tu plantel", lista de roster)

Cada fila de `renderRoster()` suma un botón "Editar" junto al botón
"Quitar" existente. Al tocarlo, la fila se expande mostrando el checkbox
"¿Es arquero?" y (si está tildado) el slider de calificación de arquero,
inline debajo de la fila. Los cambios se guardan automáticamente
(`savePlayers()` + re-render) al tocar el checkbox o soltar el slider, sin
botón de confirmar aparte. Tocar "Editar" de nuevo colapsa la fila.

### Armar el partido (selección de arqueros de hoy)

En el panel "Armar el partido", después de la lista de tildado de "quién
juega hoy" (`renderPickList()`), se calcula cuántos jugadores marcados
`isGoalkeeper` están tildados como `playing`:

- **0 o 1**: no se muestra nada nuevo. Ese arquero (si hay uno) es
  automáticamente "el arquero de hoy" para el equipo que le toque.
- **2**: no se muestra nada nuevo tampoco — los dos son automáticamente
  "los arqueros de hoy", uno por lado.
- **3 o más**: aparece una sección nueva "Elegí qué 2 arqueros juegan de
  arquero hoy" con un checkbox libre por cada arquero disponible (sin
  límite forzado al tildar, igual que la lista de checkboxes que ya usa la
  restricción de "no juntos"). El botón "Generar equipos" se deshabilita,
  con un texto de ayuda ("Elegí exactamente 2 arqueros para hoy"),
  mientras no haya exactamente 2 tildados.

Los arqueros no seleccionados (cuando hay 3+) juegan igual ese día, como
jugadores de campo normales, con su `rating` de siempre — no quedan
afuera del partido.

## Armado de equipos (`generateOptions`)

Se reutiliza el mecanismo existente de restricciones de unidad
("no juntos"), que ya arma un grafo de adyacencia entre unidades del
plantel (`adj`) a partir de `avoidPairs` y fuerza a las unidades conectadas
a equipos distintos vía backtracking.

Cuando hay exactamente 2 "arqueros de hoy" (ya sea automático con 0-2
disponibles, o elegidos manualmente con 3+):

1. Se agrega una arista temporal (solo para esta generación, no se
   persiste en `avoidPairs`) entre la unidad del primer arquero y la
   unidad del segundo, con el mismo mecanismo que ya usan las aristas de
   `avoidPairs` — así el backtracking existente los manda a equipos
   distintos sin ningún cambio a su propia lógica.
2. Si ambos arqueros ya están en el mismo grupo fijo ("juegan juntos"),
   es una contradicción — mismo tipo de aviso que ya existe hoy para
   "grupo fijo vs. no juntos": *"Los dos arqueros de hoy están en el mismo
   grupo fijo, no se pueden separar. Corregí la selección de arqueros o el
   grupo."*

Con 0 o 1 arquero de hoy, no se agrega ninguna arista nueva — el armado
sigue exactamente igual que hoy.

### Balance de poder

`combinedPower(p)` sigue calculando el poder de un jugador usando
`effectiveRating(p)` (jugador de campo) tal cual hoy. Para el cálculo de
balance dentro de `generateOptions`, se usa una variante que, únicamente
para los 0-2 jugadores que son "arquero de hoy", calcula su poder a partir
de `gkRating` en vez de `rating`/`effectiveRating`, combinado con el mismo
Elo (`eloToScore10(p.elo)`) que ya tienen — no se crea un Elo separado por
rol. El resto de los jugadores usa `combinedPower(p)` sin cambios.

## Resultado (pantalla de equipos armados)

En `renderResults()`, el nombre del arquero de cada equipo se muestra con
un ícono 🧤 al lado (mismo patrón visual que ya usan otros indicadores en
las tarjetas de jugador, como los badges de racha/goleador). Si un equipo
no tiene arquero asignado ese día (0 o 1 arquero disponible en total), no
se muestra nada especial para ese equipo — ninguna advertencia, es un caso
normal.

## Casos borde

- **Jugador marcado arquero sin `gkRating` cargado**: se usa 6.0 como
  default (ver Modelo de datos).
- **Se desmarca "¿Es arquero?" de un jugador existente**: `isGoalkeeper`
  pasa a `false`; `gkRating` se conserva en el objeto (no se borra) por si
  se vuelve a marcar más adelante, pero deja de influir en nada mientras
  esté desmarcado.
- **Jugador eliminado del plantel que era arquero**: se elimina
  normalmente junto con el resto de sus campos, igual que hoy con
  cualquier otro jugador.
- **Cambiar el formato del partido (5v5, 6v6, etc.) después de elegir los
  2 arqueros de hoy**: la selección de arqueros no depende del formato, se
  mantiene igual.

## Plan de pruebas

- Agregar un jugador nuevo marcándolo arquero con una calificación
  distinta a la de jugador de campo; confirmar que ambos valores se
  guardan por separado.
- Editar un jugador existente para marcarlo arquero después de cargado;
  confirmar que aparece el slider de calificación de arquero y que se
  guarda.
- Desmarcar "¿Es arquero?" de un jugador y confirmar que ya no cuenta como
  arquero disponible al armar el partido, sin perder su `gkRating` guardado.
- Armar equipos con exactamente 2 arqueros disponibles: confirmar que
  terminan en equipos distintos en todas las opciones generadas.
- Armar equipos con 3+ arqueros disponibles sin elegir 2: confirmar que
  "Generar equipos" queda deshabilitado hasta elegir exactamente 2.
- Armar equipos con 1 arquero disponible: confirmar que se arma sin avisos
  ni bloqueos, y que ese equipo muestra el ícono 🧤 y el otro no.
- Armar equipos con 0 arqueros disponibles: confirmar que el
  comportamiento es idéntico al que existía antes de esta feature.
- Marcar como arqueros de hoy a dos jugadores que ya están en el mismo
  grupo fijo ("juegan juntos"); confirmar que se muestra el aviso de
  contradicción y no se generan equipos.
