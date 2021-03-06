---
title: GitHub Actions のセキュリティ強化
shortTitle: セキュリティ強化
intro: '{% data variables.product.prodname_actions %} の機能を使用するための適切なセキュリティプラクティス。'
product: '{% data reusables.gated-features.actions %}'
redirect_from:
  - /actions/getting-started-with-github-actions/security-hardening-for-github-actions
versions:
  free-pro-team: '*'
  enterprise-server: '>=2.22'
type: overview
topics:
  - セキュリティ
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

### 概要

このガイドでは、特定の {% data variables.product.prodname_actions %} の機能のセキュリティ強化を設定する方法について説明します。 {% data variables.product.prodname_actions %} の概念について理解を深めるには、「[GitHub Actions の中核的概念](/actions/getting-started-with-github-actions/core-concepts-for-github-actions)」を参照してください。

### シークレットを使用する

機密性の高い値は、平文としてワークフローファイルに保存するのではなく、シークレットとして保存してください。 [シークレット](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)は、Organization{% if currentVersion == "free-pro-team@latest" or currentVersion ver_gt "enterprise-server@3.0" %}、リポジトリ、あるいは環境{% else %}あるいはリポジトリ{% endif %}のレベルで設定でき、機密情報を{% data variables.product.product_name %}に保存できるようになります。

