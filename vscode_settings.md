```json
{
    // === エディタ関連の設定 ===
    "editor.renderWhitespace": "all",                    // 空白文字（スペース、タブ）を可視化
    "editor.detectIndentation": false,                   // ファイルからインデント設定を自動検出しない
    "editor.fontFamily": "Ricty Diminished",             // エディタのフォントをRicty Diminishedに設定
    "editor.fontSize": 14,                               // エディタのフォントサイズを14pxに設定
    "editor.wordWrap": "on",                             // 長い行を自動的に折り返す
    "editor.suggestSelection": "first",                  // 候補リストの最初の項目を自動選択
    "editor.formatOnSave": false,                        // ファイル保存時の自動フォーマットを無効化（デフォルト）
    "editor.formatOnPaste": false,                       // ペースト時の自動フォーマットを無効化
    "editor.formatOnType": false,                        // 入力時の自動フォーマットを無効化（デフォルト）
    "editor.rulers": [80],                               // 80文字目に縦線（ルーラー）を表示
    "editor.columnSelection": false,                     // 矩形選択を無効化
    "editor.minimap.showSlider": "always",               // ミニマップのスライダーを常に表示
    "editor.lineNumbers": "relative",                    // 相対行番号を表示（Vim用）
    "editor.codeActionsOnSave": {
        "source.fixAll.eslint": "explicit"              // 保存時にESLintの修正を明示的に実行
    },

    // === ファイル関連の設定 ===
    "files.autoGuessEncoding": true,                     // ファイルのエンコーディングを自動推測
    "files.autoSave": "onFocusChange",                   // フォーカスが外れた時に自動保存
    "files.eol": "\n",                                   // 改行コードをLF（Unix形式）に設定
    "files.watcherExclude": {
        "**/.fvm": true                                  // .fvmフォルダをファイル監視から除外
    },

    // === ワークベンチ（UI）関連の設定 ===
    "workbench.iconTheme": "file-icons",                // ファイルアイコンテーマをfile-iconsに設定
    "workbench.colorTheme": "Default Dark+",            // カラーテーマをDefault Dark+に設定
    "workbench.startupEditor": "none",                   // 起動時にエディタを開かない
    "workbench.editor.enablePreview": false,             // プレビューモードを無効化（常に新しいタブで開く）
    "workbench.colorCustomizations": {                   // カラーのカスタマイズ設定
        "statusBar.background": "#111111",              // ステータスバーの背景色
        "statusBar.noFolderBackground": "#111111",      // フォルダ未選択時のステータスバー背景色
        "statusBar.debuggingBackground": "#111111",      // デバッグ時のステータスバー背景色
        "statusBarItem.remoteBackground": "#111111",     // リモート接続時のステータスバー背景色
        "editor.selectionBackground": "#630404",         // エディタの選択範囲の背景色
        "editor.selectionHighlightBackground": "#630404", // 選択中のテキストと同じテキストの背景色
        "button.background": "#630404",                  // ボタンの背景色
        "button.hoverBackground": "#630404",             // ボタンホバー時の背景色
        "badge.background": "#630404",                   // バッジの背景色
        "inputOption.activeBackground": "#630404",       // アクティブな入力オプションの背景色
        "input.border": "#630404",                       // 入力フィールドの境界線色
        "inputOption.activeBorder": "#630404",           // アクティブな入力オプションの境界線色
        "focusBorder": "#630404",                        // フォーカス時の境界線色
        "list.activeSelectionBackground": "#630404",     // リストのアクティブ選択項目の背景色
        "list.inactiveSelectionBackground": "#630404",   // リストの非アクティブ選択項目の背景色
        "activityBarBadge.background": "#630404"         // アクティビティバーのバッジ背景色
    },

    // === エクスプローラー関連の設定 ===
    "explorer.openEditors.visible": 0,                   // 開いているエディタの表示を無効化

    // === ターミナル関連の設定 ===
    "terminal.integrated.copyOnSelection": true,         // 選択したテキストを自動的にコピー
    "terminal.integrated.fontSize": 12,                  // ターミナルのフォントサイズを12pxに設定
    "terminal.integrated.fontFamily": "monospace",       // ターミナルのフォントをmonospaceに設定
    "terminal.integrated.rightClickBehavior": "paste",   // 右クリックでペースト動作

    // === 検索関連の設定 ===
    "search.exclude": {
        "**/.fvm": true                                  // .fvmフォルダを検索から除外
    },

    // === 言語固有の設定 ===
    // TypeScript
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode" // TypeScriptのデフォルトフォーマッターをPrettierに設定
    },

    // Python
    "[python]": {
        "editor.defaultFormatter": "ms-python.black-formatter", // PythonのデフォルトフォーマッターをBlackに設定
        "editor.formatOnSave": true                      // Pythonファイルは保存時に自動フォーマット
    },

    // Dart/Flutter
    "[dart]": {
        "editor.formatOnType": true,                     // Dartファイルで入力時に自動フォーマット
        "editor.selectionHighlight": false,              // 選択ハイライトを無効化
        "editor.suggest.snippetsPreventQuickSuggestions": false, // スニペット使用時も候補を表示
        "editor.tabCompletion": "onlySnippets",          // タブキーでスニペットのみ補完
        "editor.wordBasedSuggestions": "off"             // 単語ベースの候補を無効化
    },
    "dart.flutterSdkPath": ".fvm/flutter_sdk",          // FlutterのSDKパスをFVMのものに設定
    "dart.openDevTools": "flutter",                      // DevToolsをFlutterモードで開く
    "dart.debugExternalPackageLibraries": true,         // 外部パッケージのデバッグを有効化
    "dart.debugSdkLibraries": true,                     // SDKライブラリのデバッグを有効化

    // === 拡張機能関連の設定 ===
    "extensions.ignoreRecommendations": true,            // 拡張機能の推奨を無視

    // Prettier
    "prettier.endOfLine": "lf",                          // Prettierの改行コードをLFに設定

    // Flutter Coverage
    "flutter-coverage.lowCoverageThreshold": 75,        // カバレッジ75%未満を低カバレッジとする
    "flutter-coverage.sufficientCoverageThreshold": 80, // カバレッジ80%以上を十分なカバレッジとする

    // Markdown Preview Enhanced
    "markdown-preview-enhanced.previewTheme": "github-light.css",  // プレビューテーマをGitHub Lightに設定
    "markdown-preview-enhanced.codeBlockTheme": "github.css",       // コードブロックのテーマをGitHubに設定
    "markdown-preview-enhanced.revealjsTheme": "simple.css",        // Reveal.jsのテーマをsimpleに設定

    // Markdown PDF
    "markdown-pdf.displayHeaderFooter": false,           // PDFのヘッダー・フッターを非表示

    // Paste Image
    "pasteImage.path": "${projectRoot}/images/",        // 画像の貼り付け先パス

    // Roo Cline
    "roo-cline.allowedCommands": [                      // Roo Clineで許可するコマンドのリスト
        "npm test",
        "npm install",
        "tsc",
        "git log",
        "git diff",
        "git show"
    ],

    // Gemini Code Assist
    "geminicodeassist.project": "",                      // プロジェクト設定（空）
    "geminicodeassist.codeGenerationPaneViewEnabled": false, // コード生成ペインを無効化

    // Draw.io
    "hediet.vscode-drawio.resizeImages": null,          // 画像リサイズ設定（null）

    // Vim
    "vim.useSystemClipboard": true,                      // システムクリップボードを使用
    "vim.insertModeKeyBindings": [{                      // インサートモードのキーバインド設定
        "before": ["j", "j"],                            // jjでエスケープ
        "after": ["<Esc>"]
    }],
    "vim.hlsearch": true,                                // 検索結果のハイライトを有効化

    // === その他の設定 ===
    "security.workspace.trust.untrustedFiles": "open",   // 信頼されていないファイルを開く
    "update.showReleaseNotes": false,                    // アップデート時のリリースノートを表示しない
    "redhat.telemetry.enabled": true                     // Red Hatのテレメトリを有効化
}
```