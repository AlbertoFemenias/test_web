# Clave Maestra

App web para aprender solfeo y piano en español: dos campañas (leer notas y tocarlas en el teclado), arcade, partituras de dominio público y teoría.

## Publicar en GitHub Pages

1. Crea un repositorio nuevo en GitHub (por ejemplo `clave-maestra`).
2. Sube todos los archivos de esta carpeta a la raíz del repositorio (incluido `.nojekyll`).
3. En el repositorio: **Settings › Pages › Build and deployment**, elige *Deploy from a branch*, rama `main` y carpeta `/ (root)`. Guarda.
4. En un par de minutos estará en `https://TU-USUARIO.github.io/clave-maestra/`.

## En el iPhone

Abre el enlace en Safari › Compartir › **Añadir a pantalla de inicio**. Así:

- se abre a pantalla completa como una app y funciona sin conexión;
- el micrófono funciona (Safari te pedirá permiso la primera vez);
- Safari no borra tus datos por falta de uso (en una pestaña normal puede borrarlos si pasas semanas sin abrirla).

## Progreso

Se guarda en el propio dispositivo. Para no perderlo: **Ajustes › Contraseña de progreso**. Apunta la contraseña de 16 caracteres; con ella recuperas capítulos, estrellas, récord y XP en cualquier dispositivo. La copia en archivo incluye además la voz entrenada.

## Documentación

`ESPECIFICACION.md` contiene la especificación completa del producto: modelo musical, contenido de las campañas, sistemas de juego, diseño visual, motor de voz, formato de la contraseña y criterios de aceptación. Sirve para reconstruir la app desde cero.

## Actualizar

Sustituye `index.html` y sube el número de `CACHE` en `sw.js` (por ejemplo `clave-maestra-v11`) para que los móviles descarguen la versión nueva.
