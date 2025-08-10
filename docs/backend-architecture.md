# バックエンドアーキテクチャ解説

## プロジェクト概要

このプロジェクトは**Cloudflare Workers**上で動作する**Hono**フレームワークベースのREST APIです。タスク管理システムのサンプル実装となっています。

## 技術スタック

- **Hono**: 軽量なWebフレームワーク（Express.js系統）
- **Chanfana**: OpenAPI自動生成ライブラリ  
- **Zod**: TypeScript型バリデーション
- **Cloudflare Workers**: エッジコンピューティング環境

## ファイル構造とソースコード解説

### 1. エントリーポイント - `src/index.ts`

```typescript
import { fromHono } from "chanfana";
import { Hono } from "hono";
import { TaskCreate, TaskDelete, TaskFetch, TaskList } from "./endpoints/*";

// Honoアプリケーション初期化
const app = new Hono<{ Bindings: Env }>();

// OpenAPIドキュメント自動生成設定
const openapi = fromHono(app, {
    docs_url: "/",  // SwaggerUIがルートに表示される
});

// APIエンドポイント登録
openapi.get("/api/tasks", TaskList);          // タスク一覧取得
openapi.post("/api/tasks", TaskCreate);       // タスク作成
openapi.get("/api/tasks/:taskSlug", TaskFetch); // 単一タスク取得
openapi.delete("/api/tasks/:taskSlug", TaskDelete); // タスク削除

export default app;
```

**ポイント:**
- `Hono`インスタンスが基本的なWebサーバー機能を提供
- `fromHono`によってOpenAPI仕様書とSwagger UIが自動生成される
- 各エンドポイントはクラスベースで分離されている

### 2. 型定義 - `src/types.ts`

```typescript
import { DateTime, Str } from "chanfana";
import type { Context } from "hono";
import { z } from "zod";

// Honoのコンテキスト型（Cloudflare環境用）
export type AppContext = Context<{ Bindings: Env }>;

// Zodスキーマでのタスクモデル定義
export const Task = z.object({
    name: Str({ example: "lorem" }),     // 文字列（例付き）
    slug: Str(),                         // URL用識別子
    description: Str({ required: false }), // オプショナル説明
    completed: z.boolean().default(false), // 完了フラグ
    due_date: DateTime(),                   // 期日
});
```

**ポイント:**
- Zodを使用したランタイム型チェック
- OpenAPI仕様書への自動反映
- Chanfanaの`Str`、`DateTime`でAPI文書の詳細指定

### 3. エンドポイント実装パターン

全エンドポイントは共通のパターンで実装されています：

#### TaskCreate例 - `src/endpoints/taskCreate.ts`

```typescript
export class TaskCreate extends OpenAPIRoute {
    // OpenAPI仕様定義
    schema = {
        tags: ["Tasks"],
        summary: "Create a new Task",
        request: {
            body: {
                content: {
                    "application/json": {
                        schema: Task,  // Zodスキーマ使用
                    },
                },
            },
        },
        responses: {
            "200": {
                description: "Returns the created task",
                content: {
                    "application/json": {
                        schema: z.object({
                            series: z.object({
                                success: Bool(),
                                result: z.object({
                                    task: Task,
                                }),
                            }),
                        }),
                    },
                },
            },
        },
    };

    // リクエストハンドラー
    async handle(c: AppContext) {
        // バリデーション済みデータ取得
        const data = await this.getValidatedData<typeof this.schema>();
        const taskToCreate = data.body;

        // ビジネスロジック（ここで実際のDB操作等を行う）
        // 現在はモックデータを返却

        return {
            success: true,
            task: {
                name: taskToCreate.name,
                slug: taskToCreate.slug,
                description: taskToCreate.description,
                completed: taskToCreate.completed,
                due_date: taskToCreate.due_date,
            },
        };
    }
}
```

#### その他エンドポイント特徴

**TaskList** (`src/endpoints/taskList.ts`):
- クエリパラメータ：`page`（ページング）、`isCompleted`（完了状態フィルタ）
- 配列レスポンス：`Task.array()`

**TaskFetch** (`src/endpoints/taskFetch.ts`):
- パスパラメータ：`taskSlug`
- 404エラーハンドリング例あり

**TaskDelete** (`src/endpoints/taskDelete.ts`):
- パスパラメータでの削除対象指定
- 削除確認用のオブジェクト返却

## Honoフレームワークの特徴

1. **軽量性**: V8エンジン上での高速動作
2. **TypeScript First**: 完全な型安全性
3. **ミドルウェアサポート**: Express.js風のミドルウェアシステム
4. **エッジ対応**: Cloudflare Workers、Deno、Bun対応

## Chanfanaライブラリの利点

1. **自動ドキュメント生成**: ZodスキーマからOpenAPI仕様書自動作成
2. **型安全なバリデーション**: リクエスト/レスポンスの自動検証
3. **Swagger UI統合**: `/`でAPI仕様書をブラウザ表示

## 開発時の注意点

- 現在の実装は**モックデータ**のみ
- 実際のプロダクションでは各`handle`メソッド内でデータベース操作を実装
- Cloudflare Workers環境では[D1](https://developers.cloudflare.com/d1/)、[KV](https://developers.cloudflare.com/kv/)等の利用を検討