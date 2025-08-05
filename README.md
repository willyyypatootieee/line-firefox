# Unofficial LINE Port for Firefox

## How to Use

This is an unofficial port of the LINE Chrome extension for Firefox. To use this extension:

1. **Download or clone this repository** to your local machine
2. **Apply the patches** described in the "Patching Procedure" section below
3. **Load the extension in Firefox**:
   - Open Firefox and go to `about:debugging`
   - Click "This Firefox" in the left sidebar
   - Click "Load Temporary Add-on"
   - Select the `manifest.json` file from this project directory
4. **Use LINE**: Click the LINE icon in your Firefox toolbar to open LINE in a popup window

**Note**: This is a temporary installation. The extension will be removed when you restart Firefox. For permanent installation, you would need to package it as a proper Firefox add-on.

## Patching Procedure
### * static/js/ltsmSandbox.js

```shell
js-beautify -r ./static/js/ltsmSandbox.js
```

Replace all instances of `window.origin`, `window.location.origin`, and `location.origin` in the file with `"chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc"`

### * manifest.json
```diff
    {
    //omitted
    //omitted
    "background": {
--      "service_worker": "background.js",
++      "scripts": ["background.js"],
        "type": "module"
    },
--  "sandbox": {
--      "pages": [
--      "ltsmSandbox.html",
--      "cropperSandbox.html"
--      ]
--  },
    //omitted
    //omitted
++  "content_security_policy": {
++      "extension_pages": "script-src 'self' 'wasm-unsafe-eval'; object-src 'self';"
++  },
++  "browser_specific_settings": {
++      "gecko": {
++          "id": "LINEPorted@FoxRefire"
++      }
++  },
--  "key": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuOql7UFiY9pkxo4aAmuN2HHlZhNT5ws6knRdxYhOACJcm1sBfB7GIIMuBwtpYSb3B3m7jbrKqX2iDdgYLxE9ZmFjYgrD6p4D4H9/4FCz/a7h66vp0onNu2PmbZOEnpZKeCgUGMDDcXk673R8tPfkBbmuzQ0rvpc1Z8hWgHo1jLtnjpkTlH4vzu9FGRQFsCuqUzJPjoPpa2rozvTPpmiO2qfcqH3FJoGJbKwXIPZ74JI8cY//o6xFDVhugveN1VqoGZA8PsVliAa5fgBqDohfiv36xkuD88BqynKNn00hGibuXrj4L6mnR+9I68dhwAiXY01gihtI6KhbekToLfoJmwIDAQAB",
    "version": "3.7.0"
    }
```

### * background.js

```shell
js-beautify -r ./background.js
```

```diff
--  chrome.action.onClicked.addListener((async () => {
--  //omitted
--  //omitted
--  }));
++  if(chrome.windows){
++      chrome.action.onClicked.addListener(() => {
++          chrome.windows.create({
++              url: "index.html",
++              type: "popup",
++              width: 710,
++              height: 570
++          });
++      });
++  } else {
++      chrome.action.onClicked.addListener(() => {
++          chrome.tabs.create({url: "index.html"})
++      })
++  }
```

## Writeup

First, I performed the typical porting procedure by changing `background.service_worker` to `background.scripts` in the manifest and adding `browser_specific_settings`, then tried opening `moz-extension://Internal-UUID/index.html`.
This resulted in a CSP error during WebAssembly execution, so I added `content_security_policy` to the manifest and ran it again.
<img width="1522" height="123" alt="1" src="https://github.com/user-attachments/assets/21f30da6-24ea-492f-8830-70913804e2a4" />

A `ltsm_not_available` error was still occurring, but the stack trace didn't clearly show the direct cause, so I decided to investigate the root source of the error.
<img width="1522" height="302" alt="2" src="https://github.com/user-attachments/assets/c6cba9df-2fd8-454b-9e7e-a96b3376a1ec" />

I set a breakpoint in the `catch` block of `setup()` which calls ltsmSandbox from the debugger, and registered `console.log(e)` as a watch expression.
<img width="2473" height="671" alt="3" src="https://github.com/user-attachments/assets/b3ff9733-fa41-49ca-a858-f4c5a3d11ce9" />

An upstream error `Error: sandbox_error` was logged to the console.
<img width="1522" height="398" alt="4" src="https://github.com/user-attachments/assets/05a1d771-f41c-44d1-8f8d-f28c38277844" />

When I exported `this.sandbox` and actually executed `init()` and `sendMessage()`, `sandbox_error` occurred within `sendMessage`.
<img width="1522" height="398" alt="5" src="https://github.com/user-attachments/assets/b0e33602-6512-4111-8b2a-90a29b3349f7" />

However, tracing through the stack trace revealed that this error wasn't the root cause either, but was being thrown by code `r(new Op(_d.SANDBOX_ERROR))` within the `jp` class.
To investigate the fundamental cause further, I set a breakpoint in the constructor of the `jp` class and registered `console.log(t.data)` as a watch expression.
<img width="2473" height="1019" alt="6" src="https://github.com/user-attachments/assets/b039ad70-666c-4e82-8fc3-19384bc2244e" />


The root cause error `Invalid origin: moz-extension://Internal-UUID` was confirmed.
<img width="2473" height="606" alt="7" src="https://github.com/user-attachments/assets/3f4d9a02-84be-4c7c-a3d9-995579799375" />

I opened `ltsmSandbox.js` after formatting it with js-beautify and searched for `Invalid origin` within the file.
Looking at the code, it appears to select tokens based on the extension's origin.
Since `moz-extension://Internal-UUID` doesn't correspond to any token, an error was occurring.
<img width="1882" height="306" alt="8" src="https://github.com/user-attachments/assets/7a6569eb-bf55-445f-bc3d-c8a62a947a05" />

Therefore, I overwrote all parts in `ltsmSandbox.js` that retrieve the current origin with the original origin `"chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc"`, and it began working correctly.
<img width="1522" height="1220" alt="9" src="https://github.com/user-attachments/assets/72480461-b2fd-452e-b5ca-4cbb03fa0764" />
<img width="1522" height="1220" alt="10" src="https://github.com/user-attachments/assets/e7ac1dc7-30c9-4d88-822d-876e67213392" />

