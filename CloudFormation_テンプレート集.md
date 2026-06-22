# CloudFormation テンプレート集

**用途**: 3つのセキュリティ対策の実装例  
**テンプレート言語**: YAML  
**デプロイ方法**: AWSコンソール（CloudFormation）でアップロード  
**推奨実装順序**: ③CodeCommit → ②S3 → ①IAM Identity Center

---

## CloudFormation のデプロイ手順（共通）

CloudFormation はすべてAWSコンソールからデプロイします。

```
1. AWSコンソール → 「CloudFormation」を検索 → CloudFormation を開く
2. 左メニュー「スタック」→ 右上「スタックの作成」→「新しいリソースを使用（標準）」
3. 「テンプレートの指定」→「テンプレートファイルのアップロード」
4. 「ファイルを選択」→ ローカルに保存した YAML ファイルを選択 → 「次へ」
5. スタック名を入力、パラメータを設定 → 「次へ」
6. IAM リソースの作成を確認する場合：「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」にチェック → 「送信」
7. 「イベント」タブで進捗を確認 → ステータスが「CREATE_COMPLETE」になれば完了
```

---

## ③ CodeCommit ブランチ/リポジトリ削除保護

### テンプレート（ローカルに保存して使用）

以下の内容を `codecommit-protection.yaml` として保存してください。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CodeCommit Repository & Branch Protection'

Parameters:
  RepositoryName:
    Type: String
    Default: my-secure-repo
    Description: CodeCommit Repository Name (保護対象のリポジトリ名)

Resources:
  # リポジトリ削除保護
  RepositoryDenyDeletePolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: !Sub 'DenyCodeCommitDelete-${RepositoryName}'
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: DenyDeleteRepository
            Effect: Deny
            Action:
              - codecommit:DeleteRepository
            Resource: !Sub 'arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:${RepositoryName}'

  # Force Push 防止
  DenyForcePushPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: !Sub 'DenyForcePush-${RepositoryName}'
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: DenyForcePush
            Effect: Deny
            Action:
              - codecommit:GitPush
            Resource: !Sub 'arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:${RepositoryName}'
            Condition:
              StringLike:
                'codecommit:References': 'refs/heads/main'

Outputs:
  DenyDeletePolicyArn:
    Value: !GetAtt RepositoryDenyDeletePolicy.PolicyArn
    Description: ARN of the deny delete policy (IAM ユーザー/グループにアタッチして使用)

  DenyForcePushPolicyArn:
    Value: !GetAtt DenyForcePushPolicy.PolicyArn
    Description: ARN of the deny force push policy
```

### コンソールでの実装手順

**Step 1: YAML ファイルを準備**
- 上記テンプレートをテキストエディタ（メモ帳、VS Code 等）にコピー
- `codecommit-protection.yaml` として保存

**Step 2: CloudFormation でスタック作成**

```
AWSコンソール
→ CloudFormation → スタック → スタックの作成
→ テンプレートファイルのアップロード → codecommit-protection.yaml を選択
→ スタック名: codecommit-protection-[リポジトリ名]
→ パラメータ「RepositoryName」に実際のリポジトリ名を入力
→ IAM 作成を確認するチェックボックスにチェック
→ 送信
```

**Step 3: 作成されたポリシーを開発者グループにアタッチ**

```
AWSコンソール → IAM → ポリシー
→ 「DenyCodeCommitDelete-[リポジトリ名]」を検索
→ アクション → アタッチ
→ 開発者が所属するグループを選択してアタッチ

同様に「DenyForcePush-[リポジトリ名]」もアタッチ
```

**Step 4: Approval Rule の設定（コンソール直接）**

CodeCommit の Approval Rule は CloudFormation でもできますが、コンソールの方が簡単です。

```
AWSコンソール → CodeCommit → リポジトリ名を選択
→ 設定（左メニュー）→「承認ルールテンプレート」
→「承認ルールテンプレートを作成」
→ 名前: protect-main-branch
→ 必要な承認数: 1
→ 承認者プール: arn:aws:iam::アカウントID:root
→ ブランチフィルター: refs/heads/main
→「作成」
→ 「リポジトリへの関連付け」から対象リポジトリを選択
```

**Step 5: 複数リポジトリへの適用**

リポジトリが10個ある場合、Step 2 を各リポジトリ名でくり返します。

```
スタック名の例:
- codecommit-protection-repo1
- codecommit-protection-repo2
- ...（合計10スタック）

