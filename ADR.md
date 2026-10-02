# Architecture Decision Record

## 社員向け統合人事プラットフォーム

- **状態**: 提案（Azure リソース構成は未承認）
- **日付**: 2026-10-02

### 背景

約5,000人が利用する社員向け統合人事プラットフォームを構築する。必要な機能は社員一覧、勤怠入力、経費精算であり、経費精算では領収書ファイルを添付する。社員認証には Microsoft Entra ID を使い、Entra の `admin` グループに属するメンバーには管理者権限を付与する。社員一覧では Entra ID のユーザー属性を中心に表示する。

### 決定済みの要件

1. 認証は Microsoft Entra ID を使用する。
2. 管理者権限は Entra の `admin` グループへの所属に基づいて付与する。
3. 本番利用者の入口はインターネットからの HTTPS とし、公開入口を Web Application Firewall（WAF）で保護する。アプリケーション本体とデータサービスは VNet／Private Endpoint で隔離する。
4. 経費精算で領収書の画像・PDFを扱う。
5. 社員一覧の属性は Entra ID を中心に表示する。
6. 本番環境は Japan East を希望し、単一リージョン内のゾーン冗長化を目標とする。

### 暫定アーキテクチャ案

以下は検討候補であり、利用者によるリソース範囲の承認前のため、確定決定ではない。

- Application Gateway WAF v2 を公開 HTTPS 入口として配置し、WAF policy で保護する。
- App Service をゾーン冗長構成で実行し、Private Endpoint で受信経路を限定する。バックエンドへの送信には VNet Integration を使う。
- Azure SQL Database に勤怠・経費等の業務データを格納し、Private Endpoint 経由に限定する。
- Azure Blob Storage に領収書を保存し、ZRS、Private Endpoint、Managed Identity によるアクセスを検討する。
- Key Vault、Managed Identity、Private DNS、Log Analytics、Application Insights、Azure Monitor Private Link Scope を用いて、資格情報管理、名前解決、監視経路をプライベート化する。
- Entra アプリケーション登録に管理者用 app role を定義し、`admin` グループに割り当てる。社員一覧に必要な Microsoft Graph 権限は最小権限とし、必要な管理者同意を得る。

### 影響とトレードオフ

- インターネット経由の社員利用を許可するため、WAF の公開入口は必要となる。アプリ本体、データ、監視の接続は可能な範囲で Private Link に限定する。
- 社員一覧で Entra ID を参照するため、Graph 権限・管理者同意・属性の品質と同期要件を別途確定する必要がある。
- 5,000人という登録ユーザー数だけではピーク同時利用や DB 容量を決められない。実際の負荷試験と利用パターンに基づき SKU を選定する。
- 単一リージョン構成はゾーン障害への耐性を高めるが、リージョン障害時の DR はこの案に含まれない。
- Azure best-practices MCP と WAF サービスガイド取得はタイムアウトしたため、Microsoft Learn の公式資料で確認できた範囲を代替根拠とする。

### 未決事項

- 上記の Azure リソース範囲を計画対象として承認するか。
- ピーク同時利用、可用性目標、RTO/RPO、データ保持期間、法令・監査要件。
- Entra ID 属性の正本・同期頻度、および Microsoft Graph の具体的な権限。
- WAF policy のモードとルール調整、ドメイン・証明書の運用方法。
- SQL と App Service の具体的な SKU、容量、スケーリング設定。

### 参考資料

- [App Service の Private Endpoint](https://learn.microsoft.com/azure/app-service/overview-private-endpoint)
- [Azure SQL Database のローカル冗長性とゾーン冗長性](https://learn.microsoft.com/azure/azure-sql/database/high-availability-sla-local-zone-redundancy)
- [Azure Storage のデータ冗長性](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
- [Azure Monitor の Private Link 構成](https://learn.microsoft.com/azure/azure-monitor/fundamentals/private-link-configure)
