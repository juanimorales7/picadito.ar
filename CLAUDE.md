# Picadito.ar

Plataforma web de fútbol amateur competitivo para Mendoza, Argentina
(San Rafael, General Alvear, Capital). Producto de Appikko Tech.
Lema: "¡Ya no juegues más por la coca!"

La idea: tomar lo que hace FACEIT con Counter-Strike y League of Legends con
su ranking, y bajarlo al fútbol 5, 7 y 11. ELO, rangos, clubes, lobbies,
estadísticas.

---

## Con quién estás trabajando

Juan Ignacio Morales ("La Garza"). **Es programador principiante.** Esto importa:

- **Hablale siempre en español**, en argentino, sin tecnicismos innecesarios.
- Cuando algo falla, explicale **qué** falló y **por qué**, no solo lo arregles.
- Cuando le propongas opciones, numeralas y letralas (`1-a`, `1-b`, `2-a`...).
  Él responde con esos códigos. Es la forma de trabajo que ya tenemos.
- **Una cosa a la vez.** No le metas cinco features juntas.
- Si te pide algo que ya existe en el código, decíselo en vez de duplicarlo.
  Pasó una vez: se construyó un sistema de bloqueo entero que ya estaba hecho.

---

## Arquitectura

**Todo el proyecto es un solo archivo: `index.html`** (~800 KB). Adentro tiene
el HTML, el CSS, el JavaScript, y las imágenes y sonidos embebidos en base64.

No hay build. No hay framework. No hay dependencias que instalar. Se abre el
archivo en el navegador y funciona.

### Por qué está así

Fue la decisión correcta para arrancar: cero configuración, se sube a cualquier
hosting arrastrando un archivo. **Pero ya está en el límite de lo manejable.**
Si Juan te pide separarlo en `index.html` + `estilos.css` + `app.js`, hacelo —
es la próxima mejora grande. No lo separes por tu cuenta sin que lo pida.

### Scripts dentro del archivo

1. Un `<script>` temprano y chiquito que aplica el tema guardado antes de pintar
   (evita el flash blanco).
2. Un `<script type="module">` que importa Firebase y expone todo en
   **`window.FB`** (el puente entre el módulo y el script clásico).
3. El `<script>` principal al final: toda la lógica de la app.

Si agregás una función de Firebase nueva, **tenés que exponerla en `window.FB`**
o el script principal no la ve.

---

## Firebase

Dos bases, a propósito:

**Cloud Firestore** — lo persistente:
`usuarios`, `equipos` (los clubes), `partidos`, `solicitudes`

**Realtime Database** — lo instantáneo:
`presencia`, `chats`, `misChats`, `lecturas`, `lobbies`,
`invitacionesEquipo`, `invitacionesLobby`, `chatsEquipo`, `chatsLobby`

Listeners en vivo: `onSnapshot` para Firestore, `onValue` para RTDB.

### Reglas de seguridad

Están escritas a mano, colección por colección. **No están en modo de prueba.**
Cada vez que agregás una colección o rama nueva, hay que agregarle su regla o
va a tirar `PERMISSION_DENIED`. Ya pasó como seis veces.

Cuando le pases reglas nuevas a Juan, **pasale el archivo de reglas completo**,
no un bloque suelto. Dos veces pegó un bloque después de la llave de cierre y
quedó el JSON roto.

### Los chats se borran solos

Los mensajes con más de **3 días** se eliminan cada vez que se abre un chat.
Es a propósito: menos costo de almacenamiento y más privacidad, sin Cloud
Function (que requiere plan Blaze pago). Constante: `TRES_DIAS_MS`.

---

## Reglas de estética (importantes — son gustos ya decididos)

- **Bordes cuadrados.** `border-radius: 0` en casi todo. Juan lo pidió
  explícitamente: *"es muy parecido a la mayoría de app creadas con IA"*.
  **Única excepción:** los switches de sonido van redondeados (`20px`).
- **Nada de emojis en la interfaz.** Se sacaron todos y se reemplazaron por
  íconos de línea en SVG (`stroke`, sin `fill`). No los traigas de vuelta.