シークレットは [Libsodium sealed boxes](https://libsodium.gitbook.io/doc/public-key_cryptography/sealed_boxes) を使用するため、{% data variables.product.product_name %} に到達する前に暗号化されます。 これは、[UI を使用](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository)して、または [REST API](/rest/reference/actions#secrets) を介してシークレットが送信されたときに発生します。 このクライアント側の暗号化により、{% data variables.product.product_name %} のインフラストラクチャ内での偶発的なログ（例外ログやリクエストログなど）に関連するリスクを最小限に抑えることができます。 シークレットがアップロードされると、{% data variables.product.product_name %} はそれを復号化して、ワークフローランタイムに挿入できるようになります。

偶発的な開示を防ぐために、{% data variables.product.product_name %} は、実行ログに表示されるシークレットを編集しようとするメカニズムを使用しています。 この編集は、設定されたシークレットの完全一致、および Base64 などの値の一般的なエンコーディングを検索します。 ただし、シークレットの値を変換する方法は複数あるため、この編集は保証されません。 そのため、シークレットを確実に編集し、シークレットに関連する他のリスクを制限するために実行する必要がある、特定の予防的ステップと推奨事項は次のとおりです。

- **構造化データをシークレットとして使用しない**
    - 構造化データは、ログ内のシークレットの編集失敗の原因となる可能性があります。これは、編集が特定のシークレット値の完全一致を見つけることに大きく依存しているためです。 たとえば、JSON、XML、または YAML（または同様）の Blob を使用してシークレット値をカプセル化しないでください。シークレットが適切に編集される可能性が大幅に低下するためです。 代わりに、機密値ごとに個別のシークレットを作成します。
- **ワークフロー内で使用されるすべてのシークレットを登録する**
    - シークレットを使用してワークフロー内で別の機密値を生成する場合は、生成された値を正式に[シークレットとして登録](https://github.com/actions/toolkit/tree/main/packages/core#setting-a-secret)して、ログに表示されたときに編集されるようにする必要があります。 たとえば、秘密鍵を使用して署名済み JWT を生成し、Web API にアクセスする場合は、その JWT をシークレットとして登録してください。そうしない場合、ログ出力に入力されても編集されません。
    - シークレットの登録は、あらゆる種類の変換/エンコーディングにも適用されます。 シークレットが何らかの方法で変換された場合（Base64 や URL エンコードなど）、新しい値もシークレットとして登録してください。
- **シークレットの処理方法を監査する**
    - シークレットの使用方法を監査し、シークレットが想定どおりに処理されていることを確認します。 これを行うには、ワークフローを実行しているリポジトリのソースコードを確認し、ワークフローで使用されているアクションを確認します。 たとえば、意図しないホストに送信されていないか、またはログ出力に明示的に出力されていないかを確認します。
    - 有効/無効な入力をテストした後、ワークフローの実行ログを表示し、シークレットが正しく編集されているか、表示されていないかを確認します。 呼び出しているコマンドまたはツールがどのようにしてエラーを `STDOUT` および `STDERR` に送信するかは必ずしも明らかではなく、シークレットはその後エラーログに記録される可能性があります。 そのため、有効な入力と無効な入力をテストした後、ワークフローログを手動で確認することをお勧めします。
- **スコープが最小限の認証情報を使用する**
    - ワークフロー内で使用されている認証情報に必要な最小限の権限があることを確認し、リポジトリへの書き込みアクセスを持つすべてのユーザが、リポジトリで設定されたすべてのシークレットへの読み取りアクセスを持っていることに注意してください。
- **登録されたシークレットの監査とローテーション**
    - 登録されたシークレットを定期的に確認して、現在も必要であることを確認します。 不要になったものは削除してください。
    - シークレットを定期的にローテーションして、不正使用されたシークレットが有効である期間を短縮します。
{% if currentVersion == "free-pro-team@latest" or currentVersion ver_gt "enterprise-server@3.0" %}
- **シークレットへのアクセスのレビューを必須とすることを検討する**
    - 必須のレビュー担当者を使って環境のシークレットを保護できます。 レビュー担当者によって許可されるまで、ワークフローのジョブは環境のシークレットにアクセスできません。 緩急尾へのシークレットの保存、あるいは環境にレビュー担当者を必須とすることの詳細については、「[暗号化されたシークレット](/actions/reference/encrypted-secrets)」及び「[環境](/actions/reference/environments)」を参照してください。
{% endif %}

### サードパーティアクションを使用する

ワークフロー内の個々のジョブは、他のジョブと相互作用（および侵害）する場合があります。 たとえば、後のジョブで使用される環境変数をクエリするジョブ、後のジョブが処理する共有ディレクトリにファイルを書き込むジョブ、あるいはもっと直接的にDocker ソケットとやり取りして他の実行中のコンテナを検査してコマンドを実行するジョブなどです。

これは、ワークフロー内の 1 つのアクションへの侵害が非常に重要になる可能性があるということです。その侵害されたアクションがリポジトリに設定されたすべてのシークレットにアクセスし、`GITHUB_TOKEN` を使用してリポジトリに書き込むことができるためです。 したがって、{% data variables.product.prodname_dotcom %} のサードパーティリポジトリからアクションを調達することには大きなリスクがあります。 次のような適切な方法に従うことで、このリスクを軽減することができます。

* **アクションを完全なコミット SHA にピン止めする**

  現在、アクションを不変のリリースとして使用する唯一の方法は、アクションを完全なコミット SHA にピン止めすることです。 特定の SHA にピン止めすると、有効な Git オブジェクトペイロードに対して SHA-1 衝突を生成する必要があるため、悪意のある人がアクションのリポジトリにバックドアを追加するリスクを軽減できます。

  {% if currentVersion ver_lt "enterprise-server@3.1" %}
  {% warning %}

  **警告:** コミット SHA の短いバージョンは安全ではないため、アクションの Git リファレンスの指定には使用しないでください。 リポジトリネットワークの仕組みにより、どのユーザもリポジトリをフォークし、短い SHA と衝突するよう細工されたコミットをプッシュできます。 これにより、その SHA の後続のクローンがあいまいなコミットになるため失敗します。 その結果、短縮された SHA を使用するワークフローはすぐに失敗します。

  {% endwarning %}
  {% endif %}


* **アクションのソースコードを監査する**

  アクションが想定どおりにリポジトリとシークレットのコンテンツを処理していることを確認します。 たとえば、シークレットが意図しないホストに送信されていないか、または誤ってログに記録されていないかを確認します。

* **作成者を信頼できる場合に限り、アクションをタグにピン止めする**

  コミット SHA に対するピン止めが最も安全なオプションですが、タグを指定する方が便利で広く使用されています。 タグを指定する場合は、アクションの作成者が信頼できることを確認してください。 {% data variables.product.prodname_marketplace %} の「Verified creator」バッジは便利な判断材料で、 {% data variables.product.prodname_dotcom %} で身元が確認されたチームによって作成されたアクションであることを示しています。 作者が信頼できる場合でも、このアプローチにはリスクがあることに注意してください。悪意のある人がアクションを保存しているリポジトリにアクセスすると、タグが移動または削除される可能性があります。

### リポジトリ間のアクセスを検討する

{% data variables.product.product_name %} は、意図的に一度に単一のリポジトリに対してスコープされます。 `GITHUB_TOKEN` は、書き込みアクセスユーザと同じレベルのアクセスを許可します。書き込みアクセスを持つユーザは、ワークフローファイルを作成または変更することによってこのトークンにアクセスできるためです。 ユーザはリポジトリごとに特定の権限を持っているため、1 つのリポジトリの `GITHUB_TOKEN` に別のリポジトリへのアクセスを許可すると、慎重に実装しない場合 {% data variables.product.prodname_dotcom %} 権限モデルに影響します。 同様に、{% data variables.product.prodname_dotcom %} 認証トークンをワークフローに追加する場合は注意が必要です。これは、コラボレータに誤って広範なアクセスを付与することにより、{% data variables.product.prodname_dotcom %} アクセス許可モデルにも影響を与える可能性があるためです。

[{% data variables.product.prodname_dotcom %} ロードマップ](https://github.com/github/roadmap/issues/74)では、{% data variables.product.product_name %} 内のリポジトリ間アクセスを可能にするフローをサポートする計画がありますが、この機能はまだサポートされていません。 現在、権限のあるリポジトリ間のやり取りを実行する唯一の方法は、ワークフロー内に {% data variables.product.prodname_dotcom %} 認証トークンまたは SSH キーをシークレットとして配置することです。 多くの認証トークンタイプでは特定のリソースへの詳細なアクセスが許可されていないことから、意図したものよりはるかに広範なアクセスを許可できるため、間違ったトークンタイプを使用すると重大なリスクが生じます。

次のリストでは、ワークフロー内のリポジトリデータにアクセスするための推奨アプローチを優先度の高い順に説明しています。

1. **`GITHUB_TOKEN`**
    -  このトークンは、ワークフローを呼び出した単一のリポジトリに意図的にスコープが設定されており、リポジトリの書き込みアクセスユーザと同じレベルのアクセス権を持っています。 トークンは各ジョブが開始する前に作成され、ジョブが終了すると期限切れになります。 詳しい情報については「[GITHUB_TOKENでの認証](/actions/configuring-and-managing-workflows/authenticating-with-the-github_token)」を参照してください。
    - 可能な場合は、常に `GITHUB_TOKEN` を使用する必要があります。
2. **リポジトリのデプロイキー**
    - デプロイキーは、単一のリポジトリへの読み取りまたは書き込みアクセスを許可する唯一の認証情報タイプの 1 つであり、ワークフロー内の別のリポジトリとのやり取りに使用できます。 詳しい情報については、「[デプロイキーを管理する](/developers/overview/managing-deploy-keys#deploy-keys)」を参照してください。
    - デプロイキーは Git を使用してリポジトリにクローンおよびプッシュできるだけであり、REST または GraphQL API とのやり取りには使用できないため、要件に適さない場合があることに注意してください。
3. **{% data variables.product.prodname_github_app %} トークン**
    - {% data variables.product.prodname_github_apps %} は、選択したリポジトリにインストールでき、リポジトリ内のリソースに対する詳細な権限を持つこともできます。 Organization の内部で {% data variables.product.prodname_github_app %} を作成し、ワークフロー内でアクセスする必要があるリポジトリにインストールして、それらのリポジトリにアクセスするためのワークフロー内のインストールとして認証できます。
4. **個人アクセストークン**
    - 自分のアカウントから個人アクセストークンを使用しないでください。 これらのトークンは、アクセスできる Organization 内のすべてのリポジトリ、およびユーザアカウントのすべての個人リポジトリへのアクセスを許可します。 これにより、ワークフローが含まれているリポジトリのすべての書き込みアクセスユーザに間接的に広範なアクセス権が付与されます。 さらに、後で Organization を離れると、このトークンを使用するワークフローはすぐに中断され、この問題のデバッグが困難になる場合があります。
    - 個人アクセストークンを使用する場合は、ワークフローに必要な特定のリポジトリへのアクセスのみが許可される新しいアカウント用に生成されたものを使用してください。 このアプローチはスケーラブルではないため、デプロイキーなどの代替案を優先して避ける必要があります。
5. **ユーザアカウントの SSH キー**
    - ワークフローでは、ユーザアカウントの SSH キーを使用しないでください。 これらは、個人アクセストークンと同様に、すべての個人リポジトリと、Organization のメンバーシップを通じてアクセスできるすべてのリポジトリに読み取り/書き込み権限を付与します。  これにより、ワークフローが含まれているリポジトリのすべての書き込みアクセスユーザに間接的に広範なアクセス権が付与されます。 リポジトリのクローンまたはプッシュのみを実行する必要があり、パブリック API とやり取りする必要がないため、SSH キーを使用する場合は、代わりに個別のデプロイキーを使用する必要があります。

### セルフホストランナーを強化する

**{% data variables.product.prodname_dotcom %} でホストされた**ランナーは、一過性でクリーンな隔離された仮想マシン内でコードを実行します。つまり、この環境を永続的に危険にさらしたり、ブートストラッププロセス中にこの環境に置かれた以上の情報にアクセスしたりする方法はありません。

{% data variables.product.product_name %} の**セルフホスト**ランナーは、一過性でクリーンな仮想マシンでの実行に関する保証がなく、ワークフロー内の信頼されていないコードによって永続的に侵害される可能性があります。

そのため、{% data variables.product.product_name %} の[パブリックリポジトリにセルフホストランナーを使用することはほとんどありません](/actions/hosting-your-own-runners/about-self-hosted-runners#self-hosted-runner-security-with-public-repositories)。これは、任意のユーザがリポジトリに対してプルリクエストを開き、環境を危険にさらす可能性があるためです。 同様に、プライベートリポジトリでセルフホストランナーを使用する場合は注意してください。リポジトリをフォークして PR を開くことができるユーザ（一般にリポジトリへの読み取りアクセス権を持つユーザ）は、シークレットへのアクセスと、リポジトリへの書き込みアクセス許可を付与する、より特権的な `GITHUB_TOKEN` を取得することを含め、セルフホストのランナー環境を危険にさらすことができるためです。

セルフホストランナーがOrganizationもしくはEnterpriseのレベルで定義されているなら、{% data variables.product.product_name %}は同じランナー上で複数のリポジトリからのワークフローをスケジューリングするかもしれません。 したがって、これらの環境へのセキュリティ侵害は、大きな影響をもたらす可能性があります。 侵害の範囲を狭めるために、セルフホストランナーを個別のグループにまとめることで、境界を作ることができます。 詳しい情報については、「[グループを使用したセルフホストランナーへのアクセスの管理](/actions/hosting-your-own-runners/managing-access-to-self-hosted-runners-using-groups)」を参照してください。

次のように、セルフホストランナーマシンの環境も考慮する必要があります。
- セルフホストランナーとして設定されたマシンにはどのような機密情報が存在するか。 たとえば、SSH 秘密鍵、API アクセストークンなどです。
- マシンが機密性の高いサービスにネットワークアクセス可能か。 たとえば、Azure または AWS メタデータサービスなどです。 この環境での機密情報量は最小限に抑える必要があります。ワークフローを呼び出すことができるすべてのユーザがこの環境にアクセスできることを常に意識しておいてください。

中には、それぞれのジョブの実行後にセルフホストランナーを自動的に破棄するようなシステムを実装することで、このリスクを部分的に軽減しようとするお客様もいます。 しかし、このアプローチは意図したほどには効果的ではないかもしれません。これは、セルフホストランナーが1つのジョブだけを実行するという保証がないためです。

### {% data variables.product.prodname_actions %}イベントの監査

Organizationの管理タスクをモニタするために、監査ログを使用できます。 監査ログは、アクションの種類、実行された時刻、実行したユーザアカウントを記録します。

たとえば、監査ログを使って`action:org.update_actions_secret`イベントを追跡できます。これは、Organizationのシークレットの変更を追跡します。 ![監査ログのエントリ](/assets/images/help/repository/audit-log-entries.png)

以下の表は、監査ログにある{% data variables.product.prodname_actions %}のイベントを示します。 For more information on using the audit log, see "[Reviewing the audit log for your organization](/github/setting-up-and-managing-organizations-and-teams/reviewing-the-audit-log-for-your-organization#searching-the-audit-log)."

#### シークレット管理のイベント
| アクション                               | 説明                                                                                                                                                                           |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action:org.create_actions_secret`  | Organizationの管理者が[{% data variables.product.prodname_actions %}シークレットを作成](/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-an-organization)したときにトリガーされます。 |
| `action:org.remove_actions_secret`  | Organizationの管理者が{% data variables.product.prodname_actions %}シークレットを削除したときにトリガーされます。                                                                                        |
| `action:org.update_actions_secret`  | Organizationの管理者が{% data variables.product.prodname_actions %}シークレットを更新したときにトリガーされます。                                                                                        |
| `action:repo.create_actions_secret` | リポジトリ管理者が[{% data variables.product.prodname_actions %}シークレットを作成](/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository)したときにトリガーされます。            |
| `action:repo.remove_actions_secret` | リポジトリ管理者が{% data variables.product.prodname_actions %}シークレットを削除したときにトリガーされます。                                                                                                |
| `action:repo.update_actions_secret` | リポジトリ管理者が{% data variables.product.prodname_actions %}シークレットを更新したときにトリガーされます。                                                                                                |

#### セルフホストランナーのイベント
| アクション                                     | 説明                                                                                                                                                              |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action:org.register_self_hosted_runner`  | Organizationのオーナーが[新しいセルフホストランナーを登録](/actions/hosting-your-own-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-an-organization)したときにトリガーされます。 |
| `action:org.remove_self_hosted_runner`    | Organizationのオーナーが[セルフホストランナーを削除](/actions/hosting-your-own-runners/removing-self-hosted-runners#removing-a-runner-from-an-organization)したときにトリガーされます。          |
| `action:repo.register_self_hosted_runner` | リポジトリ管理者が[新しいセルフホストランナーを登録](/actions/hosting-your-own-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-a-repository)したときにトリガーされます。             |
| `action:repo.remove_self_hosted_runner`   | リポジトリ管理者が[セルフホストランナーを削除](/actions/hosting-your-own-runners/removing-self-hosted-runners#removing-a-runner-from-a-repository)したときにトリガーされます。                      |

#### セルフホストランナーグループのイベント
| アクション                                     | 説明                                                                                                                                                                                                |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action:org.runner_group_created`         | Organizationの管理者が[セルフホストランナーグループを作成](/actions/hosting-your-own-runners/managing-access-to-self-hosted-runners-using-groups#creating-a-self-hosted-runner-group-for-an-organization)したときにトリガーされます。 |
| `action:org.runner_group_removed`         | Organizationの管理者がセルフホストランナーグループを削除したときにトリガーされます。                                                                                                                                                  |
| `action:org.runner_group_renamed`         | Organizationの管理者がセルフホストランナーグループの名前を変更したときにトリガーされます。                                                                                                                                               |
| `action:org.runner_group_runners_added`   | Organizationの管理者が[セルフホストランナーをグループに追加](/actions/hosting-your-own-runners/managing-access-to-self-hosted-runners-using-groups#moving-a-self-hosted-runner-to-a-group)したときにトリガーされます。                 |
| `action:org.runner_group_runners_removed` | Organizationの管理者がセルフホストランナーをグループから削除したときにトリガーされます。                                                                                                                                                |