パラメータの RepositoryName をリポジトリごとに変更するだけです。
```

---

## ②S3 オブジェクトロック

### 重要前提: 既存バケットへのロック追加は不可

S3 のオブジェクトロックは **バケット作成時のみ有効化できます**。  
現在の15個のバケットには後付けできないため、以下の手順が必要です。

```
現在の流れ:
既存バケット → 新規ロック対応バケットを作成 → オブジェクトをコピー → 切り替え
```

### テンプレート（ローカルに保存して使用）

以下の内容を `s3-object-lock.yaml` として保存してください。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'S3 Bucket with Object Lock'

Parameters:
  BucketName:
    Type: String
    Description: 新しいS3バケット名（グローバルでユニーク）

  LockMode:
    Type: String
    Default: GOVERNANCE
    AllowedValues:
      - GOVERNANCE
      - COMPLIANCE
    Description: 'GOVERNANCE=管理者が解除可能 / COMPLIANCE=誰も解除不可'

  RetentionDays:
    Type: Number
    Default: 30
    MinValue: 1
    MaxValue: 36500
    Description: オブジェクトの保持期間（日数）

Resources:
  SecureS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      ObjectLockEnabled: true
      VersioningConfiguration:
        Status: Enabled
      ObjectLockConfiguration:
        ObjectLockEnabled: Enabled
        Rule:
          DefaultRetention:
            Mode: !Ref LockMode
            Days: !Ref RetentionDays
      Tags:
        - Key: Security
          Value: ObjectLocked

Outputs:
  BucketName:
    Value: !Ref SecureS3Bucket

  ObjectLockMode:
    Value: !Ref LockMode

  RetentionDays:
    Value: !Ref RetentionDays
```

### コンソールでの実装手順

**Phase 1: 新しいロック対応バケットを作成（CloudFormation）**

```
AWSコンソール → CloudFormation → スタックの作成
→ s3-object-lock.yaml をアップロード
→ スタック名: s3-lock-[既存バケット名]
→ パラメータ設定:
   BucketName: [既存バケット名]-secured（既存と別名にする）
   LockMode: GOVERNANCE
   RetentionDays: 30
→ 送信 → CREATE_COMPLETE を確認

※ バケット15個分くり返す（15スタック作成）
```

**Phase 2: 既存バケットからオブジェクトをコピー（S3 Batch Operations）**

大量のオブジェクトをコピーする場合は **S3 Batch Operations** を使います。

```
事前準備: S3 インベントリレポートの設定
AWSコンソール → S3 → 既存バケットを選択
→「管理」タブ →「インベントリ設定」→「インベントリ設定を作成」
→ 名前: inventory-for-migration
→ 保存先バケット: 任意のバケット（ログ用）
→ フォーマット: CSV
→ 頻度: 毎日
→「保存」→ 翌日にレポートが生成されるのを待つ

（インベントリなしでコピーする場合は次の方法）
```

インベントリを待てない場合は **S3コンソールから直接コピー** できます（少数オブジェクトの場合）:

```
AWSコンソール → S3 → 既存バケット
→ オブジェクトを選択（チェックボックス）
→「アクション」→「コピー」
→ コピー先: 新しいロック対応バケット
→「コピー」
```

**Phase 3: オブジェクトロックが適用されているか確認（コンソール）**

```
AWSコンソール → S3 → 新しいロック対応バケット
→ オブジェクトを1つクリック
→「プロパティ」タブ
→「オブジェクトロック」セクションを確認
→ 保持モードと保持期限日が表示されていれば OK
```

**Phase 4: バケット切り替え（コンソール）**

アプリケーションのバケット名を新バケット名に変更する作業です。  
この作業はシステムによって異なるので、開発チームと連携してください。

---

## ①IAM Identity Center への移行

### CloudFormation 対応状況

| 設定項目 | コンソール作業 | CloudFormation |
|---------|--------------|---------------|
| IDC 有効化 | ✅ 必須（コンソールのみ） | ❌ 非対応 |
| PermissionSet 作成 | - | ✅ 対応 |
| ユーザー/グループ作成 | ✅ コンソール推奨 | △ 部分的 |
| ユーザーへのPermissionSet割り当て | ✅ コンソール推奨 | △ 部分的 |

**IDC インスタンスの有効化は必ずコンソール最初のステップで行います。**

### Phase 1: IDC の有効化（コンソール）