- **Dos temas**, con más de 300 variables CSS cada uno:
  - `:root` = tema oscuro (verde muy oscuro, es el original)
  - `[data-tema="claro"]` = tema claro, pero **verde, no blanco**
  - Cambian **todos** los colores, no solo el fondo.
  - **Los colores de rango NO se tematizan** (#cd7f32 bronce, #c0c0c0 plata,
    #f59e0b oro, #a855f7 platino, #5fd3f3 diamante). Son datos, no decoración.
- **Nada de `alert()` de Chrome.** Hay `avisar(mensaje, tipo)` con tipos
  `'exito'`, `'error'`, `'info'`, y `confirmar()` para los sí/no.
- **Nombres en español.** Funciones, variables, clases CSS, IDs: todo en
  español (`dibujarSlots`, `escucharChat`, `panel-izquierdo`). Es consistente
  en todo el archivo, no lo rompas con nombres en inglés.
- Layout: rail de navegación a la izquierda (estilo Instagram), panel derecho
  colapsable (estilo Discord), los dos a toda la altura, con reflow real vía
  `--ancho-izq` / `--ancho-der` en CSS Grid.

---

## Verificación antes de entregar

**Esto no es opcional.** El archivo es enorme y un error de sintaxis lo rompe
entero sin aviso. Corré esto después de cada cambio:

```bash
# 1. Divs balanceados
python3 -c "h=open('index.html').read(); print('divs:', h.count('<div'), h.count('</div>'))"

# 2. El script principal parsea
python3 -c "
h=open('index.html').read()
i=h.rfind('<script>'); j=h.find('</script>', i)
open('/tmp/main.js','w').write(h[i+8:j])
" && node -c /tmp/main.js

# 3. El script del tema también parsea (es otro script aparte)
python3 -c "
h=open('index.html', encoding='utf-8').read()
i=h.find('var guardado = null')
ini=h.rfind('<script>', 0, i); fin=h.find('</script>', ini)
open('/tmp/tema.js','w').write(h[ini+8:fin])
" && node -c /tmp/tema.js
```

Además, revisá a mano:

- **Variables de tema:** toda variable usada tiene que estar definida en
  **los dos** temas. Y ninguna puede referenciarse a sí misma
  (`--ficha-b: var(--ficha-b)` pasó y dejó la ficha transparente).
- **Funciones llamadas desde HTML** (`onclick`, `oninput`, `onchange`,
  `oncontextmenu`, `ondrop`) tienen que existir en el JS.
- **Funciones llamadas desde el propio JS** también. `escudoHTML` se llamaba en
  6 lugares y nunca se había definido, porque la verificación solo miraba los
  `onclick`. Revisá las dos cosas.
- **`getElementById('x')`** tiene que tener su `id="x"` en el HTML.
  (`res-rival` aparece pero es benigno: está en código que quedó para el
  matchmaking futuro y nunca se llama.)

---

## Lo que falta

1. **Invitaciones**: hacerlas distintas, por la página, y agregar **link de
   invitación**. (Quedó pedido y sin hacer.)
2. **Roles de club**: creador = "10", asistente = "5", tercero = "arquero",
   resto = "jugadores". Pendiente de definir si son solo etiquetas o permisos
   reales.
3. **Matchmaking real** — el grande. Partidos agendados, carga de resultado al
   capitán, sistema de honor/reputación estilo LoL, MMR.
   → **Está especificado en detalle en `matchmaking-spec.txt`**, en la raíz del
   proyecto: modelo de datos, algoritmo de emparejamiento, fórmula de ELO,
   reglas de seguridad y 4 fases de implementación con criterios de aceptación.
   Leelo antes de tocar nada de esto. **Implementá de a una fase**, no todas
   juntas, y esperá la aprobación de Juan entre fase y fase.
4. **Mapa de canchas** — hoy es una página de "Próximamente".
5. **Separar el archivo** en HTML + CSS + JS.
6. **Ordenar las reglas de Firebase** — se parchearon muchas veces de a poco.
7. **Proteger el ELO** con Cloud Functions (necesita plan Blaze, pago).

---

## Cosas que NO hay que hacer

- No agregues frameworks, bundlers ni `npm install` de nada. El proyecto se
  abre en el navegador y punto.
- No reemplaces `avisar()` por `alert()`.
- No redondees bordes.
- No metas emojis.
- No renombres cosas al inglés.
- No toques los colores de rango.
- No entregues sin correr la verificación de arriba.
