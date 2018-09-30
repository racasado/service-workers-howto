# service-workers-howto

## Register/Activation

#### When to register Service Worker
Se debería esperar a que la página principal esté cargada...
```
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('sw.js');
  }
}
```

#### Wait until service Worker has been downloaded and registered
Si no esperas, el objeto `serviceWorkerRegistration` es undefined y es el objeto clave
```
navigator.serviceWorker.ready.then(function(serviceWorkerRegistration) {
  // rest of code comes here
}
```

#### Force activation... or not
Cuando una versión de serviceWorker sustituye a una antigua, tienen que esperar a que todas las páginas que utilizaban ese service worker se cierren.
Alternativamente se puede forzar el refresco con `skipWaiting`.
Si la nueva versión es **MUY DIFERENTE** a la anterior, se podría solicitar al usuario que refresque manualmante.
Se debería mandar un mensaje a todas las ventanas del usario y en esa ventana solicitar el update.
Ver [https://www.youtube.com/watch?v=CPP9ew4Co0M](https://www.youtube.com/watch?v=CPP9ew4Co0M) en el minuto 14:58.
```
self.addEventListener('install', function(event) {
  self.skipWaiting();
});
```

#### Mandar mensajes del service worker a los clientes
```
// Desde el ServiceWorker
const allClients = await clients.matchAll();
allClients.forEach(client => {
  client.postMessage({type: 'update});
});

// Desde cualquiera de las páginas abiertas
navigator.serviceWorker.addEventListener('message', (event) => {
  if(event.data.type == 'update') {
    // Show promt and trigger page refresh
  }
}
```

#### Unregister all serviceWorkers

```
navigator.serviceWorker.getRegistrations()
  .then((resgistrations) => {
    resgistrations.forEach((registration) => {
      registration.unregister();
    });
  });
}
```

#### Make service worker check its version or force update

```
versionCheck();

async function versionCheck() {
  const response = await fetch(SW_VERSION_URL);
  const version = await.response.text();
  if (version !== VERSION) {
    self.registration.update();
  }
}
```


#### Other things to consider
Set cache headers to `sw.js`
It will refresh after 24 hours anyhow
Probably browser are now ignoring cache so you always get the latest version

```
cache-control: max-age=120
```



## Execution

#### Listado ventanas controladas por un SW
```
const allClients = await clients.matchAll({
  includeUncontrolled: true
});

for (const client of allClients) {
  const url = new URL(client.url);
  console.log(url);
}
```

## Subscription

#### subscribirme a eventos push
La subscripción de un serviceWorker genera una serviceWorkerRegistration.
Se puede conseguir con `navigator.serviceWorker.getRegistration()` o con `navigator.serviceWorker.ready.then(function(serviceWorkerRegistration) {}`.
El [serviceWorkerRegistration](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration) tiene un objeto `pushManager`.
El [PushMagager](https://developer.mozilla.org/en-US/docs/Web/API/PushManager) tiene un método `subscribe`. Antes se llamaba `register`, ahora deprecado.
Mirar la opción `userVisibleOnly` pues puede condicionar el tipo de mensajes que el serviceWorker pueda recibir.  Por ahora parece obligatorio enviar un `true o el browser dará un error.
**No me queda claro si para hacer el `subscribe()` es necesario obtener primero un `Notification.permission === "granted"`**.

```
serviceWorkerRegistration.pushManager
.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(publicVapidKey)
})
.then(async function(subscription) {
  await fetch("/wp_register.php", {
    method: "POST",
    body: JSON.stringify(subscription),
    headers: {
      "content-type": "application/json"
    }
  });
};
```

#### Check if pushManager can be used

```
if (!('PushManager' in window)) {
  // Push isn't supported on this browser, disable or hide UI.
  return;
}
```

#### Borrar a un usuario de una subscription

```
navigator.serviceWorker.ready.then(function(reg) {
  reg.pushManager.getSubscription().then(function(subscription) {
    subscription.unsubscribe().then(function(successful) {
      // You've successfully unsubscribed
    }).catch(function(e) {
      // Unsubscription failed
    })
  })        
});
```

#### escuchar un push
```
self.addEventListener('push', function(event) {
  var data = {};
  if (event.data) {
    data = event.data.json();
  }
});
```

#### El estado de mi subscripción a eventos push
```
serviceWorkerRegistration.pushManager
.permissionState({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(publicVapidKey)
})
.then(state => {console.log('PushManager.permissionState()', state);});
```

## Notifications


#### Do I have permission?

```
if (Notification.permission === "granted") {
  // If it's okay let's create a notification
  var notification = new Notification("Hi there!");
}
```

#### Request permission

```
// Con promesas
Notification.requestPermission().then(function(result) {
  console.log(result);
});

// Con callbacks
Notification.requestPermission(function(permission) {
  if (permission === "granted") {
    var notification = new Notification("Hi there!");
  }
});
```

#### Desde el service worker
Obviamente esto se puede hacer si se tienen permisos de `Notificaciones`.  En caso contrario los pide.
```
navigator.serviceWorker.ready.then(function(registration) {
  registration.showNotification('Vibration Sample', {
    body: 'Buzz! Buzz!',
    icon: '../images/touch/chrome-touch-icon-192x192.png',
    vibrate: [200, 100, 200, 100, 200, 100, 200],
    tag: 'vibration-sample'
  });
});
```

#### Desde el service worker, consigue una lista de todas las notificaciones
```
var options = { tag : 'user_alerts' };

navigator.serviceWorker.ready.then(function(registration) {
  registration.getNotifications(options).then(function(notifications) {
    // do something with your notifications
  }) 
});
```





#### Make notification clickable
```
// desde un script
notification.onclick = function(event) {
  event.preventDefault(); // prevent the browser from focusing the Notification's tab
  window.open('http://www.mozilla.org', '_blank');
}

// desde el service worker
self.addEventListener('notificationclick', function(event) {
  console.log('Which action: ', event.action);
  console.log('On notification click: ', event.notification.tag);
  event.notification.close();
});
```





#### Listen to events from Notifications

```
self.addEventListener('notificationclick', function(event) {
  console.log('Which action: ', event.action);
  console.log('On notification click: ', event.notification.tag);
  event.notification.close();
});
```

#### Open a window from the notification

```
self.addEventListener('notificationclick', function(event) {
  console.log('On notification click: ', event.notification.tag);
  event.notification.close();

  // This looks to see if the current is already open and
  // focuses if it is
  event.waitUntil(clients.matchAll({
    type: "window"
  }).then(function(clientList) {
    for (var i = 0; i < clientList.length; i++) {
      var client = clientList[i];
      if (client.url == '/' && 'focus' in client)
        return client.focus();
    }
    if (clients.openWindow)
      return clients.openWindow('/');
  }));
});
```






## other things

`importScripts`
dont use global state
always check response.ok for caches.  Yo may cache a 404.
test for api
```
if (self.skipWaiting) {
  self.skipWaiting();
}
```
npm sw-test-env
sw generators
```
sw-precache
sw-toolbox
generate-service.worker
...
```




