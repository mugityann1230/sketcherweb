# sketcherweb アーキテクチャ（全体構成・データモデル・API方針）

## 0. 方針（おすすめ）
- フロント：2D作図・編集に集中
- バック：Region検証とGmshでメッシュ生成（安定・最短ルート）
- “編集用(Sketch)” と “解析用(Region)” を分離する

---

## 1. 全体構成（概略）
### Frontend（React + TypeScript）
- Canvas描画（パン/ズーム/スナップ）
- ツール（線分/円/円弧/選択）
- Sketchの状態管理（Undo/Redo）
- Region生成リクエスト or ローカル生成（※MVPではローカル生成でもOK）
- Mesh生成APIを叩いて `.msh` をDL

### Backend（FastAPI）
- `/mesh`：Region受け取り→Gmsh実行→`.msh`返却
- （任意）`/validate`：Region妥当性検証だけ（将来分離したい場合）

---

## 2. データモデル

### 2.1 Sketch（編集用）
- version: string（例 "0.1"）
- units: "mm" | "m"（MVPはmm推奨）
- points: Point[]
- edges: Edge[]（Line / Arc / Circle）

#### Point
- id: string
- x: number
- y: number

#### Line
- id: string
- type: "line"
- a: pointId
- b: pointId

#### Circle（MVPでは“穴用”としてあってもOK）
- id: string
- type: "circle"
- center: pointId
- radius: number

#### Arc（おすすめ：3点式を入力にして内部は center/r/angle に正規化）
- id: string
- type: "arc"
- start: pointId
- end: pointId
- mid: pointId
- （内部表現として center/r/startAngle/endAngle を持っても良い）

#### Tolerance
- eps: number（例 1e-6 * unit）

> 注意：CircleはLoop抽出の都合上、最終的に “エッジ列” として扱える必要がある  
> （Circleは「Arc 360°」扱いに正規化するのが楽）

---

### 2.2 Region（解析用）
- version: string
- unit: string
- outer: Loop（1つ）
- holes: Loop[]（0..n）
- mesh: { globalSize: number, ... }（MVPはglobalSizeのみ）

#### Loop
- edges: OrientedEdge[]（順序付き）
- winding: "CCW" | "CW"（外周/穴の判定にも利用）
- bbox: {minx, miny, maxx, maxy}（任意：高速化用）

#### OrientedEdge
- edgeId: string
- dir: 1 | -1（向き）

---

## 3. 幾何処理パイプライン（推奨）
1) Sketch → 端点接続グラフ構築（epsでマージ）
2) 閉ループ抽出（DFS等）
3) ループ妥当性（未閉/自己交差/重複/極小）
4) outer/holes判定（面積符号・包含関係）
5) Region確定

---

## 4. API設計

### 4.1 POST /mesh（MVP必須）
- Request: Region(JSON)
- Response: `.msh`（v4、Content-Typeは `application/octet-stream` 推奨）

#### エラー（例）
- 400: invalid region（理由メッセージ付き）
- 500: gmsh failure（ログIDなど返す）

### 4.2 （任意）POST /validate
- Request: Sketch or Region
- Response: { ok: boolean, errors: [...] }
- ※MVPはフロント側でvalidateしてもOK。将来の堅牢化用。

---

## 5. Gmsh連携方針（MVP）
- Regionから `.geo` を生成してGmsh CLI実行
- 2Dメッシュ（三角形）を基本
- 出力：`.msh v4` 固定

#### メッシュサイズ
- globalSizeのみ（後で境界近傍を細かく等を追加）

---

## 6. ディレクトリ構成（例）
- frontend/
  - src/
    - canvas/
    - tools/
    - state/
    - core/（型定義・共通）
- backend/
  - app/
    - main.py
    - mesh_service.py
    - gmsh_runner.py
- shared/（任意：共通スキーマを置く場合）
- docs/

---

## 7. テスト方針（特に重要）
- geometry（loop/region）はユニットテスト必須
- 例題（穴あき板/ドーナツ/未閉/自己交差）を固定データとして持つ
- backendは `/mesh` を軽い統合テスト（Gmsh有無でskipできるようにしても良い）

---

## 8. セキュリティ/運用（最低限）
- `/mesh` は入力サイズを制限（点数・辺数・最大文字数など）
- Gmsh実行はタイムアウト必須（無限ループ回避）
- 生成ファイルは一時ディレクトリで管理し、終了時に削除
