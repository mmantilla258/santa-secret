# Secret Santa 2026

Aplicación web para organizar un intercambio de regalos ("amigo secreto") entre **8 familiares**.
El usuario (Miguel) está aprendiendo: explica los cambios en español, en pasos cortos, y pide
confirmación antes de decisiones que no se puedan deshacer.

## Stack y restricciones

- **Frontend:** HTML + CSS + JavaScript vanilla. Todo vive en un solo `index.html` mientras sea manejable.
  Si crece demasiado, se puede separar en `styles.css` y `app.js`, pero **sin frameworks, sin npm y sin paso de build**.
- **Hosting:** GitHub Pages (sitio estático, repositorio público). No hay servidor propio.
- **Correo:** EmailJS (SDK v4 por CDN: `@emailjs/browser@4`), plan gratuito. Todo lo secreto (PIN, sorteo) va por correo.
- **WhatsApp:** solo un botón "Compartir por WhatsApp" con un enlace `wa.me` que comparte la URL de la página.
  Sin API de Meta ni tokens (decidido 2026-11-02: no hay opción automática gratis sin servidor).
- **Base de datos:** Firebase (Firestore), proyecto `santa-secret-2026-45ed4`, plan Spark. SDK modular por CDN
  fijado en `12.19.0` (`https://www.gstatic.com/firebasejs/12.19.0/firebase-*.js`) con `<script type="module">`.
  - Datos: `deseos/{nombre}` = `{ opciones: [{ texto, link, precio }], combos: [{ opciones: [índices] }], actualizado }`
    (máx. 10 opciones y 20 combos; los documentos viejos `{ deseo }` se leen como "Opción 1")
    · `correos/{nombre}` = `{ correo, creado }` (solo se crea una vez)
    · `config/sorteo` = `{ realizado, fecha }`.
  - Las reglas viven en `firestore.rules` (copia en el repo); Miguel las pega a mano en la consola de Firebase.
    Si se cambian, recordarle que las vuelva a publicar.
- Idioma: comentarios del código en **español**. La página tiene selector de idioma (banderas arriba a la izquierda):
  **español, inglés y neerlandés**. Todo texto visible va en el diccionario `TEXTOS` (claves `es`/`en`/`nl`) y se usa
  con `data-i18n="clave"` en el HTML o `t("clave")` / `mostrarEstado("clave", …)` en el script: al agregar un texto,
  traducirlo a los tres. Los correos (plantillas de EmailJS) siguen solo en español.
- Debe verse bien en celular (la mayoría de familiares entrará desde el teléfono).

## Seguridad (obligatorio)

- **Nunca** poner secretos en el código: el repositorio es público. Solo pueden ir en el frontend
  valores diseñados para ser públicos: la Public Key de EmailJS y el objeto `firebaseConfig`.
- Una versión antigua tenía un token de Meta/WhatsApp en texto plano. Ya se abandonó WhatsApp;
  si aparece algún token de Meta en el código o el historial, avisar a Miguel para que lo revoque.
- Hoy el PIN se genera y valida **en el navegador** (`pinActual`), así que cualquiera puede verlo
  con F12. Es aceptable solo como primera barrera. La protección real de los datos debe venir de
  las **reglas de seguridad de Firestore**, nunca de ocultar cosas en el JavaScript.
- Los correos de los familiares ya **no** están en el código: cada uno registra el suyo la primera vez que entra
  (se guarda en `correos/{nombre}` solo después de validar el PIN). Solo el del organizador está fijo en `CORREOS_FIJOS`.
  Con la Opción B, cualquiera que conozca el nombre exacto podría leer ese correo.

## Flujo actual (autenticación por PIN, ya implementado)

1. El usuario elige su nombre en `#participanteSelect` (opciones generadas desde `PARTICIPANTES`).
   Si aún no tiene correo guardado, aparece `#zonaRegistro` para escribirlo (solo la primera vez).
2. "Enviarme el código" genera un PIN de 4 dígitos y lo envía por EmailJS a su correo.
3. Reglas de tiempo (no cambiarlas sin preguntar):
   - El PIN vale **5 minutos** desde el último envío.
   - Botón "Reenviar código" con **cooldown de 60 s**.
   - A los **8 minutos** del primer envío se genera un PIN completamente nuevo.
   - El PIN queda ligado al usuario: el selector se bloquea; "No soy yo / cambiar nombre" reinicia todo.
   - Un PIN validado no se puede reutilizar.
4. Con el PIN correcto se muestra `#zonaRegalo`, donde irá la lógica de Firebase.

## EmailJS

Constantes al inicio del `<script>`: `EMAILJS_PUBLIC_KEY`, `EMAILJS_SERVICE_ID`, `EMAILJS_TEMPLATE_ID`
y (por agregar) `EMAILJS_TEMPLATE_SORTEO`. El plan gratuito tiene pocas plantillas: no crear más de estas dos.

| Plantilla | Variables (deben coincidir exactamente con `{{...}}` en EmailJS) |
|---|---|
| PIN (`EMAILJS_TEMPLATE_ID`) | `to_email`, `to_name`, `pin`, `minutos` |
| Sorteo (`EMAILJS_TEMPLATE_SORTEO`) | `to_email`, `to_name`, `amigo`, `deseo`, `presupuesto`, `fecha_entrega` |

