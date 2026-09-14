# Votación entre amigos (reemplazo del Google Forms)

## Resumen

Hoy la calificación (1-10) de cada jugador la carga a mano quien administra
el plantel, en el panel "Tu plantel". El dueño del grupo venía juntando esos
puntajes por Google Forms y cargándolos manualmente. Esta feature permite
que los propios amigos del plantel voten (califiquen) a los demás jugadores
directamente desde la app, sin pasar por Forms ni por el admin.

## Objetivos

- Los jugadores del plantel pueden puntuar (1.0-10.0) a sus compañeros desde
  un link simple, sin necesidad de usuario/contraseña.
- El promedio de los votos recibidos por un jugador reemplaza automáticamente
  su calificación base, la cual sigue alimentando el sistema de Elo/OVR ya
  existente (que sube o baja según partidos ganados/perdidos).
- No se pierde ni se recalcula manualmente nada del historial: al cambiar la
  calificación base de un jugador, todo el historial de Elo se recalcula solo
  porque `recalculateAllElo()` ya reconstruye el Elo desde cero en cada render.

## No-objetivos

- No se implementa un sistema de autenticación con contraseña/PIN (decisión
  explícita: es un grupo de amigos de confianza, no hace falta).
- No se restringe el acceso a la app completa a quien tenga el código de
  grupo — el link de votación es una capa de conveniencia, no de seguridad.
  Cualquiera con el código puede, si quiere, entrar a la app completa y no
  solo a votar. Es el mismo nivel de confianza que ya tiene hoy toda la app.
- No se agrega un umbral mínimo de votos para que el promedio "cuente"; con
  un solo voto recibido ya reemplaza la calificación manual.
- No se permite editar el voto después de enviarlo.

## Arquitectura

Un único archivo (`index.html`), sin build step, tal como está hoy. Se agrega
un modo alternativo activado por query param:

- `armaequipo.tomasgima7.workers.dev/?votar=<codigo-de-grupo>` → pantalla de
  votación standalone (oculta el resto de los paneles de la app).
- Sin el parámetro `votar` → comportamiento actual sin cambios.

Se descarta un archivo HTML separado (`votar.html`) para no duplicar CSS ni
la inicialización de Firebase entre dos archivos.

## Modelo de datos (Firestore)

Se agrega un campo nuevo al mismo documento de grupo que ya existe
(`armaequipos/{codigo}`), junto a `playersJson` y `matchesJson`:

```
votesJson: JSON.stringify([
  { voterId: "p123abc", targetId: "p456def", score: 7.5 },
  { voterId: "p123abc", targetId: "p789ghi", score: 6.0 },
  ...
])
```

- Cada entrada es un voto de una persona (`voterId`) sobre otra
  (`targetId`) con un puntaje (`score`, 1.0-10.0).
- **No** hay un campo separado de "quién ya votó": se infiere. Si existe al
  menos una entrada con ese `voterId` en `votesJson`, esa persona ya votó y
  no puede volver a enviar votos.
- Los votos, igual que `players` y `matches` (pero a diferencia de `groups`/
  `avoidPairs`), sí se persisten en Firestore y en localStorage — no aplica
  la política de "nunca guardado" que se implementó para las restricciones.

## Cálculo de la calificación base

En `recalculateAllElo()` (o justo antes, en `renderAll()`), para cada
jugador:

1. Si tiene 1 o más votos recibidos en `votesJson` (entradas donde
   `targetId === p.id`), su `rating` efectivo pasa a ser el promedio de esos
   votos.
2. Si no tiene votos, se usa el `rating` manual que ya tiene guardado (el que
   se le puso al agregarlo, o el último valor manual).
3. Ese `rating` efectivo alimenta `initialElo(rating)` exactamente igual que
   hoy, y `recalculateAllElo()` reproduce todo el historial de partidos para
   recalcular el Elo/OVR de punta a punta. No hace falta tocar esa función
   más que leer el nuevo `rating` efectivo en vez del campo manual crudo.

El campo `p.rating` manual (el que se edita con el slider en "Tu plantel")
se conserva sin cambios — es el valor de respaldo antes de que haya votos.
El promedio de votos vive en una función derivada, no pisa el campo manual
guardado (así si algún día se sacan los votos, se puede volver al valor
manual sin perder el dato).

## Pantalla de votación

Flujo, en `?votar=<codigo>`:

1. **Elegir quién sos**: lista con los nombres del plantel actual. Los
   nombres que ya emitieron su voto se muestran atenuados con la etiqueta
   "ya votaste" y no son clickeables.
2. **Ballot**: al elegir un nombre, se muestra la lista de todos los demás
   jugadores del plantel (no incluye a quien está votando). Cada jugador
   tiene un slider 1.0-10.0 sin valor precargado. El votante puede dejar
   sliders sin tocar para los jugadores que no conoce — esos no se envían
   como voto.
3. **Envío**: un solo botón "Enviar votos" que manda todos los puntajes
   marcados de una sola vez. Tras enviar, ese `voterId` queda bloqueado para
   siempre (no hay edición ni reenvío).
4. Contador visible arriba: "Ya votaron X de Y" (cuenta jugadores distintos
   con al menos un voto emitido, sobre el total del plantel).

No hay paso de confirmación adicional ni resumen antes de enviar — mandar el
voto es la acción final.

## Cambios en el panel de admin ("Tu plantel")

Al lado de cada jugador en la lista del roster, se agrega un texto chico
indicando el estado de sus votos:

- Con votos: `★ 7.2 (5 votos)`
- Sin votos: `sin votos todavía (usando calificación manual)`

El slider manual de calificación sigue existiendo tal cual está hoy (se usa
al agregar un jugador nuevo, y como respaldo mientras no tiene votos).

## Casos borde

- **Jugador nuevo agregado después de que otros ya votaron**: no tiene votos
  de quienes ya enviaron su ballot (no pueden volver a votar). Su promedio
  se arma solo con los votos de quienes voten de ahí en adelante. No requiere
  manejo especial.
- **Jugador eliminado del plantel que tenía votos**: sus votos recibidos
  quedan huérfanos en `votesJson` pero no afectan a nadie (se filtran por
  `targetId` existente al calcular promedios, igual que ya se hace con el
  historial de partidos y jugadores eliminados).
- **Alguien vota por otra persona a propósito** (eligiendo un nombre que no
  es el suyo): posible, aceptado como riesgo conocido dado el nivel de
  confianza del grupo (ver No-objetivos).

## Plan de pruebas

- Votar por un jugador desde cero (sin votos previos) y confirmar que su
  OVR se recalcula usando el promedio en vez del valor manual.
- Confirmar que el historial de partidos ya jugados sigue produciendo el
  mismo delta de Elo relativo aunque cambie el punto de partida (`rating`).
- Confirmar que un `voterId` que ya votó no puede volver a enviar un ballot
  (ni editar valores existentes).
- Confirmar que saltear jugadores en el ballot no genera votos con score
  vacío/inválido en `votesJson`.
- Confirmar que la app sin `?votar=` sigue funcionando exactamente igual
  que antes de esta feature.
