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
  ```json
  {
    "type": "create_object",
    "params": {
      "type": "CUBE",
      "location": [0, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1],
      "name": "MyCube"
    }
  }
  ```

- **レスポンス**は`status`と`result`または`message`を持つJSONオブジェクトです
  ```json
  {
    "status": "success",
    "result": {
      "name": "MyCube",
      "type": "MESH",
      "location": [0, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1]
    }
  }
  ```

### アーキテクチャ

#### Blenderアドオン (addon.py)

Blenderアドオンは以下の主要なコンポーネントで構成されています：

1. **BlenderMCPServer**: TCPソケットサーバーを実装し、Claudeからのコマンドを受信して処理します
   - ソケット接続の管理: `start()`, `stop()`, `_process_server()`メソッドでソケット接続のライフサイクルを管理
   - JSONコマンドの解析: 受信したバイトデータをJSONオブジェクトに変換し、コマンドタイプとパラメータを抽出
   - コマンド実行とレスポンス送信: `execute_command()`メソッドでコマンドを実行し、結果をJSONとしてクライアントに返送

   ```python
   # サーバーの初期化
   def __init__(self, host='localhost', port=9876):
       self.host = host
       self.port = port
       self.running = False
       self.socket = None
       self.client = None
       self.command_queue = []
       self.buffer = b''  # 不完全なデータ用のバッファ
   ```

   ```python
   # コマンド実行の中核部分
   def _execute_command_internal(self, command):
       cmd_type = command.get("type")
       params = command.get("params", {})
       
       # ハンドラーの取得と実行
       handler = handlers.get(cmd_type)
       if handler:
           result = handler(**params)
           return {"status": "success", "result": result}
       else:
           return {"status": "error", "message": f"Unknown command type: {cmd_type}"}
   ```

2. **コマンドハンドラー**: 様々なBlender操作を実行するメソッド
   - `get_scene_info`: シーン情報の取得（オブジェクト名、タイプ、位置など）
     ```python
     def get_scene_info(self):
         scene_info = {
             "name": bpy.context.scene.name,
             "object_count": len(bpy.context.scene.objects),
             "objects": [],
             "materials_count": len(bpy.data.materials),
         }
         
         # オブジェクト情報の収集
         for i, obj in enumerate(bpy.context.scene.objects):
             if i >= 10:
                 break
             obj_info = {
                 "name": obj.name,
                 "type": obj.type,
                 "location": [round(float(obj.location.x), 2), 
                             round(float(obj.location.y), 2), 
                             round(float(obj.location.z), 2)],
             }
             scene_info["objects"].append(obj_info)
         
         return scene_info
     ```
   
   - `create_object`: 新しいオブジェクトの作成（立方体、球体、円柱など）
     ```python
     def create_object(self, type="CUBE", name=None, location=(0, 0, 0), rotation=(0, 0, 0), scale=(1, 1, 1), ...):
         # オブジェクトタイプに基づいて作成
         if type == "CUBE":
             bpy.ops.mesh.primitive_cube_add(location=location, rotation=rotation, scale=scale)
         elif type == "SPHERE":
             bpy.ops.mesh.primitive_uv_sphere_add(location=location, rotation=rotation, scale=scale)
         # ... その他のオブジェクトタイプ
         
         # 作成されたオブジェクトの取得と名前設定
         obj = bpy.context.view_layer.objects.active
         if name:
             obj.name = name
         
         # 結果の返却
         result = {
             "name": obj.name,
             "type": obj.type,
             "location": [obj.location.x, obj.location.y, obj.location.z],
             "rotation": [obj.rotation_euler.x, obj.rotation_euler.y, obj.rotation_euler.z],
             "scale": [obj.scale.x, obj.scale.y, obj.scale.z],
         }
         return result
     ```
   
   - `modify_object`: 既存オブジェクトの修正（位置、回転、スケールなど）
   - `delete_object`: オブジェクトの削除
   - `get_object_info`: オブジェクト情報の取得（詳細なプロパティ、マテリアル、メッシュデータなど）
   - `execute_code`: 任意のPythonコードの実行（強力だが潜在的に危険）
   - `set_material`: マテリアルの設定（色、テクスチャなど）
     ```python
     def set_material(self, object_name, material_name=None, create_if_missing=True, color=None):
         # オブジェクトの取得
         obj = bpy.data.objects.get(object_name)
         
         # マテリアルの作成または取得
         if material_name:
             mat = bpy.data.materials.get(material_name)
             if not mat and create_if_missing:
                 mat = bpy.data.materials.new(name=material_name)
         else:
             mat_name = f"{object_name}_material"
             mat = bpy.data.materials.get(mat_name)
             if not mat:
                 mat = bpy.data.materials.new(name=mat_name)
             material_name = mat_name
         
         # マテリアルノードの設定
         if mat:
             if not mat.use_nodes:
                 mat.use_nodes = True
             
             # Principled BSDFノードの取得または作成
             principled = mat.node_tree.nodes.get('Principled BSDF')
             if not principled:
                 principled = mat.node_tree.nodes.new('ShaderNodeBsdfPrincipled')
                 # ... ノードの接続
             
             # 色の設定
             if color and len(color) >= 3:
                 principled.inputs['Base Color'].default_value = (
                     color[0], color[1], color[2],
                     1.0 if len(color) < 4 else color[3]
                 )
         
         # マテリアルをオブジェクトに割り当て
         if mat:
             if not obj.data.materials:
                 obj.data.materials.append(mat)
             else:
                 obj.data.materials[0] = mat
         
         return {
             "status": "success",
             "object": object_name,
             "material": mat.name,
             "color": color if color else None
         }
     ```