`deseo` llega como HTML (opciones con precio/link y combos, armado por `deseoParaCorreo` con el texto escapado):
en la plantilla debe ir con **tres llaves** `{{{deseo}}}` para que se vean los saltos de línea y los links.

Siempre manejar el error de `emailjs.send` y mostrarlo en pantalla; nunca mostrar "enviado ✅" si falló.

## Hoja de ruta (en este orden)

1. ✅ Autenticación por PIN con EmailJS.
2. ✅ **Decisión tomada (2026-11-02): Opción B** — se mantiene el PIN de EmailJS para el acceso.
   Miguel prioriza lo más fácil y gratis sobre la seguridad máxima. Consecuencia aceptada: Firestore no
   sabe quién es quién, así que alguien con conocimientos técnicos podría leer asignaciones ajenas.
   (Descartada la Opción A —Firebase Authentication— entre otras cosas porque el plan Spark solo
   permite 5 correos de enlace al día.)
   - EmailJS gratuito **no permite restringir dominios**: la lista de dominios queda vacía y solo se
     mantiene desactivada la opción "Allow EmailJS API for non-browser applications".
3. ✅ Conectar Firebase (proyecto, `firebaseConfig`, Firestore en modo producción con reglas propias).
4. ✅ Cada usuario guarda su **deseo de regalo** como **opciones** (texto + mini pestañas 🔗 Link y 💲 Precio, botón
   "＋ Agregar otra opción") y, opcionalmente, **combos**: marca ☑ varias opciones, ve la suma contra el presupuesto
   (aviso si se pasa, pero deja guardar) y la guarda como "Combo N". Editable mientras el plazo esté abierto.
   El correo del sorteo lleva todas las opciones y, si hay, los combos.
   - **Fecha del sorteo: domingo 8 de noviembre de 2026, 8:00 a. m. hora de Colombia** (`FECHA_SORTEO` en
     `index.html`, `timestamp.value(1794142800000)` en `firestore.rules`, y textos del HTML: cambiar los tres juntos).
     Desde esa hora ya no se pueden editar deseos. La página muestra una cuenta regresiva.
   - No hay servidor: el sorteo no se ejecuta solo; Miguel lo lanza desde la página a esa hora.
   - Presupuesto **$250.000 COP** y entrega el **24 de diciembre** (`PRESUPUESTO` y `FECHA_ENTREGA` en `index.html`).
   - **Participantes** (`PARTICIPANTES`): Alejandro, Luddy Laguado, Ismael Moncada, Miguel Angel, Blanca,
     Estefania, Bart, Angelica. El nombre es el ID en Firestore: cambiarlo deja huérfanos sus datos.
   - El participante 1 (correo `mmantilla258@gmail.com`, el organizador) aparece como **Alejandro** (`ORGANIZADOR`).
5. ✅ **Sorteo** (implementado; panel del organizador visible solo para `ORGANIZADOR`):
   - Antes de `FECHA_SORTEO` es **modo prueba** (repetible, no escribe en Firebase). Desde esa hora es real y
     escribe `config/sorteo = { realizado: true, fecha }` (las reglas solo lo permiten desde la fecha).
   - Las asignaciones **no se guardan** en Firestore: solo viajan en los correos (8 envíos por sorteo).
   - `PAREJAS_EXCLUIDAS` (ambos sentidos): Alejandro no puede dar ni recibir de Luddy Laguado, Ismael Moncada,
     Miguel Angel ni Blanca.
   - El sorteo no se hace si alguien aún no registró su correo (muestra quiénes faltan).
   Requisitos originales:
   - Permutación aleatoria donde nadie se saca a sí mismo (derangement). Usar Fisher-Yates con
     `crypto.getRandomValues` y reintentar hasta cumplir las restricciones.
   - Soportar una lista opcional de parejas excluidas (por ejemplo, esposos que no se regalan entre sí).
   - Preferir un ciclo único (A→B→C→…→A) para que nadie quede en un par cerrado, si las exclusiones lo permiten.
   - Solo el organizador puede ejecutarlo, una sola vez (o con confirmación explícita para repetir).
   - **El organizador no debe ver los resultados en pantalla:** el sorteo guarda las asignaciones y
     envía a cada participante su correo con la plantilla de sorteo, sin mostrar nada en la UI ni en `console.log`.
6. Vista del participante tras iniciar sesión: a quién le regala, el deseo de esa persona,
   presupuesto y fecha de entrega.
7. Reglas de seguridad de Firestore: cada usuario lee y edita solo su propio deseo; lee solo su
   propia asignación y el deseo de su amigo secreto; nadie lee el sorteo completo.

## Cómo trabajar en este proyecto

- Antes de cambios grandes (Firebase, sorteo, reglas), presentar un plan breve y esperar aprobación.
- Cambios pequeños e incrementales; después de cada uno, decir exactamente cómo probarlo en el navegador.
- Para probar en local: abrir `index.html` con un servidor simple (por ejemplo `python -m http.server` o la
  extensión Live Server de VS Code), no con doble clic, porque los módulos ES de Firebase no cargan con `file://`.
- Miguel no tiene Node.js: para probar en local usar `python -m http.server 8000` y abrir `http://localhost:8000`.
- No hacer `git push` sin que Miguel lo pida; GitHub Pages publica automáticamente lo que se sube.
- Mantener los textos de la interfaz cálidos y navideños (🎅🎁), pero sin exagerar.
