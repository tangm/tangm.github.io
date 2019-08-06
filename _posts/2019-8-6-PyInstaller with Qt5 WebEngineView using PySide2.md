---
layout: post
title: PyInstaller with Qt5 WebEngineView using PySide2
---

As I'm writing this (Aug 2019), there are a number of teething issues and rotating knives if you try and package up a cross-platform (Mac, Linux, Windows) app with a Qt `WebEngineView` using the official Python Qt bindings from PySide2 and PyInstaller. I'll go through some of the issues I encountered to hopefully save you some grey hairs. 

Note that PyInstaller is not a cross-compiler, so running on separate platforms is still required. We'll get it working for windows, before moving on to other OSes.

I'm using
* PySide2 5.13.0
* Python 3.7.4
* Virtualenv 16.7.2
* PyInstaller develop @ [b5826b4a82ecc0f021ab584d0ce68a82e64ffca5](https://github.com/pyinstaller/pyinstaller/tree/b5826b4a82ecc0f021ab584d0ce68a82e64ffca5)
* Windows 10 for the most part, but Mac and Linux will come in later in the post

You can follow along with a simple app at [github](https://github.com/tangm/pyinstaller-pyside2webview-sample).

At a high level, the WebEngineView communicates with the python backend via Qt's QWebChannel (Websockets under the covers). On the python side, objects are made available via `QWebChannel#registerObject`. 

```python
import os
from pathlib import Path
from PySide2.QtWidgets import QApplication
from PySide2.QtWebEngineWidgets import QWebEngineView, QWebEnginePage
from PySide2.QtWebChannel import QWebChannel
from PySide2.QtCore import QUrl, Slot, QObject, QUrl

data_dir = Path(os.path.abspath(os.path.dirname(__file__))) / 'data'

class Handler(QObject):
    def __init__(self, *args, **kwargs):
        super(Handler, self).__init__(*args, **kwargs)

    @Slot(str, result=str)
    def sayHello(self, name):        
        return f"Hello from the other side, {name}"

class WebEnginePage(QWebEnginePage):
    def __init__(self, *args, **kwargs):
        super(WebEnginePage, self).__init__(*args, **kwargs)

    def javaScriptConsoleMessage(self, level, message, lineNumber, sourceId):
        print("WebEnginePage Console: ", level, message, lineNumber, sourceId)

if __name__ == "__main__":
    
    # Set up the main application
    app = QApplication([])
    app.setApplicationDisplayName("Greetings from the other side")

    # Use a webengine view
    view = QWebEngineView()
    view.resize(500,200)

    # Set up backend communication via web channel
    handler = Handler()
    channel = QWebChannel()
    # Make the handler object available, naming it "backend"
    channel.registerObject("backend", handler)

    # Use a custom page that prints console messages to make debugging easier
    page = WebEnginePage()
    page.setWebChannel(channel)
    view.setPage(page)

    # Finally, load our file in the view
    url = QUrl.fromLocalFile(f"{data_dir}/index.html")
    view.load(url)
    view.show()

    app.exec_()
```

On the HTML view, the registered objects are made available from the JS QWebChannel (Note the `qwebchannel.js` script being loaded)

```HTML
<html>
<head>
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width,initial-scale=1.0">
	<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
	<meta content="IE=edge" http-equiv="X-UA-Compatible">
	<script src="qrc:///qtwebchannel/qwebchannel.js"></script>
	<title>Hello</title>
</head>
<body style="text-align: center; font-size: 1.5rem">
	<h2>
		My name is: <input id="nameInput">
	</h2>
<input type="file">
	<div id="result">
	</div>
	<button id="sayHello" style="font-size: 1.5rem">
		Say Hello
	</button>
	<script>
		document.addEventListener('contextmenu', event => event.preventDefault());
		document.addEventListener('DOMContentLoaded', function () {
			
			const nameInput = document.getElementById('nameInput');
			const result = document.getElementById('result'); 
			const sayHello = document.getElementById('sayHello');
			
			// Obtain the exposed python object interface
			const getBackend = new Promise((resolve, reject) => {
				new QWebChannel(qt.webChannelTransport, 
					(channel) => resolve(channel.objects.backend));
			})

			// Show a preview when an image file is selected
			sayHello.addEventListener('click', function(){
				result.textContent = '';
				getBackend.then((backend) => {
					backend.sayHello(nameInput.value, (prediction) => {
						result.textContent = prediction;
					});							
				})
			});
		});
	</script>
</body>
</html>
```


To get this up and running
* Create a virtualenv and activate it
* Install deps using pip from requirements.txt
* `python main.py`

If everything goes well, you should have a `QtWebEngineView` that will give you messages from the other side.

So far so good, now we get to the real hair pulling part. Making this work with PyInstaller. At a high level with lots of hand-waving, PyInstaller 
* Analyses your source files for imports
* Bundles the imports into the executable, and supporting files a single distribution folder (you can opt for a single file as well)
* Uses a bootloader to bootstrap your application, manipulating the module search paths to point to the packaged modules
* Along the way, it provides *hooks* for modifying the packaging behaviour during the bundling as well as the runtime behaviour during the bootloader bootstrapping

Details can be found at https://pythonhosted.org/PyInstaller/operating-mode.html and https://pythonhosted.org/PyInstaller/advanced-topics.html

The repository already has all the fixes, but if you want to follow along you can start from the tag 'before-pyinstaller-fixes'. It will contain a **pristine** `main.spec` file generated by `pyinstaller main.py`:

Let's kick it off trying to build the executable
* `pyinstaller main.spec`
* run the `main` executable in the `dist/main` folder

### Could not import module 'PySide2.QtPrintSupport'

It will give you 
```
Traceback (most recent call last):
  File "main.py", line 5, in <module>
    from PySide2.QtWebEngineWidgets import QWebEngineView, QWebEnginePage
ImportError: could not import module 'PySide2.QtPrintSupport'
[6184] Failed to execute script main
```

So the archive couldn't find the dependent module `PySide2.QtPrintSupport`, likely because it wasn't imported using a standard python mechanism. To let PyInstaller know about this module, we need to add a *hidden import*, in the `main.spec`:
```
              datas=[],
-             hiddenimports=[],
+             hiddenimports=['PySide2.QtPrintSupport'],
              hookspath=[],
              runtime_hooks=[],
              excludes=[],
```

Carrying on our merry way, `pyinstaller main.spec` and running the generated executable again

### Could not find QtWebEngineProcess.exe on Windows

It will give you 
```
Could not find QtWebEngineProcess.exe
```

`QtWebEngineProcess.exe` is the Qt support file needed to actually launch the WebEngineView. Whereever it is being copied to though, Qt can't find it. I'll save you the googling, but [PYSIDE-642](https://bugreports.qt.io/browse/PYSIDE-642?focusedCommentId=461015&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-461015) indicates what the expected layout is meant to be:

```
ls -la dist\app\PySide2
total 16661
drwxr-xr-x 1 fran 197610       0 May 20 13:09 ./
drwxr-xr-x 1 fran 197610       0 May 20 13:08 ../
-rw-r--r-- 1 fran 197610      21 May 18 09:53 qt.conf
-rwxr-xr-x 1 fran 197610 3468288 May 18 09:55 QtCore.pyd*
-rwxr-xr-x 1 fran 197610 4029440 May 18 09:55 QtGui.pyd*
-rwxr-xr-x 1 fran 197610 1010176 May 18 09:55 QtNetwork.pyd*
-rwxr-xr-x 1 fran 197610  336384 May 18 09:53 QtPrintSupport.pyd*
-rwxr-xr-x 1 fran 197610  472576 May 18 09:55 QtSql.pyd*
-rwxr-xr-x 1 fran 197610   58368 May 18 09:55 QtWebChannel.pyd*
-rwxr-xr-x 1 fran 197610  109056 May 18 09:55 QtWebEngineCore.pyd*
-rwxr-xr-x 1 fran 197610   19456 May 18 09:53 QtWebEngineProcess.exe*
-rwxr-xr-x 1 fran 197610  308224 May 18 09:55 QtWebEngineWidgets.pyd*
-rwxr-xr-x 1 fran 197610 7089152 May 18 09:55 QtWidgets.pyd*
drwxr-xr-x 1 fran 197610       0 May 20 13:09 resources/
drwxr-xr-x 1 fran 197610       0 May 20 13:09 translations/
```

but instead we have:

```
ls -la dist\main\PySide2
total 13924
drwxrwxrwx 1 mtan mtan    4096 Aug  6 13:51 .
drwxrwxrwx 1 mtan mtan    4096 Aug  6 13:51 ..
drwxrwxrwx 1 mtan mtan    4096 Aug  6 13:51 PySide2
-rwxrwxrwx 1 mtan mtan 3320320 Jul 30 15:55 QtCore.pyd
-rwxrwxrwx 1 mtan mtan 3750400 Jul 30 15:55 QtGui.pyd
-rwxrwxrwx 1 mtan mtan  937472 Jul 30 15:55 QtNetwork.pyd
-rwxrwxrwx 1 mtan mtan  261120 Jul 30 15:55 QtPrintSupport.pyd
-rwxrwxrwx 1 mtan mtan   54272 Jul 30 15:55 QtWebChannel.pyd
-rwxrwxrwx 1 mtan mtan  106496 Jul 30 15:55 QtWebEngineCore.pyd
-rwxrwxrwx 1 mtan mtan  294400 Jul 30 15:55 QtWebEngineWidgets.pyd
-rwxrwxrwx 1 mtan mtan 5523968 Jul 30 15:55 QtWidgets.pyd
drwxrwxrwx 1 mtan mtan    4096 Aug  6 13:51 plugins
drwxrwxrwx 1 mtan mtan    4096 Aug  6 13:51 translations
```

There's no `qt.conf`, nor any `QtWebEngineProcess`. If you google for just `Could not find QtWebEngineProcess.exe`, chances are you'll find posts (some even resolved) for `PyQt5`, which `!== PySide2`. As far as I can tell, `PyQt5` was around before the official Qt-blessed bindings, so much supporting infrastructure is geared towards `PyQt5`. Including some bad fixes...

How do these files even end up there? Let's have a look at the pyinstaller included hook - https://github.com/pyinstaller/pyinstaller/blob/b5826b4a82ecc0f021ab584d0ce68a82e64ffca5/PyInstaller/hooks/hook-PySide2.QtWebEngineWidgets.py

```
...
# Find the additional files necessary for QtWebEngine.
datas = (collect_data_files('PySide2', True, os.path.join('Qt', 'resources')) +
         collect_data_files('PySide2', True, os.path.join('Qt', 'translations')) +
         [x for x in collect_data_files('PySide2', False, os.path.join('Qt', 'bin'))
          if x[0].endswith('QtWebEngineProcess.exe')])

```

In a hook, ```datas``` is an array of 2-tuples indicating supporting resource files to be copied, of the form ```(src, dest)```, e.g. ```[('somefile.exe', '.'), ('someotherfile.data', 'blah/someotherfile.data')]``` would copy the file `somefile.exe` to the root of the PyInstaller distribution folder and `someotherfile.data` to the `blah` dir in the distribution folder. 

```collect_data_files('PySide2', True, os.path.join('Qt', 'resources'))``` returns an array of 2-tuples to the effect of "copy all files in the `Qt/resources` subfolder of the installed `PySide2` module (`virtualenv site package dir>\PySide2\Qt\resources`". 

Except if we were to look at the appropriate folder:

```
ls -la venv/Lib/site-packages/PySide2/Qt/resources
ls: cannot access 'venv/Lib/site-packages/PySide2/Qt/resources': No such file or directory
```

there's no such dir. The hook is kind of broken, as you can see from the tests in the pyinstaller package https://github.com/pyinstaller/pyinstaller/blob/b5826b4a82ecc0f021ab584d0ce68a82e64ffca5/tests/functional/test_libraries.py#L408

```
@xfail(True, reason="Hook is old and needs updating.")
@importorskip('PySide2')
def test_PySide2_QWebEngine(pyi_builder, data_dir):
    pyi_builder.test_source(get_QWebEngine_html('PySide2', data_dir),
                            **USE_WINDOWED_KWARG)
```

if we look further in the file though, we find that `PyQt5` seems to be working fine, and the [`PyQt5` web engine hook]( https://github.com/pyinstaller/pyinstaller/blob/b5826b4a82ecc0f021ab584d0ce68a82e64ffca5/PyInstaller/hooks/hook-PyQt5.QtWebEngineWidgets.py) seems to have lots more stuff and is alot newer than the [`PySide2` hook](https://github.com/pyinstaller/pyinstaller/blob/b5826b4a82ecc0f021ab584d0ce68a82e64ffca5/PyInstaller/hooks/hook-PySide2.QtWebEngineWidgets.py) .

Long story short, we need to incoroporate the updated changes made in `PyQt5` to accommodate newer versions into the `PySide2` hook. You can find the cobbled together version at 

We also need to tell PyInstaller about our user-supplied hook dir. In `main.spec`

```
@@ -8,7 +8,7 @@ a = Analysis(['main.py'],
              binaries=[],
              datas=[],
              hiddenimports=['PySide2.QtPrintSupport'],
-             hookspath=[],
+             hookspath=['hooks'],
              runtime_hooks=[],
              excludes=[],
              win_no_prefer_redirects=False,
````

Carrying on our merry way, `pyinstaller main.spec` and running the generated executable again

Success! At least for some of you. If you're running the generated executable from the root of the repository, something like `dist\main\main`, it'll work great. If you first change to `dist\main` before running it, you'll get a sad face in the webengineview saying it couldn't find the file.

### Copying our actual resource files

If we take a look at the `dist\main` dir, we'll find that we haven't actually copied our `data\index.html` in. Let's add it in `main.spec`, with the `datas` keyword (remember the `datas` from the `PySide2` hook above?).

```
 a = Analysis(['main.py'],
              pathex=['C:\\Users\\mtan\\projects\\pyinstaller-pyside2webview-sample'],
              binaries=[],
-             datas=[],
+             datas=[('data', 'data')],
              hiddenimports=['PySide2.QtPrintSupport'],
              hookspath=['hooks'],
              runtime_hooks=[],
```              

Now we have to go cross platform. Let's start with Linux first. If you're on a recent version of Windows 10, you already have access to a Linux environment with a combination of something like [VcXSrv](https://sourceforge.net/projects/vcxsrv/) and [Windows Subsystem for Linux (WSL)](https://lifehacker.com/how-to-get-started-with-the-windows-subsystem-for-linux-1828952698) . If you're going down the WSL route:
* Remember to set the `DISPLAY` env var, if you've set `VcXSrv` to display number 0 it would be something like `export DISPLAY=:0`
* You can access your C drive at `/mnt/c` a la cygwin.

Run through the gauntlet of creating a new virtual env, installing the requirements, then running `pyinstaller main.spec` and finally the generated executable in `dist/main/main`.

... and lo and behold, success! 

Will it go so smoohtly on MacOS? Three's the charm?

Run through the gauntlet of creating a new virtual env again, installing the requirements, but I'll spare you the suspense:

```
Could not find QtWebEngineProcess on Mac
```

From our favourite [PYSIDE-642](https://bugreports.qt.io/browse/PYSIDE-642?focusedCommentId=452047&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-452047) which gave us the previous recommended layout for windows, it turns out that an outdated PyInstaller *runtime* hook is to blame. 

```
# See ``pyi_rth_qt5.py`: use a "standard" PyQt5 layout.
if sys.platform == 'darwin':
    os.environ['QTWEBENGINEPROCESS_PATH'] = os.path.normpath(os.path.join(
        sys._MEIPASS, '..', 'MacOS', 'PySide2', 'Qt', 'lib',
        'QtWebEngineCore.framework', 'Helpers', 'QtWebEngineProcess.app',
        'Contents', 'MacOS', 'QtWebEngineProcess'
    ))
```

It's alot harder to overwrite a runtime hook, because PyInstaller provided runtime hooks are executed _after_ user provided runtime hooks according to [PyInstaller documentation](https://pythonhosted.org/PyInstaller/when-things-go-wrong.html?highlight=runtime#changing-runtime-behavior), so we'll have to overwrite it ourselves:

```
# Some hackery required for pyInstaller
if sys.platform == 'darwin':
    os.environ['QTWEBENGINEPROCESS_PATH'] = os.path.normpath(os.path.join(
        sys._MEIPASS, 'PySide2', 'lib',
        'QtWebEngineCore.framework', 'Helpers', 'QtWebEngineProcess.app',
        'Contents', 'MacOS', 'QtWebEngineProcess'
    ))

```
It would be nice if runtime hooks could be overridden, but there is an [open issue](https://github.com/pyinstaller/pyinstaller/issues/3390) for it.

And after running `pyinstaller main.spec` , the generated mac executable should run quite nicely.

### Conclusion

A mini rant first. Most of these issues stem from outdated hooks provided by the `PyInstaller` package. Don't get me wrong, I have immense respect for the folks who maintain `PyInstaller`, it's an amazing library considering the number of ways you can glue python apps together! The reality is that its practically impossible that such a widely used, open source, infrastructure geared library whose main use case is integration with the universe of other 3rd party libraries will be able to keep up with changes across the board. 

As long as there are mechanisms in place to extend `PyInstaller` (like [overriding runtime hooks...](https://github.com/pyinstaller/pyinstaller/issues/3390)), the *onus should really be on the third parties to provide hooks*, not `PyInstaller`. To their credit, `PySide2` has a [page for `PyInstaller`](https://doc.qt.io/qtforpython/deployment-pyinstaller.html), but the existence of issues like [PYSIDE-642](https://bugreports.qt.io/browse/PYSIDE-642) means there's probably room to do more. 

As a disclaimer, I've not used python or Qt for that matter in anger, so any clarification is welcome!

For bonus points, here are somethings you can do to make your ~~web~~ desktop app more spiffy.
* Add an icon (used for the executable and taskbar)
* Use `hdiutil` on a mac to create a dmg 

### Resources
PyInstaller - https://pythonhosted.org/PyInstaller/
PySide2 - https://wiki.qt.io/Qt_for_Python

