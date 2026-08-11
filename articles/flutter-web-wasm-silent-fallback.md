---
title: "Flutter Web の --wasm は黙って劣化する — skwasmが動いていないことを検出する"
emoji: "🕵️"
type: "tech"
topics: ["flutter", "flutterweb", "webassembly", "skwasm", "docker"]
published: false
---

## TL;DR

- `--wasm` ビルドには2段階の無言フォールバックがある。**どちらもエラーを
  出さない**
  - 段階1: Wasm(dart2wasm) → JS(dart2js/CanvasKit)。`kIsWasm` で判定できる
  - 段階2: skwasmのマルチスレッド → シングルスレッド。
    `crossOriginIsolated` で判定できる
- 段階1でFirefox/Safariが弾かれるのは「WasmGC未対応だから」ではない。
  両ブラウザともWasmGC自体には対応済みで、Flutter側が**既知のバグを理由に
  ブラウザエンジン単位でwasm自体を試みないようハードコードしている**、
  というのが実際の中身だった
- `kIsWasm` と `crossOriginIsolated` の2つをConsoleで見るだけで、今どの
  状態で動いているか分かる
- 実測(Edge・Chrome・Firefox、2026年8月11日時点): Edge・ChromeはCOOP/COEP
  ヘッダーの有無に応じてマルチスレッド↔シングルスレッドをきれいに
  切り替えた。Firefoxはヘッダーの有無に関わらず常に段階1で脱落する
- 前回記事の訂正: COOP/COEPヘッダーの検証を一度もしていなかった。
  結果的にはリポジトリの配信サーバーは最初からヘッダーを返しており、
  前回のEdge計測値はマルチスレッドskwasmのものだったと確認できた

## 背景

