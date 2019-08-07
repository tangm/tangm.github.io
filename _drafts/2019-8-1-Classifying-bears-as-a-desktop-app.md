---
layout: post
title: Classifying Bears as a desktop app
---

Weekly Wrangle: Teddy Bear Classification on the Desktop

I recently finished fast.ai's excellent [deep learning for coders] course and wanted to explore alternative options for packaging up a predictive model. The techniques used by the course run on python and use their own fast.ai library, which under the covers makes use of pytorch, pandas and more.

The idea was to have a desktop interface, as opposed to a webapp, for [classifying teddy, grizzly and brown bears]. I'm going to go through what I had to do to make this happen. At a low level, what's required for a fast.ai + pyInstaller + Qt5 WebView build. 

To say I ran into troubles would be an understatement, so I'll save the scenic route for another post. This will just be the express version. You can skip ahead in the repository at []

What you'll need:
* Specialised knowledge of teddy bear classification
* Python 3 on Windows (do *NOT* install it from the app store)
* Virtualenv
* Some patience

Where it comes to tech choices, the main factors were:
* Python backend, so a GUI with python bindings
* Near nil GUI knowledge, so a HTML based webview

Let's build the actual app first, then we'll get to the packaging

Here's the python app in a single file so you can see how it hangs together:

And the frontend JS

Nothing fancy, here's some ascii

```
---------------------    Qt5 WebChannel (WebSockets)  ---------------------
| Qt5 WebEngineView | ------------------------------> | Actual Python app |
---------------------                                 ---------------------
```

In general
* Python app brings up WebEngine view, displays ```index.html```
* Selecting an image in the WebEngineView invokes the ```classify``` method with the selected image
** ```classify``` uses the fast.ai library to make a prediction based on the image
* The classification result is returned and displayed

Some points to note:
* Communication between the web view and app is done via Qt5's provided QWebChannel (websocket) interface. Python object interfaces are exposed over the channel. Methods on the interface can be invoked, and their result returned via a callback. 
* It's not clear what form the method parameters take, the documentation for QWebChannel just states that it is JSON serialized. For a binary string, I found it was received as a unicode string, which needed some... massaging. If you're sending typical JSON payloads you should be just fine.
* I'll explain the pyInstaller+pytorch hackery a little later, but just go along with it for now

To get it up and running,
* Create a virtualenv (```virtualenv teddys```)
* Switch in to (```teddys\Scripts\activate```)
* Install deps (```pip install -r requirements.txt```)
* Run it! (```python main.py```)

Here's some images for you to try. This is based on Jeremy H's pretrained model, so I'd say it's a pretty good teddy bear classifier, some would even say best in the world.

Thing of beauty. Up to this point, it's been pretty straightforward. If users of this app are savvy enough to do a pip install we're home free. 

But we want more, we want to distribute it! As an executable! PyInstaller was already installed as part of the requirements list, so...

* ```pyinstaller main.spec```

This will build an executable and all its required files in a ```dist``` subfolder. And finally, drumroll please,

* ```dist\main\main```

Voila, an executable desktop app for teddy bear classification.

... 

In reality, just don't do it. The size of the folder is crazy.




Yes, I could have used Electron as described in [https://github.com/fyears/electron-python-example], but I wanted to try and get away from a separate process running. 

