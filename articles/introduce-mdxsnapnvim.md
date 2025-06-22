---
title: "マークダウン執筆で画像挿入が面倒だったのでNeovimプラグインを開発した話"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim", "markdown", "mdx"]
published: false
---

[![not-by-ai](/images/introduce-mdxsnapnvim/not-by-ai.png)](https://notbyai.fyi/)

はじめに、多くの`Vimmer`は`Zenn`や個人ブログの記事を執筆するときも`Vim`で執筆すると思います。もちろん手慣れたテキストエディターなのでサクサク記事を書けますが、ただ唯一の欠点と言っていいほどに画像の挿入の面倒臭さに悩まされました。

従来の画像挿入、例えば`Zenn`の記事を執筆する際の画像挿入をワークフローをあげると、自分は以下の工程で画像の追加をしていました。

```
1. ファイルマネージャーで画像ディレクトリを開く
2. 画像ディレクトリにインターネットからダウンロードしていた画像を任意の名前で保存する
3. Neovimで画像要素を手打ちする
```

この通りに三工程あって画像を挿入するだけのことに一苦労です。今回はこのワークフローを`Neovim`から離れずにコマンド一つで行えるようにしたプラグインを開発しました。

:::message
この記事の対象は`Neovim`ユーザーに限ります。`mdxsnap.nvim`は`lua`製のプラグインであるため`Vim`では動作しません。
:::

## mdxsnap.nvim

`mdxsnap.nvim`はクリップボードに保存された画像をマークダウンもしくは、MDX形式の文書にコピペするプラグインです。

注目すべき特徴は主にこれら５つが挙げられます。

- 覚えるコマンドは一つだけ `:PasteImage [filename]`
- プロジェクト単位でのプロファイル分け (`ProjectOverrides`)
- 画像の保存場所に任意のディレクトリパスを使用可能 (`PastePath`, `DefaultPastePath`)
- Markdownの`![]()`画像形式構文を任意の形式に置き換える (`customTextFormat`)
- MDXで画像用のコンポーネントや関数が不足している際に自動インポート (`customImports`)

これらの機能によって、様々なディレクトリ構造を取る`CMS`やブログサービスに対応しています。

https://github.com/HidemaruOwO/mdxsnap.nvim

### インストール

通常のインストール方法でインストールできます。

- Example for `lazy.nvim`

```lua
{
  'HidemaruOwO/mdxsnap.nvim',
  config = function()
    require('mdxsnap').setup()
  end
}
```

- Example for `jetpack.vim (packer style)`

```lua
{
	"HidemaruOwO/mdxsnap.nvim",
	cmd = "PasteImage",
	ft = { "markdown", "mdx" },
	config = function()
        require('mdxsnap').setup()
	end,
},

```

依存関係について`OS`標準のクリップボードツールが必要なためインストールされていない場合はインストールもしくは、`OS`の更新を必要としてます。

- Windows: `Get-Clipboard`
- macOS: `pbpaste`, `osascript`, `sips`
- Linux (X11): `xclip`
- Linux (Wayland): `wl-paste`

### 使い方

前述の通り`mdxsnap.nvim`は覚えるコマンドは一つだけでシンプルに設計されています。

```vim
" 引数なしで実行できて、この場合はクリップボードの画像をランダムな文字列で保存します
:PasteImage

" クリップボードの画像を引数で指定したファイル名で保存します。人気があるのはこちらでしょう
:PasteImage [filename]
```

**動作デモ**

- マークダウン

以前は面倒な手順を３ステップ踏まないと画像挿入がご覧の通り、たった１コマンドで瞬時に貼り付けられます。我ながらチョー便利に感じられました。

:::details マークダウンのスナップ

![マークダウンのスナップ](/images/introduce-mdxsnapnvim/snap_markdown.gif)

:::

- MDX

実装の際工夫した機能として、`:PasteImage`コマンドを実行したのちに、ひょこっと`mdx`ドキュメントの上部にて`astro:assets`と`mdxsnapとプロジェクトのディレクトリの互換レイヤー`をインポートしているのをご覧になったと思います。また、考えられるバグの一つは重複インポートで、こちらについては`checkRegex`で対策しています。

:::details MDXのスナップ

![MDXのスナップ](/images/introduce-mdxsnapnvim/snap_mdx.gif)

:::

### 設定

`mdxsnap.nvim`は設定なしで動作しますが、ユーザーが任意の`CMS`向けの設定をする前提で設計したため使用体験はとても悪くやはり設定するべきです。

- 設定項目のサンプル

```lua
require("mdxsnap").setup({
  -- 画像保存のデフォルトパス
  DefaultPastePath = "snaps", -- デフォルト: "snaps/images/posts"
  DefaultPastePathType = "relative",       -- デフォルト: "relative" ("absolute"も指定可能)
  -- ファイル内に存在させるグローバルなカスタムインポート文
  customImports = {
    {
      line = 'import { Image } from "astro:assets";', -- 完全なインポート行
      checkRegex = 'astro:assets',                   -- 既存インポートをチェックする文字列/正規表現
    },
  },
  -- 挿入される画像参照テキストのグローバル形式
  customTextFormat = "![%s](%s)", -- デフォルト: Markdown画像形式 "![alt](src)"
  -- 特定プロジェクト用のデフォルト設定オーバーライド
  ProjectOverrides = {
    {
      -- プロジェクトディレクトリ名でマッチ
      matchType = "projectName",             -- "projectName" または "projectPath"
      matchValue = "my-astro-blog",        -- プロジェクトルートディレクトリ名
      PastePath = "src/assets/blog-images", -- このプロジェクト用カスタムパス
      PastePathType = "relative",
      customImports = { -- このプロジェクト用グローバルcustomImportsをオーバーライド
        { line = 'import { BlogImage } from "@/components/BlogImage.astro";', checkRegex = "@/components/BlogImage.astro" },
      },
      customTextFormat = '<BlogImage alt="%s" src="%s" />', -- グローバルcustomTextFormatをオーバーライド
    },
       -- 必要に応じてさらにルールを追加
  },
})
```

まず、覚えておくべき設定項目は

- `customTextFormat`: Markdownの`![]()`画像形式構文を任意の形式に置き換えます。

これによって`mdxsnap`が`:PasteImage`コマンド実行時に出力するマークダウンの出力を変更することができます。

たとえば、`<img>`タグで画像を読み込んだり、独自に実装されたコンポーネントで画像を読み込むことができます。
この際に代入詞の`%s`は`alt -> src`の順番で書いてください。(`mdxsnap`の次のアップデートでは`{{alt}}, {{src}}`のようなディレクティブで書けるようにするべきですね・・・)

```tsx
<img alt="%s" src="%s" />

<BlogImage alt="%s" src="%s" />
```

- `customImports`: MDXで画像用のコンポーネントや関数が不足している際に自動インポートします。

  - `line`: インポートする全文です。
  - `checkRegex`: インポートの重複対策として既存のインポートを検出するために必要なフレーズです。

:::message
もし、あなたの`MDX`を使用するマイクロブログでコンポーネント経由で画像が読み込まれない場合には`mdxsnap`からのディレクトリパスの出力を確認と実際にマークダウンレンダリングで解決されるディレクトリパスを確認してください。

たとえば、自分のブログでは`/src/images/posts/[article name]/[image].png`のディレクトリパスが解決されて画像がレンダリングされる仕組みでしたが、`mdxsnap.nvim`の出力は`/client/src/images/posts/[article name]/[image].png`だったので、マークダウンパーサーと`mdxsnap`の出力の互換性を持つレイヤーを実装する必要がありました。

```lua
{
	matchType = "projectName",
	matchValue = "portfolio",
	PastePath = "client/src/images/posts/",
	PastePathType = "relative",
	customImports = {
		{
			line = 'import { Image } from "astro:assets";',
			checkRegex = "astro:assets",
		},
		{
			line = 'import { ImportImage } from "@/lib/functions";',
			checkRegex = "@/lib/functions",
		},
	},
	customTextFormat = '<Image alt="%s" src={ImportImage("%s")} />',
},
```

```ts
export function ImportImage(path: string) {
  const images = import.meta.glob("/src/images/**/*.{png,jpg,jpeg,webp,gif}");

  // マークダウンパーサーは client ディレクトリを考慮しないが mdxsnap のマークダウンに出力する画像パスではプロジェクトルートからのパスを出力してしまうので、 client ディレクトリも含んでしまって取り除く必要があった。

  // 参考: Directory Structure
  // .
  // ├── README.md
  // ├── bun.lockb
  // ├── client
  // │   ├── README.md
  // │   ├── astro.config.mjs
  // │   ├── biome.jsonc
  // │   ├── bun.lock
  // │   ├── dist
  // │   ├── package.json
  // │   ├── public
  // │   ├── src
  // │   ├── tailwind.config.js
  // │   └── tsconfig.json
  // ├── package.json
  // ├── server
  // │   ├── web-server.ts
  // │   ├── commands
  // │   ├── computer-server.ts
  // │   └── lib
  // └── tsconfig.json

  const trim = path.replace(/^\/client\//, "/");

  return images[trim]();
}
```

:::

- `ProjectOverrides`: プロジェクト単位で`customImports`や`customTextFormat`、`PastePath`などの変更を可能にします。

  - `matchType`: プロジェクトを上書きする際にプロジェクトを判定するロジックを選択します。`"projectName"`もしくは`"projectPath"`が代入可能です。

    - `"projectName"`: (推奨) プロジェクトルートのディレクトリ名で判断します。ここがプロジェクトルートかの判断は`.git`や`.svn`などのディレクトリの存在を基準に判断します。
    - `"projectPath"`: プロジェクトの絶対パスを指定して設定を上書きします。`git`などのDev-Opsを使用してないプロジェクトでの利用などを考慮しています。

  - `matchValue`: ここには`matchType`に対応した任意の値が代入されます。`matchType`が`"projectName"`の場合はプロジェクトルートのディレクトリ名 (`portfolio`, `dotfiles`, `zenn-articles`など)。`"projectPath"`の場合はプロジェクトがあるディレクトリの絶対パスを代入します。

  - `PastePath`: 記事に使用する画像ファイルの保存場所を決める設定項目です。

PastePathで指定したディレクトリの下に、以下の構造で画像が保存されます：

```
プロジェクトルート/
└── [PastePathで指定したディレクトリ]/
    └── [マークダウンファイル名]/
        └── [画像ファイル].png
```

- `PastePathType`: 記事の画像ファイルの保存先を決める`PastePath`のパスを絶対パスか相対パスか決めます。代入する値は`"relative"`か`"absolute"`です。

  - `"relative"`: (推奨) 画像の保存先をプロジェクトルートからの相対パスに指定します。
  - `"absolute"`: 画像の保存先を絶対パスに指定します。

これらが他のプラグインにはない`mdxsnap.nvim`の柔軟性に貢献している機能です。

#### Zennでmdxsnapをつかう

`mdxsnap`は仕組み上`Zenn`の記事を書くことにも貢献できて、現にこの記事の画像挿入は`mdxsnap`で行われています。

`Zenn CLI`向けのサンプルは以下の通りです。`ProjectOverrides`にこの記述を追記したのちに、`matchValue`の`"zenn-articles"`という値をご自身のリポジトリ名にご変更ください。

```lua
{
    matchType = "projectName",
    matchValue = "zenn-articles", -- Replace with your actual repository name
    PastePath = "images",
    PastePathType = "relative",
},
```

- Directory Structure

```
.
├── articles
│   └── introduce-mdxsnapnvim.md
├── books
├── images
│   └── introduce-mdxsnapnvim
│       ├── github.png
│       ├── neovim.png
│       ├── not-by-ai.png
│       └── zenn-setting.png
├── README.md
├── bun.lock
└── package.json
```

## 今後の課題

このプラグインはほぼ一日で完成させたものなので、どうしても粗い実装などが見つかります。`v1.1.0`にリリースにあたって修正項目などをリストアップします。

- `customImports`の代入詞

`customImports`では`alt`テキストおよび`src`画像ファイルパスの入力に`%s`を仮置きの代入詞として使用していますが、あまりヒューマンライクな使用ではないため`{{alt}}, {{src}}`のような代入詞を実装する必要があります。

- ファイルコピーペーストの効率的な実装

現在`mdxsnap`がコピペする際の実装の一部として、クリップボード内の画像をプラグインに受け渡しする目的とクリップボード内のファイルをリネームするために任意のOSのコマンドで`~/.local/share/nvim/mdxsnap_tmp`に画像を作成したのちに`lua`の標準の機能でファイルコピーなどを行うため、あまり良い実装とは呼べません。`vim.api`もしくは`lua`標準の機能で外部依存を減らした実装にシフトチェンジしたいですね。`OS`のクリップボードツールなどを極力呼び出さずに`vim.api`のクリップボード関連のユーティリティなどがありましたらそちらで直接データを受け渡しできる構造に実装したいですね。（一工夫すれば`temp`ファイル作らなくても受け渡しできそう）

- `macOS`環境で`gif`画像を挿入できない

`macOS`でのクリップボード周りの実装は、直接クリップボード内のデータを出力する方法が分からなかったので、`osascript`で`png`形式で出力して受け渡ししているので、`gif`画像を貼り付けると静止画になってしまいます。自分のリサーチ不足でこのような実装になってしまったので悔やむ限りです。

- そもそもコードが汚い

はじめて作った`Neovim`プラグインにしてもコードがとても汚いです。手っ取り早く作りたかったため`VibeCoding`をしましたが、ノウハウなどがないため`LLM`に対して満足にカウンセリングもできずご覧の次第です。これからじっくりコードリーディングを行って自分主体で改修を進めたいと考えています。

また、`mdxsnap`を使用していてバグ発見、もしくは「こんな機能が欲しいな」「こうしたらいいじゃないの」のようなな提案がございましたら`GitHub`の`issue`ページで気軽にご連絡ください。 記事をご覧いただきありがとうございました！

https://github.com/HidemaruOwO/mdxsnap.nvim/issues