```
AWSコンソール → 「IAM Identity Center」を検索
→ 「IAM Identity Center を有効にする」ボタンをクリック
→ 「有効にする」を確認

※ 組織（AWS Organizations）を使っている場合は組織全体で有効化するか確認が出る
※ シングルアカウントの場合はそのまま有効化
```

有効化後、左メニューに「ダッシュボード」が表示されることを確認。

### Phase 2: PermissionSet 作成（CloudFormation）

以下の内容を `idc-permission-sets.yaml` として保存してください。

**事前確認: SSO Instance ARN の取得**
```
AWSコンソール → IAM Identity Center → 設定（左メニュー）
→「インスタンスの詳細」セクションの「インスタンス ARN」をコピー
例: arn:aws:sso:::instance/ssoins-xxxxxxxxxx
```

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'IAM Identity Center Permission Sets with 15-minute session'

Parameters:
  SSOInstanceArn:
    Type: String
    Description: IAM Identity Center のインスタンス ARN（コンソールの設定画面から取得）

Resources:
  # 開発者用 PermissionSet（セッション15分）
  DeveloperPermissionSet:
    Type: AWS::SSO::PermissionSet
    Properties:
      Name: Developer
      Description: '開発者用（セッション15分）'
      InstanceArn: !Ref SSOInstanceArn
      SessionDuration: PT15M   # ISO 8601 形式: 15分
      # 既存のカスタムポリシー5個をここにインラインで記載するか
      # ManagedPolicies に既存ポリシーのARNを列挙する

  # ポリシー1個目（例）
  AdminPermissionSet:
    Type: AWS::SSO::PermissionSet
    Properties:
      Name: Admin
      Description: '管理者用（セッション15分）'
      InstanceArn: !Ref SSOInstanceArn
      SessionDuration: PT15M
      ManagedPolicies:
        - arn:aws:iam::aws:policy/AdministratorAccess

Outputs:
  DeveloperPermissionSetArn:
    Value: !GetAtt DeveloperPermissionSet.PermissionSetArn

  AdminPermissionSetArn:
    Value: !GetAtt AdminPermissionSet.PermissionSetArn
```

**CloudFormation でデプロイ**

```
AWSコンソール → CloudFormation → スタックの作成
→ idc-permission-sets.yaml をアップロード
→ スタック名: idc-permission-sets
→ パラメータ「SSOInstanceArn」に先ほどコピーした ARN を貼り付け
→ 送信 → CREATE_COMPLETE を確認
```

### Phase 3: グループ・ユーザー作成（コンソール）

```
AWSコンソール → IAM Identity Center
→ 左メニュー「グループ」→「グループを作成」
→ グループ名: Developers（既存の IAM グループ名に合わせる）
→「グループを作成」

→ 左メニュー「ユーザー」→「ユーザーを追加」
→ ユーザー名・メールアドレスを入力
→ グループ「Developers」に追加
→「送信」（招待メールが送信される）

※ 既存 IAM ユーザーの数だけくり返す
```

### Phase 4: PermissionSet をアカウントに割り当て（コンソール）

```
AWSコンソール → IAM Identity Center
→ 左メニュー「AWSアカウント」
→ 割り当て対象のAWSアカウントを選択
→「ユーザーとグループを割り当て」
→「グループ」タブ → Developers グループを選択 → 「次へ」
→ PermissionSet「Developer」を選択 → 「次へ」
→「送信」

※ ここで初めて「セッション15分」の制限が適用される
```

### Phase 5: テスト（コンソール）

```
テスト用アクセスポータルURL の確認:
AWSコンソール → IAM Identity Center → ダッシュボード
→「AWSアクセスポータルのURL」をコピー（例: https://xxxx.awsapps.com/start）

テスト手順:
1. ブラウザのシークレットウィンドウでポータルURLを開く
2. 招待されたユーザー情報でログイン
3. AWSアカウントとロールが表示されることを確認
4. コンソールにアクセス → 15分後に自動ログアウトされることを確認
```

---

## 📋 まとめ: 実装順序と時間目安

| # | 対策 | コンソール作業 | CF作業 | 実装時間 |
|---|-----|------------|-------|--------|
| 1️⃣ | CodeCommit 保護 | ポリシーのアタッチ・Approval Rule | ✅ 主体 | 4-6時間 |
| 2️⃣ | S3 オブジェクトロック | バケット切り替え・オブジェクトコピー | ✅ 主体 | 6-8時間 |
| 3️⃣ | IAM Identity Center | IDC有効化・ユーザー作成・割り当て | PermissionSet のみ | 3-5日間 |
