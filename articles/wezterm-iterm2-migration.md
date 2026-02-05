---
title: 'iTerm2を捨ててWezTermにした - tmux不要のLua設定全公開'
emoji: ''
type: 'tech'
topics: ['wezterm', 'terminal', 'iterm2', 'lua', 'cli']
published: false
---

# はじめに

長年iTerm2を愛用してきましたが、WezTermに完全移行しました。この記事では移行の動機、詳細な設定方法、そして実際に使っている設定ファイル全体を公開します。

## 移行の動機

iTerm2に不満があったわけではありませんが、以下の点でWezTermに魅力を感じました。

- **クロスプラットフォーム対応**: macOS/Linux/Windowsで同一設定が使える
- **Lua設定の柔軟性**: コードで設定を記述できるため、条件分岐や関数化が可能
- **Rust製による高速性**: 起動・描画が高速
- **tmux代替機能**: Workspace/Pane/Tab管理が組み込み
- **GitHubで設定を管理**: dotfilesとして`.lua`ファイルをバージョン管理

iTerm2のプロファイル設定（GUI操作）に対し、WezTermは全てコードで記述するため、再現性と可搬性が高いのが決定的でした。

# WezTermとは

[WezTerm](https://wezfurlong.org/wezterm/)は、Rust製のGPUアクセラレーション対応ターミナルエミュレータです。

**主な特徴:**

- **クロスプラットフォーム**: macOS、Linux、Windows、FreeBSDに対応
- **Lua設定**: 設定ファイルをLuaで記述（`wezterm.lua`）
- **マルチプレクサ機能**: tmux不要でWorkspace/Tab/Pane管理
- **Ligature対応**: Nerd Fontsの合字を美しく描画
- **画像表示**: Kitty/iTerm2プロトコルに対応
- **SSHドメイン**: リモートサーバーのタブ管理

公式ドキュメントは非常に充実しており、[設定リファレンス](https://wezfurlong.org/wezterm/config/files.html)を見れば大抵の疑問は解決します。

# インストール

```bash
# Homebrewでインストール
brew install --cask wezterm

# バージョン確認
wezterm --version
```

インストール後、`~/.config/wezterm/wezterm.lua`に設定ファイルを配置します。

```bash
mkdir -p ~/.config/wezterm
cd ~/.config/wezterm
```

# 設定ファイルの構成

WezTermは単一の`wezterm.lua`でも動作しますが、可読性のために2ファイルに分割しています。

```
~/.config/wezterm/
├── wezterm.lua      # メイン設定（外観、タブ、一般設定）
└── keybinds.lua     # キーバインド専用
```

この分割により、キーバインドだけを変更したい時に見通しが良くなります。

## wezterm.lua（メイン設定）

```lua
local wezterm = require("wezterm")
local config = wezterm.config_builder()

-- 基本設定
config.automatically_reload_config = true
config.font_size = 12.0
config.use_ime = true  -- 日本語入力対応

-- 透過設定
config.window_background_opacity = 0.3
config.macos_window_background_blur = 20  -- macOSのみ有効

----------------------------------------------------
-- タブバーのカスタマイズ
----------------------------------------------------
-- タイトルバーを非表示（タブバーのみ表示）
config.window_decorations = "RESIZE"
config.show_tabs_in_tab_bar = true
config.hide_tab_bar_if_only_one_tab = true  -- タブが1つなら非表示

-- タブバーの透過設定
config.window_frame = {
  inactive_titlebar_bg = "none",
  active_titlebar_bg = "none",
}

config.window_background_gradient = {
  colors = { "#000000" },
}

-- タブUIの簡素化
config.show_new_tab_button_in_tab_bar = false  -- 追加ボタン非表示
config.show_close_tab_button_in_tabs = false   -- 閉じるボタン非表示

-- タブ間の境界線を非表示
config.colors = {
  tab_bar = {
    inactive_tab_edge = "none",
  },
}

----------------------------------------------------
-- タブの矢印型カスタマイズ（Nerd Fonts使用）
----------------------------------------------------
local SOLID_LEFT_ARROW = wezterm.nerdfonts.ple_lower_right_triangle
local SOLID_RIGHT_ARROW = wezterm.nerdfonts.ple_upper_left_triangle

wezterm.on("format-tab-title", function(tab, tabs, panes, config, hover, max_width)
  local background = "#5c6d74"
  local foreground = "#FFFFFF"
  local edge_background = "none"

  if tab.is_active then
    background = "#ae8b2d"  -- アクティブタブは金色
    foreground = "#FFFFFF"
  end

  local edge_foreground = background
  local title = "   " .. wezterm.truncate_right(tab.active_pane.title, max_width - 1) .. "   "

  return {
    { Background = { Color = edge_background } },
    { Foreground = { Color = edge_foreground } },
    { Text = SOLID_LEFT_ARROW },
    { Background = { Color = background } },
    { Foreground = { Color = foreground } },
    { Text = title },
    { Background = { Color = edge_background } },
    { Foreground = { Color = edge_foreground } },
    { Text = SOLID_RIGHT_ARROW },
  }
end)

----------------------------------------------------
-- キーバインド読み込み
----------------------------------------------------
config.disable_default_key_bindings = true
config.keys = require("keybinds").keys
config.key_tables = require("keybinds").key_tables
config.leader = { key = "q", mods = "CTRL", timeout_milliseconds = 2000 }

return config
```

### ポイント解説

**1. 透過設定**

```lua
config.window_background_opacity = 0.3
config.macos_window_background_blur = 20
```

背景透過率30%、macOSのブラー効果20で、デスクトップ壁紙がうっすら見える美しい外観になります。

**2. タブの矢印型デザイン**

Nerd Fontsの三角形記号を使い、tmuxのステータスラインのような矢印型タブを実現しています。アクティブタブは`#ae8b2d`（金色）で視認性を高めています。

**3. Leader Key**

```lua
config.leader = { key = "q", mods = "CTRL", timeout_milliseconds = 2000 }
```

tmuxの`Ctrl-b`や`Ctrl-a`に相当するLeader Keyを`Ctrl-q`に設定しています。2秒以内に次のキーを押すことで、複雑な操作を2ストロークで実行できます。

## keybinds.lua（キーバインド設定）

```lua
local wezterm = require("wezterm")
local act = wezterm.action

-- アクティブなキーテーブルをステータス表示
wezterm.on("update-right-status", function(window, pane)
  local name = window:active_key_table()
  if name then
    name = "TABLE: " .. name
  end
  window:set_right_status(name or "")
end)

return {
  keys = {
    ----------------------------------------------------
    -- Workspace管理（tmuxのセッションに相当）
    ----------------------------------------------------
    -- Workspace切り替え
    { key = "w", mods = "LEADER", action = act.ShowLauncherArgs({ flags = "WORKSPACES", title = "Select workspace" }) },
    -- Workspace名前変更
    { key = "$", mods = "LEADER", action = act.PromptInputLine({
      description = "(wezterm) Set workspace title:",
      action = wezterm.action_callback(function(win, pane, line)
        if line then
          wezterm.mux.rename_workspace(wezterm.mux.get_active_workspace(), line)
        end
      end)
    }) },
    -- Workspace新規作成
    { key = "W", mods = "LEADER|SHIFT", action = act.PromptInputLine({
      description = "(wezterm) Create new workspace:",
      action = wezterm.action_callback(function(window, pane, line)
        if line then
          window:perform_action(act.SwitchToWorkspace({ name = line }), pane)
        end
      end)
    }) },

    ----------------------------------------------------
    -- Tab操作
    ----------------------------------------------------
    { key = "Tab", mods = "CTRL", action = act.ActivateTabRelative(1) },
    { key = "Tab", mods = "SHIFT|CTRL", action = act.ActivateTabRelative(-1) },
    { key = "{", mods = "LEADER", action = act.MoveTabRelative(-1) },
    { key = "}", mods = "LEADER", action = act.MoveTabRelative(1) },
    { key = "t", mods = "SUPER", action = act.SpawnTab("CurrentPaneDomain") },
    { key = "w", mods = "SUPER", action = act.CloseCurrentTab({ confirm = true }) },

    -- Cmd + 数字でタブ直接切り替え
    { key = "1", mods = "SUPER", action = act.ActivateTab(0) },
    { key = "2", mods = "SUPER", action = act.ActivateTab(1) },
    { key = "3", mods = "SUPER", action = act.ActivateTab(2) },
    { key = "4", mods = "SUPER", action = act.ActivateTab(3) },
    { key = "5", mods = "SUPER", action = act.ActivateTab(4) },
    { key = "6", mods = "SUPER", action = act.ActivateTab(5) },
    { key = "7", mods = "SUPER", action = act.ActivateTab(6) },
    { key = "8", mods = "SUPER", action = act.ActivateTab(7) },
    { key = "9", mods = "SUPER", action = act.ActivateTab(8) },

    ----------------------------------------------------
    -- Pane操作（tmux風）
    ----------------------------------------------------
    -- Pane分割
    { key = "d", mods = "LEADER", action = act.SplitVertical({ domain = "CurrentPaneDomain" }) },
    { key = "r", mods = "LEADER", action = act.SplitHorizontal({ domain = "CurrentPaneDomain" }) },
    -- Pane閉じる
    { key = "x", mods = "LEADER", action = act.CloseCurrentPane({ confirm = true }) },
    -- Pane移動（hjkl）
    { key = "h", mods = "LEADER", action = act.ActivatePaneDirection("Left") },
    { key = "l", mods = "LEADER", action = act.ActivatePaneDirection("Right") },
    { key = "k", mods = "LEADER", action = act.ActivatePaneDirection("Up") },
    { key = "j", mods = "LEADER", action = act.ActivatePaneDirection("Down") },
    -- Pane選択UI
    { key = "[", mods = "CTRL|SHIFT", action = act.PaneSelect },
    -- Paneズーム切り替え（全画面化）
    { key = "z", mods = "LEADER", action = act.TogglePaneZoomState },

    ----------------------------------------------------
    -- コピー＆ペースト
    ----------------------------------------------------
    { key = "c", mods = "SUPER", action = act.CopyTo("Clipboard") },
    { key = "v", mods = "SUPER", action = act.PasteFrom("Clipboard") },
    -- コピーモード起動（Vim風）
    { key = "[", mods = "LEADER", action = act.ActivateCopyMode },

    ----------------------------------------------------
    -- その他
    ----------------------------------------------------
    { key = "p", mods = "SUPER", action = act.ActivateCommandPalette },
    { key = "Enter", mods = "ALT", action = act.ToggleFullScreen },
    { key = "+", mods = "CTRL", action = act.IncreaseFontSize },
    { key = "-", mods = "CTRL", action = act.DecreaseFontSize },
    { key = "0", mods = "CTRL", action = act.ResetFontSize },
    { key = "r", mods = "SHIFT|CTRL", action = act.ReloadConfiguration },

    ----------------------------------------------------
    -- キーテーブル起動
    ----------------------------------------------------
    -- Paneリサイズモード
    { key = "s", mods = "LEADER", action = act.ActivateKeyTable({ name = "resize_pane", one_shot = false }) },
    -- Pane移動モード（1秒間有効）
    { key = "a", mods = "LEADER", action = act.ActivateKeyTable({ name = "activate_pane", timeout_milliseconds = 1000 }) },
  },

  ----------------------------------------------------
  -- キーテーブル定義
  ----------------------------------------------------
  key_tables = {
    -- Paneリサイズモード
    resize_pane = {
      { key = "h", action = act.AdjustPaneSize({ "Left", 5 }) },
      { key = "l", action = act.AdjustPaneSize({ "Right", 5 }) },
      { key = "k", action = act.AdjustPaneSize({ "Up", 5 }) },
      { key = "j", action = act.AdjustPaneSize({ "Down", 5 }) },
      { key = "Enter", action = "PopKeyTable" },
      { key = "Escape", action = "PopKeyTable" },
    },

    -- Pane移動モード
    activate_pane = {
      { key = "h", action = act.ActivatePaneDirection("Left") },
      { key = "l", action = act.ActivatePaneDirection("Right") },
      { key = "k", action = act.ActivatePaneDirection("Up") },
      { key = "j", action = act.ActivatePaneDirection("Down") },
    },

    -- コピーモード（Vim風）
    copy_mode = {
      { key = "h", mods = "NONE", action = act.CopyMode("MoveLeft") },
      { key = "j", mods = "NONE", action = act.CopyMode("MoveDown") },
      { key = "k", mods = "NONE", action = act.CopyMode("MoveUp") },
      { key = "l", mods = "NONE", action = act.CopyMode("MoveRight") },
      { key = "w", mods = "NONE", action = act.CopyMode("MoveForwardWord") },
      { key = "b", mods = "NONE", action = act.CopyMode("MoveBackwardWord") },
      { key = "e", mods = "NONE", action = act.CopyMode("MoveForwardWordEnd") },
      { key = "0", mods = "NONE", action = act.CopyMode("MoveToStartOfLine") },
      { key = "$", mods = "NONE", action = act.CopyMode("MoveToEndOfLineContent") },
      { key = "^", mods = "NONE", action = act.CopyMode("MoveToStartOfLineContent") },
      { key = "f", mods = "NONE", action = act.CopyMode({ JumpForward = { prev_char = false } }) },
      { key = "t", mods = "NONE", action = act.CopyMode({ JumpForward = { prev_char = true } }) },
      { key = "F", mods = "NONE", action = act.CopyMode({ JumpBackward = { prev_char = false } }) },
      { key = "T", mods = "NONE", action = act.CopyMode({ JumpBackward = { prev_char = true } }) },
      { key = "g", mods = "NONE", action = act.CopyMode("MoveToScrollbackTop") },
      { key = "G", mods = "NONE", action = act.CopyMode("MoveToScrollbackBottom") },
      { key = "H", mods = "NONE", action = act.CopyMode("MoveToViewportTop") },
      { key = "L", mods = "NONE", action = act.CopyMode("MoveToViewportBottom") },
      { key = "M", mods = "NONE", action = act.CopyMode("MoveToViewportMiddle") },
      { key = "b", mods = "CTRL", action = act.CopyMode("PageUp") },
      { key = "f", mods = "CTRL", action = act.CopyMode("PageDown") },
      { key = "d", mods = "CTRL", action = act.CopyMode({ MoveByPage = 0.5 }) },
      { key = "u", mods = "CTRL", action = act.CopyMode({ MoveByPage = -0.5 }) },
      { key = "v", mods = "NONE", action = act.CopyMode({ SetSelectionMode = "Cell" }) },
      { key = "V", mods = "NONE", action = act.CopyMode({ SetSelectionMode = "Line" }) },
      { key = "v", mods = "CTRL", action = act.CopyMode({ SetSelectionMode = "Block" }) },
      { key = "o", mods = "NONE", action = act.CopyMode("MoveToSelectionOtherEnd") },
      { key = "y", mods = "NONE", action = act.Multiple({
        { CopyTo = "ClipboardAndPrimarySelection" },
        { CopyMode = "Close" }
      }) },
      { key = "Escape", mods = "NONE", action = act.CopyMode("Close") },
      { key = "c", mods = "CTRL", action = act.CopyMode("Close") },
      { key = "q", mods = "NONE", action = act.CopyMode("Close") },
    },
  },
}
```

### キーバインドの設計思想

**1. tmux互換のキーバインド**

tmuxユーザーが違和感なく移行できるよう、以下の操作をtmuxと同様に設定しています。

| 操作              | tmux          | WezTerm       |
| ----------------- | ------------- | ------------- |
| 垂直分割          | `Ctrl-b %`    | `Ctrl-q d`    |
| 水平分割          | `Ctrl-b "`    | `Ctrl-q r`    |
| Pane移動          | `Ctrl-b hjkl` | `Ctrl-q hjkl` |
| Paneズーム        | `Ctrl-b z`    | `Ctrl-q z`    |
| Workspace切り替え | `Ctrl-b s`    | `Ctrl-q w`    |

**2. macOSネイティブキーバインド**

`Cmd`キーを使ったmacOS標準操作も残しています。

```lua
{ key = "t", mods = "SUPER", action = act.SpawnTab("CurrentPaneDomain") },
{ key = "w", mods = "SUPER", action = act.CloseCurrentTab({ confirm = true }) },
{ key = "c", mods = "SUPER", action = act.CopyTo("Clipboard") },
{ key = "v", mods = "SUPER", action = act.PasteFrom("Clipboard") },
```

**3. キーテーブル（モーダル操作）**

`resize_pane`と`activate_pane`のキーテーブルで、連続的な操作を簡潔に実行できます。

```lua
-- Ctrl-q s でリサイズモード起動
{ key = "s", mods = "LEADER", action = act.ActivateKeyTable({ name = "resize_pane", one_shot = false }) },
```

リサイズモード中は`hjkl`だけでPaneサイズを連続調整でき、`Enter`で抜けます。

# 外観カスタマイズ詳細

## タブの矢印型デザイン

Nerd Fontsの三角形記号を使ったタブデザインは、以下の仕組みで実現しています。

```lua
local SOLID_LEFT_ARROW = wezterm.nerdfonts.ple_lower_right_triangle
local SOLID_RIGHT_ARROW = wezterm.nerdfonts.ple_upper_left_triangle

wezterm.on("format-tab-title", function(tab, tabs, panes, config, hover, max_width)
  local background = "#5c6d74"  -- 非アクティブタブの背景色
  local foreground = "#FFFFFF"  -- テキスト色
  local edge_background = "none"

  -- アクティブタブの色変更
  if tab.is_active then
    background = "#ae8b2d"
    foreground = "#FFFFFF"
  end

  local edge_foreground = background
  local title = "   " .. wezterm.truncate_right(tab.active_pane.title, max_width - 1) .. "   "

  return {
    { Background = { Color = edge_background } },
    { Foreground = { Color = edge_foreground } },
    { Text = SOLID_LEFT_ARROW },  -- 左の三角形
    { Background = { Color = background } },
    { Foreground = { Color = foreground } },
    { Text = title },  -- タブタイトル
    { Background = { Color = edge_background } },
    { Foreground = { Color = edge_foreground } },
    { Text = SOLID_RIGHT_ARROW },  -- 右の三角形
  }
end)
```

この設定により、各タブが矢印型で表示されます。

## 透過設定の調整

背景透過度とブラー効果は環境に応じて調整可能です。

```lua
-- 透過度: 0.0（完全透明）～ 1.0（不透明）
config.window_background_opacity = 0.3

-- macOSのブラー効果: 0～100
config.macos_window_background_blur = 20
```

デスクトップ壁紙が明るい場合は透過度を下げ、暗い場合は上げると視認性が向上します。

# Workspace管理（tmuxのセッション代替）

WezTermの**Workspace**は、tmuxのセッションに相当する概念です。プロジェクトごとにWorkspaceを作成し、Tab/Paneを管理できます。

## Workspace操作

| 操作              | キーバインド     | 説明                    |
| ----------------- | ---------------- | ----------------------- |
| Workspace切り替え | `Ctrl-q w`       | 既存Workspaceを選択     |
| Workspace新規作成 | `Ctrl-q Shift-W` | 名前を入力して作成      |
| Workspace名前変更 | `Ctrl-q $`       | 現在のWorkspace名を変更 |

## 使用例

```bash
# プロジェクトごとにWorkspaceを作成
Ctrl-q Shift-W -> "project-a"
Ctrl-q Shift-W -> "project-b"

# プロジェクト切り替え
Ctrl-q w -> "project-a"を選択
```

Workspaceは自動的に保存され、WezTerm再起動後も復元されます。tmuxのセッション管理ツール（tmuxinator等）が不要になります。

# コピーモード（Vimライク操作）

`Ctrl-q [`でコピーモードに入ると、Vimキーバインドでバッファを操作できます。

## 主な操作

| キー       | 動作              |
| ---------- | ----------------- |
| `h/j/k/l`  | カーソル移動      |
| `w/b/e`    | 単語移動          |
| `0/$`      | 行頭/行末         |
| `gg/G`     | バッファ先頭/末尾 |
| `f{char}`  | 文字検索（前方）  |
| `t{char}`  | 文字の前まで移動  |
| `v`        | 文字単位選択      |
| `V`        | 行単位選択        |
| `Ctrl-v`   | 矩形選択          |
| `y`        | ヤンク（コピー）  |
| `Escape/q` | コピーモード終了  |

tmux copy-modeと同等の操作感が得られるため、tmuxなしでもストレスなく作業できます。

# iTerm2との比較

実際に移行して感じたメリットとデメリットを整理します。

## メリット

| 項目                   | iTerm2                                | WezTerm                         |
| ---------------------- | ------------------------------------- | ------------------------------- |
| 設定の可搬性           | プロファイルのエクスポート/インポート | `.lua`ファイルをgit管理         |
| 設定の柔軟性           | GUI操作中心                           | Luaコードで条件分岐・関数化可能 |
| マルチプレクサ         | tmux必須                              | Workspace/Tab/Pane内蔵          |
| クロスプラットフォーム | macOSのみ                             | macOS/Linux/Windows対応         |
| 起動速度               | 高速                                  | さらに高速（Rust製）            |
| Nerd Fonts対応         | 良好                                  | 合字表示が美しい                |

**特に良かった点:**

1. **設定のバージョン管理**: dotfilesに`.lua`を含めるだけで、新しいマシンでも即座に環境再現
2. **tmux不要**: Workspace/Pane管理がネイティブで、レイテンシがない
3. **Luaの柔軟性**: 条件分岐でOS別設定、関数で設定を再利用可能

```lua
-- OS別フォント設定の例
if wezterm.target_triple == "x86_64-pc-windows-msvc" then
  config.font = wezterm.font("Consolas")
else
  config.font = wezterm.font("JetBrains Mono")
end
```

## デメリット

| 項目         | 課題                                 |
| ------------ | ------------------------------------ |
| エコシステム | iTerm2に比べて情報が少ない           |
| プラグイン   | iTerm2のような豊富なプラグインはない |
| GUI設定      | 全てコードで記述（GUIなし）          |
| 学習コスト   | Lua習得が必要                        |

GUIに慣れている場合、最初は設定ファイルの編集に戸惑うかもしれません。ただし、公式ドキュメントが充実しており、設定例も豊富なため、数日で慣れました。

# 参考にした記事・リソース

この記事の執筆にあたり、以下を参考にしました。

- [WezTerm公式ドキュメント](https://wezfurlong.org/wezterm/)
- [モテるターミナルにカスタマイズしよう（WezTerm）](https://zenn.dev/mozumasu/articles/mozumasu-wezterm-customization) by @mozumasu
- [WezTerm Config Files](https://wezfurlong.org/wezterm/config/files.html)
- [WezTerm Key Tables](https://wezfurlong.org/wezterm/config/key-tables.html)

mozumasuさんの記事はタブカスタマイズの実装で特に参考になりました。

# まとめ

iTerm2からWezTermへの移行は、設定の可搬性とtmux不要の快適さを求めるエンジニアにとって有力な選択肢です。

**移行をおすすめする人:**

- dotfilesで環境を完全管理したい
- tmuxのオーバーヘッドを減らしたい
- クロスプラットフォームで同一設定を使いたい
- コードで設定を記述したい

**移行を控えた方が良い人:**

- GUI設定を好む
- iTerm2のエコシステムに深く依存している
- 学習コストをかけたくない

この記事で公開した設定ファイルは、そのままコピーして使えます。WezTermの導入を検討している方の参考になれば幸いです。

## 次のステップ

- [WezTermのSSHドメイン機能](https://wezfurlong.org/wezterm/config/lua/SshDomain.html)でリモートサーバー管理
- [カラースキームのカスタマイズ](https://wezfurlong.org/wezterm/config/appearance.html)
- [起動時のレイアウト自動復元](https://wezfurlong.org/wezterm/config/lua/wezterm/on.html)

WezTermの可能性はまだまだ広がります。
