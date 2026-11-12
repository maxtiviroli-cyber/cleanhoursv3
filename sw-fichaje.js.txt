// Servap · Fichaje — permite abrir la app sin conexión.
// Solo gestiona los archivos de fichaje y las fuentes; el resto de la web no se toca.
var CACHE = 'servap-fichaje-v1';
var ARCHIVOS = ['fichaje.html', 'fichaje.webmanifest', 'icon-fichaje-192.png', 'icon-fichaje-512.png'];

self.addEventListener('install', function (e) {
  e.waitUntil(caches.open(CACHE).then(function (c) { return c.addAll(ARCHIVOS); }).then(function () { return self.skipWaiting(); }));
});

self.addEventListener('activate', function (e) {
  e.waitUntil(caches.keys().then(function (ks) {
    return Promise.all(ks.filter(function (k) { return k.indexOf('servap-fichaje-') === 0 && k !== CACHE; })
      .map(function (k) { return caches.delete(k); }));
  }).then(function () { return self.clients.claim(); }));
});

function esArchivoFichaje(url) {
  if (url.origin !== self.location.origin) return false;
  var nombre = url.pathname.split('/').pop();
  return nombre === 'fichaje' || ARCHIVOS.indexOf(nombre) >= 0;
}

self.addEventListener('fetch', function (e) {
  var req = e.request;
  if (req.method !== 'GET') return;
  var url = new URL(req.url);

  // Archivos de la app: primero internet (para recibir mejoras), si no hay, la copia guardada
  if (esArchivoFichaje(url)) {
    e.respondWith(fetch(req).then(function (res) {
      if (res && res.ok) {
        var copia = res.clone();
        var clave = url.pathname.split('/').pop() === 'fichaje' ? 'fichaje.html' : req;
        caches.open(CACHE).then(function (c) { c.put(clave, copia); });
      }
      return res;
    }).catch(function () {
      return caches.match(req, { ignoreSearch: true }).then(function (r) {
        return r || caches.match('fichaje.html');
      });
    }));
    return;
  }

  // Fuentes de Google: la copia guardada primero
  if (url.hostname === 'fonts.googleapis.com' || url.hostname === 'fonts.gstatic.com') {
    e.respondWith(caches.match(req).then(function (r) {
      return r || fetch(req).then(function (res) {
        var copia = res.clone();
        caches.open(CACHE).then(function (c) { c.put(req, copia); });
        return res;
      });
    }));
  }
});