3. **Poly Haven統合**: 高品質な3Dアセット、テクスチャ、HDRIの検索とダウンロード
   - `get_polyhaven_categories`: アセットカテゴリの取得
   - `search_polyhaven_assets`: アセットの検索
   - `download_polyhaven_asset`: アセットのダウンロードとインポート
   - `set_texture`: テクスチャの適用

   ```python
   def download_polyhaven_asset(self, asset_id, asset_type, resolution="1k", file_format=None):
       # アセット情報の取得
       files_response = requests.get(f"https://api.polyhaven.com/files/{asset_id}")
       files_data = files_response.json()
       
       # アセットタイプに応じた処理
       if asset_type == "hdris":
           # HDRIのダウンロードと適用
           file_url = files_data["hdri"][resolution][file_format]["url"]
           # ... HDRIのダウンロードと適用
       elif asset_type == "textures":
           # テクスチャのダウンロードとマテリアル作成
           downloaded_maps = {}
           for map_type in files_data:
               if map_type not in ["blend", "gltf"]:
                   file_url = files_data[map_type][resolution][file_format]["url"]
                   # ... テクスチャのダウンロードと処理
           # ... マテリアルの作成と設定
       elif asset_type == "models":
           # モデルのダウンロードとインポート
           file_url = files_data[file_format][resolution][file_format]["url"]
           # ... モデルのダウンロードとインポート
   ```

4. **Hyper3D Rodin統合**: AIによる3Dモデル生成
   - `create_rodin_job`: モデル生成ジョブの作成
   - `poll_rodin_job_status`: ジョブステータスの確認
   - `import_generated_asset`: 生成されたアセットのインポート

   ```python
   def create_rodin_job_main_site(self, text_prompt=None, images=None, bbox_condition=None):
       # Rodin APIの呼び出し
       files = [
           *[("images", (f"{i:04d}{img_suffix}", img)) for i, (img_suffix, img) in enumerate(images or [])],
           ("tier", (None, "Sketch")),
           ("mesh_mode", (None, "Raw")),
       ]
       if text_prompt:
           files.append(("prompt", (None, text_prompt)))
       if bbox_condition:
           files.append(("bbox_condition", (None, json.dumps(bbox_condition))))
       
       # APIリクエスト
       response = requests.post(
           "https://hyperhuman.deemos.com/api/v2/rodin",
           headers={"Authorization": f"Bearer {bpy.context.scene.blendermcp_hyper3d_api_key}"},
           files=files
       )
       return response.json()
   ```

5. **Blender UI**: アドオンの設定と制御のためのユーザーインターフェース
   - `BLENDERMCP_PT_Panel`: サイドバーのパネル
   - `BLENDERMCP_OT_StartServer`: サーバー開始ボタン
   - `BLENDERMCP_OT_StopServer`: サーバー停止ボタン

   ```python
   class BLENDERMCP_PT_Panel(bpy.types.Panel):
       bl_label = "Blender MCP"
       bl_idname = "BLENDERMCP_PT_Panel"
       bl_space_type = 'VIEW_3D'
       bl_region_type = 'UI'
       bl_category = 'BlenderMCP'
       
       def draw(self, context):
           layout = self.layout
           scene = context.scene
           
           # UI要素の描画
           layout.prop(scene, "blendermcp_port")
           layout.prop(scene, "blendermcp_use_polyhaven", text="Use assets from Poly Haven")
           layout.prop(scene, "blendermcp_use_hyper3d", text="Use Hyper3D Rodin 3D model generation")
           
           # Hyper3D設定
           if scene.blendermcp_use_hyper3d:
               layout.prop(scene, "blendermcp_hyper3d_mode", text="Rodin Mode")
               layout.prop(scene, "blendermcp_hyper3d_api_key", text="API Key")
           
           # サーバー制御ボタン
           if not scene.blendermcp_server_running:
               layout.operator("blendermcp.start_server", text="Start MCP Server")
           else:
               layout.operator("blendermcp.stop_server", text="Stop MCP Server")
               layout.label(text=f"Running on port {scene.blendermcp_port}")
   ```

