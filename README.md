# LINE Firefox用非公式ポート

## パッチ手順
### * static/js/ltsmSandbox.js

```shell
js-beautify -r ./static/js/ltsmSandbox.js
```

ファイル中の全ての`window.origin`、`window.location.origin`、`location.origin`を`"chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc"`に置き換え

### * manifest.json
```diff
    {
    //省略
    //省略
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
    //省略
    //省略
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
--  //省略
--  //省略
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

まず、一般的なポート手順であるマニフェスト内のbackground.service_workerをbackground.scriptsに変更しbrowser_specific_settingsの追加を行って`moz-extension://Internal-UUID/index.html`を開いてみる。
このままではWebAssembly実行でCSPエラーが発生したため更にマニフェストにcontent_security_policyを追加して再実行。
<img width="1522" height="123" alt="1" src="https://github.com/user-attachments/assets/21f30da6-24ea-492f-8830-70913804e2a4" />

まだ、ltsm_not_availableというエラーが発生していたが、スタックトレースからは直接の原因箇所が明らかではなかったため、大元のエラー箇所を探ることにする
<img width="1522" height="302" alt="2" src="https://github.com/user-attachments/assets/c6cba9df-2fd8-454b-9e7e-a96b3376a1ec" />

デバッガーからltsmSandboxを呼び出している`setup()`の`catch`内にブレークポイントを設定して、監視式に`console.log(e)`と登録してみる
<img width="2473" height="671" alt="3" src="https://github.com/user-attachments/assets/b3ff9733-fa41-49ca-a858-f4c5a3d11ce9" />

コンソールにさらに上流のエラーと見られる`Error: sandbox_error`が記録された
<img width="1522" height="398" alt="4" src="https://github.com/user-attachments/assets/05a1d771-f41c-44d1-8f8d-f28c38277844" />

更に`this.sandbox`をエクスポートしてみて実際に`init()`、`sendMessage()`を実行してみると`sendMessage`内で`sandbox_error`が発生していた
<img width="1522" height="398" alt="5" src="https://github.com/user-attachments/assets/b0e33602-6512-4111-8b2a-90a29b3349f7" />

しかしスタックトレースから辿ったところこのエラーも根本原因のエラーではなく、`jp`クラス内で`r(new Op(_d.SANDBOX_ERROR))`というコードによってスローされていることが分かった。
更に根本的な原因を探るため`jp`クラスのコンストラクタ内にブレークポイントを設定し、監視式に`console.log(t.data)`を登録
<img width="2473" height="1019" alt="6" src="https://github.com/user-attachments/assets/b039ad70-666c-4e82-8fc3-19384bc2244e" />


根本原因とみられるエラー`Invalid origin: moz-extension://Internal-UUID`が確認できた
<img width="2473" height="606" alt="7" src="https://github.com/user-attachments/assets/3f4d9a02-84be-4c7c-a3d9-995579799375" />

`ltsmSandbox.js`をjs-beautifyでインデントを整えてから開き、ファイル内で`Invalid origin`と検索してみる
コードを見ると拡張機能のオリジンに応じて使用するトークンを選択しているようだ。
`moz-extension://Internal-UUID`はトークンと対応していないため、エラーが発生していることが分かった。
<img width="1882" height="306" alt="8" src="https://github.com/user-attachments/assets/7a6569eb-bf55-445f-bc3d-c8a62a947a05" />

そこで、`ltsmSandbox.js`内の全ての現在のオリジンを取得している部分の処理を本来のオリジン`"chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc"`に上書きしてみたところ、正しく動作するようになった。
<img width="1522" height="1220" alt="9" src="https://github.com/user-attachments/assets/72480461-b2fd-452e-b5ca-4cbb03fa0764" />
<img width="1522" height="1220" alt="10" src="https://github.com/user-attachments/assets/e7ac1dc7-30c9-4d88-822d-876e67213392" />

