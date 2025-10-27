// sw.js - सरल Service Worker (cache-first strategy for core assets)
const CACHE_NAME = 'nazeer-naps-v1';
const CORE_ASSETS = [
  './',
  './index.html',
  './manifest.json',
  './icons/icon-192.png',
  './icons/icon-512.png'
];

// Install - cache core assets
self.addEventListener('install', event => {
  self.skipWaiting();
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(CORE_ASSETS))
      .catch(err => console.warn('SW install cache failed:', err))
  );
});

// Activate - clean up old caches
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.map(key => {
          if (key !== CACHE_NAME) return caches.delete(key);
        })
      )
    ).then(() => self.clients.claim())
  );
});

// Fetch - cache-first strategy
self.addEventListener('fetch', event => {
  // navigation requests: serve index.html (SPA friendly)
  if (event.request.mode === 'navigate') {
    event.respondWith(
      caches.match('./index.html').then(resp => {
        return resp || fetch(event.request).then(r => {
          return caches.open(CACHE_NAME).then(cache => {
            cache.put('./index.html', r.clone());
            return r;
          });
        }).catch(() => caches.match('./index.html'));
      })
    );
    return;
  }

  // other requests: try cache first, then network, then cache fallback
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;
      return fetch(event.request).then(networkResp => {
        // put successful network responses in cache
        return caches.open(CACHE_NAME).then(cache => {
          // Avoid caching opaque requests (like cross-origin non-CORS resources)
          try {
            if (networkResp && networkResp.status === 200) {
              cache.put(event.request, networkResp.clone());
            }
          } catch (e) { /* ignore caching errors */ }
          return networkResp;
        });
      }).catch(() => {
        // fallback: try to return a cached navigation or image placeholder if needed
        return caches.match('./index.html');
      });
    })
  );
});
