---
title: "マークダウン執筆で画像挿入が面倒だったのでNeovimプラグインを開発した話"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim", "markdown", "mdx"]
published: true
published_at: 2025-06-24 09:03
---

はじめに、私は多くの`Vimmer`は`Zenn`や個人ブログの記事を執筆するときも`Vim`で執筆する傾向にあると考えています。もちろん手慣れたテキストエディターなのでサクサク記事を書けますが、ただ唯一の欠点と言っていいほどに画像の挿入の面倒臭さに私は悩まされました。

従来の画像挿入、例えば`Zenn`の記事を執筆する際の画像挿入ワークフローをあげると、私は以下の工程で画像の追加をしていました。

```
1. ファイルマネージャーで画像ディレクトリを開く
2. 画像ディレクトリにインターネットからダウンロードしていた画像を任意の名前で保存する
3. Neovimで画像要素を手打ちする
```

この通りに三工程あって画像を挿入するだけのことに一苦労でした。今回は私がこのワークフローを`Neovim`から離れずにコマンド一つで行えるようにしたプラグインを開発しました。

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

依存関係についは、`OS`標準のクリップボードツールが必要なため、インストールされていない場合はインストール、もしくは`OS`の更新が必要となります。

- Windows: `Get-Clipboard`
- macOS: `pbpaste`, `osascript`, `sips`
- Linux (X11): `xclip`
- Linux (Wayland): `wl-paste`

### 使い方

前述の通り`mdxsnap.nvim`は覚えるコマンドが一つだけでシンプルに設計されています。

```vim
" 引数なしで実行できて、この場合はクリップボードの画像をランダムな文字列で保存します
:PasteImage

" クリップボードの画像を引数で指定したファイル名で保存します。人気があるのはこちらでしょう
:PasteImage [filename]
```

**動作デモ**

- マークダウン

以前は面倒な手順を３ステップ踏まないと画像挿入ができませんでしたが、`mdxsnap`を使うことでご覧の通り、たった１コマンドで瞬時に貼り付けられるようになりました。我ながらチョー便利に感じられました。

:::details マークダウンのスナップ

![マークダウンのスナップ](/images/introduce-mdxsnapnvim/snap_markdown.gif)

:::

- MDX

実装の際に工夫した機能として、`:PasteImage`コマンドを実行した後に、ひょこっと`mdx`ドキュメントの上部に`astro:assets`と`mdxsnapとプロジェクトのディレクトリの互換レイヤー`がインポートされるのをご覧になったと思います。また、考えられるバグの一つは重複インポートで、こちらについては`checkRegex`で対策しています。

:::details MDXのスナップ

![MDXのスナップ](/images/introduce-mdxsnapnvim/snap_mdx.gif)

:::

### 設定

`mdxsnap.nvim`は設定なしで動作しますが、ユーザーが任意の`CMS`向けの設定をする前提で設計されたため、使用体験はとても悪く、やはり設定するべきです。

:::details 設定のサンプル

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

:::

重要な設定として`customTextFormat`と`customImports`が挙げられます。これらは他のプラグインにはない`mdxsnap`の特徴と言えます。

- `customTextFormat`: Markdownの`![]()`画像形式構文を任意の形式に置き換えます。

これによって、`mdxsnap`が`:PasteImage`コマンド実行時に出力するマークダウンを変更することができます。

たとえば、`<img>`タグで画像を読み込んだり、独自に実装されたコンポーネントで画像を読み込むことができます。
この際に代入詞の`%s`は`alt -> src`の順番で記述してください。(`mdxsnap`の次のアップデートでは`{{alt}}, {{src}}`のようなディレクティブで記述できるようにするべきですね・・・)

```tsx
<img alt="%s" src="%s" />

<BlogImage alt="%s" src="%s" />
```

- `customImports`: MDXで画像用のコンポーネントや関数が不足している際に自動インポートします。
  - `line`: インポートする全文です。
  - `checkRegex`: インポートの重複対策として既存のインポートを検出するために必要なフレーズです。

:::message
もし、あなたの`MDX`を使用するマイクロブログでコンポーネント経由で画像が読み込まれない場合には`mdxsnap`からのディレクトリパスの出力を確認と実際にマークダウンレンダリングで解決されるディレクトリパスを確認してください。

たとえば、私のブログでは`/src/images/posts/[article name]/[image].png`のディレクトリパスが解決されて画像がレンダリングされる仕組みでしたが、`mdxsnap.nvim`の出力は`/client/src/images/posts/[article name]/[image].png`だったので、マークダウンパーサーと`mdxsnap`の出力の互換性を持つレイヤーを実装する必要がありました。

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

最後に覚えておくべき設定は`ProjectOverrides`です。この機能は平たくいえばプロファイル機能で、プロジェクトに適応した`customImports`などの設定を行えます。

