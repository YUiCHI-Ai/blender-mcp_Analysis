# BlenderMCP - Blender Model Context Protocol 連携

BlenderMCPは、Model Context Protocol (MCP)を通じてBlenderをClaude AIに接続し、Claudeが直接Blenderと対話して制御できるようにします。この連携により、プロンプトによる3Dモデリング、シーン作成、操作が可能になります。

## リリースノート (1.1.0)

- Poly Haven APIを通じたアセットのサポートを追加
- Hyper3D Rodinを使用した3Dモデルのプロンプト生成をサポート
- 新規ユーザーは、インストールセクションに直接進めます。既存ユーザーは、以下の点を確認してください
- 最新のaddon.pyファイルをダウンロードして古いものと置き換え、Blenderに追加してください
- ClaudeからMCPサーバーを削除して再度追加すれば、準備完了です！

## 機能

- **双方向通信**: ソケットベースのサーバーを通じてClaude AIをBlenderに接続
- **オブジェクト操作**: Blenderで3Dオブジェクトの作成、修正、削除
- **マテリアル制御**: マテリアルと色の適用と修正
- **シーン検査**: 現在のBlenderシーンに関する詳細情報の取得
- **コード実行**: ClaudeからBlenderで任意のPythonコードを実行

## コンポーネント

システムは主に2つのコンポーネントで構成されています：

1. **Blenderアドオン (`addon.py`)**: Blender内にソケットサーバーを作成し、コマンドを受信して実行するBlenderアドオン
2. **MCPサーバー (`src/blender_mcp/server.py`)**: Model Context Protocolを実装し、Blenderアドオンに接続するPythonサーバー

## 技術的な詳細

### 通信プロトコル

システムはTCPソケットを介したシンプルなJSONベースのプロトコルを使用します：

- **コマンド**は`type`と任意の`params`を持つJSONオブジェクトとして送信されます
- **レスポンス**は`status`と`result`または`message`を持つJSONオブジェクトです

### アーキテクチャ

#### Blenderアドオン (addon.py)

Blenderアドオンは以下の主要なコンポーネントで構成されています：

1. **BlenderMCPServer**: TCPソケットサーバーを実装し、Claudeからのコマンドを受信して処理します
   - ソケット接続の管理
   - JSONコマンドの解析
   - コマンド実行とレスポンス送信

2. **コマンドハンドラー**: 様々なBlender操作を実行するメソッド
   - `get_scene_info`: シーン情報の取得
   - `create_object`: 新しいオブジェクトの作成
   - `modify_object`: 既存オブジェクトの修正
   - `delete_object`: オブジェクトの削除
   - `get_object_info`: オブジェクト情報の取得
   - `execute_code`: 任意のPythonコードの実行
   - `set_material`: マテリアルの設定
   - Poly Havenアセット関連の機能
   - Hyper3D Rodin 3Dモデル生成関連の機能

3. **Blender UI**: アドオンの設定と制御のためのユーザーインターフェース
   - ポート設定
   - Poly Havenの有効化/無効化
   - Hyper3D Rodinの有効化/無効化
   - サーバーの開始/停止

#### MCPサーバー (server.py)

MCPサーバーは以下のコンポーネントで構成されています：

1. **BlenderConnection**: Blenderアドオンとの通信を管理するクラス
   - 接続の確立と維持
   - コマンドの送信とレスポンスの受信
   - エラー処理

2. **FastMCP**: Model Context Protocolを実装するサーバー
   - ツールとリソースの登録
   - Claudeとの通信

3. **ツール実装**: Claudeが使用できる様々なツール
   - シーン情報の取得
   - オブジェクトの作成、修正、削除
   - マテリアルの設定
   - Blenderコードの実行
   - Poly Havenアセットの検索とダウンロード
   - Hyper3D Rodinによる3Dモデル生成

## 動作の流れ

1. **初期化**:
   - Blenderアドオンがインストールされ、有効化される
   - ユーザーがBlenderのサイドバーから「Connect to Claude」をクリック
   - BlenderMCPServerがTCPソケットを開き、接続を待機

2. **MCPサーバー起動**:
   - `uvx blender-mcp`コマンドでMCPサーバーが起動
   - MCPサーバーがClaudeに登録され、利用可能なツールとリソースを公開

3. **コマンド実行**:
   - ユーザーがClaudeに指示を出す（例：「赤い立方体を作成して」）
   - Claudeが適切なMCPツールを選択（例：`create_object`と`set_material`）
   - MCPサーバーがBlenderアドオンにコマンドを送信
   - Blenderアドオンがコマンドを実行し、結果を返す
   - MCPサーバーが結果をClaudeに返し、Claudeがユーザーに結果を表示

4. **統合機能**:
   - **Poly Haven**: 高品質な3Dアセット、テクスチャ、HDRIをダウンロードして使用
   - **Hyper3D Rodin**: テキスト説明や参照画像から3Dモデルを生成

## インストール

### 前提条件

- Blender 3.0以降
- Python 3.10以降
- uvパッケージマネージャー