#### MCPサーバー (server.py)

MCPサーバーは以下のコンポーネントで構成されています：

1. **BlenderConnection**: Blenderアドオンとの通信を管理するクラス
   - `connect()`: Blenderアドオンへの接続
   - `disconnect()`: 接続の切断
   - `send_command()`: コマンドの送信とレスポンスの受信
   - `receive_full_response()`: 完全なレスポンスの受信（複数のチャンクを処理）

   ```python
   @dataclass
   class BlenderConnection:
       host: str
       port: int
       sock: socket.socket = None
       
       def connect(self) -> bool:
           """Blenderアドオンのソケットサーバーに接続"""
           if self.sock:
               return True
           
           try:
               self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
               self.sock.connect((self.host, self.port))
               logger.info(f"Connected to Blender at {self.host}:{self.port}")
               return True
           except Exception as e:
               logger.error(f"Failed to connect to Blender: {str(e)}")
               self.sock = None
               return False
   ```

   ```python
   def send_command(self, command_type: str, params: Dict[str, Any] = None) -> Dict[str, Any]:
       """Blenderにコマンドを送信し、レスポンスを返す"""
       if not self.sock and not self.connect():
           raise ConnectionError("Not connected to Blender")
       
       command = {
           "type": command_type,
           "params": params or {}
       }
       
       try:
           # コマンドの送信
           self.sock.sendall(json.dumps(command).encode('utf-8'))
           
           # レスポンスの受信
           response_data = self.receive_full_response(self.sock)
           response = json.loads(response_data.decode('utf-8'))
           
           if response.get("status") == "error":
               raise Exception(response.get("message", "Unknown error from Blender"))
           
           return response.get("result", {})
       except Exception as e:
           # エラー処理
           self.sock = None
           raise Exception(f"Communication error with Blender: {str(e)}")
   ```

2. **FastMCP**: Model Context Protocolを実装するサーバー
   - サーバーの初期化と設定
   - ツールとリソースの登録
   - サーバーのライフサイクル管理

   ```python
   # MCPサーバーの作成
   mcp = FastMCP(
       "BlenderMCP",
       description="Blender integration through the Model Context Protocol",
       lifespan=server_lifespan
   )
   
   @asynccontextmanager
   async def server_lifespan(server: FastMCP) -> AsyncIterator[Dict[str, Any]]:
       """サーバーの起動と終了のライフサイクルを管理"""
       try:
           # 起動時の処理
           logger.info("BlenderMCP server starting up")
           
           # Blenderへの接続確認
           try:
               blender = get_blender_connection()
               logger.info("Successfully connected to Blender on startup")
           except Exception as e:
               logger.warning(f"Could not connect to Blender on startup: {str(e)}")
           
           yield {}
       finally:
           # 終了時の処理
           global _blender_connection
           if _blender_connection:
               logger.info("Disconnecting from Blender on shutdown")
               _blender_connection.disconnect()
               _blender_connection = None
           logger.info("BlenderMCP server shut down")
   ```

