# Part 98: Progressive Web App (PWA) และ Offline Support

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Service Worker สำหรับ offline support
- Web App Manifest
- Background sync
- Push notifications จาก Phoenix

---

## 1. Web App Manifest

```json
// priv/static/manifest.json
{
  "name": "My Phoenix App",
  "short_name": "MyApp",
  "description": "A Phoenix-powered Progressive Web App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "orientation": "any",
  "icons": [
    {
      "src": "/images/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/images/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "shortcuts": [
    {
      "name": "New Post",
      "url": "/posts/new",
      "icons": [{"src": "/images/new-post.png", "sizes": "96x96"}]
    }
  ]
}
```

```heex
<!-- In root.html.heex -->
<head>
  <link rel="manifest" href="/manifest.json" />
  <meta name="theme-color" content="#6366f1" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="default" />
  <link rel="apple-touch-icon" href="/images/icon-192.png" />
</head>
```

---

## 2. Service Worker

```javascript
// priv/static/sw.js
const CACHE_NAME = 'my-app-v1'
const STATIC_ASSETS = [
  '/',
  '/assets/app.css',
  '/assets/app.js',
  '/offline.html'
]

// Install: cache static assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => self.skipWaiting())
  )
})

// Activate: clean old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(key => key !== CACHE_NAME)
          .map(key => caches.delete(key))
      )
    ).then(() => self.clients.claim())
  )
})

// Fetch strategy: Network first with cache fallback
self.addEventListener('fetch', (event) => {
  const { request } = event

  // API calls: network only
  if (request.url.includes('/api/')) {
    event.respondWith(fetch(request))
    return
  }

  // Static assets: cache first
  if (request.destination === 'image' ||
      request.destination === 'style' ||
      request.destination === 'script') {
    event.respondWith(
      caches.match(request).then(cached =>
        cached || fetch(request).then(response => {
          const clone = response.clone()
          caches.open(CACHE_NAME).then(cache => cache.put(request, clone))
          return response
        })
      )
    )
    return
  }

  // HTML pages: network first, fallback to cache or offline page
  event.respondWith(
    fetch(request)
      .then(response => {
        const clone = response.clone()
        caches.open(CACHE_NAME).then(cache => cache.put(request, clone))
        return response
      })
      .catch(() =>
        caches.match(request)
          .then(cached => cached || caches.match('/offline.html'))
      )
  )
})

// Background sync
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-pending-actions') {
    event.waitUntil(syncPendingActions())
  }
})

async function syncPendingActions() {
  const db = await openDB()
  const pending = await db.getAll('pendingActions')

  for (const action of pending) {
    try {
      await fetch(action.url, {
        method: action.method,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(action.data)
      })
      await db.delete('pendingActions', action.id)
    } catch (err) {
      console.error('Sync failed:', err)
    }
  }
}
```

---

## 3. Service Worker Registration

```javascript
// assets/js/pwa.js

export function registerServiceWorker() {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', async () => {
      try {
        const registration = await navigator.serviceWorker.register('/sw.js', {
          scope: '/'
        })
        console.log('SW registered:', registration.scope)

        // Check for updates
        registration.addEventListener('updatefound', () => {
          const newWorker = registration.installing
          newWorker.addEventListener('statechange', () => {
            if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
              showUpdatePrompt()
            }
          })
        })
      } catch (err) {
        console.error('SW registration failed:', err)
      }
    })
  }
}

function showUpdatePrompt() {
  // Show "New version available, reload?" prompt
  const banner = document.createElement('div')
  banner.className = 'update-banner fixed bottom-4 right-4 bg-indigo-600 text-white px-4 py-2 rounded'
  banner.innerHTML = `
    <span>New version available!</span>
    <button onclick="window.location.reload()" class="ml-2 underline">Reload</button>
  `
  document.body.appendChild(banner)
}
```

---

## 4. Push Notifications ด้วย Phoenix

```elixir
# mix.exs: {:web_push_encryption, "~> 0.3"}

# Generate VAPID keys once:
# {:ok, {public, private}} = :crypto.generate_key(:ecdh, :prime256v1)

defmodule MyApp.PushNotifications do
  alias MyApp.{Repo, PushSubscription}

  def subscribe(user_id, subscription_params) do
    %PushSubscription{}
    |> PushSubscription.changeset(%{
      user_id: user_id,
      endpoint: subscription_params["endpoint"],
      p256dh: subscription_params["keys"]["p256dh"],
      auth: subscription_params["keys"]["auth"]
    })
    |> Repo.insert(on_conflict: :nothing)
  end

  def send_to_user(user_id, payload) do
    subscriptions = Repo.all(
      from ps in PushSubscription,
      where: ps.user_id == ^user_id
    )

    Enum.each(subscriptions, fn sub ->
      send_push(sub, payload)
    end)
  end

  defp send_push(subscription, payload) do
    message = %{
      endpoint: subscription.endpoint,
      keys: %{
        p256dh: subscription.p256dh,
        auth: subscription.auth
      }
    }

    case WebPushEncryption.send_web_push(
      Jason.encode!(payload),
      message,
      Application.get_env(:my_app, :vapid_private_key),
      "mailto:admin@myapp.com"
    ) do
      {:ok, _} -> :ok
      {:error, %{status_code: 410}} ->
        # Subscription expired, remove it
        Repo.delete(subscription)
      {:error, reason} ->
        require Logger
        Logger.warning("Push failed: #{inspect(reason)}")
    end
  end
end

# Controller
defmodule MyAppWeb.PushSubscriptionController do
  use MyAppWeb, :controller

  def create(conn, %{"subscription" => params}) do
    user_id = conn.assigns.current_user.id

    case MyApp.PushNotifications.subscribe(user_id, params) do
      {:ok, _} -> json(conn, %{ok: true})
      {:error, _} -> conn |> put_status(422) |> json(%{error: "Failed"})
    end
  end

  def vapid_public_key(conn, _) do
    json(conn, %{key: Application.get_env(:my_app, :vapid_public_key)})
  end
end
```

---

## 5. Offline-capable LiveView

```javascript
// LiveView JS hook for offline detection
export const OfflineDetector = {
  mounted() {
    this.updateStatus()

    window.addEventListener('online', () => this.updateStatus())
    window.addEventListener('offline', () => this.updateStatus())
  },

  updateStatus() {
    const online = navigator.onLine
    this.el.dataset.online = online

    if (!online) {
      this.el.classList.add('offline-mode')
      this.pushEvent('offline', {})
    } else {
      this.el.classList.remove('offline-mode')
      this.pushEvent('online', {})
      // Trigger background sync
      if ('serviceWorker' in navigator && 'SyncManager' in window) {
        navigator.serviceWorker.ready
          .then(reg => reg.sync.register('sync-pending-actions'))
      }
    }
  }
}
```

---

## สรุป

```
PWA Components:
├── Manifest: installability, icons, shortcuts
├── Service Worker: offline caching, background sync
├── HTTPS: required for SW and push
└── Responsive: works on mobile

Cache Strategies:
├── Cache first: static assets (images, fonts)
├── Network first: HTML pages
├── Stale while revalidate: semi-dynamic content
└── Network only: API calls, real-time data

Push Notifications:
├── VAPID keys: server identity
├── web-push library: encrypt and send
├── SW push event: display notification
└── Subscription expiry: handle 410 Gone

Service Worker lifecycle:
├── install: cache assets
├── activate: clean old caches
├── fetch: intercept requests
└── sync: background data sync
```

---

*ก่อนหน้า: [Part 97](part_97.md) | ต่อไป: [Part 99 - Machine Learning Production](part_99.md)*
