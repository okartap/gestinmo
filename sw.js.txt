// Service Worker de Patrimer — PUNTO 3 (25/08/2026)
//
// Qué hace, en corto: la primera vez que la app se abre con internet, guarda
// una copia de sí misma (el HTML, el manifest y los iconos) dentro del propio
// móvil/tablet. Si la próxima vez no hay cobertura, la app abre igual usando
// esa copia guardada, en vez de quedarse en blanco.
//
// Estrategia elegida a propósito — "red primero, copia guardada de reserva":
// como esta app se actualiza varias veces por semana (Claude entrega un
// archivo nuevo para sustituir al anterior), NO conviene que el Service
// Worker se quede enganchado para siempre a la primera copia que guardó.
// Cada vez que hay internet, se pide la versión más reciente al servidor y
// esa es la que se ve y la que se vuelve a guardar. Solo si de verdad no hay
// cobertura, se usa la última copia que se pudo guardar. Así nunca hace
// falta "desinstalar y reinstalar" para ver los cambios nuevos.
//
// Subir un número aquí (v1 → v2...) SOLO hace falta si algún día se quiere
// forzar a limpiar cachés antiguas de golpe; para el uso normal, con
// sustituir el archivo de la app es suficiente, esto se auto-gestiona solo.
const CACHE_NAME = 'patrimer-shell-v1';

// Archivos que se intentan guardar de entrada. Se listan varios nombres
// posibles del archivo principal porque no sabemos con qué nombre exacto
// queda publicado en GitHub Pages (gestinmo.html, index.html...) — los que no
// existan simplemente se ignoran, no rompen el resto.
const APP_SHELL = [
  './',
  './index.html',
  './gestinmo.html',
  './manifest.json',
  './icon-192.png',
  './icon-512.png',
  './icon-512-maskable.png'
];

self.addEventListener('install', function (event) {
  self.skipWaiting();
  event.waitUntil(
    caches.open(CACHE_NAME).then(function (cache) {
      return Promise.all(
        APP_SHELL.map(function (url) {
          return cache.add(url).catch(function () {
            // Archivo que no existe con ese nombre en este sitio — normal,
            // se ignora sin romper la instalación del resto.
          });
        })
      );
    })
  );
});

self.addEventListener('activate', function (event) {
  event.waitUntil(
    caches.keys().then(function (nombres) {
      return Promise.all(
        nombres
          .filter(function (nombre) { return nombre !== CACHE_NAME; })
          .map(function (nombre) { return caches.delete(nombre); })
      );
    }).then(function () { return self.clients.claim(); })
  );
});

self.addEventListener('fetch', function (event) {
  const req = event.request;

  // Solo se gestiona lo propio de la app (GET, mismo origen). Todo lo demás
  // (Firebase, jsPDF, tipografías de Google...) sigue su camino normal, sin
  // que este Service Worker intervenga.
  if (req.method !== 'GET' || new URL(req.url).origin !== self.location.origin) {
    return;
  }

  event.respondWith(
    fetch(req)
      .then(function (respuestaRed) {
        // Hay internet: se usa la versión de verdad, y de paso se actualiza
        // la copia guardada con esta misma respuesta para la próxima vez.
        const copia = respuestaRed.clone();
        caches.open(CACHE_NAME).then(function (cache) { cache.put(req, copia); });
        return respuestaRed;
      })
      .catch(function () {
        // Sin internet: se sirve la última copia guardada, si existe.
        return caches.match(req).then(function (respuestaGuardada) {
          return respuestaGuardada || Response.error();
        });
      })
  );
});
