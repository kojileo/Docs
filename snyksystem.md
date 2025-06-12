# Azure DevOps + Snyk 脆弱性解析システム構築手順書

本手順書では、Azure DevOpsでSnykを活用した脆弱性解析システムを、コスト最適化とセキュリティ要件を満たしながら構築する方法を説明します。

## システム構成概要

```
製品コードリポジトリ (課金対象外)
    ↓ ミラーリング
セキュリティプロジェクト (Snyk課金対象)
    ↓ 脆弱性解析
Snyk WebUI / Pipeline解析
    ↓ 結果通知
PR マージ制御
```

## 前提条件

- Azure DevOps組織の管理者権限
- Snykアカウントとプロジェクト作成権限
- 製品コードを管理している既存のAzure DevOpsプロジェクト

---

## 手順1: セキュリティプロジェクトの作成

### 1.1 新しいプロジェクトの作成

1. Azure DevOps組織にアクセス
2. 「New project」をクリック
3. プロジェクト設定：
   - **Project name**: `SecurityCheck` (任意の名前)
   - **Visibility**: Private
   - **Version control**: Git
   - **Work item process**: Agile (任意)

### 1.2 セキュリティリポジトリの作成

1. 作成したプロジェクト内で「Repos」に移動
2. 「New repository」をクリック
3. リポジトリ設定：
   - **Repository name**: `security-check-[ADOプロジェクト]-[Repo名]`
   - **Add a README**: チェック
   - **Add .gitignore**: None

---

## 手順2: Variable Groupの設定

### 2.1 Snyk APIトークンの取得

