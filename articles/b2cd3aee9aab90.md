---
title: "Denoでfaviconに必要なファイルを自動生成するCLIを作った"
emoji: "🌟"
type: "tech"
topics: ["deno", "typescript"]
published: true
---

## 成果

https://github.com/sabigara/faviconize

```bash
deno run --unstable --allow-read --allow-write --allow-ffi \
  "https://deno.land/x/faviconize/cmd.ts" \
  -i "path/to/icon.<svg or png>" \
  -o "path/to/outdir/"
```

## これは何か

1 枚の svg または png ファイルから favicon 用のファイル群を自動生成してくれる CLI ツールです。

どんなファイルをどんな形式で用意するのが正解なのかは正直よくわかっていない（し考えたくない）ので、この記事を参考にしました。

https://evilmartians.com/chronicles/how-to-favicon-in-2021-six-files-that-fit-most-needs

👇 生成されるファイル

```md
- favicon.ico
- icon-192.png
- icon-512.png
- apple-touch-icon.png
- manifest-webmanifest
```

## モチベーション

- Web サービスやサイトを作るたびに favicon を用意するのが面倒だったので
- Deno を使ってみたかったので
- 本業が煮詰まっていたので

## 使ったライブラリ

- [Cliffy](https://github.com/c4spar/deno-cliffy)
- [resvg-js](https://github.com/yisibl/resvg-js)
- [imagemagick-deno](https://github.com/lumeland/imagemagick-deno)

## 実装方法

- `cliffy` でコマンドラインオプションを処理する。
- （入力が svg だった場合） `resvg` で png としてレンダリングする。
- `imagemagick-deno` でリサイズしたり、余白を足したり（`apple-touch-icon`）、フォーマットを変換し（`favicon.icon`）たりする。
- ファイルを出力する。

### Cliffy

> The Framework for Building Interactive Commandline Tools with Deno

👇 こんな感じでコマンドライン引数やオプションを定義・処理できるのでとても便利でした。

```typescript
await new Command()
  .name("Faviconize")
  .version("0.0.1")
  .description("Generate favicons")
  .option(
    "-i --input <input:file>",
    "Path to the source file. Available formats: [.png, .svg]",
    {
      required: true,
      value: (val) => {
        // バリデーション、トランスフォーム
        return val;
      },
    }
  )
  .option(
    "-o --output <output:file>",
    "Path to output a directory that contains the generated favicons.",
    {
      default: "." as const,
    }
  )
  .action((options) => {
    // 実際の処理・・・
    // optionsは指定した型で渡してくれる
  })
  .parse(Deno.args);
```

`<input:file>` の部分は他にも `string` や `number` などの型を指定できます。これを見て「また`action` 内で型を絞らないといけないのか・・・😩」と思ってしまいそうですが、おそらく[Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)によって型を判断して渡してくれます。

### resvg-js

入力として svg を使いたかったので導入しました。API はシンプルだしドキュメント通りに動いてくれたので特に書くことはありません（それ以上にありがたいことはない）。

### imagemagick-deno

[ImageMagick](https://imagemagick.org/index.php)の[wasm 版](https://github.com/dlemstra/magick-wasm)を deno で動かせるようにしたものという認識です。

リサイズ・alpha の除去・`ico` への変換（という表現は正しい？）など一通りできるのですが、コールバックベースの API はやっぱり使いにくいです。しかし `Promise` でラップしようとしたらなぜか動かなかったのでがんばってそのまま実装しました（wasm の制限？）。

代替として[ImageScript](https://github.com/matmen/ImageScript)も試してみました。より使いやすい API で画像処理ができそうでしたが、自分の環境ではリサイズ時にガビガビになってしまったため採用は断念しました。僕の使い方が間違っている可能性もあります。

## Deno の感想

これだけ小さいツールでは判断できないことも多いですが、Deno を使った感想も書いておきます。

### 初速が速い

TypeScript やリンターなどのツール周りの設定をしなくてもいいので、**コードを書き始めるまでが速い**のはいいことだと思います。「JavaScript はどこでも動く」は本来強みであるはずなのに、ツールチェインの複雑さのせいで逆に障害になっている感があります。Web 標準へ準拠する姿勢なのもそういう意味ではいいのではないでしょうか。

Node でも[tsx](https://github.com/esbuild-kit/tsx)などを使えば ts ファイルを直接実行（？）できますが、ランタイム側でサポートしてくれているのはありがたいです。

### 公開が簡単

npm 向けの `package.json` の設定は非常に込み入っていてうんざりさせられますが、Deno のモジュールの公開はシンプルかつ簡単にできました。`main` と `module` と `exports` と `types` と `typesVersions` を設定する必要はありません。

[deno.land への公開](https://deno.land/add_module)も簡単だし、バージョニングが必要なければ[github.com](https://github.com)からインポートしても完全に正しいです。

ただ、この辺のシンプルさゆえに import 文に URL をすべて記述しないといけなかったりするので、npm から雑にダウンロードするのと比べると若干摩擦を感じる気もします（[It seems unwieldy to import URLs everywhere](https://deno.land/manual@v1.30.2/basics/modules#it-seems-unwieldy-to-import-urls-everywhere)で紹介されてるワークフローもどうなんだ？と思う）。

## TODO

- 画像・svg の最適化？
- Web 版？
- Node 版？

## まとめ

- Deno 初心者なので間違ってる部分があったら優しく教えてください。
- [個人ブログ](https://rubiq.vercel.app)もやっているのでよければ読んでください。
