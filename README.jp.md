# my-boilerplate-go

このリポジトリは、私個人が使用するGo言語テンプレートリポジトリのコレクションです。すべてのテンプレートリポジトリは公開されており、自由に利用できます。これらのテンプレートにインスピレーションを与えたアーキテクチャリファレンスは以下に記載されています。もしこれらのテンプレートがあなたのプロジェクトに役立つと感じましたら、GitHub Starをいただけると嬉しいです！

[English](README.md) | [日本語](README.jp.md)

## ソフトウェアアーキテクチャパターン

### データ中心

1. **CQRSアーキテクチャ**:  
データストアの読み取り操作と書き込み操作を分離します。読み取りと書き込みのワークロードを独立してスケールし、それぞれを最適化できます。

### レイヤード

1. **レイヤード（n層）アーキテクチャ**:  
ソフトウェアを論理レイヤーに分離します。[詳細を表示](docs/jp/layered-architecture.md)
    - [i7s7-ymp/go-layered](https://github.com/i7s7-ymp/go-layered)

### コンポーネントベース

1. **マイクロカーネル**:  
最小限の機能コアを、拡張機能や顧客固有の部分から分離します。[詳細を表示](docs/jp/microkernel-architecture.md)
    - 作業中

### サービス指向

1. **マイクロサービス**:  
このアーキテクチャは、独立してデプロイ可能な小さなモジュラーサービスのスイートとしてソフトウェアアプリケーションを設計します。[詳細を表示](docs/jp/microservices-architecture.md)
   - 作業中

### 分散システム

1. **スペースベース**:
大規模分散システムにおけるデータ一貫性、信頼性のあるパフォーマンス、スケーラビリティの問題を解決します。[詳細を表示](docs/jp/space-based-architecture.md)
   - 作業中

### ドメイン駆動

1. **DDD**:  
使用する技術よりもドメインロジックと複雑性に焦点を当てます。[詳細を表示](docs/jp/domain-driven-design.md)
   - 作業中

### イベント駆動

1. **イベント駆動**:  
イベントの生成、検出、消費、および反応を促進します。[詳細を表示](docs/jp/event-driven-architecture.md)
   - 作業中

### 関心の分離

1. **MVP**:  
Model-View-Controller（MVC）パターンの派生で、データ管理、ユーザーインターフェース、制御フローの関心事を分離することを目的とします。[詳細を表示](docs/jp/mvp-architecture.md)
   - 作業中

### インタープリター

1. **インタープリター**:  
高水準言語で書かれ、インタープリターが実行可能コードに変換します。[詳細を表示](docs/jp/interpreter-pattern.md)
   - 作業中

### 並行性

1. **オーケストレーション**: 
サービス間の相互作用を指示する中央コーディネーター（オーケストレーターと呼ばれることが多い）です。オーケストレーターはサービス間の制御フローとデータフローの管理を担当します。[詳細を表示](docs/jp/orchestration-pattern.md)
   - 作業中

## 参考文献

- [ByteByteGo - Top 5 Software Architectural Patterns](https://bytebytego.com/guides/top-5-software-architectural-patterns/)