3. **ツール実装**: Claudeが使用できる様々なツール
   - `@mcp.tool()` デコレータを使用して各ツールを定義
   - 各ツールはBlenderアドオンの対応するコマンドを呼び出す

   ```python
   @mcp.tool()
   def get_scene_info(ctx: Context) -> str:
       """現在のBlenderシーンに関する詳細情報を取得"""
       try:
           blender = get_blender_connection()
           result = blender.send_command("get_scene_info")
           
           # Blenderから送られてきたものをJSON表現として返す
           return json.dumps(result, indent=2)
       except Exception as e:
           logger.error(f"Error getting scene info from Blender: {str(e)}")
           return f"Error getting scene info: {str(e)}"
   ```

   ```python
   @mcp.tool()
   def create_object(
       ctx: Context,
       type: str = "CUBE",
       name: str = None,
       location: List[float] = None,
       rotation: List[float] = None,
       scale: List[float] = None,
       # トーラス特有のパラメータ
       align: str = "WORLD",
       major_segments: int = 48,
       minor_segments: int = 12,
       mode: str = "MAJOR_MINOR",
       major_radius: float = 1.0,
       minor_radius: float = 0.25,
       abso_major_rad: float = 1.25,
       abso_minor_rad: float = 0.75,
       generate_uvs: bool = True
   ) -> str:
       """Blenderシーンに新しいオブジェクトを作成"""
       try:
           # グローバル接続の取得
           blender = get_blender_connection()
           
           # 欠落パラメータのデフォルト値設定
           loc = location or [0, 0, 0]
           rot = rotation or [0, 0, 0]
           sc = scale or [1, 1, 1]
           
           params = {
               "type": type,
               "location": loc,
               "rotation": rot,
           }
           
           if name:
               params["name"] = name
   
           if type == "TORUS":
               # トーラスの場合、スケールは使用されない
               params.update({
                   "align": align,
                   "major_segments": major_segments,
                   "minor_segments": minor_segments,
                   "mode": mode,
                   "major_radius": major_radius,
                   "minor_radius": minor_radius,
                   "abso_major_rad": abso_major_rad,
                   "abso_minor_rad": abso_minor_rad,
                   "generate_uvs": generate_uvs
               })
               result = blender.send_command("create_object", params)
               return f"Created {type} object: {result['name']}"
           else:
               # トーラス以外のオブジェクトの場合、スケールを含める
               params["scale"] = sc
               result = blender.send_command("create_object", params)
               return f"Created {type} object: {result['name']}"
       except Exception as e:
           logger.error(f"Error creating object: {str(e)}")
           return f"Error creating object: {str(e)}"
   ```

4. **アセット生成戦略**: Blenderでのアセット作成の推奨戦略を定義
   ```python
   @mcp.prompt()
   def asset_creation_strategy() -> str:
       """Blenderでのアセット作成の推奨戦略を定義"""
       return """When creating 3D content in Blender, always start by checking if integrations are available:
       
       0. Before anything, always check the scene from get_scene_info()
       1. First use the following tools to verify if the following integrations are enabled:
           1. PolyHaven
               Use get_polyhaven_status() to verify its status
               If PolyHaven is enabled:
               - For objects/models: Use download_polyhaven_asset() with asset_type="models"
               - For materials/textures: Use download_polyhaven_asset() with asset_type="textures"
               - For environment lighting: Use download_polyhaven_asset() with asset_type="hdris"
           2. Hyper3D(Rodin)
               ...
       """
   ```

## 動作の流れ

1. **初期化**:
   - Blenderアドオンがインストールされ、有効化される
   - ユーザーがBlenderのサイドバーから「Connect to Claude」をクリック
   - BlenderMCPServerがTCPソケットを開き、接続を待機
   ```python
   # Blenderアドオンの起動
   def start(self):
       self.running = True
       self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
       self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
       
       try:
           self.socket.bind((self.host, self.port))
           self.socket.listen(1)
           self.socket.setblocking(False)
           # タイマーの登録
           bpy.app.timers.register(self._process_server, persistent=True)
           print(f"BlenderMCP server started on {self.host}:{self.port}")
       except Exception as e:
           print(f"Failed to start server: {str(e)}")
           self.stop()
   ```

2. **MCPサーバー起動**:
   - `uvx blender-mcp`コマンドでMCPサーバーが起動
   - MCPサーバーがClaudeに登録され、利用可能なツールとリソースを公開
   ```python
   # メイン実行
   def main():
       """MCPサーバーを実行"""
       mcp.run()
   
   if __name__ == "__main__":
       main()
   ```

3. **コマンド実行**:
   - ユーザーがClaudeに指示を出す（例：「赤い立方体を作成して」）
   - Claudeが適切なMCPツールを選択（例：`create_object`と`set_material`）
   - MCPサーバーがBlenderアドオンにコマンドを送信
   - Blenderアドオンがコマンドを実行し、結果を返す
   - MCPサーバーが結果をClaudeに返し、Claudeがユーザーに結果を表示

   ```
   ユーザー → Claude → MCPサーバー → Blenderアドオン → Blender
                ↑                 ↑
                └─────────────────┘
                     結果の返送
   ```