1. [Snyk Web Console](https://app.snyk.io) にログイン
2. 「Settings」→「General」→「Auth Token」
3. 「Generate token」でAPIトークンを生成・コピー

### 2.2 Variable Groupの作成

1. セキュリティプロジェクトの「Pipelines」→「Library」に移動
2. 「+ Variable group」をクリック
3. Variable group設定：
   - **Variable group name**: `snyk-security-variables`
   - 以下の変数を追加：

| Variable name | Value | Secure |
|---------------|--------|---------|
| `snyk-api-token` | (手順2.1で取得したトークン) | ✅ |
| `COMMIT_USER_EMAIL` | `pipeline@azuredevops.com` | ❌ |
| `COMMIT_USER_NAME` | `Azure DevOps Pipeline` | ❌ |

### 修正版手順3: セキュリティプロジェクトのパイプライン作成

#### 3.1 リポジトリを空の状態で作成

1. セキュリティプロジェクトの「Repos」→「New repository」
2. リポジトリ設定：
   - **Repository name**: `product-code-mirror`
   - **Add a README**: ❌ **チェックしない**（重要）
   - **Add .gitignore**: None

#### 3.2 空リポジトリに初期ファイルを直接追加

**以下のファイルを製品リポジトリ側で作成後、セキュリティプロジェクトに手動アップロード：**

**ファイル名**: `snyk-security-analysis.yml`

```yaml
# このファイルは製品リポジトリで作成し、セキュリティプロジェクトにアップロードする
# snyk-security-analysis.yml
# Snyk脆弱性解析を実行し、結果を元のPRにコメントするパイプライン

trigger: none  # 手動トリガーのみ

parameters:
- name: sourceBranch
  type: string
  displayName: 'ソースブランチ名'
- name: originalPRNumber
  type: string
  displayName: '元PRの番号'
- name: originalProject
  type: string
  displayName: '元プロジェクト名'
- name: originalRepo
  type: string
  displayName: '元リポジトリ名'

variables:
- group: snyk-security-variables

jobs:
- job: SnykAnalysis
  displayName: 'Snyk脆弱性解析ジョブ'
  pool:
    vmImage: 'ubuntu-latest'
    
  steps:
  - checkout: self
    displayName: 'ソースコードチェックアウト'
    ref: ${{ parameters.sourceBranch }}
    
  - task: NodeTool@0
    displayName: 'Node.js環境セットアップ'
    inputs:
      versionSpec: '18.x'
      
  - script: |
      echo "Snyk CLIをインストール中..."
      npm install -g snyk
      echo "Snyk認証設定中..."
      snyk auth $(snyk-api-token)
      echo "Snyk設定完了"
    displayName: 'Snyk CLIインストール・認証'
    
  - script: |
      echo "依存関係の脆弱性スキャンを開始..."
      snyk test --severity-threshold=critical --json > snyk-results.json
      echo $? > snyk-exit-code.txt
      echo "依存関係スキャン完了"
    displayName: '依存関係脆弱性スキャン'
    continueOnError: true
    
  - script: |
      echo "Dockerfileの存在チェック中..."
      if [ -f "Dockerfile" ]; then
        echo "Dockerfileが見つかりました。コンテナスキャンを開始..."
        snyk container test . --severity-threshold=critical --json > snyk-container-results.json
        echo $? > snyk-container-exit-code.txt
        echo "コンテナスキャン完了"
      else
        echo "Dockerfileが見つかりません。コンテナスキャンをスキップ"
        echo "0" > snyk-container-exit-code.txt
        echo "{}" > snyk-container-results.json
      fi
    displayName: 'コンテナ脆弱性スキャン'
    continueOnError: true
    
  - task: PowerShell@2
    displayName: 'スキャン結果評価・PRコメント投稿'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "スキャン結果の評価を開始..."
        
        $exitCode = [int](Get-Content snyk-exit-code.txt -ErrorAction SilentlyContinue)
        $containerExitCode = [int](Get-Content snyk-container-exit-code.txt -ErrorAction SilentlyContinue)
        
        Write-Host "依存関係スキャン終了コード: $exitCode"
        Write-Host "コンテナスキャン終了コード: $containerExitCode"
        
        try {
          $results = Get-Content snyk-results.json -ErrorAction SilentlyContinue | ConvertFrom-Json
          $containerResults = Get-Content snyk-container-results.json -ErrorAction SilentlyContinue | ConvertFrom-Json
        } catch {
          Write-Warning "結果ファイルの解析に失敗: $_"
          $results = @{}
          $containerResults = @{}
        }
        
        $criticalVulns = 0
        $highVulns = 0
        $mediumVulns = 0
        $lowVulns = 0
        
        if ($results.vulnerabilities) {
          $criticalVulns += ($results.vulnerabilities | Where-Object { $_.severity -eq "critical" }).Count
          $highVulns += ($results.vulnerabilities | Where-Object { $_.severity -eq "high" }).Count
          $mediumVulns += ($results.vulnerabilities | Where-Object { $_.severity -eq "medium" }).Count
          $lowVulns += ($results.vulnerabilities | Where-Object { $_.severity -eq "low" }).Count
        }
        
        if ($containerResults.vulnerabilities) {
          $criticalVulns += ($containerResults.vulnerabilities | Where-Object { $_.severity -eq "critical" }).Count
          $highVulns += ($containerResults.vulnerabilities | Where-Object { $_.severity -eq "high" }).Count
          $mediumVulns += ($containerResults.vulnerabilities | Where-Object { $_.severity -eq "medium" }).Count
          $lowVulns += ($containerResults.vulnerabilities | Where-Object { $_.severity -eq "low" }).Count
        }
        
        Write-Host "脆弱性カウント - Critical: $criticalVulns, High: $highVulns, Medium: $mediumVulns, Low: $lowVulns"
        
        $statusIcon = if ($criticalVulns -gt 0) { "❌" } else { "✅" }
        $statusMessage = if ($criticalVulns -gt 0) { "**Critical脆弱性が検出されました。マージをブロックします。**" } else { "Critical脆弱性は検出されませんでした。" }
        
        $comment = @"
        ## 🔒 セキュリティ脆弱性解析結果
        
        $statusIcon $statusMessage
        
        ### 脆弱性サマリー
        - **Critical:** $criticalVulns 件
        - **High:** $highVulns 件
        - **Medium:** $mediumVulns 件
        - **Low:** $lowVulns 件
        
        ### スキャン詳細
        - 依存関係スキャン: $(if ($exitCode -eq 0) { "✅ 正常完了" } else { "⚠️ 脆弱性検出" })
        - コンテナスキャン: $(if ($containerExitCode -eq 0) { "✅ 正常完了" } else { "⚠️ 脆弱性検出" })
        
        詳細な情報はSnykダッシュボードで確認してください。
        
        ---
        *この結果は自動生成されました。*
        "@
        
        Write-Host "PRコメントを投稿中..."
        
        $headers = @{
          'Authorization' = "Bearer $(System.AccessToken)"
          'Content-Type' = 'application/json'
        }
        
        $body = @{
          comments = @(
            @{
              parentCommentId = 0
              content = $comment
              commentType = 1
            }
          )
          status = 1
        } | ConvertTo-Json -Depth 3
        
        # ★要変更: 実際の組織名に置き換え
        $orgUrl = "https://dev.azure.com/your-org"
        $uri = "$orgUrl/${{ parameters.originalProject }}/_apis/git/repositories/${{ parameters.originalRepo }}/pullRequests/${{ parameters.originalPRNumber }}/threads?api-version=6.0"
        
        try {
          Invoke-RestMethod -Uri $uri -Method Post -Headers $headers -Body $body
          Write-Host "PRコメントの投稿が成功しました"
        } catch {
          Write-Warning "PRコメントの投稿に失敗しました: $_"
          Write-Host "URI: $uri"
        }
        
        if ($criticalVulns -gt 0) {
          Write-Host "##vso[task.logissue type=error]Critical脆弱性が $criticalVulns 件検出されました"
          Write-Host "##vso[task.complete result=Failed;]Critical脆弱性によりマージがブロックされました"
          exit 1
        }
        
        Write-Host "セキュリティ解析が正常に完了しました"
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

#### 3.3 完全クリーンな初期化方式

**修正版の初期セットアップパイプライン:**

```yaml
# azure-pipelines-setup-mirror.yml (完全クリーン版)
# セキュリティミラーリポジトリを完全にクリーンな状態で初期化

trigger: none

variables:
  # ★要変更: 実際のセキュリティプロジェクトURLに置き換え
  securityProjectUrl: 'https://dev.azure.com/your-org/security-analysis'
  mirrorRepoName: 'product-code-mirror'
  COMMIT_USER_EMAIL: 'pipeline@azuredevops.com'
  COMMIT_USER_NAME: 'Azure DevOps Pipeline'

jobs:
- job: CompleteCleanSetup
  displayName: '完全クリーンセットアップ'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  - checkout: self
    displayName: '製品コードチェックアウト'
    persistCredentials: true
    
  - task: PowerShell@2
    displayName: '完全クリーンな履歴でセキュリティリポジトリ初期化'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "完全クリーンなセキュリティリポジトリ初期化を開始..."
        
        # Git設定
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        
        try {
          # 一時ディレクトリでクリーンなリポジトリを作成
          $tempDir = "clean-security-repo"
          Write-Host "一時ディレクトリでクリーンリポジトリを作成: $tempDir"
          
          New-Item -ItemType Directory -Path $tempDir -Force
          Set-Location $tempDir
          
          # 新しいGitリポジトリを初期化
          git init
          git config user.email "$(COMMIT_USER_EMAIL)"
          git config user.name "$(COMMIT_USER_NAME)"
          
          # セキュリティプロジェクト用のパイプラインファイルを作成
          Write-Host "セキュリティ解析用パイプラインファイルを作成中..."
          
          # 製品リポジトリからsnyk-security-analysis.ymlをコピー
          $pipelineFile = "$(Build.SourcesDirectory)/snyk-security-analysis.yml"
          if (Test-Path $pipelineFile) {
            Copy-Item $pipelineFile "azure-pipelines.yml"
            Write-Host "パイプラインファイルをコピー完了"
          } else {
            # パイプラインファイルが存在しない場合は最小限のファイルを作成
            @"
# Minimal pipeline for security repository
trigger: none
pool:
  vmImage: 'ubuntu-latest'
steps:
- script: echo "Security repository initialized"
"@ | Out-File "azure-pipelines.yml" -Encoding UTF8
            Write-Host "最小限のパイプラインファイルを作成"
          }
          
          # 製品コードをコピー
          Write-Host "製品コードをコピー中..."
          $sourceDir = "$(Build.SourcesDirectory)"
          Get-ChildItem $sourceDir -Exclude @(".git", $tempDir) | Copy-Item -Destination . -Recurse -Force
          
          # 初期コミット（パイプラインユーザーのみ）
          Write-Host "初期コミットを作成中..."
          git add .
          $commitMessage = "Initial security repository setup - Build $(Build.BuildNumber)"
          git commit -m $commitMessage
          
          # リモートを追加してプッシュ（既存履歴を完全に上書き）
          Write-Host "セキュリティリポジトリに完全上書きプッシュ中..."
          git remote add origin $securityRepoUrl
          git push origin main --force
          
          Write-Host "✅ 完全クリーンなセキュリティリポジトリ初期化完了"
          
          # 履歴確認
          Write-Host "=== 初期化後のコミット履歴 ==="
          git log --oneline --format="%h %an <%ae> %s" -n 5
          Write-Host "================================"
          
        } catch {
          Write-Error "❌ 完全クリーン初期化でエラーが発生: $_"
          throw
        } finally {
          # クリーンアップ
          Set-Location "$(Build.SourcesDirectory)"
          if (Test-Path $tempDir) {
            Remove-Item $tempDir -Recurse -Force -ErrorAction SilentlyContinue
          }
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

## 修正版手順のポイント

### ✅ 完全にクリーンな履歴を保証

1. **空リポジトリ作成**: README等を自動作成しない
2. **一時ディレクトリ方式**: 既存履歴の影響を受けない新しいGitリポジトリを作成
3. **Force Push**: 既存の履歴（もしあれば）を完全に上書き
4. **パイプラインユーザーのみ**: 最初から最後まで `pipeline@azuredevops.com` のみ

### ✅ セキュリティプロジェクトでのパイプライン作成手順

1. **セキュリティプロジェクトのパイプライン作成時**:
   - 「Existing Azure Pipelines YAML file」を選択
   - `/azure-pipelines.yml` を選択（初期化時に作成済み）
   - **新しいYAMLファイルは作成しない**

2. **履歴の確認**:
   ```bash
   git log --format="%an <%ae> %s" --all
   ```
   すべてのコミットが `Azure DevOps Pipeline <pipeline@azuredevops.com>` になる

### ✅ 検証方法

初期化完了後、以下で履歴を確認：

```powershell
# セキュリティリポジトリで実行
git log --oneline --format="%h %an <%ae> %s"
```

**期待する出力例:**
```
a1b2c3d Azure DevOps Pipeline <pipeline@azuredevops.com> Initial security repository setup - Build 20250612.1
```

## 手順4: 製品リポジトリでのパイプライン設定

### 4.1 初期ミラーリングパイプラインの作成

製品コードリポジトリで以下のファイルを作成：

**ファイル名**: `azure-pipelines-mirror.yml`

```yaml
# azure-pipelines-mirror.yml
# 製品コードをセキュリティプロジェクトにミラーリングするパイプライン
# mainブランチやdevelopブランチの更新時に自動実行される

# トリガー設定：指定ブランチの更新時に実行
trigger:
  branches:
    include:
    - main      # mainブランチの更新を監視
    - develop   # developブランチの更新を監視
  paths:
    exclude:
    - README.md    # READMEの更新はトリガー対象外
    - docs/*       # ドキュメントの更新はトリガー対象外

# パイプライン設定変数
variables:
  # ★要変更: 実際のセキュリティプロジェクトURLに置き換え
  securityProjectUrl: 'https://dev.azure.com/your-org/security-analysis'
  mirrorRepoName: 'product-code-mirror'
  COMMIT_USER_EMAIL: 'pipeline@azuredevops.com'    # ミラーリポジトリのコミットユーザーEmail
  COMMIT_USER_NAME: 'Azure DevOps Pipeline'        # ミラーリポジトリのコミットユーザー名

jobs:
- job: MirrorToSecurityRepo
  displayName: 'セキュリティリポジトリへのミラーリング'
  pool:
    vmImage: 'ubuntu-latest'  # Ubuntu最新版を使用
  
  steps:
  # ソースコードをチェックアウト（認証情報を保持）
  - checkout: self
    displayName: '製品コードチェックアウト'
    persistCredentials: true  # git操作に必要な認証情報を保持
    
  # Gitユーザー設定（ミラーリポジトリでのコミット用）
  - task: PowerShell@2
    displayName: 'Gitユーザー設定'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "Gitユーザー設定を開始..."
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        Write-Host "Gitユーザー設定完了"
        Write-Host "Email: $(COMMIT_USER_EMAIL)"
        Write-Host "Name: $(COMMIT_USER_NAME)"
        
  # セキュリティリポジトリへのミラーリング実行
  - task: PowerShell@2
    displayName: 'セキュリティリポジトリミラーリング'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "ミラーリング処理を開始..."
        
        # セキュリティリポジトリのURL構築
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        
        try {
          # セキュリティリポジトリをクローン
          Write-Host "セキュリティリポジトリをクローン中..."
          git clone $securityRepoUrl security-repo
          Set-Location security-repo
          
          # 現在のブランチ名を取得
          $sourceBranch = "$(Build.SourceBranchName)"
          Write-Host "ソースブランチ: $sourceBranch"
          
          # 対象ブランチに切り替え（存在しない場合は作成）
          try {
            git checkout $sourceBranch
            Write-Host "既存ブランチ '$sourceBranch' にチェックアウト"
          } catch {
            git checkout -b $sourceBranch
            Write-Host "新しいブランチ '$sourceBranch' を作成"
          }
          
          # 既存の内容をクリア（.gitディレクトリは除く）
          Write-Host "既存ファイルをクリア中..."
          Get-ChildItem -Force -Exclude ".git" | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
          
          # 製品コードをセキュリティリポジトリにコピー
          Write-Host "製品コードをコピー中..."
          $sourceDir = "$(Build.SourcesDirectory)"
          Copy-Item -Path "$sourceDir/*" -Destination . -Recurse -Force -Exclude @(".git")
          
          # 変更があるかチェック
          git add .
          $changes = git status --porcelain
          
          if ($changes) {
            # 変更がある場合のみコミット・プッシュ
            $commitMessage = "Mirror update from $(Build.Repository.Name) - $(Build.SourceBranch) (Build: $(Build.BuildNumber))"
            Write-Host "変更を検出。コミット中..."
            Write-Host "コミットメッセージ: $commitMessage"
            
            git commit -m $commitMessage
            git push origin $sourceBranch --force
            Write-Host "✅ ミラーリングが正常に完了しました"
          } else {
            Write-Host "変更が検出されませんでした。ミラーリングをスキップします"
          }
          
        } catch {
          Write-Error "❌ ミラーリングでエラーが発生しました: $_"
          throw
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)  # Azure DevOps認証トークン
```

### 4.2 PR用セキュリティチェックパイプラインの作成

**ファイル名**: `azure-pipelines-pr-security.yml`

```yaml
# azure-pipelines-pr-security.yml
# プルリクエスト時にセキュリティチェックを実行するパイプライン
# PR作成時に自動実行され、Snyk解析結果に基づいてマージを制御する

# トリガー設定
trigger: none  # 通常のCI/CDトリガーは無効

# プルリクエストトリガー設定
pr:
  branches:
    include:
    - main      # mainブランチへのPRを監視
    - develop   # developブランチへのPRを監視

# パイプライン設定変数
variables:
  # ★要変更: 実際のセキュリティプロジェクトURLに置き換え
  securityProjectUrl: 'https://dev.azure.com/your-org/security-analysis'
  mirrorRepoName: 'product-code-mirror'
  COMMIT_USER_EMAIL: 'pipeline@azuredevops.com'    # ミラーリポジトリのコミットユーザーEmail
  COMMIT_USER_NAME: 'Azure DevOps Pipeline'        # ミラーリポジトリのコミットユーザー名

jobs:
- job: SecurityCheck
  displayName: 'セキュリティ脆弱性チェック'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  # PRのソースコードをチェックアウト
  - checkout: self
    displayName: 'PRソースコードチェックアウト'
    persistCredentials: true
    
  # Gitユーザー設定
  - task: PowerShell@2
    displayName: 'Gitユーザー設定'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "Gitユーザー設定を開始..."
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        Write-Host "Gitユーザー設定完了"
        
  # PRブランチをセキュリティリポジトリにミラー
  - task: PowerShell@2
    displayName: 'PRブランチミラーリング'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "PRブランチミラーリングを開始..."
        
        # PR情報の取得
        $prNumber = "$(System.PullRequest.PullRequestNumber)"
        $branchName = "pr-$prNumber"
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        
        Write-Host "PR番号: $prNumber"
        Write-Host "ミラーブランチ名: $branchName"
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        
        try {
          # セキュリティリポジトリをクローン
          Write-Host "セキュリティリポジトリをクローン中..."
          git clone $securityRepoUrl security-repo
          Set-Location security-repo
          
          # PRブランチを作成または切り替え
          Write-Host "PRブランチ '$branchName' を作成・切り替え中..."
          git checkout -B $branchName
          
          # 既存の内容をクリア
          Write-Host "既存ファイルをクリア中..."
          Get-ChildItem -Force -Exclude ".git" | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
          
          # PRの内容をコピー
          Write-Host "PRコードをコピー中..."
          $sourceDir = "$(Build.SourcesDirectory)"
          Copy-Item -Path "$sourceDir/*" -Destination . -Recurse -Force -Exclude @(".git")
          
          # 変更をコミット
          Write-Host "変更をコミット中..."
          git add .
          $changes = git status --porcelain
          
          # 変更がある場合や新しいブランチの場合はコミット
          if ($changes -or !(git rev-parse --verify $branchName 2>$null)) {
            $commitMessage = "PR #$prNumber mirror from $(Build.Repository.Name) (Build: $(Build.BuildNumber))"
            Write-Host "コミットメッセージ: $commitMessage"
            
            git commit -m $commitMessage --allow-empty
            git push origin $branchName --force
            Write-Host "✅ PRブランチミラーリング完了: $branchName"
          } else {
            Write-Host "変更が検出されませんでした: $branchName"
          }
          
          # 後続処理用に変数設定
          echo "##vso[task.setvariable variable=prBranchName]$branchName"
          
        } catch {
          Write-Error "❌ PRブランチミラーリングでエラーが発生: $_"
          throw
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
      
  # セキュリティプロジェクトのSnyk解析パイプラインをトリガー
  - task: TriggerBuild@4
    displayName: 'Snyk解析パイプライントリガー'
    inputs:
      definitionIsInCurrentTeamProject: false              # 他プロジェクトのパイプライン
      tfsServer: '$(securityProjectUrl)'                   # セキュリティプロジェクトURL
      teamProject: 'security-analysis'                     # ★要変更: セキュリティプロジェクト名
      buildDefinition: 'snyk-security-analysis'           # Snyk解析パイプライン名
      queueBuildForUserThatTriggeredBuild: true           # 同じユーザーで実行
      buildParameters: |
        sourceBranch: $(prBranchName)
        originalPRNumber: $(System.PullRequest.PullRequestNumber)
        originalProject: $(System.TeamProject)
        originalRepo: $(Build.Repository.Name)
      waitForQueuedBuildsToFinish: true                   # パイプライン完了まで待機
      treatPartiallySucceededBuildAsSuccessful: false     # 部分成功も失敗として扱う
      authenticationMethod: 'OAuth Token'                 # 認証方式
      password: '$(System.AccessToken)'                   # 認証トークン
```

### 4.3 初期セットアップパイプラインの作成

**ファイル名**: `azure-pipelines-setup-mirror.yml`

```yaml
# azure-pipelines-setup-mirror.yml
# セキュリティミラーリポジトリの初期セットアップ用パイプライン
# 一回限りの実行で、ミラーリポジトリの履歴をリセットしてパイプラインユーザーのみの履歴で開始

# 手動実行のみ（自動トリガーなし）
trigger: none

# パイプライン設定変数
variables:
  # ★要変更: 実際のセキュリティプロジェクトURLに置き換え
  securityProjectUrl: 'https://dev.azure.com/your-org/security-analysis'
  mirrorRepoName: 'product-code-mirror'
  COMMIT_USER_EMAIL: 'pipeline@azuredevops.com'
  COMMIT_USER_NAME: 'Azure DevOps Pipeline'

jobs:
- job: SetupSecurityMirror
  displayName: 'セキュリティミラーリポジトリ初期セットアップ'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  # 製品コードをチェックアウト
  - checkout: self
    displayName: '製品コードチェックアウト'
    persistCredentials: true
    
  # セキュリティミラーリポジトリの初期化
  - task: PowerShell@2
    displayName: 'セキュリティミラーリポジトリ初期化'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "セキュリティミラーリポジトリの初期化を開始..."
        
        # Git設定
        Write-Host "Gitユーザー設定中..."
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        
        # セキュリティリポジトリのURL
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        
        try {
          # セキュリティリポジトリをクローン
          Write-Host "セキュリティリポジトリをクローン中..."
          git clone $securityRepoUrl security-repo
          Set-Location security-repo
          
          # 既存の履歴を削除して新しい履歴を開始
          Write-Host "既存のGit履歴を削除中..."
          Remove-Item -Path ".git" -Recurse -Force -ErrorAction SilentlyContinue
          
          # 新しいGitリポジトリとして初期化
          Write-Host "新しいGitリポジトリとして初期化中..."
          git init
          git config user.email "$(COMMIT_USER_EMAIL)"
          git config user.name "$(COMMIT_USER_NAME)"
          
          # リモートリポジトリを設定
          git remote add origin $securityRepoUrl
          
          # 既存ファイルをクリア
          Write-Host "既存ファイルをクリア中..."
          Get-ChildItem -Force -Exclude ".git" | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
          
          # 現在のソースコードをコピー
          Write-Host "製品コードをコピー中..."
          $sourceDir = "$(Build.SourcesDirectory)"
          Copy-Item -Path "$sourceDir/*" -Destination . -Recurse -Force -Exclude @(".git")
          
          # 初期コミット
          Write-Host "初期コミットを作成中..."
          git add .
          $commitMessage = "Initial mirror setup from $(Build.Repository.Name) (Build: $(Build.BuildNumber))"
          git commit -m $commitMessage
          
          # mainブランチにプッシュ（履歴を上書き）
          Write-Host "リモートリポジトリにプッシュ中..."
          git push origin main --force
          
          Write-Host "✅ セキュリティミラーリポジトリの初期化が正常に完了しました"
          
        } catch {
          Write-Error "❌ 初期化でエラーが発生しました: $_"
          throw
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

### 4.4 クリーンアップパイプラインの作成

**ファイル名**: `azure-pipelines-cleanup.yml`

```yaml
# azure-pipelines-cleanup.yml
# セキュリティリポジトリの古いPRブランチを定期的にクリーンアップするパイプライン
# 週1回実行され、7日以上古いPRブランチを自動削除してリポジトリサイズを最適化

# 定期実行スケジュール設定
schedules:
- cron: "0 2 * * 0"  # 毎週日曜日 2:00 AM (UTC) に実行
  displayName: 週次クリーンアップ
  branches:
    include:
    - main  # mainブランチでのみスケジュール実行
  always: true  # 変更がなくても実行

# 手動トリガーは無効
trigger: none

# パイプライン設定変数
variables:
  # ★要変更: 実際のセキュリティプロジェクトURLに置き換え
  securityProjectUrl: 'https://dev.azure.com/your-org/security-analysis'
  mirrorRepoName: 'product-code-mirror'
  COMMIT_USER_EMAIL: 'pipeline@azuredevops.com'
  COMMIT_USER_NAME: 'Azure DevOps Pipeline'
  # クリーンアップ対象の日数（この日数より古いブランチを削除）
  CLEANUP_DAYS: 7

jobs:
- job: CleanupSecurityRepo
  displayName: '古いPRブランチクリーンアップ'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  # 古いPRブランチの削除処理
  - task: PowerShell@2
    displayName: '古いPRブランチ削除'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "古いPRブランチのクリーンアップを開始..."
        
        # Git設定
        Write-Host "Gitユーザー設定中..."
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        
        # セキュリティリポジトリのURL
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        $cleanupDays = [int]"$(CLEANUP_DAYS)"
        $cutoffDate = (Get-Date).AddDays(-$cleanupDays)
        
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        Write-Host "クリーンアップ基準日: $($cutoffDate.ToString('yyyy-MM-dd HH:mm:ss'))"
        Write-Host "削除対象: $cleanupDays 日以上古いPRブランチ"
        
        try {
          # セキュリティリポジトリをクローン
          Write-Host "セキュリティリポジトリをクローン中..."
          git clone $securityRepoUrl security-repo
          Set-Location security-repo
          
          # リモートブランチ一覧を取得
          Write-Host "リモートブランチ一覧を取得中..."
          git fetch --all
          
          # PR用ブランチ（pr-で始まるブランチ）を検索
          $branchInfo = git for-each-ref --format="%(refname:short) %(committerdate:iso8601)" refs/remotes/origin/
          $prBranches = $branchInfo | Where-Object { $_ -match "^origin/pr-\d+" }
          
          Write-Host "PR用ブランチ数: $($prBranches.Count)"
          
          $deletedCount = 0
          $skippedCount = 0
          
          foreach ($branchLine in $prBranches) {
            try {
              # ブランチ名と最終コミット日時を分割
              $parts = $branchLine -split ' ', 2
              $fullBranchName = $parts[0]
              $branchName = $fullBranchName -replace '^origin/', ''
              $commitDateStr = $parts[1]
              
              # 日時をパース
              $commitDate = [DateTime]::Parse($commitDateStr)
              
              Write-Host "チェック中: $branchName (最終コミット: $($commitDate.ToString('yyyy-MM-dd HH:mm:ss')))"
              
              # 古いブランチかどうか判定
              if ($commitDate -lt $cutoffDate) {
                Write-Host "  → 削除対象: $branchName"
                
                # リモートブランチを削除
                git push origin --delete $branchName
                Write-Host "  → ✅ 削除完了: $branchName"
                $deletedCount++
              } else {
                Write-Host "  → 保持: $branchName (まだ新しい)"
                $skippedCount++
              }
              
            } catch {
              Write-Warning "  → ⚠️ ブランチ処理でエラー: $branchLine - $_"
              $skippedCount++
            }
          }
          
          # クリーンアップ結果をサマリー表示
          Write-Host ""
          Write-Host "==================== クリーンアップ完了 ===================="
          Write-Host "削除したブランチ数: $deletedCount"
          Write-Host "保持したブランチ数: $skippedCount"
          Write-Host "合計処理ブランチ数: $($deletedCount + $skippedCount)"
          Write-Host "========================================================="
          
          # 統計情報をパイプライン変数として設定
          echo "##vso[task.setvariable variable=DeletedBranches]$deletedCount"
          echo "##vso[task.setvariable variable=SkippedBranches]$skippedCount"
          
          Write-Host "✅ クリーンアップが正常に完了しました"
          
        } catch {
          Write-Error "❌ クリーンアップでエラーが発生しました: $_"
          throw
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
      
  # クリーンアップ結果の通知（オプション）
  - task: PowerShell@2
    displayName: 'クリーンアップ結果サマリー'
    inputs:
      targetType: 'inline'
      script: |
        $deletedCount = "$(DeletedBranches)"
        $skippedCount = "$(SkippedBranches)"
        
        Write-Host "📊 セキュリティリポジトリクリーンアップサマリー"
        Write-Host "日時: $(Get-Date)"
        Write-Host "削除ブランチ: $deletedCount 個"
        Write-Host "保持ブランチ: $skippedCount 個"
        
        # 必要に応じてSlackやTeamsへの通知を追加
        # 例: Webhook通知、メール送信など
```

---

## 手順5: パイプラインの作成と設定

### 5.1 製品リポジトリでのパイプライン作成

1. 製品コードリポジトリの「Pipelines」→「Create Pipeline」

2. **初期セットアップパイプライン**の作成：
   - 「Azure Repos Git」→ 製品リポジトリを選択
   - 「Existing Azure Pipelines YAML file」
   - Path: `/azure-pipelines-setup-mirror.yml`
   - 名前を「Setup Security Mirror」に変更
   - **一度だけ手動実行**

3. **ミラーリングパイプライン**の作成：
   - 同様の手順で `/azure-pipelines-mirror.yml` を選択
   - 名前を「Mirror to Security Repo」に変更

4. **PRセキュリティチェックパイプライン**の作成：
   - 同様の手順で `/azure-pipelines-pr-security.yml` を選択
   - 名前を「PR Security Check」に変更

5. **クリーンアップパイプライン**の作成：
   - 同様の手順で `/azure-pipelines-cleanup.yml` を選択
   - 名前を「Security Repo Cleanup」に変更

### 5.2 パイプライン権限の設定

各パイプラインで以下の設定を行う：

1. 「Edit」→「...」→「Settings」
2. 「Processing of YAML files」を「YAML files from any repository in the organization」に設定
3. 「Build job authorization scope」を「Project Collection」に設定

---

## 手順6: ブランチ保護ポリシーの設定

### 6.1 製品リポジトリでのブランチポリシー設定

1. 製品リポジトリの「Repos」→「Branches」
2. `main` ブランチの「...」→「Branch policies」
3. 以下のポリシーを設定：

#### Build validation（必須）
```
Build pipeline: PR Security Check
Path filter: /* (すべてのファイル)
Trigger: Automatic
Policy requirement: Required
Build expiration: Immediately when <source branch> is updated
Display name: Security Vulnerability Check
```

#### Status check requirements（推奨）
```
Status to check: Security Analysis
Policy requirement: Required
Reset conditions: Source branch is updated
```

#### Require a minimum number of reviewers（推奨）
```
Minimum number of reviewers: 1
When new changes are pushed: Reset all approval votes
Allow requestors to approve their own changes: ❌
```

### 6.2 developブランチにも同様の設定を適用

---

## 手順7: Snyk WebUIでの設定

### 7.1 Snyk組織の設定

1. [Snyk WebUI](https://app.snyk.io) にログイン
2. 組織を選択または作成

### 7.2 セキュリティリポジトリの連携

1. 「Add project」をクリック
2. 「Azure Repos」を選択
3. Azure DevOpsアカウントを認証
4. セキュリティプロジェクトの `product-code-mirror` リポジトリを選択
5. 「Add selected repositories」

### 7.3 プロジェト設定の調整

1. 追加したプロジェクトの「Settings」
2. 「Notifications」でアラート設定を調整
3. 「Integrations」でSlack/Teams連携を設定（オプション）

---

## 手順8: 動作確認

### 8.1 初期セットアップの確認

1. 製品リポジトリで「Setup Security Mirror」パイプラインを手動実行
2. セキュリティプロジェクトの `product-code-mirror` リポジトリで履歴を確認
3. 全コミットが `Azure DevOps Pipeline` ユーザーになっていることを確認

### 8.2 ミラーリングの確認

1. 製品リポジトリで何らかの変更をmainブランチにプッシュ
2. 「Mirror to Security Repo」パイプラインが自動実行されることを確認
3. セキュリティリポジトリに変更が反映されることを確認

### 8.3 PR時のセキュリティチェック確認

1. 製品リポジトリで新しいブランチを作成
2. テスト用の脆弱性（例：古いバージョンの依存関係）を含む変更を追加
3. mainブランチに対してPRを作成
4. 以下が実行されることを確認：
   - 「PR Security Check」パイプラインの自動実行
   - セキュリティリポジトリにPRブランチのミラー作成
   - 「snyk-security-analysis」パイプラインの実行
   - PRへのコメント投稿
   - Critical脆弱性がある場合のマージブロック

---

## 手順9: 運用設定

### 9.1 通知設定

#### Slackへの通知（オプション）
セキュリティプロジェクトのパイプラインに以下を追加：

```yaml
# Slack通知用のタスク（snyk-security-analysis.ymlの最後に追加）
- task: PowerShell@2
  displayName: 'Slack通知送信'
  condition: always()  # 成功・失敗に関わらず実行
  inputs:
    targetType: 'inline'
    script: |
      $webhook = "$(SLACK_WEBHOOK_URL)"  # Variable Groupに追加
      if (-not $webhook) {
        Write-Host "Slack Webhook URLが設定されていません"
        return
      }
      
      $criticalCount = "$(CriticalVulnerabilities)"  # 前のタスクで設定
      $status = if ($criticalCount -gt 0) { "danger" } else { "good" }
      $message = if ($criticalCount -gt 0) { 
        "🚨 Critical脆弱性 $criticalCount 件検出 - PR #${{ parameters.originalPRNumber }}" 
      } else { 
        "✅ 脆弱性チェック完了 - PR #${{ parameters.originalPRNumber }}" 
      }
      
      $payload = @{
        channel = "#security"
        username = "Security Bot"
        text = $message
        color = $status
      } | ConvertTo-Json
      
      try {
        Invoke-RestMethod -Uri $webhook -Method POST -Body $payload -ContentType "application/json"
        Write-Host "Slack通知送信完了"
      } catch {
        Write-Warning "Slack通知送信失敗: $_"
      }
```

### 9.2 ダッシュボード作成

Azure DevOpsで監視用ダッシュボードを作成：

1. 「Overview」→「Dashboards」→「New Dashboard」
2. 以下のウィジェットを追加：
   - **Build History**: セキュリティチェックパイプラインの実行履歴
   - **Pull Request**: 製品リポジトリのPR状況
   - **Work Items**: セキュリティ関連のタスク管理
   - **Markdown**: セキュリティポリシーや連絡先情報

### 9.3 定期レポート設定

月次セキュリティレポート用パイプライン（オプション）：

**ファイル名**: `azure-pipelines-security-report.yml`

```yaml
# azure-pipelines-security-report.yml
# 月次セキュリティレポートを生成するパイプライン

schedules:
- cron: "0 9 1 * *"  # 毎月1日 9:00 AM
  displayName: Monthly Security Report
  branches:
    include:
    - main

trigger: none

variables:
- group: snyk-security-variables

jobs:
- job: GenerateSecurityReport
  displayName: '月次セキュリティレポート生成'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  - task: PowerShell@2
    displayName: 'セキュリティメトリクス収集'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "月次セキュリティレポートを生成中..."
        
        # Snyk APIからプロジェクト情報を取得
        $headers = @{
          'Authorization' = "token $(snyk-api-token)"
          'Content-Type' = 'application/json'
        }
        
        try {
          # プロジェクト一覧取得
          $projects = Invoke-RestMethod -Uri "https://snyk.io/api/v1/projects" -Headers $headers
          
          # 脆弱性サマリー生成
          $totalVulns = 0
          $criticalVulns = 0
          $highVulns = 0
          
          foreach ($project in $projects.projects) {
            $issues = Invoke-RestMethod -Uri "https://snyk.io/api/v1/project/$($project.id)/issues" -Headers $headers
            $totalVulns += $issues.issues.vulnerabilities.Count
            $criticalVulns += ($issues.issues.vulnerabilities | Where-Object { $_.severity -eq "critical" }).Count
            $highVulns += ($issues.issues.vulnerabilities | Where-Object { $_.severity -eq "high" }).Count
          }
          
          $report = @"
# 月次セキュリティレポート - $(Get-Date -Format 'yyyy年MM月')

## サマリー
- 総脆弱性数: $totalVulns
- Critical: $criticalVulns
- High: $highVulns
- Low/Medium: $($totalVulns - $criticalVulns - $highVulns)

## 推奨アクション
$(if ($criticalVulns -gt 0) { "🚨 Critical脆弱性の即座対応が必要" } else { "✅ Critical脆弱性なし" })

---
生成日時: $(Get-Date)
"@
          
          Write-Host $report
          
          # レポートをファイルに保存
          $report | Out-File -FilePath "security-report.md"
          
          # 必要に応じてメール送信やファイル保存
          
        } catch {
          Write-Error "レポート生成エラー: $_"
        }
```

---

## 手順10: トラブルシューティング

### 10.1 よくある問題と解決方法

#### 問題1: パイプライン間の認証エラー
**症状**: TriggerBuild@4で「401 Unauthorized」エラー
**解決方法**:
1. Project Settings → Service connections → OAuth認証の確認
2. パイプライン権限を「Project Collection」に変更
3. Personal Access Tokenの有効期限確認

#### 問題2: Snyk認証エラー
**症状**: `snyk auth` コマンドが失敗
**解決方法**:
1. Snyk APIトークンの有効性確認
2. Variable Groupでトークンが正しく設定されているか確認
3. Snyk組織の権限確認

#### 問題3: PRコメント投稿失敗
**症状**: 脆弱性結果がPRにコメントされない
**解決方法**:
1. Azure DevOps REST API URLの組織名確認
2. `System.AccessToken` の権限確認
3. API URLのバージョン指定確認

#### 問題4: ミラーリングでGit認証エラー
**症状**: `git clone` や `git push` で認証失敗
**解決方法**:
1. `persistCredentials: true` の設定確認
2. `SYSTEM_ACCESSTOKEN` の環境変数設定確認
3. セキュリティプロジェクトへのアクセス権限確認

### 10.2 ログの確認方法

1. **パイプライン実行ログ**: Azure DevOps → Pipelines → 該当パイプライン → 実行履歴
2. **Snyk解析ログ**: セキュリティプロジェクトのパイプライン実行ログ
3. **Git操作ログ**: パイプラインの各タスクの詳細ログ
4. **API呼び出しログ**: PowerShellタスクの出力ログ

### 10.3 デバッグ用設定

デバッグ時は以下の環境変数を追加：

```yaml
variables:
  System.Debug: true  # 詳細ログ出力
```

---

## 手順11: セキュリティと運用の考慮事項

### 11.1 アクセス制御

- **セキュリティプロジェクト**: 最小限のユーザーのみアクセス許可
- **Snyk APIトークン**: 定期的なローテーション（90日毎推奨）
- **Personal Access Token**: 有効期限の監視

### 11.2 コスト最適化

- **Snyk利用料**: セキュリティプロジェクトのみで課金
- **Azure DevOps**: パイプライン実行時間の最適化
- **ストレージ**: 古いPRブランチの自動削除

### 11.3 災害復旧

- **バックアップ**: セキュリティプロジェクトのエクスポート
- **設定文書**: 本手順書の定期更新
- **連絡先**: セキュリティチーム・Snyk管理者の連絡先リスト

---

## まとめ

本手順書に従って構築することで、以下が実現されます：

✅ **コスト最適化**: 製品リポジトリでのSnyk課金を回避
✅ **自動セキュリティチェック**: PR時のCritical脆弱性自動検出
✅ **マージ制御**: Critical脆弱性があるPRのマージブロック
✅ **履歴管理**: ミラーリポジトリのコミット履歴統一
✅ **運用効率**: 自動化されたクリーンアップとレポート生成

**重要**: すべてのYAMLファイル内の「★要変更」マークがある箇所は、実際の環境に合わせて変更してください。

**次のステップ**: 本システムの運用開始後、定期的な脆弱性レビューと設定の最適化を実施することを推奨します。