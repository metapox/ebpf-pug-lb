# API仕様設計

## 概要
eBPF L7 Rate Limiterの制御・監視・設定を行うためのAPI仕様

## API構成

### 1. Control API (REST)
ユーザー空間からのリアルタイム制御用REST API

### 2. CLI Interface
コマンドライン操作用インターフェース

### 3. Configuration API
設定ファイルベースの静的設定

## REST API仕様

### Base URL
```
http://localhost:8080/api/v1
```

### 認証
```
Authorization: Bearer <token>
```

### エンドポイント一覧

#### 1. クライアント管理

##### GET /clients
全クライアントの一覧取得
```json
{
  "clients": [
    {
      "client_id": "client-001",
      "rate_limit": 1000,
      "algorithm": "token_bucket",
      "status": "active",
      "created_at": "2024-08-10T06:30:00Z"
    }
  ],
  "total": 1
}
```

##### POST /clients
新規クライアント登録
```json
// Request（アルゴリズム指定なし、パラメータのみ）
{
  "client_id": "client-002",
  "rate_limit": 500,
  "burst_size": 100,
  "priority": "normal"
}

// Response
{
  "client_id": "client-002",
  "status": "created",
  "message": "Client registered successfully"
}
```

##### GET /clients/{client_id}
特定クライアントの詳細取得
```json
{
  "client_id": "client-001",
  "config": {
    "rate_limit": 1000,
    "algorithm": "token_bucket",
    "burst_size": 200,
    "priority": "high"
  },
  "stats": {
    "requests_total": 15420,
    "requests_allowed": 15200,
    "requests_dropped": 220,
    "current_tokens": 150,
    "last_request": "2024-08-10T06:29:45Z"
  }
}
```

##### PUT /clients/{client_id}
クライアント設定更新
```json
// Request
{
  "rate_limit": 1500,
  "burst_size": 300
}

// Response
{
  "client_id": "client-001",
  "status": "updated",
  "message": "Configuration updated successfully"
}
```

##### DELETE /clients/{client_id}
クライアント削除
```json
{
  "client_id": "client-001",
  "status": "deleted",
  "message": "Client removed successfully"
}
```

#### 2. 統計情報

##### GET /stats/global
全体統計情報
```json
{
  "global_stats": {
    "total_requests": 1000000,
    "total_allowed": 950000,
    "total_dropped": 50000,
    "active_clients": 150,
    "uptime_seconds": 86400
  },
  "performance": {
    "avg_processing_time_ns": 1200,
    "max_processing_time_ns": 5000,
    "packets_per_second": 50000
  }
}
```

##### GET /stats/clients
全クライアントの統計情報
```json
{
  "client_stats": [
    {
      "client_id": "client-001",
      "requests_total": 10000,
      "requests_allowed": 9500,
      "requests_dropped": 500,
      "drop_rate": 0.05
    }
  ]
}
```

##### GET /stats/clients/{client_id}
特定クライアントの詳細統計
```json
{
  "client_id": "client-001",
  "time_series": {
    "interval": "1m",
    "data": [
      {
        "timestamp": "2024-08-10T06:25:00Z",
        "requests": 100,
        "allowed": 95,
        "dropped": 5
      }
    ]
  }
}
```

#### 3. システム制御

##### GET /system/status
システム状態取得
```json
{
  "status": "running",
  "algorithm": "token_bucket",  // システム全体で使用中のアルゴリズム
  "ebpf_programs": {
    "tc_ingress": "loaded",
    "tc_egress": "loaded"
  },
  "maps": {
    "client_config": "active",
    "rate_limit_state": "active",
    "statistics": "active"
  },
  "version": "0.1.0"
}
```

##### PUT /system/algorithm
システム全体のアルゴリズム変更（要再起動）
```json
// Request
{
  "algorithm": "token_bucket",
  "restart_required": true
}

// Response
{
  "status": "updated",
  "message": "Algorithm updated. System restart required.",
  "restart_required": true
}
```

##### POST /system/reload
設定リロード
```json
{
  "status": "reloaded",
  "message": "Configuration reloaded successfully",
  "affected_clients": 25
}
```

##### POST /system/reset
統計情報リセット
```json
{
  "status": "reset",
  "message": "Statistics reset successfully"
}
```

## CLI Interface

### 基本コマンド構造
```bash
ebpf-pug-lb [GLOBAL_OPTIONS] <COMMAND> [COMMAND_OPTIONS]
```

### グローバルオプション
```bash
--config, -c <FILE>     設定ファイルパス
--verbose, -v           詳細出力
--json                  JSON形式出力
--help, -h              ヘルプ表示
```

### コマンド一覧

#### 1. システム制御
```bash
# システム開始（アルゴリズム指定）
ebpf-pug-lb start --interface eth0 --algorithm token_bucket

# システム停止
ebpf-pug-lb stop

# 状態確認（使用中アルゴリズム表示）
ebpf-pug-lb status

# 設定リロード
ebpf-pug-lb reload
```

#### 2. クライアント管理
```bash
# クライアント一覧
ebpf-pug-lb client list

# クライアント追加（アルゴリズム指定なし）
ebpf-pug-lb client add --id client-001 --rate-limit 1000 --burst-size 100

# クライアント更新
ebpf-pug-lb client update --id client-001 --rate-limit 1500

# クライアント削除
ebpf-pug-lb client remove --id client-001

# クライアント詳細
ebpf-pug-lb client show --id client-001
```

#### 3. 統計情報
```bash
# 全体統計
ebpf-pug-lb stats global

# クライアント統計
ebpf-pug-lb stats client --id client-001

# リアルタイム監視
ebpf-pug-lb monitor --interval 1s

# 統計リセット
ebpf-pug-lb stats reset
```

## エラーレスポンス

### HTTP エラーコード
- `400 Bad Request`: 不正なリクエスト
- `401 Unauthorized`: 認証エラー
- `404 Not Found`: リソースが見つからない
- `409 Conflict`: 競合エラー（既存クライアントID等）
- `500 Internal Server Error`: サーバー内部エラー

### エラーレスポンス形式
```json
{
  "error": {
    "code": "CLIENT_NOT_FOUND",
    "message": "Client with ID 'client-001' not found",
    "details": {
      "client_id": "client-001",
      "suggestion": "Use 'GET /clients' to list available clients"
    }
  }
}
```

## WebSocket API (リアルタイム監視)

### エンドポイント
```
ws://localhost:8080/api/v1/ws/monitor
```

### メッセージ形式
```json
{
  "type": "stats_update",
  "timestamp": "2024-08-10T06:30:00Z",
  "data": {
    "client_id": "client-001",
    "requests_per_second": 150,
    "drop_rate": 0.02
  }
}
```

## SDK/ライブラリ

### Go SDK例
```go
client := ebpflb.NewClient("http://localhost:8080")
err := client.AddClient("client-001", &ebpflb.ClientConfig{
    RateLimit: 1000,
    Algorithm: "token_bucket",
})
```

### Python SDK例
```python
from ebpf_pug_lb import Client

client = Client("http://localhost:8080")
client.add_client("client-001", rate_limit=1000, algorithm="token_bucket")
```

## 次のステップ
1. 設定ファイル形式設計
2. エラーハンドリング設計
3. API実装プロトタイプ作成
