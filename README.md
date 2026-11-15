# browser-engineering-python

Pavel Panchekha、Chris Harrelson『Web Browser Engineering』（[browser.engineering](https://browser.engineering/)）に沿って、PythonでWebブラウザを書いたリポジトリ。本の1章から11章（Adding Visual Effects）まで、1章ごとに1コミットで進めた。コードは本の実装に沿っており、処理の説明を日本語のコメントで書き足している。

## できること

- HTTP/1.0でページを取得する（`http` と `https`、ポート指定、TLSは標準ライブラリの `ssl`）。`Set-Cookie` を保存して送り返し、`SameSite=Lax` のCookieは別サイトからのPOSTに付けない
- HTMLを解析して木を作る。自己終了タグと、省略された `<html>`・`<head>`・`<body>` の補完に対応
- CSSを解析してスタイルを当てる。タグセレクタと子孫セレクタ、`style` 属性、`<link rel=stylesheet>`、セレクタの詳細度による並べ替え、`font-size` などの継承
- ブロックレイアウトとインラインレイアウト（行の折り返し、太字・斜体・文字サイズ）
- SkiaでページとブラウザのUIを描き、SDL2のウィンドウに表示する（10章まではTkinter、11章でSkiaとSDL2に置き換えた）
- タブ、アドレスバー、戻るボタン、リンクのクリック、下矢印キーでのスクロール
- `<input>` と `<button>`、フォームのPOST送信
- dukpyでJavaScriptを実行する。`console.log`・`document.querySelectorAll`・`getAttribute`・`addEventListener`・`innerHTML` への代入・`preventDefault` を用意した
- `Content-Security-Policy` の `default-src` を読み、許可されていない読み込み元のスクリプトとスタイルシートを読み込まない
- `opacity`・`mix-blend-mode`・`border-radius`・`overflow: clip` の描画

`server.py` は本の8章・10章で使うゲストブックのサーバで、ポート8000で待ち受ける。Cookieによるセッション、CSRF対策のノンス、`html.escape` による出力のエスケープ、CSPヘッダを入れてある。ログインの処理は書いていないため、ノンスが発行されず、書き込みは受け付けられない。

## 動かし方

依存パッケージの一覧ファイルは無い。`import` しているのは `skia`・`sdl2`・`OpenGL`・`dukpy` で、PyPIのパッケージ名は次のとおり（WindowsではSDL2のDLLとして `pysdl2-dll` も要る）。

```sh
pip install skia-python pysdl2 pysdl2-dll PyOpenGL dukpy

# browser.css と runtime.js をカレントディレクトリから読むので、リポジトリのルートで実行する
python server.py                          # 別の端末で起動しておく
python browser.py http://localhost:8000/
```

Python 3.11で、ローカルの `server.py` からページを取得し、CSPで外部スクリプトを止めたうえでレイアウトと描画命令の生成まで通ることを確かめた（ウィンドウ表示は目視していない）。

## 構成

| ファイル | 役割 |
|---|---|
| `browser.py` | ブラウザ本体。`URL`（HTTP）、`HTMLParser`・`CSSParser`、`BlockLayout`・`LineLayout`・`TextLayout`・`InputLayout`（レイアウト）、`Draw*`・`Blend`（描画命令）、`JSContext`、`Tab`・`Chrome`・`Browser`（画面とイベント） |
| `browser.css` | ブラウザの既定スタイルシート |
| `runtime.js` | ページのスクリプトより先に読み込むJSのランタイム（`document`・`Node`・`Event`） |
| `server.py` | ゲストブックのサーバ |
| `comment.js` / `comment.css` | ゲストブックのページが読み込むスクリプトとスタイル |

## 設計・実装のポイント

本の設計をそのまま実装しているので、ここに挙げるのも本の考え方である。

- 取得 → HTML解析 → スタイル計算 → レイアウト → 描画命令のリスト（display list）→ 描画、と段階を分ける。スクロールしても描画命令を作り直さず、ずらして描くだけで済む
- JavaScriptとPythonの間ではDOMのノードを直接渡さず、整数のハンドルで参照する（`JSContext.get_handle`、`runtime.js` の `Node`）
- イベントは先にJS側のリスナーを呼び、`preventDefault` が呼ばれなかったときだけリンク遷移やフォーム送信などの既定動作をする
- `opacity` や `mix-blend-mode` は `Blend` 命令で子の描画命令を包み、Skiaの `saveLayer` で合成する。不透明かつ合成モードの指定が無いときは `saveLayer` を省く
- 11章でSkiaに移ったあとも、UIのクリック判定用に整数座標の `Rect` クラスを残し、描画するときだけ `skia.Rect` に変換している

## 学んだこと

- HTTPのリクエストとレスポンスを、ソケットに文字列を書いて読むところから組み立てる方法
- HTML・CSSの解析からレイアウト、描画までのブラウザの処理の流れと、それぞれの段階が持つデータ構造（DOM木、レイアウト木、描画命令のリスト）
- Cookie・SameSite・CSRFのノンス・CSPが、それぞれどの攻撃をどこで止める仕組みなのか

## 参考

- Pavel Panchekha、Chris Harrelson『Web Browser Engineering』 https://browser.engineering/