**Macの場合、uvを以下のようにインストール**
```bash
brew install uv
```

**Windowsの場合**
```bash
powershell -c "irm https://astral.sh/uv/install.ps1 | iex" 
```
その後
```bash
set Path=C:\Users\nntra\.local\bin;%Path%
```

その他のインストール手順は公式ウェブサイトにあります: [uvのインストール](https://docs.astral.sh/uv/getting-started/installation/)

**⚠️ uvをインストールする前に進まないでください**

### Claude for Desktop 統合

[セットアップ手順ビデオを見る](https://www.youtube.com/watch?v=neoK_WMq92g) (uvがすでにインストールされていると仮定)

Claude > 設定 > 開発者 > 設定を編集 > claude_desktop_config.json に以下を含めます:

```json
{
    "mcpServers": {
        "blender": {
            "command": "uvx",
            "args": [
                "blender-mcp"
            ]
        }
    }
}
```

### Cursor 統合

uvxを通じてblender-mcpを永続的にインストールせずに実行します。Cursor設定 > MCPに移動し、これをコマンドとして貼り付けます。

```bash
uvx blender-mcp
```

[Cursorセットアップビデオ](https://www.youtube.com/watch?v=wgWsJshecac)

**⚠️ MCPサーバーのインスタンスは1つだけ実行してください（CursorまたはClaude Desktop、両方ではない）**

### Blenderアドオンのインストール

1. このリポジトリから`addon.py`ファイルをダウンロード
2. Blenderを開く
3. 編集 > 設定 > アドオン に移動
4. 「インストール...」をクリックし、`addon.py`ファイルを選択
5. 「Interface: Blender MCP」の横にあるボックスをチェックしてアドオンを有効化

## 使用方法

### 接続の開始
![サイドバーのBlenderMCP](assets/addon-instructions.png)

1. Blenderで、3Dビューのサイドバーに移動（表示されていない場合はNキーを押す）
2. 「BlenderMCP」タブを見つける
3. Poly Haven APIからアセットを使用したい場合は「Use assets from Poly Haven」チェックボックスをオンにする（オプション）
4. 「Connect to Claude」をクリック
5. ターミナルでMCPサーバーが実行されていることを確認

### Claudeでの使用

設定ファイルがClaudeに設定され、アドオンがBlenderで実行されていれば、Blender MCPのツールを示すハンマーアイコンが表示されます。

![サイドバーのBlenderMCP](assets/hammer-icon.png)

#### 機能

- シーンとオブジェクト情報の取得
- 形状の作成、削除、修正
- オブジェクトのマテリアルの適用または作成
- Blenderで任意のPythonコードを実行
- [Poly Haven](https://polyhaven.com/)を通じて適切なモデル、アセット、HDRIをダウンロード
- [Hyper3D Rodin](https://hyper3d.ai/)を通じたAI生成3Dモデル

### コマンド例

Claudeに依頼できることの例：

- 「ダンジョンの中に、金の壺を守るドラゴンがいる低ポリシーンを作成して」[デモ](https://www.youtube.com/watch?v=DqgKuLYUv00)
- 「HDRI、テクスチャ、岩や植物などのモデルを使って、Poly Havenからビーチの雰囲気を作成して」[デモ](https://www.youtube.com/watch?v=I29rn92gkC4)
- 参照画像を与えて、それからBlenderシーンを作成する[デモ](https://www.youtube.com/watch?v=FDRb03XPiRo)
- 「Hyper3Dを通じて庭のノームの3Dモデルを生成して」
- 「現在のシーンに関する情報を取得して、それからthreejsのスケッチを作成して」[デモ](https://www.youtube.com/watch?v=jxbNI5L7AH8)
- 「この車を赤くメタリックにして」
- 「球体を作成して、立方体の上に配置して」
- 「照明をスタジオのようにして」
- 「カメラをシーンに向けて、アイソメトリックにして」

## トラブルシューティング

- **接続の問題**: Blenderアドオンサーバーが実行されていて、MCPサーバーがClaudeに設定されていることを確認してください。ターミナルでuvxコマンドを実行しないでください。最初のコマンドが通らないこともありますが、その後は機能し始めます。
- **タイムアウトエラー**: リクエストを簡素化するか、小さなステップに分けてみてください。
- **Poly Haven統合**: Claudeの動作が時々不安定になることがあります。
- **再起動してみましたか？**: 接続エラーが続く場合は、ClaudeとBlenderサーバーの両方を再起動してみてください。

## 制限事項とセキュリティに関する考慮事項

- `execute_blender_code`ツールはBlenderで任意のPythonコードを実行できるため、強力ですが潜在的に危険です。本番環境では注意して使用してください。常に作業を保存してから使用してください。
- Poly Havenはモデル、テクスチャ、HDRI画像のダウンロードが必要です。使用したくない場合は、Blenderのチェックボックスでオフにしてください。
- 複雑な操作は、より小さなステップに分解する必要があるかもしれません。

## 免責事項

これはサードパーティの統合であり、Blenderによって作成されたものではありません。