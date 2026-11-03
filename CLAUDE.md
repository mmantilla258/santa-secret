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
  - Datos: `deseos/{nombre}` = `{ deseo, actualizado }` · `config/sorteo` = `{ presupuesto, fecha_entrega, realizado }`.
  - Las reglas viven en `firestore.rules` (copia en el repo); Miguel las pega a mano en la consola de Firebase.
    Si se cambian, recordarle que las vuelva a publicar.
- Idioma: toda la interfaz, mensajes y comentarios del código en **español**.
- Debe verse bien en celular (la mayoría de familiares entrará desde el teléfono).

## Seguridad (obligatorio)

- **Nunca** poner secretos en el código: el repositorio es público. Solo pueden ir en el frontend
  valores diseñados para ser públicos: la Public Key de EmailJS y el objeto `firebaseConfig`.
- Una versión antigua tenía un token de Meta/WhatsApp en texto plano. Ya se abandonó WhatsApp;
  si aparece algún token de Meta en el código o el historial, avisar a Miguel para que lo revoque.
- Hoy el PIN se genera y valida **en el navegador** (`pinActual`), así que cualquiera puede verlo
  con F12. Es aceptable solo como primera barrera. La protección real de los datos debe venir de
  las **reglas de seguridad de Firestore**, nunca de ocultar cosas en el JavaScript.
- Los correos de los familiares hoy están en el objeto `correos` del código (visibles en el repo).
  Idealmente moverlos a Firestore cuando se integre Firebase.

## Flujo actual (autenticación por PIN, ya implementado)

1. El usuario elige su nombre en `#participanteSelect`.
2. "Solicitar PIN" genera un PIN de 4 dígitos y lo envía por EmailJS al correo del objeto `correos`.
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
4. ✅ Cada usuario guarda su **deseo de regalo** (máx. 200 caracteres; editable mientras `config/sorteo.realizado != true`).
   - **Fecha del sorteo: domingo 8 de noviembre de 2026, 8:00 a. m. hora de Colombia** (`FECHA_SORTEO` en
     `index.html`, `timestamp.value(1794142800000)` en `firestore.rules`, y textos del HTML: cambiar los tres juntos).
     Desde esa hora ya no se pueden editar deseos. La página muestra una cuenta regresiva.
   - No hay servidor: el sorteo no se ejecuta solo; Miguel lo lanza desde la página a esa hora.
   - Presupuesto **$250.000 COP** y entrega el **24 de diciembre** (`PRESUPUESTO` y `FECHA_ENTREGA` en `index.html`).
   - **Fase de prueba interna:** Familiar2..8 usan alias de Gmail (`mmantilla258+familiarN@gmail.com`) que llegan
     a la bandeja del organizador. Antes de invitar a la familia, cambiar nombres y correos reales (en `correos` y en el `<select>`).
   - El participante 1 (correo `mmantilla258@gmail.com`, el organizador) aparece como **Alejandro** (`ORGANIZADOR`).
5. ✅ **Sorteo** (implementado; panel del organizador visible solo para `ORGANIZADOR`):
   - Antes de `FECHA_SORTEO` es **modo prueba** (repetible, no escribe en Firebase). Desde esa hora es real y
     escribe `config/sorteo = { realizado: true, fecha }` (las reglas solo lo permiten desde la fecha).
   - Las asignaciones **no se guardan** en Firestore: solo viajan en los correos (8 envíos por sorteo).
   - `PAREJAS_EXCLUIDAS` (ambos sentidos): Alejandro no puede dar ni recibir de Familiar2, 3, 4 y 5.
     Al poner los nombres reales, actualizar también esta lista.
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
