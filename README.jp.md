# my-boilerplate-go

このリポジトリは、私個人が使用するGo言語テンプレートリポジトリのコレクションです。すべてのテンプレートリポジトリは公開されており、自由に利用できます。これらのテンプレートにインスピレーションを与えたアーキテクチャリファレンスは以下に記載されています。もしこれらのテンプレートがあなたのプロジェクトに役立つと感じましたら、GitHub Starをいただけると嬉しいです！

[English](README.md) | [日本語](README.jp.md)

## ソフトウェアアーキテクチャパターン

### データセントリック

1. [**CQRSアーキテクチャ**](docs/jp/cqrs-architecture.md):  
データストアの読み取り操作と書き込み操作を分離します。読み取りと書き込みのワークロードを独立してスケールし、それぞれを最適化できます。

### レイヤード

1. [**レイヤード（n層）アーキテクチャ**](docs/jp/layered-architecture.md):  
ソフトウェアを論理レイヤーに分離します。
    - [i7s7-ymp/go-layered](https://github.com/i7s7-ymp/go-layered)

### コンポーネントベース

1. [**マイクロカーネル**](docs/jp/microkernel-architecture.md):  
最小限の機能コアを、拡張機能や顧客固有の部分から分離します。
    - 作業中

### サービス指向

1. [**マイクロサービス**](docs/jp/microservices-architecture.md):  
このアーキテクチャは、独立してデプロイ可能な小さなモジュラーサービスのスイートとしてソフトウェアアプリケーションを設計します。
   - [i7s7-ymp/go-microservice](https://github.com/i7s7-ymp/go-microservice.git)

### 分散システム

1. [**スペースベース**](docs/jp/space-based-architecture.md):
大規模分散システムにおけるデータ一貫性、信頼性のあるパフォーマンス、スケーラビリティの問題を解決します。
   - 作業中

### ドメイン駆動

1. [**DDD**](docs/jp/domain-driven-design.md):  
使用する技術よりもドメインロジックと複雑性に焦点を当てます。
   - 作業中

### イベント駆動

1. [**イベント駆動**](docs/jp/event-driven-architecture.md):  
イベントの生成、検出、消費、および反応を促進します。
   - 作業中

### 関心の分離

1. [**MVP**](docs/jp/mvp-architecture.md):  
Model-View-Controller（MVC）パターンの派生で、データ管理、ユーザーインターフェース、制御フローの関心事を分離することを目的とします。
   - 作業中

### インタープリター

1. [**インタープリター**](docs/jp/interpreter-pattern.md):  
高水準言語で書かれ、インタープリターが実行可能コードに変換します。
   - 作業中

### 並行性

1. [**オーケストレーション**](docs/jp/orchestration-pattern.md): 
サービス間の相互作用を指示する中央コーディネーター（オーケストレーターと呼ばれることが多い）です。オーケストレーターはサービス間の制御フローとデータフローの管理を担当します。
   - 作業中

## 参考文献

- [ByteByteGo - Top 5 Software Architectural Patterns](https://bytebytego.com/guides/top-5-software-architectural-patterns/)
