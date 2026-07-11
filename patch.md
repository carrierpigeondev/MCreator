# Patch Note

TL;DR: Update Jetbrains Runtime with JCEF from b329.117 to *something* newer (I chose b508.16 because it is the newest as of this writeup) as it fixes incompatibility with the Fedora Linux distro.

```diff
- def jbr25_linux_64 = "jbrsdk_jcef-25.0.2-linux-x64-b329.117"
+ def jbr25_linux_64 = "jbrsdk_jcef-25.0.3-linux-x64-b508.16"
```

Applied to `/platform/setup.gradle`.

```diff
  import org.cef.browser.CefBrowser;
  import org.cef.browser.CefFrame;
  import org.cef.callback.CefCallback;
+ import org.cef.callback.CefResourceReadCallback;
+ import org.cef.callback.CefResourceSkipCallback;
  import org.cef.handler.CefResourceHandler;
+ import org.cef.misc.BoolRef;
  import org.cef.misc.IntRef;
+ import org.cef.misc.LongRef;
  import org.cef.misc.StringRef;
  import org.cef.network.CefRequest;
  import org.cef.network.CefResponse;

  @@ -86,6 +90,11 @@ public CefClassLoaderSchemeHandler(CefBrowser browser, CefFrame frame, String sc
		return true;
	}

+	@Override public boolean open(CefRequest request, BoolRef handleRequest, CefCallback callback) {
+		handleRequest.set(false);
+		return false;
+	}
+
	@Override public void getResponseHeaders(CefResponse response, IntRef responseLength, StringRef redirectUrl) {
		response.setMimeType(contentType);
		response.setStatus(200);
@@ -108,6 +117,16 @@ public CefClassLoaderSchemeHandler(CefBrowser browser, CefFrame frame, String sc
		}
	}

+	@Override public boolean read(byte[] dataOut, int bytesToRead, IntRef bytesRead, CefResourceReadCallback callback) {
+		bytesRead.set(-1);
+		return false;
+	}
+
+	@Override public boolean skip(long bytesToSkip, LongRef bytesSkipped, CefResourceSkipCallback callback) {
+		bytesSkipped.set(-2L);
+		return false;
+	}
+
	@Override public void cancel() {
		closeStream();
	}
```

Applied to `/src/main/java/net/mcreator/ui/chromium/CefClassLoaderSchemeHandler.java`.


# What is the issue?

Howdy yall. I am not at all a Java developer. I have only now began to dip my toes into Java with MCreator modding once again. It's been a good couple years since I seriously used the program, and in
that time I've been doing a little C#+Unity and Zig for game development. As a result, I did not attempt to fully implement everything as it probably should be implemented, given that the solution
seems like it could have breaking changes across MCreator as a result, doing it properly is way out of my depth.

Despite that, I managed to fix the issue I aimed to solve.

I noticed a few other users were having issues with loading MCreator on Fedora based systems, all seeming to be related to a JCEF failure to initialize, affecting procedure rendering.

Specifically, starting MCreator gives the following log, which I reproduced on my Fedora 44 x86_64 machine using the latest version of MCreator on the website (2026.2):

```log
$ MCREATOR_CEF_DEBUG=1 ./mcreator.sh
2026-07-11-01:32:55 [main/INFO] [Launcher] Starting MCreator 2026.2.27711 - 202600227711
2026-07-11-01:32:55 [main/INFO] [Launcher] Java version: 25.0.2+10-b329.117, VM: OpenJDK 64-Bit Server VM, vendor: JetBrains s.r.o.
2026-07-11-01:32:55 [main/INFO] [Launcher] Current JAVA_HOME for running instance: /home/USER/.local/share/MCreator20262/jdk
2026-07-11-01:32:55 [main/DEBUG] [Preferences Manager] Loading preferences from core
2026-07-11-01:32:55 [main/INFO] [Launcher] Installation path: /home/USER/.local/share/MCreator20262
2026-07-11-01:32:55 [main/INFO] [Launcher] User home of MCreator: /home/USER/.mcreator
...
2026-07-11-01:32:56 [Application-Loader/INFO] [Theme Loader] Using MCreator UI theme: default_dark
2026-07-11-01:32:56 [Application-Loader/DEBUG] [net.mcreator.ui.chromium.WebView] Preloading CEF WebView
2026-07-11-01:32:56 [Application-Loader/INFO] [CEF] Initializing JCEF in OSR (60 FPS) mode
2026-07-11-01:32:56 [Application-Loader/DEBUG] [CEF] JCEF arguments: [--force-device-scale-factor=1.0, --disable-extensions, --disable-default-apps, --disable-sync, --disable-speech-api, --mute-audio, --disable-gaia-services, --disable-gpu-compositing, --disable-features=TranslateUI,WebUSB,SpareRendererForSitePerProcess,WebBluetooth,WebSerial,MediaRouter,NewUsbBackend,WebHID]
2026-07-11-01:32:56 [Application-Loader/DEBUG] [CEF] Deleting orphaned CEF cache folder /tmp/mcreator-cef-cache-16711042836117812750
2026-07-11-01:33:01 [Application-Loader/ERROR] [CEF] Failed to initialize JCEF in time. Things may not work properly.
2026-07-11-01:33:06 [Application-Loader/ERROR] [net.mcreator.ui.chromium.WebView] Failed to preload WebView in time
```