- `ProjectOverrides`: プロジェクト単位で`customImports`や`customTextFormat`、`PastePath`などの変更を可能にします。
  - `matchType`: プロジェクトを上書きする際に、プロジェクトを判定するロジックを選択します。`"projectName"`もしくは`"projectPath"`が指定可能です。
    - `"projectName"`: (推奨) プロジェクトルートのディレクトリ名で判断します。ここがプロジェクトルートであるかの判断は、`.git`や`.svn`などのディレクトリの存在を基準に判断します。
    - `"projectPath"`: プロジェクトの絶対パスを指定して設定を上書きします。`git`などのDev-Opsを使用していないプロジェクトでの利用などを考慮しています。

  - `matchValue`: ここには`matchType`に対応した任意の値が指定されます。`matchType`が`"projectName"`の場合はプロジェクトルートのディレクトリ名 (`portfolio`, `dotfiles`, `zenn-articles`など)です。`"projectPath"`の場合はプロジェクトが存在するディレクトリの絶対パスを指定します。

  - `PastePath`: 記事に使用する画像ファイルの保存場所を決める設定項目です。

`PastePath`で指定したディレクトリの下に、以下の構造で画像が保存されます：

```
プロジェクトルート/
└── [PastePathで指定したディレクトリ]/
    └── [マークダウンファイル名]/
        └── [画像ファイル].png
```

- `PastePathType`: 記事の画像ファイルの保存先を決める`PastePath`のパスが、絶対パスか相対パスかを決めます。指定する値は`"relative"`か`"absolute"`です。
  - `"relative"`: (推奨) 画像の保存先をプロジェクトルートからの相対パスに指定します。
  - `"absolute"`: 画像の保存先を絶対パスに指定します。

これらの設定は、`mdxsnap.nvim`が他の`CMS`などへの対応といった柔軟性に貢献している機能です。

### Zennでmdxsnapをつかう

`mdxsnap`は仕組み上、`Zenn`の記事を書くことにも貢献できます。現にこの記事の画像挿入は`mdxsnap`で行われています。

`Zenn CLI`向けのサンプルは以下の通りです。`ProjectOverrides`にこの記述を追記したのちに、`matchValue`の`"zenn-articles"`という値をご自身のリポジトリ名にご変更ください。

```lua
{
    matchType = "projectName",
    matchValue = "zenn-articles", -- この項目をあなたのリポジトリ名に置き換えてください。
    -- :%s/zenn-articles/YOUR_REPO_NAME/g
    PastePath = "images",
    PastePathType = "relative",
}, -- ProjectOverrides に追加
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

このプラグインはほぼ一日で完成させたものなので、どうしても粗い実装などが見つかります。`v1.1.0`へのリリースにあたって修正項目などをリストアップします。

- `customImports`の代入詞

`customImports`では、`alt`テキストおよび`src`画像ファイルパスの入力に`%s`を仮置きの代入詞として使用していますが、あまりヒューマンフレンドリーな使用法ではないため、`{{alt}}, {{src}}`のような代入詞を実装する必要があります。

- ファイルコピーペーストの効率的な実装

現在、`mdxsnap`がコピペする際の実装の一部として、クリップボード内の画像をプラグインに受け渡しする目的と、クリップボード内のファイルをリネームするために任意のOSのコマンドで`~/.local/share/nvim/mdxsnap_tmp`に画像を作成した後に、`lua`の標準機能でファイルコピーなどを行っているため、あまり良い実装とは呼べません。`vim.api`もしくは`lua`標準機能で外部依存を減らした実装にシフトしたいですね。`OS`のクリップボードツールなどを極力呼び出さずに、`vim.api`のクリップボード関連ユーティリティなどがあればそちらで直接データを受け渡しできる構造に実装したいですね。（一工夫すれば`temp`ファイルを作らなくても受け渡しできそう）

また、`mdxsnap`を使用していてバグ発見、もしくは「こんな機能が欲しいな」「こうしたらいいじゃないの」のような提案などございましたら`GitHub`の`issue`ページで気軽にご連絡ください。 ここまで記事をご覧いただきありがとうございました！

こちらのプラグインを気に入られましたら是非ともスターをお付けください ⭐️

https://github.com/HidemaruOwO/mdxsnap.nvim

## 追記 4月23日

ある日に`refactor/overall-1`ブランチでコードベースを全面的にリファクタリングしました。ただ、受験勉強でE2Eテストできる余裕がないので、もしお都合がよければ代わりにテストお願いします。以下の環境でどれか一つだけでも大丈夫なので基本的な機能の確認次第`owner@v-sli.me`にメールをお願いします。またREADMEなどで感謝をしたいのでGitHubのIDを教えていただけると助かります🙏🏻🙏🏻
これらすべての動作確認次第masterブランチにマージします。

- macos
- wayland
- x11
- windows
