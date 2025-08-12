# 設定ファイル形式設計

## 概要
eBPF L7 Rate Limiterの設定ファイル形式とスキーマ定義

## 設定ファイル形式

### 主設定ファイル: `config.yaml`

```yaml
# eBPF L7 Rate Limiter Configuration
version: "1.0"

# システム設定
system:
  # ネットワークインターフェース
  interface: "eth0"
  
  # eBPFプログラム設定
  ebpf:
    # TCフック設定
    tc:
      ingress: true
      egress: false
      priority: 1
    
    # XDPフック設定（オプション）
    xdp:
      enabled: false
      mode: "native"  # native, generic, offload
  
  # ログ設定
  logging:
    level: "info"  # trace, debug, info, warn, error
    format: "json"  # json, text
    output: "/var/log/ebpf-pug-lb.log"
  
  # パフォーマンス設定
  performance:
    # Map サイズ
    max_clients: 10000
    max_concurrent_requests: 100000
    
    # 統計情報更新間隔
    stats_interval: "1s"
    
    # メモリ使用量制限
    memory_limit: "256MB"

# API設定
api:
  # REST API
  rest:
    enabled: true
    bind_address: "0.0.0.0"
    port: 8080
    
    # TLS設定
    tls:
      enabled: false
      cert_file: "/etc/ssl/certs/server.crt"
      key_file: "/etc/ssl/private/server.key"
    
    # 認証設定
    auth:
      enabled: true
      type: "bearer"  # bearer, basic, none
      token: "${API_TOKEN}"
  
  # WebSocket
  websocket:
    enabled: true
    path: "/api/v1/ws"
  
  # CLI
  cli:
    enabled: true
    socket_path: "/var/run/ebpf-pug-lb.sock"

# デフォルトレート制限設定
defaults:
  # システム全体で使用するアルゴリズム（統一）
  algorithm: "token_bucket"  # システム全体で固定
  
  # デフォルトレート制限
  rate_limit: 1000  # requests per second
  
  # デフォルトバーストサイズ
  burst_size: 100
  
  # デフォルト優先度
  priority: "normal"  # high, normal, low
  
  # デフォルトアクション
  action: "drop"  # drop, delay, log_only

# クライアント設定
clients:
  # 個別クライアント設定（アルゴリズムは統一、パラメータのみ個別）
  - client_id: "api-client-001"
    description: "High priority API client"
    
    # 識別子設定
    identifiers:
      - type: "header"
        name: "X-Client-ID"
        value: "api-client-001"
      - type: "header"
        name: "Authorization"
        pattern: "Bearer token-001-.*"
    
    # レート制限設定（アルゴリズムは統一、パラメータのみ個別）
    rate_limiting:
      rate_limit: 5000      # このクライアント専用の制限値
      burst_size: 500       # このクライアント専用のバーストサイズ
      priority: "high"      # 優先度
      action: "drop"        # アクション
    
    # 時間帯別設定
    schedules:
      - name: "business_hours"
        cron: "0 9-17 * * 1-5"  # 平日9-17時
        rate_limit: 5000
      - name: "off_hours"
        cron: "0 0-8,18-23 * * *"  # その他の時間
        rate_limit: 1000
  
  - client_id: "mobile-app"
    description: "Mobile application clients"
    
    identifiers:
      - type: "header"
        name: "User-Agent"
        pattern: "MobileApp/.*"
      - type: "header"
        name: "X-API-Key"
        value: "mobile-api-key"
    
    rate_limiting:
      rate_limit: 100       # パラメータのみ個別設定
      burst_size: 50        # アルゴリズムは統一
      priority: "normal"
      action: "drop"

# グローバルルール
global_rules:
  # IP別制限
  - name: "ip_rate_limit"
    type: "ip_based"
    rate_limit: 10000
    window: "1m"
    action: "drop"
  
  # 地理的制限
  - name: "geo_blocking"
    type: "geo_based"
    allowed_countries: ["JP", "US", "EU"]
    action: "drop"
  
  # DDoS保護
  - name: "ddos_protection"
    type: "anomaly_detection"
    threshold_multiplier: 10.0
    action: "drop"

# 監視・アラート設定
monitoring:
  # メトリクス収集
  metrics:
    enabled: true
    interval: "10s"
    
    # Prometheus設定
    prometheus:
      enabled: true
      port: 9090
      path: "/metrics"
  
  # アラート設定
  alerts:
    - name: "high_drop_rate"
      condition: "drop_rate > 0.1"
      duration: "5m"
      action: "webhook"
      webhook_url: "https://alerts.example.com/webhook"
    
    - name: "system_overload"
      condition: "cpu_usage > 0.8"
      duration: "2m"
      action: "email"
      email: "admin@example.com"

# 開発・デバッグ設定
development:
  # デバッグモード
  debug: false
  
  # テストモード
  test_mode: false
  
  # プロファイリング
  profiling:
    enabled: false
    port: 6060
  
  # トレーシング
  tracing:
    enabled: false
    jaeger_endpoint: "http://localhost:14268/api/traces"
```

