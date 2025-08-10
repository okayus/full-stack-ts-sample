# Wrangler設定解説 - `wrangler.jsonc`

## Wranglerとは

**Wrangler**はCloudflare Workers開発用の公式CLIツールです。ローカル開発、デプロイ、設定管理を行います。

## 設定ファイル詳細解説

### 基本設定

```jsonc
{
    "$schema": "node_modules/wrangler/config-schema.json",
    "name": "backend",
    "main": "src/index.ts",
    "compatibility_date": "2025-08-10"
}
```

- **`$schema`**: VSCodeでの自動補完・バリデーション有効化
- **`name`**: Worker名（デプロイ時のドメインに影響：`backend.{subdomain}.workers.dev`）
- **`main`**: エントリーポイントファイル
- **`compatibility_date`**: Cloudflare Workers APIの互換性日付（重要）

### 監視設定

```jsonc
{
    "observability": {
        "enabled": true
    }
}
```

- **ログ・メトリクス収集**を有効化
- Cloudflareダッシュボードでパフォーマンス監視可能

## コメントアウト済み設定オプション

### Smart Placement（スマート配置）

```jsonc
// "placement": { "mode": "smart" }
```

- **地理的最適化**: ユーザーに最も近いデータセンターで実行
- レイテンシ削減効果
- 必要時にコメント解除

### 環境変数

```jsonc
// "vars": { "MY_VARIABLE": "production_value" }
```

- **ランタイム環境変数**設定
- TypeScriptの`Env`型と連携
- 機密情報は[Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)を使用

### 静的アセット

```jsonc
// "assets": { "directory": "./public/", "binding": "ASSETS" }
```

- **静的ファイル配信**（画像、CSS、JS等）
- Worker内で`ASSETS`バインディングでアクセス可能

### Service Bindings（マイクロサービス連携）

```jsonc
// "services": [{ "binding": "MY_SERVICE", "service": "my-service" }]
```

- **Worker間通信**
- マイクロサービスアーキテクチャ実現
- 他のWorkerのAPIを直接呼び出し可能

## 実際の開発での活用

### 1. 開発フロー

```bash
# ローカル開発サーバー起動
wrangler dev

# プロダクションデプロイ
wrangler publish
```

### 2. 環境別設定

通常は環境別にファイルを分割：

```
wrangler.jsonc          # 開発環境
wrangler.prod.jsonc     # 本番環境
```

### 3. 型安全性の確保

`src/types.ts`の`Env`型とwrangler設定の`vars`は対応させる：

```typescript
// wrangler.jsonc
// "vars": { "DATABASE_URL": "..." }

// types.ts (自動生成される)
interface Env {
    DATABASE_URL: string;
}
```

## プロダクション運用時の推奨設定

```jsonc
{
    "name": "backend-prod",
    "main": "src/index.ts",
    "compatibility_date": "2025-08-10",
    "placement": { "mode": "smart" },
    "observability": { "enabled": true },
    "vars": {
        "ENVIRONMENT": "production"
    }
    // 機密情報はwrangler secretコマンドで別途設定
}
```

## 参考リンク

- [Wrangler Configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Smart Placement](https://developers.cloudflare.com/workers/configuration/smart-placement/)
- [Environment Variables](https://developers.cloudflare.com/workers/wrangler/configuration/#environment-variables)
- [Static Assets](https://developers.cloudflare.com/workers/static-assets/binding/)