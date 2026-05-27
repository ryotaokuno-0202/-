# **講義文字起こし＆要約デモの仕様書**

Version: 0.1.0  
作成日: 2026-05-27  
作成者: Developer Team

## **概要**

* ユーザー導線  
  * 講義名を入力（任意・推奨）  
  * 音声ファイルをファイル選択で投入  
  * 実行ボタン押下で処理開始  
  * 文字起こしと要約を表示  
  * 文字起こし・要約・要点をコピー可能  
* 出力  
  * transcription: 文字起こしテキスト  
  * summary: 要約テキスト  
  * keyPoints: 要点（3〜5個の箇条書き）  
* エラー対応  
  * ファイル未選択  
  * APIキー未設定  
  * 外部API呼び出しエラー  
* 公式リファレンス  
  * Gemini API ドキュメント: [https://ai.google.dev/gemini-api/docs/models?hl=ja](https://ai.google.dev/gemini-api/docs/models?hl=ja)

---

## **範囲と目的**

* 本仕様は、フロントエンド＋バックエンドの実装ガイドラインと、API連携の契約を定義する。  
* 目的は、講義の音声データから文字起こしと要約を自動生成し、要点を抽出してUI上で提供すること。

---

## **用語集**

* transcription: 文字起こし結果  
* summary: 要約結果  
* keyPoints: 要点の箇条書き  
* Gemini API: Google Gemini の要約 API  
* Speech-to-Text: 音声から文字起こしを行うサービス

---

## **アーキテクチャ概要**

* クライアント側（フロントエンド）  
  * 音声ファイルと講義名を受け取り、バックエンドへ送信する  
  * 文字起こし・要約・要点を表示・コピー機能を提供  
* サーバー側（バックエンド）  
  * 音声ファイルを受け取り、文字起こしを実施（例：Google Cloud Speech-to-Text）  
  * 文字起こし結果を Gemini API へ送信して要約を取得  
  * 要点の抽出（要約から3〜5点を抽出）  
  * クライアントへ JSON で返却  
* 外部サービス連携  
  * Speech-to-Text: 文字起こし  
  * Gemini API: 要約  
  * 認証・課金・権限管理は各サービスの推奨パターンに従う

---

## **機能要件**

* 入力  
  * lectureName: string（任意、推奨）  
  * audioFile: File（wav/mp3/m4a 等、サイズは後述の制限を参照）  
* 処理  
  * 実行ボタンを押すとバックエンドで処理を開始  
* 出力  
  * transcription: string（文字起こし）  
  * summary: string（要約）  
  * keyPoints: string\[\]（3〜5件の要点、箇条書き表示）  
* コピー機能  
  * 文字起こし／要約／要点それぞれにコピー可能ボタンを設置  
* エラー表示  
  * ファイルなしエラー  
  * APIキーなしエラー  
  * API呼び出しエラー  
  * 予期せぬエラー

---

## **非機能要件**

* パフォーマンス  
  * 音声ファイルの長さ・サイズに応じた処理時間を許容範囲内に抑える設計  
* セキュリティ  
  * APIキー・認証情報はサーバー側で管理  
  * ファイルアップロードは検証済みの MIME タイプとサイズ制限を実装  
* 可用性  
  * 外部APIのリトライ機能と適切なエラーメッセージを提供  
* アクセシビリティ  
  * コピー機能、エラーメッセージは読み上げソースにも対応を想定

---

## **API・インターフェース定義**

* フロントエンド → バックエンド  
  * エンドポイント: POST /process  
  * コンテンツタイプ: multipart/form-data  
  * リクエストデータ  
    * lectureName: string（任意）  
    * audioFile: File（必須）  
  * 例:  
    * lectureName: "機械学習入門 第3回"  
    * audioFile: アップロード音声ファイル  
* バックエンド → 外部サービス  
  * Speech-to-Text  
    * 入力: 音声ファイル（ファイルパス or バイナリストリーム）  
    * 出力: transcription  
  * Gemini API  
    * 入力: transcription  
    * 出力: summary  
* バックエンド → フロントエンド（成功時のレスポンス）  
  * HTTP 200  
  * レスポンス例: { "transcription": "文字起こしテキスト...", "summary": "要約テキスト...", "keyPoints": \["要点1", "要点2", "要点3"\] }  
* エラーレスポンス  
  * 400: クライアントエラー（例: 音声ファイル未指定）  
  * 401/403: APIキー未設定  
  * 500: 内部エラー/外部APIエラー  
  * 例: { "message": "音声ファイルを選択してください。" }

---

## **データモデルとフォーマット**

* 送信データ  
  * multipart/form-data  
  * fields: lectureName (string, optional)  
  * file: audioFile (binary)  
* 返却データ（JSON）  
  * transcription: string  
  * summary: string  
  * keyPoints: string\[\] (3〜5点)  
* ローカルストレージ/キャッシュ  
  * 必要に応じて実装、現状はセッション動作を想定

---

## **画面/UI設計の指針**

* 構成  
  * 入力エリア: 講義名テキスト、音声ファイル選択  
  * 操作用エリア: 実行ボタン  
  * 出力エリア: 文字起こし、要約、要点  
  * コピーエリア: 各セクション横にコピーボタン  
  * エラー表示エリア: ページ上部または該当エリア近く  
* ユーザー体験  
  * 進捗状況を示すローディング表示  
  * 成功/エラー時のトースト通知  
  * コピー成功時のフィードバック

---

## **エラーハンドリングの方針**

* フロントエンド  
  * ファイル未選択時は即時エラーメッセージを表示  
  * サーバーエラー時はレスポンスの message を表示  
* バックエンド  
  * ファイル検証エラーは 400、メッセージを返却  
  * Gemini API キー未設定は 401/403、適切なメッセージを返却  
  * 外部 API 呼び出し失敗は 500、リトライポリシーを検討  
* ロギング  
  * エラー時にはサーバーサイドで詳細ログを残す

---

## **セキュリティと認証**

* 認証情報の管理  
  * GEMINI\_API\_KEY、GEMINI\_MODEL などは環境変数で管理  
  * APIキーはクライアントには露出させない  
* データ保護  
  * 音声ファイルのアップロード後は不要時に削除  
  * 通信はHTTPSを前提  
* 権限  
  * サービスアカウントの最小権限原則

---

## **テスト計画**

* ユニットテスト  
  * 音声ファイル検証、要点抽出のロジック、エラーメッセージの表示  
* 統合テスト  
  * /process のエンドツーエンドテスト（モック音声・モック Gemini API で検証）  
* 受け入れテスト  
  * 実際の音声ファイルを用いて、文字起こし・要約・要点が正しく表示されることを確認

---

## **実行・導入手順**

* 前提  
  * Node.js 環境 (例: 18.x)  
  * Google Cloud Speech-to-Text クレデンシャル設定  
  * Gemini API キー（環境変数 GEMINI\_API\_KEY 等）  
* ローカル開発  
  * フロントエンド  
    * 静的ファイルサーバで実行（例: npm run dev など）  
  * バックエンド  
    * npm install  
    * .env ファイルに以下を設定  
      * GEMINI\_API\_KEY=your\_api\_key  
      * GEMINI\_MODEL=models/your-gemini-model  
      * PORT=3000  
  * 実行  
    * node server.js または npm start  
* デプロイ  
  * CI/CD パイプラインを設定  
  * 環境変数の管理（秘密情報ストレージを使用）

---

## **依存関係**

* フロントエンド  
  * HTML/CSS/JavaScript  
  * fetch API, navigator.clipboard  
* バックエンド  
  * Node.js  
  * Express  
  * Multer (ファイルアップロード)  
  * @google-cloud/speech (Speech-to-Text)  
  * node-fetch または axios  
* 外部サービス  
  * Google Gemini API

---

## **公式URL・重要ポイント**

* Gemini API ドキュメント（日本語）: [https://ai.google.dev/gemini-api/docs/models?hl=ja](https://ai.google.dev/gemini-api/docs/models?hl=ja)  
* 重要ポイント  
  * 認証と鍵管理を適切に実施  
  * モデル名・エンドポイントは公式ドキュメントを参照  
  * 入力サイズ・トークン上限、言語設定に注意  
  * 料金・制限（レートリミット等）を把握

---

## **変更履歴**

* v0.1.0: 初版仕様書作成

---

## **付録**

* 推奨ディレクトリ構成案  
  * project/  
    * SPEC.md  
    * client/  
      * index.html  
      * main.js  
      * styles.css  
    * server/  
      * server.js  
      * package.json  
    * .env.example  
* 参考リンク  
  * Gemini API: [https://ai.google.dev/gemini-api/docs/models?hl=ja](https://ai.google.dev/gemini-api/docs/models?hl=ja)

