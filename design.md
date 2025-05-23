# BigQuery Metadata API Server 設計ドキュメント

## 1. アプリケーションの概要

本アプリケーションは、Google Cloud BigQueryに存在するデータセット、テーブル、およびスキーマ情報を取得し、ローカルにキャッシュしてAPI経由で提供するPythonサーバーです。
主な目的は、生成AIなどがBigQueryの構造を迅速に把握できるようにすることです。

**主な機能:**

*   **メタデータ取得:** 指定された複数のGCPプロジェクトから、BigQueryのデータセット一覧、各データセット内のテーブル一覧、および各テーブルのスキーマ情報を取得します。
*   **ローカルキャッシュ:** 取得したメタデータをローカルにキャッシュし、APIリクエストに対して高速なレスポンスを提供します。キャッシュは定期的に、または手動で更新可能です。
*   **APIエンドポイント:**
    *   `/datasets`: 全ての監視対象プロジェクトに含まれるデータセットの一覧を返します。
    *   `/<dataset>/tables`: 指定されたデータセットに含まれるテーブルの一覧を返します。
    *   `/search?key=<keyword>`: 指定されたキーワードに一致するデータセット名、テーブル名、またはカラム名を持つメタデータを検索し、結果を返します。
*   **設定:** GCP認証情報（サービスアカウントキーJSONファイルパス）と対象のプロジェクトIDリストを設定ファイルまたは環境変数で管理します。

## 2. 構成モジュール

アプリケーションは以下の主要モジュールで構成されます。

*   **`main.py`**:
    *   FastAPIアプリケーションのインスタンス化とルーティング定義。
    *   APIエンドポイントの実装。
    *   アプリケーション起動時の初期化処理（キャッシュ読み込みなど）。
*   **`bigquery_client.py`**:
    *   `google-cloud-bigquery`ライブラリを使用し、GCPと通信します。
    *   指定されたプロジェクトIDに基づいてデータセット、テーブル、スキーマ情報を取得する関数を提供します。
    *   認証処理を担当します。
*   **`cache_manager.py`**:
    *   メタデータのキャッシュ（PythonオブジェクトまたはJSON形式）を管理します。
    *   キャッシュの読み込み、書き込み、更新（有効期限管理を含む）ロジックを実装します。
    *   キャッシュの保存先は、ファイルシステム（例: JSONファイル）またはインメモリ（例: `cachetools`ライブラリを使用）を選択可能にします。
    *   定期的なキャッシュ更新のスケジューリング機能（オプション）。
*   **`search_engine.py`**:
    *   キャッシュされたメタデータ（データセット、テーブル、カラム情報）に対して検索を実行します。
    *   キーワードに基づいて、関連するメタデータエントリを効率的に検索するロジックを実装します。
*   **`config.py`**:
    *   アプリケーションの設定値（GCPサービスアカウントキーのパス、プロジェクトIDリスト、キャッシュ設定など）を管理します。
    *   環境変数または設定ファイル（例: `.env`）から設定を読み込みます。
*   **`models.py`**:
    *   Pydanticを使用し、APIレスポンスや内部データ構造（Dataset, Table, Columnなど）のモデルを定義します。
    *   データのバリデーションを行います。

## 3. 依存ライブラリ

本アプリケーションは以下のPythonライブラリに依存します。

*   **`fastapi`**: 高性能なWebフレームワーク。APIエンドポイントの構築に使用します。
*   **`uvicorn[standard]`**: ASGIサーバー。FastAPIアプリケーションを実行するために使用します。
*   **`google-cloud-bigquery`**: Google Cloud BigQuery APIと対話するための公式クライアントライブラリ。
*   **`pydantic`**: データバリデーションと設定管理のためのライブラリ。APIのデータモデル定義に使用します。
*   **`python-dotenv`** (オプション): `.env`ファイルから環境変数を読み込むために使用します。設定管理に役立ちます。
*   **`cachetools`** (オプション): インメモリキャッシュを効率的に実装するために使用します。ファイルベースキャッシュの場合は不要です。
*   **`google-auth`**: Google Cloud認証を処理するために`google-cloud-bigquery`が内部で使用します。明示的なインストールが必要な場合もあります。