4. **統合機能の詳細**:
   - **Poly Haven**: 
     - APIを通じてアセットカテゴリを取得
     - アセットを検索
     - 選択したアセットをダウンロードしてBlenderにインポート
     - HDRIの場合は環境マップとして設定
     - テクスチャの場合はマテリアルを作成
     - モデルの場合はシーンにインポート
   
   - **Hyper3D Rodin**: 
     - テキスト説明や参照画像からモデル生成ジョブを作成
     - ジョブのステータスを定期的に確認
     - 生成が完了したらモデルをダウンロードしてBlenderにインポート
     - モデルの位置、スケール、回転を調整

## コマンド例と実装詳細

### シーン情報の取得

```python
# MCPサーバー側
@mcp.tool()
def get_scene_info(ctx: Context) -> str:
    blender = get_blender_connection()
    result = blender.send_command("get_scene_info")
    return json.dumps(result, indent=2)

# Blenderアドオン側
def get_scene_info(self):
    scene_info = {
        "name": bpy.context.scene.name,
        "object_count": len(bpy.context.scene.objects),
        "objects": [],
        "materials_count": len(bpy.data.materials),
    }
    
    # オブジェクト情報の収集
    for i, obj in enumerate(bpy.context.scene.objects):
        if i >= 10:
            break
        obj_info = {
            "name": obj.name,
            "type": obj.type,
            "location": [round(float(obj.location.x), 2), 
                        round(float(obj.location.y), 2), 
                        round(float(obj.location.z), 2)],
        }
        scene_info["objects"].append(obj_info)
    
    return scene_info
```

### オブジェクトの作成

```python
# MCPサーバー側
@mcp.tool()
def create_object(ctx: Context, type: str = "CUBE", name: str = None, ...) -> str:
    blender = get_blender_connection()
    params = {"type": type, "location": location or [0, 0, 0], ...}
    result = blender.send_command("create_object", params)
    return f"Created {type} object: {result['name']}"

# Blenderアドオン側
def create_object(self, type="CUBE", name=None, ...):
    # オブジェクトタイプに基づいて作成
    if type == "CUBE":
        bpy.ops.mesh.primitive_cube_add(location=location, rotation=rotation, scale=scale)
    elif type == "SPHERE":
        bpy.ops.mesh.primitive_uv_sphere_add(location=location, rotation=rotation, scale=scale)
    # ... その他のオブジェクトタイプ
    
    # 作成されたオブジェクトの取得と名前設定
    obj = bpy.context.view_layer.objects.active
    if name:
        obj.name = name
    
    # 結果の返却
    result = {
        "name": obj.name,
        "type": obj.type,
        "location": [obj.location.x, obj.location.y, obj.location.z],
        # ...
    }
    return result
```

### マテリアルの設定

```python
# MCPサーバー側
@mcp.tool()
def set_material(ctx: Context, object_name: str, material_name: str = None, color: List[float] = None) -> str:
    blender = get_blender_connection()
    params = {"object_name": object_name}
    if material_name:
        params["material_name"] = material_name
    if color:
        params["color"] = color
    result = blender.send_command("set_material", params)
    return f"Applied material to {object_name}: {result.get('material_name', 'unknown')}"

# Blenderアドオン側
def set_material(self, object_name, material_name=None, create_if_missing=True, color=None):
    # オブジェクトの取得
    obj = bpy.data.objects.get(object_name)
    
    # マテリアルの作成または取得
    if material_name:
        mat = bpy.data.materials.get(material_name)
        if not mat and create_if_missing:
            mat = bpy.data.materials.new(name=material_name)
    else:
        mat_name = f"{object_name}_material"
        mat = bpy.data.materials.get(mat_name)
        if not mat:
            mat = bpy.data.materials.new(name=mat_name)
        material_name = mat_name
    
    # マテリアルノードの設定
    if mat:
        if not mat.use_nodes:
            mat.use_nodes = True
        
        # Principled BSDFノードの取得または作成
        principled = mat.node_tree.nodes.get('Principled BSDF')
        # ... ノードの設定
        
        # 色の設定
        if color and len(color) >= 3:
            principled.inputs['Base Color'].default_value = (
                color[0], color[1], color[2],
                1.0 if len(color) < 4 else color[3]
            )
    
    # マテリアルをオブジェクトに割り当て
    if mat:
        if not obj.data.materials:
            obj.data.materials.append(mat)
        else:
            obj.data.materials[0] = mat
    
    return {
        "status": "success",
        "object": object_name,
        "material": mat.name,
        "color": color if color else None
    }
```

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