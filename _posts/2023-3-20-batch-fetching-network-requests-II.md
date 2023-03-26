---
title:  "JS: Batch Fetching Network Requests for Images - Part II"
date:   2023-3-20 10:00:00 +0800
categories: javascript
tags:
  - Service Worker
  - JavaScript
  - Python
  - PHP
---

This article is the second part of the post. I tested the batch fetching idea in Synology's Photo Station web interface. Check out the [first part](/javascript/2023/03/20/batch-fetching-network-requests-I.html) for some background information. In part I, I also provided a performance comparison of the target app with and without batch fetching.

<!-- omit in toc -->
## Table of Contents

- [How to achieve batch fetching on img tags](#how-to-achieve-batch-fetching-on-img-tags)
- [Service worker to the rescue](#service-worker-to-the-rescue)
- [Intercept network requests](#intercept-network-requests)
- [Handle the batch fetch message from service worker](#handle-the-batch-fetch-message-from-service-worker)
- [Render the thumbnail images in DOM](#render-the-thumbnail-images-in-dom)
- [The backend](#the-backend)
- [Ending notes](#ending-notes)

### How to achieve batch fetching on img tags

Normally, when the DOM contains an `img` tag, the browser will schedule a fetch request based on the `src` attribute of that tag. However, it's the browser's job to make those fetch requests and populate the tags after the requests are resolved. The user or programmer has no control on the ordering of the requests once the browser starts to queue up those requests on your behalf. Each request's response timing is also subject to various conditions, the network traffic, the server, and the asset in request.

As a result, to batch fetching images, the `img` tags have to be dynamically built and rendered. I think there are multiple ways to achieve the desired effect. For example, initiate batch fetch request in the first place when the page is loaded and create the `img` tags only after the response comes in. However, this method is not really an option in this case. I do not want to change JavaScript code written by the Synology programmers and this method will surely mess up with a lot of the original code. If I had gone this route, I probably would've needed to rewrite everything anyway since I do not have access to the source code. The JS code is only shipped in the minified version (obviously).

Then, what is the alternative? Service Workers of course!

### Service worker to the rescue

The service worker API is one of the many Web APIs available to programmer writing web applications. It can execute JavaScript code in a different thread than the one used to run JS code referenced in the DOM. There are many use cases for service worker, such as push notifications, network requests caching, background fetching, network requests intercepting, and etc. The documentation can be found in the [MDN web docs](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API). This blog will discuss network requests intercepting in particular to achieve batch fetching.

Firstly, service worker has to be installed explicitly before it can starts working. Normally, you would have code such as

```javascript
// the following code can be inserted in the script tag of your DOM
// or contained in a separate .js file that is linked by the DOM

window.addEventListener('load', () => {
  // Is service worker available?
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker
      .register('sw.js', { scope: './'})
      .then(() => {
        console.log('Service worker registered!');
      }).catch((error) => {
        console.warn('Error registering service worker:');
        console.warn(error);
      });
  }
  // ...other stuff
}
```

`sw.js` is a file containing JavaScript code that you want the service worker to execute and the optional scope can bound the service worker to a specific origin.

### Intercept network requests

To intercept network requests, add an event listener for the fetch event in the `sw.js` file. If the service worker is installed successfully, the event listener's callback function will be invoked every time the browser makes a fetch request. Since all fetch requests will be intercepted, the code should be set up to filter only the targeted fetches.

```javascript
// sw.js

self.addEventListener("fetch", (event) => {
  const request = event.request;
  // intercepting fetches for webapi/thumb.php
  if (request.method !== "GET" 
      || request.url.match(/webapi\/thumb.php/) == null)
    return;

  // ...record individual fetch and trigger batch fetch

  // instruct the sw to fetch a placeholder image instead of 
  // a real thumbnail
  event.respondWith(
    return fetch("some/path/to/placeholder.jpg");
  );
});
```

Here, the service worker is told to handle the fetches with request url containing `webapi/thumb.php`. Each request for the thumbnail will be instructed to fetch some placeholder image can be responded really fast. The call to `event.respondWith` is used to achieved that. Although each thumb request will be responded right away, we need to keep track of what has been requested along the way so that we could perform a batch fetch in some future time.

To record the thumb requests coming in, some data structure will be needed. Here, I declared a global variable `kTHUMB_SET` of type `Set` to store the unique id of the thumbnails being requested. Every time a fetch request comes in, the code will insert an entry in `kTHUMB_SET`. Then, if the set accumulates enough fetch requests, service worker will trigger a batch fetch.

```javascript
// sw.js

// global vars
const kBATCH_SIZE = 30;
const kTHUMB_SET = new Set();

self.addEventListener("fetch", (event) => {
  // ...previous code

  // record individual fetch
  // Add request to the request set
  const url_params = new URLSearchParams(request.url);
  kTHUMB_SET.add(url_params.get('id'));

  // Trigger a batch fetch
  if (kTHUMB_SET.size >= kBATCH_SIZE) {
    const thumb_reqs = new Set(kTHUMB_SET); 
    kTHUMB_SET.clear();

    event.waitUntil(
      (async() => {
        const client = await clients.get(event.clientID);
        client.postMessage({ thumb_ids: thumb_reqs });
      })()
    );
  }

  // event.respondWith(...)
}
```

When `kTHUMB_SET`'s size reaches 30 requests, service worker will send a message to the client so that the client can execute the batch fetch request. You can treat the client as the thread executing JS code linked in the DOM. Different from the service worker, client has access to the DOM. Since the service worker records all thumbnail fetch requests, it is responsible for letting the client know what thumbnails should be batch fetched when delivering the message. Here, the code will invoke `postMessage` to pass the unique thumbnail ids.

### Handle the batch fetch message from service worker

The client should install an event listener for the `message` event in the `navigator.serviceWorker` object in order to process the message sent by service worker. The following is the "normal" context where JS is run to execute code linked in the DOM. In this case, the client will first retrieve the unique thumbnail ids in the message, perform a fetch request with these ids and update the corresponding `img` tags once the request is responded.

```javascript
window.addEventListener('load', () => {
  if ("serviceWorker" in navigator)
    // ...install service worker

    navigator.serviceWorker.addEventListener("message", (event) => {
      batch_fetch_thumbs(event.data.thumb_ids);
    });
}

// call a batch fetch for thumbnails 
function batch_fetch_thumbs(thumb_ids) {
  let form_data = new FormData();
  form_data.append("thumb_ids", thumb_ids);
  fetch("webapi/batch_thumb.php", {
    method: "post",
    body: form_data
  })
  .then(response => response.json())
  .then(data => {
    update_thumb_src(data);
  })
  .catch(e => console.warn(e));
}
```

### Render the thumbnail images in DOM

Since the client is expected to receive response containing multiple thumbnails from each batch fetch request, the response data is actually another data structure containing such information. I've created a customized backend API in PHP so that the batch fetch is responded with a JSON response. The JSON response contains key/value pairs where the key represents unique thumbnail id and the value represents base64 encoded thumbnail image file. Normally when an `img` tag is created in DOM, the `src` attribute specifies the path to the image file (the path could be a network url or a path in the server's filesystem). Here, since the image is base64 encoded data, the `src` should be written as `data:image/jpg;base64,+encoded_image_data`.

```javascript
function update_thumb_src(img_data) {
  Object.keys(img_data).forEach(thumb_id => {
    const target_tag = kIMG_TAGS.get(thumb_id);
    tag.setAttribute("src", 
      "data:image/jpg;base64," + img_data[thumb_id]);
  }
}
```

### The backend

The bulk of the solutions rely on two new JavaScript files I wrote and I didn't have to alter any of the JS code written by Synology. However, to hook up everything, I had to write some code in the backend as well. For example, I mentioned earlier I had to create a customized API in PHP that handles the batch request. I called it `batch_fetch.php`. That part is a bit of reverse engineering the original thumb API plus creating a custom JSON response that encodes thumbnail image files into base64 text. 

Also, I used python to encode images into base64 text is at the bottom. This should be fairly straightforward and there is no requirement for other 3rd party libraries. The code is invoked in a child process by calling `shell_exec` in my `batch_fetch.php`.

```python
import argparse
import base64
import json
import os.path
import sys

def main():
  parser.add_argument("-f", "--files", action="extend", nargs="+",
                      type=str, help="Files to convert", required=True)
  args = parser.parse_args()
  result = {}

  for path in args.files:
    filename = path.split("/")[-1]
  
    if (not os.path.isfile(path)):
      # report error
    else:
      with open(path, "rb") as file:
        base64_bytes = base64.b64encode(file.read())
        base64_str = base64_bytes.decode("utf-8")
        result[filename] = base64_str

  output = json.dumps(result)
  sys.stdout.write(output)

if __name__ == "__main__":
  try:
    main()
  except Exception as e:
    sys.stderr.write(f"{__file__} error: {e.__str__()}")
```

### Ending notes

That is! The process of working out this solution is quite interesting since any code I wrote cannot be tested locally and had to be deployed in the Synology NAS directly. This is expected cause I couldn't recreate a Photo Station environment in my local PC. Plus, I'm not affiliated with Synology so it took me some time to figure out what goes where and how. However, I felt the result after implementing the solution really satisfying and I probably could extend the life of my NAS for some more time.

There is some restriction in writing and deploying code in the Synology's NAS. The linux OS is not installed with any package manager (no apt, apt-get) but fortunately, it's installed with python 3.8 and vim. However, I still had to install a lightweight package manager `ipkg` to install `git` so that I could revert back to the default environment. 

The code I provided in the blog is not the complete solution and there are many tweaks to account for different cases (such as handling the residual number of thumbnail requests not adding up to a batch size). I'm also testing some of the remaining corner cases. If you're interested in learning more about this solution, feel free to contact me. Cheers!
