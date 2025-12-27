# sketcherweb 開発手順（起動方法・コマンド集）

このプロジェクトは frontend（Web UI）と backend（メッシュ生成API）で構成する。

---

## 0. 前提（必要なもの）
### 必須
- Node.js（推奨：20以上）
- Python（推奨：3.11以上）
- Git

### Gmsh（どれか1つ）
- 方式A：ローカルにGmshをインストール（開発が楽）
- 方式B：DockerでGmshを実行（環境差を減らせる）

---

## 1. リポジトリ構成（想定）
- frontend/
- backend/
- docs/

※ もし構成が違う場合は、この文書を更新すること（これが正本）

---

## 2. 初回セットアップ

### 2.1 frontend セットアップ
```bash
cd frontend
npm install
