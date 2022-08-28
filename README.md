# msg

```ts
// @ts-nocheck
function Parent(targetWindowOrIframeEl, targetOrigin, afterConnectedCallback) {
  var postMessageQueue = [];
  var connected = false;
  var handlers = {};
  var targetWindowIsIframeElement;

  function getIframeOrigin(iframe) {
    return iframe.src.match(/(.*?\/\/.*?)\//)[1];
  }

  function post(type, content) {
    var message;
    // Message object can be constructed from 'type' and 'content' arguments or it can be passed
    // as the first argument.
    if (arguments.length === 1 && typeof type === 'object' && typeof type.type === 'string') {
      message = type;
    } else {
      message = {
        type: type,
        content: content
      };
    }
    if (connected) {
      var tWindow = getTargetWindow();
      tWindow.postMessage(JSON.stringify(message), targetOrigin);
    } else {
      // else queue up the messages to send after connection complete.
      postMessageQueue.push(message);
    }
  }

  function addListener(messageName, func) {
    handlers[messageName] = func;
  }

  function removeListener(messageName) {
    delete handlers[messageName];
  }

  function removeAllListeners() {
    handlers = {};
  }

  function getTargetWindow() {
    if (targetWindowIsIframeElement) {
      var tWindow = targetWindowOrIframeEl.contentWindow;
      if (!tWindow) {
        throw "IFrame element needs to be added to DOM before communication " +
        "can be started (.contentWindow is not available)";
      }
      return tWindow;
    }
    return targetWindowOrIframeEl;
  }

  function receiveMessage(message) {
    var messageData;
    if (message.source === getTargetWindow() && (targetOrigin === '*' || message.origin === targetOrigin)) {
      messageData = message.data;
      if (typeof messageData === 'string') {
        messageData = JSON.parse(messageData);
      }
      if (handlers[messageData.type]) {
        handlers[messageData.type](messageData.content);
      } else {
        console.log("cant handle type: " + messageData.type);
      }
    }
  }

  function disconnect() {
    connected = false;
    removeAllListeners();
    window.removeEventListener('message', receiveMessage);
  }

  try {
    targetWindowIsIframeElement = targetWindowOrIframeEl.nodeName === 'IFRAME';
  } catch (e) {
    targetWindowIsIframeElement = false;
  }

  if (targetWindowIsIframeElement) {
    if (!targetOrigin || typeof targetOrigin === 'function') {
      afterConnectedCallback = targetOrigin;
      targetOrigin = getIframeOrigin(targetWindowOrIframeEl);
    }
  }

  if (targetOrigin === 'file://') {
    targetOrigin = '*';
  }

  addListener('__init__', function () {
    connected = true;

    post({
      type: '__init__',
      origin: window.location.origin
    });

    // give the user a chance to do things now that we are connected
    // note that is will happen before any queued messages
    if (afterConnectedCallback && typeof afterConnectedCallback === "function") {
      afterConnectedCallback();
    }

    // Now send any messages that have been queued up ...
    while (postMessageQueue.length > 0) {
      post(postMessageQueue.shift());
    }
  });

  window.addEventListener('message', receiveMessage, false);

  // Public API.
  return {
    post: post,
    addListener: addListener,
    removeListener: removeListener,
    removeAllListeners: removeAllListeners,
    disconnect: disconnect,
    getTargetWindow: getTargetWindow,
    targetOrigin: targetOrigin
  };
}

export default Parent;
```

```ts
// @ts-nocheck
function IFrameEndpoint() {
  var listeners = {};
  var isInitialized = false;
  var connected = false;
  var postMessageQueue = [];

  function postToParent(message) {
    window.parent.postMessage(JSON.stringify(message), '*');
  }

  function post(type, content) {
    var message;
    // Message object can be constructed from 'type' and 'content' arguments or it can be passed
    // as the first argument.
    if (arguments.length === 1 && typeof type === 'object' && typeof type.type === 'string') {
      message = type;
    } else {
      message = {
        type: type,
        content: content
      };
    }
    if (connected) {
      postToParent(message);
    } else {
      postMessageQueue.push(message);
    }
  }

  function addListener(type, fn) {
    listeners[type] = fn;
  }

  function removeListener(type) {
    delete listeners[type];
  }

  function removeAllListeners() {
    listeners = {};
  }

  function getListenerNames() {
    return Object.keys(listeners);
  }

  function messageListener(message) {
    debugger;
    // Anyone can send us a message. Only pay attention to messages from parent.
    if (message.source !== window.parent) return;
    var messageData = message.data;
    if (typeof messageData === 'string') messageData = JSON.parse(messageData);

    if (!connected && messageData.type === '__init__') {
      connected = true;
      while (postMessageQueue.length > 0) {
        post(postMessageQueue.shift());
      }
    }

    if (connected && listeners[messageData.type]) {
      listeners[messageData.type](messageData.content);
    }
  }

  function disconnect() {
    connected = false;
    removeAllListeners();
    window.removeEventListener('message', messageListener);
  }

  /**
   Initialize communication with the parent frame. This should not be called until the app's custom
   listeners are registered (via our 'addListener' public method) because, once we open the
   communication, the parent window may send any messages it may have queued. Messages for which
   we don't have handlers will be silently ignored.
   */
  function initialize() {
    if (isInitialized) {
      return;
    }
    isInitialized = true;
    if (window.parent === window) return;

    postToParent({
      type: '__init__'
    });
    window.addEventListener('message', messageListener, false);
  }

  // Public API.
  return {
    initialize: initialize,
    getListenerNames: getListenerNames,
    addListener: addListener,
    removeListener: removeListener,
    removeAllListeners: removeAllListeners,
    disconnect: disconnect,
    post: post
  };
}

var instance = null;

function getIFrameEndpoint() {
  if (!instance) {
    instance = new IFrameEndpoint();
  }
  return instance;
}

export default getIFrameEndpoint;
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Title</title>
</head>
<body>
<iframe id="c1" src="http://localhost:8000/examples/c1.html" frameborder="0"></iframe>
<!--<iframe id="c2" src="http://localhost:8001/examples/c2.html" frameborder="0"></iframe>-->
<script src="http://localhost:3000/packages/iframe-message/dist/iframe-message.global.js"></script>
<script>
  var phone1 = new window.IM.default.Parent(document.getElementById('c1'), function () {
    console.log("c1 --- connection with iframe established");
  });
  phone1.post('testMessage', 'c1 --- abc');
  phone1.addListener('response', function (content) {
    console.log("parent received response: " + content);
  });

  // var phone2 = new window.IM.default.Parent(document.getElementById('c2'), function () {
  //   console.log("c2 --- connection with iframe established");
  // });
  // phone2.post('testMessage', 'c2 --- abc');
  // phone2.addListener('response', function (content) {
  //   console.log("parent received response: " + content);
  // });
</script>
</body>
</html>
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>c1</title>
</head>
<body>
<script src="http://localhost:8000/packages/iframe-message/dist/iframe-message.global.js"></script>
<script>
  var phone = window.IM.default.Child();
  phone.addListener('testMessage', function (content) {
    console.log("c1 --- iframe received message: " + content);
    phone.post('response', 'got it');
  });
  // Initialize connection after all message listeners are added!
  phone.initialize();
</script>
</body>
</html>

```