## 環境変数設定

### `.env` ファイル例
```bash
# API認証トークン
API_TOKEN=your-secret-token-here

# データベース接続
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ebpf_pug_lb
DB_USER=ebpf_user
DB_PASSWORD=secure_password

# 外部サービス
WEBHOOK_URL=https://alerts.example.com/webhook
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx

# TLS証明書パス
TLS_CERT_PATH=/etc/ssl/certs/server.crt
TLS_KEY_PATH=/etc/ssl/private/server.key

# ログ設定
LOG_LEVEL=info
LOG_FORMAT=json
```

## 設定スキーマ定義

### JSON Schema (`config-schema.json`)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "eBPF L7 Rate Limiter Configuration",
  "type": "object",
  "required": ["version", "system"],
  "properties": {
    "version": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+$"
    },
    "system": {
      "type": "object",
      "required": ["interface"],
      "properties": {
        "interface": {
          "type": "string",
          "description": "Network interface name"
        },
        "ebpf": {
          "type": "object",
          "properties": {
            "tc": {
              "type": "object",
              "properties": {
                "ingress": {"type": "boolean"},
                "egress": {"type": "boolean"},
                "priority": {"type": "integer", "minimum": 1}
              }
            }
          }
        }
      }
    },
    "clients": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["client_id", "identifiers", "rate_limiting"],
        "properties": {
          "client_id": {
            "type": "string",
            "pattern": "^[a-zA-Z0-9_-]+$"
          },
          "identifiers": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["type", "name"],
              "properties": {
                "type": {
                  "type": "string",
                  "enum": ["header", "query", "path", "ip"]
                },
                "name": {"type": "string"},
                "value": {"type": "string"},
                "pattern": {"type": "string"}
              }
            }
          },
          "rate_limiting": {
            "type": "object",
            "required": ["algorithm", "rate_limit"],
            "properties": {
              "algorithm": {
                "type": "string",
                "enum": ["token_bucket", "fixed_window", "sliding_window"]
              },
              "rate_limit": {
                "type": "integer",
                "minimum": 1
              },
              "burst_size": {
                "type": "integer",
                "minimum": 1
              }
            }
          }
        }
      }
    }
  }
}
```

## 設定ファイル管理

### 設定ファイル配置
```
/etc/ebpf-pug-lb/
├── config.yaml              # メイン設定
├── clients/                 # クライアント別設定
│   ├── api-clients.yaml
│   ├── mobile-clients.yaml
│   └── web-clients.yaml
├── rules/                   # ルール設定
│   ├── global-rules.yaml
│   └── security-rules.yaml
├── schemas/                 # スキーマ定義
│   └── config-schema.json
└── templates/               # 設定テンプレート
    ├── basic-config.yaml
    └── advanced-config.yaml
```

### 設定検証コマンド
```bash
# 設定ファイル検証
ebpf-pug-lb config validate --config /etc/ebpf-pug-lb/config.yaml

# 設定ファイル生成
ebpf-pug-lb config generate --template basic --output config.yaml

# 設定差分確認
ebpf-pug-lb config diff --current config.yaml --new config-new.yaml
```

## 動的設定更新

### ホットリロード対応項目
- クライアント設定の追加・削除・更新
- レート制限値の変更
- 優先度の変更
- ログレベルの変更

### 再起動が必要な項目
- ネットワークインターフェース変更
- eBPFプログラム設定変更
- APIバインドアドレス変更

## 設定例

### 最小設定例
```yaml
version: "1.0"
system:
  interface: "eth0"
api:
  rest:
    enabled: true
    port: 8080
clients:
  - client_id: "default"
    identifiers:
      - type: "header"
        name: "X-Client-ID"
        value: "default"
    rate_limiting:
      algorithm: "token_bucket"
      rate_limit: 1000
```

### 本番環境設定例
```yaml
version: "1.0"
system:
  interface: "eth0"
  logging:
    level: "warn"
    format: "json"
    output: "/var/log/ebpf-pug-lb.log"
  performance:
    max_clients: 50000
    max_concurrent_requests: 1000000
api:
  rest:
    enabled: true
    bind_address: "0.0.0.0"
    port: 8080
    tls:
      enabled: true
      cert_file: "/etc/ssl/certs/server.crt"
      key_file: "/etc/ssl/private/server.key"
    auth:
      enabled: true
      type: "bearer"
      token: "${API_TOKEN}"
monitoring:
  metrics:
    enabled: true
    prometheus:
      enabled: true
      port: 9090
```

## 次のステップ
1. エラーハンドリング設計
2. 設定ファイルパーサー実装
3. 設定検証ロジック実装
