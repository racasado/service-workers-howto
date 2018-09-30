# service-workers-howto related APIs

* Push API: https://developer.mozilla.org/en-US/docs/Web/API/Push_API
* Notifications API: https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API
* Channel Messaging API: https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API
* Web Workers API: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API


#### Para escuchar un mensaje

```
var eventMethod = window.addEventListener ? 'addEventListener' : 'attachEvent';
var eventer = window[eventMethod];
var messageEvent = eventMethod === 'attachEvent' ? 'onmessage' : 'message';

function handleMessage(data) {
  var parts = data.split('@@');
  if (parts[0] === 'referrer') {
    uiActions.setLocation(parts[1]);
  }
  
  if (parts[0] === 'params') {
  // Params from referrer
  }

  if (parts[0] === 'ueCookiesPolicy' && parts[1] == 'true') {
    sessionActions.acceptCookies('ueCookiesPolicy');
  }
}

// Listen to message from child window
eventer(messageEvent, function(e) {
  if (typeof e.data !== 'string' || e.data.indexOf('@@') < 0) { return; }
    handleMessage(e.data);
  }, false);
},
```