前回、[Flutter Web の CanvasKit と skwasm、実測して比較してみた](https://zenn.dev/amam96/articles/flutter-web-canvaskit-vs-skwasm)
という記事を書いた。その終盤で、`--wasm` ビルドをFirefoxで開いたら
Network タブ上は `main.dart.wasm` ではなく `main.dart.js` が読み込まれて
いた、つまり**黙ってCanvasKitにフォールバックしていた**ことに気づいた。
「未対応ブラウザには自動でJSにフォールバックする」という仕様通りではある
ものの、Network タブを目視で確認しない限り気づけない、という話で記事を
締めた。

今回はその続き。前回見つけたフォールバックには実はもう1段階あって、
そちらは前回検証すらしていなかった。それに気づいて掘り直した、という
個人的な検証記録。前回同様、誰かに何かを強く主張したいわけではなく、
「`--wasm`って思ったより罠があるんだな」というのが伝われば十分、という
温度感で書いている。

- **段階1(前回気づいた方)**: Wasm(dart2wasm) → JS(dart2js/CanvasKit)
- **段階2(今回の本題)**: skwasmのマルチスレッド実行 → シングルスレッド
  実行

skwasmのマルチスレッド描画には `SharedArrayBuffer` が必要で、そのために
配信サーバーが `Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy`
(COOP/COEP)ヘッダーを返している必要がある。これが欠けていると、skwasm自体
は動くのだが、描画がUIスレッドに相乗りするシングルスレッド動作に**エラー
なく**フォールバックする。

前回の記事では、このヘッダーの有無を一度も確認せずに計測していた。つまり
**前回のベンチは実は段階2に落ちた状態で計測していた可能性があった**。

:::message
結論を先に書いておくと、このリポジトリのベンチ用配信サーバー
(`tool/serve_static.dart`)は、ベンチツール一式を追加した最初のコミット
時点から既にCOOP/COEPヘッダーを返していた。つまり前回のEdge計測は
実際にはマルチスレッドskwasmで行われていたと考えてよい(後述の実測で
確認済み)。**数値自体を訂正する必要はなかった**が、それを確認する手段が
なかったこと自体が今回のテーマになる。
:::

## 段階1: Wasm → JS フォールバック

`--wasm` でビルドしても、実行時にWasmGCが使えないと判断されると、
自動的にJS(dart2js/CanvasKit)版が読み込まれる。これは
[公式ドキュメント](https://docs.flutter.dev/platform-integration/web/wasm)
に明記されている仕様だが、ここでの「使えない」の中身が意外だった。

### WasmGC自体はFirefox/Safariも対応済み

WasmGCは [Chrome/Chromium 119+](https://caniuse.com/wasm-gc) だけでなく、
Firefox 120+、Safari(WebKit)も既に対応している。にもかかわらず
`docs.flutter.dev` は次のように明記している(2026年8月11日時点で確認)。

- **Chromium/Chrome(119+)**: 完全サポート
- **Firefox(120+)**: WasmGC自体はサポート発表済みだが、**既知のバグにより
  現在は動作しない**([Firefox bug 1788206](https://bugzilla.mozilla.org/show_bug.cgi?id=1788206))
- **Safari**: WasmGC実装済みだが、**既知のバグにより互換性がない**
  ([WebKit bug 267291](https://bugs.webkit.org/show_bug.cgi?id=267291))
- **iOS上の全ブラウザ**: WebKit強制のため、Chrome for iOSを含め動作しない

つまり「WasmGCに対応していないから動かない」のではなく、「WasmGCには
対応しているが、関連する既知のバグがあるので、Flutter側が意図的に
無効化している」というのが正確な理解になる。

両バグとも2026年8月11日時点で **NEW(未解決)**。中身を覗くと、どちらも
「wasm自体が動かない」バグではなく、**Canvasの内容をやり取りする際の
パフォーマンス劣化**に関するバグだった。

- Firefox bug 1788206: `OffscreenCanvas.transferToImageBitmap` が
  内部的にコピーを伴い、Chromeが0〜0.1msで終わる処理にFirefoxは
  25〜30msかかる、という再現ケースが報告されている
- WebKit bug 267291: 異なるcanvas間・canvas種別間での画像ソースのやり取りが
  遅い、という趣旨のバグ

つまり「動かせなくはないが、動かすと著しく遅くなる」ため、Flutterのチームが
リリース前に無効化を選んだ、という話に読める。

### ビルド成果物を読むと、無効化はハードコードされている

実際にビルドして`flutter_bootstrap.js`を読むと、ブラウザエンジンごとの
許可リストがそのまま書かれている。

```js
var _ = {blink: !0, gecko: !1, webkit: !1, unknown: !1};
```

`blink`(Chromium系)だけが `true`。`gecko`(Firefox)・`webkit`(Safari)は
デフォルトで `false` に固定されている。つまりFlutterのローダーは、
ブラウザが実際にWasmGCを実行できるかを機能検出する前に、**エンジンの
種類だけを見てwasmを試すかどうかを決めている**。(アプリ側で
`wasmAllowList` を明示的に上書きすれば変更できるが、デフォルトでは
このリストが使われる。)

### kIsWasmで判定する

実行中のバイナリが本当にdart2wasmでコンパイルされたものかどうかは、
`package:flutter/foundation.dart` の
[`kIsWasm`](https://api.flutter.dev/flutter/foundation/kIsWasm-constant.html)
というコンパイル時定数で判定できる。実体は
`bool.fromEnvironment('dart.tool.dart2wasm')`。JSにフォールバックした
場合はJS版バイナリ自体が動くので、この値は自動的に `false` になる。

## 段階2: マルチスレッド → シングルスレッド フォールバック

skwasmのマルチスレッド描画は `SharedArrayBuffer` を使う。ブラウザで
`SharedArrayBuffer` を有効にするには、ページが
[クロスオリジン分離(cross-origin isolation)](https://developer.mozilla.org/en-US/docs/Web/API/Window/crossOriginIsolated)
された状態、つまり配信サーバーが以下の2つのヘッダーを返している必要が
ある。

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

(`credentialless` でも成立する。後述)

これも実際にビルドして読むと、`flutter_bootstrap.js` に直球の分岐が
書かれている。

```js
skwasmSingleThreaded: e.enableWimp || !n.crossOriginIsolated ||
  n.isChromeExtension || e.forceSingleThreadedSkwasm,
```

`window.crossOriginIsolated` が `false` なら、問答無用でシングルスレッド
に倒す。例外もエラーメッセージも出ない。skwasm自体は動いているので、
`kIsWasm` を見ただけでは段階2の劣化には気づけない。

### crossOriginIsolatedで判定する

`dart:js_interop` で `window.crossOriginIsolated` を直接読めば判定できる。

```dart
@JS('crossOriginIsolated')
external bool get _crossOriginIsolated;
```

`kIsWasm` と組み合わせると、実行中の状態を3つに分類できる。

| 状態 | kIsWasm | crossOriginIsolated |
|---|---|---|
| CanvasKit(JS) ※段階1で脱落 | false | (無関係) |
| skwasmシングルスレッド ※段階2で脱落 | true | false |
| skwasmマルチスレッド ※想定通り | true | true |

## 実際にどう確認するか

`kIsWasm` と `crossOriginIsolated` さえ見えれば判定できるので、テスト
アプリの起動時にこの2つを `window` へ書き出すようにした。

```dart
// main() の起動時に1回呼ぶ
void exposeRendererStatusToJs() {
  globalContext.setProperty('__benchKIsWasm'.toJS, kIsWasm.toJS);
  globalContext.setProperty(
      '__benchCrossOriginIsolated'.toJS, _crossOriginIsolated.toJS);
}
```

あとはDevToolsのConsoleで叩くだけ。

```js
JSON.stringify({
  kIsWasm: window.__benchKIsWasm,
  crossOriginIsolated: window.__benchCrossOriginIsolated,
})
```

これで今開いているページが「CanvasKit(JS)にフォールバック済みか」
「skwasmだとしてシングルスレッドかマルチスレッドか」がその場で分かる。
(手元のテストアプリでは、この状態を常時表示する色分けバッジも作って
みたが、記事としては上のConsoleコマンドだけ覚えておけば十分。)

一つ注意点がある。**「CanvasKit(JS)」の状態は、`--wasm`ビルドが
フォールバックした結果なのか、そもそも`--wasm`を付けずにビルドしたのかを、
実行中のアプリの内部だけからは区別できない**。どちらもコンパイル後の
バイナリはただのJS版であり、`kIsWasm`はfalseにしかならないからだ。
「自分が何をビルド・デプロイしたか」を把握した上で、想定と食い違って
いないかを確認する、という前提が要る。

## ブラウザ別マトリクス(実測)

### 検証環境

| 項目 | 値 |
|---|---|
| Flutter | 3.44.9 |
| Dart | 3.12.2 |
| 開発環境 | Docker (Debian bookworm-slim ベース、Flutter SDKはバージョン固定) |
| ホストOS | Windows 11 + WSL2 (Ubuntu 24.04) |
| マシン | Intel Core i7-1185G7 (第11世代) / RAM 32GB / Intel Iris Xe Graphics(内蔵GPU) |
| 計測日 | 2026-08-11 |
| Edge | 151.0.4129.72 (公式ビルド, 64ビット) |
| Chrome | 151.0.7922.76 (公式ビルド, 64ビット, cohort: Stable) |
| Firefox | 153.0.3 |

### 構成A: COOP/COEPヘッダーなし

| ブラウザ | ビルド | kIsWasm | crossOriginIsolated | 判定結果 | 読み込まれたmainファイル |
|---|---|---|---|---|---|
| Edge | CanvasKit | false | false | canvasKitJs | main.dart.js |
| Edge | skwasm | true | false | **skwasmSingleThread** | main.dart.mjs |
| Chrome | CanvasKit | false | false | canvasKitJs | main.dart.js |
| Chrome | skwasm | true | false | **skwasmSingleThread** | main.dart.mjs |
| Firefox | CanvasKit | false | false | canvasKitJs | main.dart.js |
| Firefox | skwasm | false | false | canvasKitJs | main.dart.js |

### 構成B: COOP/COEPヘッダーあり(`require-corp`)

| ブラウザ | ビルド | kIsWasm | crossOriginIsolated | 判定結果 | 読み込まれたmainファイル |
|---|---|---|---|---|---|
| Edge | CanvasKit | false | true | canvasKitJs | main.dart.js |
| Edge | skwasm | true | true | **skwasmMultiThread** | main.dart.mjs |
| Chrome | CanvasKit | false | true | canvasKitJs | main.dart.js |
| Chrome | skwasm | true | true | **skwasmMultiThread** | main.dart.mjs |
| Firefox | CanvasKit | false | true | canvasKitJs | main.dart.js |
| Firefox | skwasm | false | true | canvasKitJs | main.dart.js |

Edge・Chromeは、ヘッダーの有無だけで橙↔緑がきれいに切り替わった。段階2の
フォールバックが実ブラウザでも確認できたことになる。

Firefoxはヘッダーの有無に関わらず一貫して赤(`canvasKitJs`)だった。これは
段階1で先に脱落しているため、段階2(ヘッダーの有無)を観測する以前の
状態にあるということ。段階1と段階2は独立した別の問題であることが、この
対比から見て取れる。

### コラム: require-corpだとCanvasKitのgstatic取得が壊れるのでは?

検証を始める前は、こう懸念していた。CanvasKitは
`https://www.gstatic.com/flutter-canvaskit/...` から読み込まれる。
`Cross-Origin-Embedder-Policy: require-corp` は、クロスオリジンの
サブリソースに対応ヘッダーを要求するので、これが原因でCanvasKit自体の
取得が失敗するかもしれない、と。

`curl -I` でgstatic側のレスポンスを見ると、この懸念は的外れだった。

```
$ curl -sI https://www.gstatic.com/flutter-canvaskit/<revision>/chromium/canvaskit.js
access-control-allow-origin: *
cross-origin-resource-policy: cross-origin
```

`Cross-Origin-Resource-Policy: cross-origin` と
`Access-Control-Allow-Origin: *` が既に付与されている。つまりgstatic側は
`require-corp` を前提にした設計になっている。

`require-corp` と `credentialless` の両方を3ブラウザで実測したが、
`crossOriginIsolated` はどちらも `true` になり、コンソールにエラーも
出なかった(Chromeで無害な情報レベルのIssueが1件出た程度)。事前の懸念は
実測では再現しなかった、というのが結論。

### 未検証 — Safari / iOS

Safari・iOS実機は手元にないため実測していない。以下は
**公式ドキュメントとWebKit Issueに基づく伝聞**であり、実測値ではない。

| ブラウザ | 根拠 | 予想される判定結果 |
|---|---|---|
| Safari (macOS) | [WebKit bug 267291](https://bugs.webkit.org/show_bug.cgi?id=267291)。2026年8月11日時点でNEW(未解決) | canvasKitJs |
| Safari (iOS/iPadOS) | 全ブラウザがWebKit強制のため上記と同条件 | canvasKitJs |
| Chrome for iOS | 同上(iOSは全ブラウザWebKit強制) | canvasKitJs |

エンジン側のバグは修正されうる。この記事の内容は2026年8月11日時点の
バージョン・Issueの状態に基づいている。上記リンク先で最新状況を確認して
ほしい。

## まとめ

`--wasm` ビルドには段階1(Wasm→JS)・段階2(マルチ→シングルスレッド)の
2段階の無言フォールバックがある。どちらもエラーを出さないので、
Network タブを目視するか、`kIsWasm` / `crossOriginIsolated` を明示的に
確認しない限り気づけない。

- 段階1の原因は「WasmGC未対応」ではなく「既知のバグを理由にした
  ブラウザエンジン単位でのハードコードされた無効化」だった
- 段階2は配信サーバーのCOOP/COEPヘッダー次第で、実機のEdge・Chromeで
  実際にオン/オフを切り替えられることを確認した
- 前回記事のベンチ環境は元々ヘッダーを返しており、Edgeの計測値は
  マルチスレッドskwasmのものだったと確認できた。ただしそれを
  「確認していなかった」こと自体が今回の出発点になっている
- `window.__benchKIsWasm` / `__benchCrossOriginIsolated` をConsoleで見る、
  というだけの単純なやり方だが、同じ手法は他のFlutter Webプロジェクトにも
  そのまま応用できるはず

### 今回の限界

- Safari・iOSは実機がないため実測していない(公式ドキュメント・Issueに
  基づく予想のみ)
- skwasmのシングルスレッド版とマルチスレッド版のパフォーマンス差(build/
  raster/fps)の再計測は今回のスコープに含めていない
- Firefox/Safariのエンジン側バグは修正されうる。この記事は2026年8月11日
  時点の状態に基づいている