The error to note is `[CEF] Failed to initialize JCEF in time. Things may not work properly.`

The five second wait between `[CEF] Deleting orphaned CEF cache folder /tmp/mcreator-cef-cache-{}` and the error was a consistent occurance.

The behavior following the error was a normal load into the MCreator UI. The issue that results from this is not a fatal error, however, when one goes to open a procedure (whether it already existed
or was newly created), the Blockly UI would not render at all (neither the side bar nor procedure blocks).

Let me preface this by saying **I have no clue what caused this to fail.**

## What is the (half-baked) solution?

First, I updated the Jetbrains Runtime with JCEF to the newest version to see if it might fix anything. It had JCEF in the name :shrug:

```diff
- def jbr25_linux_64 = "jbrsdk_jcef-25.0.2-linux-x64-b329.117"
+ def jbr25_linux_64 = "jbrsdk_jcef-25.0.3-linux-x64-b508.16"
```

Applied to `/platform/setup.gradle`.

When it redownloaded, I tried to run MCreator again directly, without recompiling, and it initialized with no errors. I really hoped that the problem was solved by just updating the Jetbrains
Runtime, however, when I tried to open a procedure:

```log
2026-07-11-01:02:17 [Thread-104/ERROR] [STDERR] java.lang.AbstractMethodErrorException in thread "Thread-107"
2026-07-11-01:02:17 [Thread-107/ERROR] [STDERR] java.lang.AbstractMethodErrorException in thread "Thread-146"
2026-07-11-01:02:17 [Thread-146/ERROR] [STDERR] java.lang.AbstractMethodErrorException in thread "Thread-150"
2026-07-11-01:02:17 [Thread-150/ERROR] [STDERR] java.lang.AbstractMethodError2026-07-11-01:02:17 [Thread-154/ERROR]
[CEF] Uncaught ReferenceError: Blockly is not defined (source: http://mcreator/blockly/blockly.html, line: 3)
2026-07-11-01:02:27 [DesktopUtils/INFO] [net.mcreator.util.DesktopUtils] Trying to use Desktop.getDesktop().browse() with https://mcreator.net/wiki/section/procedure-system
026-07-11-01:02:27 [WebView-Callback-Thread/WARN] [net.mcreator.ui.chromium.WebView] JS execution timed out after 10 seconds
2026-07-11-01:02:27 [Thread-2755/ERROR] [CEF] Uncaught ReferenceError: Blockly is not defined (source: http://mcreator/blockly/blockly.html, line: 3)
```
```java
	@Override public boolean open(CefRequest request, BoolRef handleRequest, CefCallback callback) {
		handleRequest.set(false);
		return false;

Confused, I tried to recompile and:

```
> Compilation failed; see the compiler output below.
  Note: /home/USER/Projects/MCreator/src/main/java/net/mcreator/ui/chromium/CefClassLoaderSchemeHandler.java uses or overrides a deprecated API.
  Note: Recompile with -Xlint:deprecation for details.
  /home/USER/Projects/MCreator/src/main/java/net/mcreator/ui/chromium/CefClassLoaderSchemeHandler.java:44: error: CefClassLoaderSchemeHandler is not abstract and does not override abstract method skip(long,LongRef,CefResourceSkipCallback) in CefResourceHandler
  class CefClassLoaderSchemeHandler implements CefResourceHandler {
  ^
  1 error
```

There was a missing method. I checked in IntelliJ IDEA and it also warned me about two other methods. In all it told me to implement:

```java
open(cefRequest: CefRequest, boolRef: BoolRef, cefCallback: cefCallback): boolean
read(bytes: byte[], i: int, intRef: IntRef, cefResourceReadCallback: CefResourceReadCallback): boolean
skip(l: long, longRef: LongRef, cefResourceSkipCallback: CefResourceSkipCallback): boolean
```

I started by going to the `CefResourceHandler` class which is implemented by `CefClassLoaderSchemeHandler`, and saw that the three methods had implementations inside of `CefResourceHandlerAdapter`
that IDEA told me was a part of the Jetbrains Runtime.

I ctrl-c ctrl-v'd those methods as overrides into `CefClassLoaderSchemeHandler` as such:

```java
	@Override public boolean open(CefRequest request, BoolRef handleRequest, CefCallback callback) {
		handleRequest.set(false);
		return false;
	}

    @Override public boolean read(byte[] dataOut, int bytesToRead, IntRef bytesRead, CefResourceReadCallback callback) {
		bytesRead.set(-1);
		return false;
	}

	@Override public boolean skip(long bytesToSkip, LongRef bytesSkipped, CefResourceSkipCallback callback) {
		bytesSkipped.set(-2L);
		return false;
	}
```

Compiled and ran MCreator and the Blockly UI appears (i.e., the side bar shows up and procedures actually render) :D

# Why "half-baked"?

Simply because I do not know the ramifications of this change to other parts of the program that use JCEF, such as those that can

1) fail silently

or

2) fail loudly, but I haven't triggered the path that would cause it yet.

Furthermore, I did the bare minimum to get my setup of Fedora working - I have **no clue why it didn't / doesn't work on Fedora in the first place.**

Therefore, there could be something wrong / incomplete by using the `CefResourceHandlerAdapter` implementation of the three new methods in `CefClassLoaderSchemeHandler` of which I am simply unaware.

At any rate, this fixed my issue :P
