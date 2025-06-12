非常に良いアプローチです！最初に空の処理でパイプラインの動作を確認してから、実際のSnyk解析を有効化する方が安全ですね。

## 段階的テスト用の修正版パイプライン

### テスト用Snyk解析パイプライン（空処理版）

# snyk-security-analysis-test.yml
# テスト用のSnyk解析パイプライン（空処理版）
# 実際のSnyk解析は行わず、パイプライン構成とPRコメント機能のみテスト

# 手動トリガーのみ（製品リポジトリから呼び出される）
trigger: none

# 製品リポジトリからのパラメータを受け取る
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

# Variable Groupから環境変数を取得（テスト段階ではSnykトークン不要）
variables:
- group: snyk-security-variables

jobs:
- job: TestSecurityAnalysis
  displayName: 'テスト用セキュリティ解析ジョブ'
  pool:
    vmImage: 'ubuntu-latest'
    
  steps:
  # 指定されたブランチをチェックアウト
  - checkout: self
    displayName: 'ソースコードチェックアウト'
    ref: ${{ parameters.sourceBranch }}
    
  # ソースコード情報の表示（デバッグ用）
  - task: PowerShell@2
    displayName: 'ソースコード情報表示'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "==================== ソースコード情報 ===================="
        Write-Host "現在のディレクトリ: $(Get-Location)"
        Write-Host "ファイル一覧:"
        
        $files = Get-ChildItem -Recurse -File | Select-Object -First 20
        foreach ($file in $files) {
          Write-Host "  $($file.FullName.Replace((Get-Location), '.'))"
        }
        
        Write-Host "========================================================="
        
        # パラメータ情報を表示
        Write-Host "==================== パラメータ情報 ===================="
        Write-Host "ソースブランチ: ${{ parameters.sourceBranch }}"
        Write-Host "元PR番号: ${{ parameters.originalPRNumber }}"
        Write-Host "元プロジェクト: ${{ parameters.originalProject }}"
        Write-Host "元リポジトリ: ${{ parameters.originalRepo }}"
        Write-Host "========================================================="
    
  # 依存関係ファイルの検出（Snyk解析対象の確認）
  - task: PowerShell@2
    displayName: '依存関係ファイル検出'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "依存関係ファイルの検出を開始..."
        
        # 検出対象のファイルパターン
        $dependencyFiles = @(
          "package.json",           # Node.js
          "package-lock.json",
          "yarn.lock",
          "requirements.txt",       # Python
          "Pipfile",
          "Pipfile.lock",
          "poetry.lock",
          "pom.xml",               # Java
          "build.gradle",
          "gradle.lockfile",
          "Gemfile",               # Ruby
          "Gemfile.lock",
          "go.mod",                # Go
          "go.sum",
          "Cargo.toml",            # Rust
          "Cargo.lock",
          "composer.json",         # PHP
          "composer.lock",
          "*.csproj",              # .NET
          "*.fsproj",
          "*.vbproj",
          "packages.config",
          "project.json"
        )
        
        $foundFiles = @()
        
        foreach ($pattern in $dependencyFiles) {
          $files = Get-ChildItem -Path . -Recurse -Name $pattern -ErrorAction SilentlyContinue
          if ($files) {
            $foundFiles += $files
            Write-Host "✅ 検出: $pattern"
            foreach ($file in $files) {
              Write-Host "    $file"
            }
          }
        }
        
        # Dockerfileの検出
        $dockerfiles = Get-ChildItem -Path . -Recurse -Name "Dockerfile*" -ErrorAction SilentlyContinue
        if ($dockerfiles) {
          Write-Host "🐳 Dockerfileを検出:"
          foreach ($dockerfile in $dockerfiles) {
            Write-Host "    $dockerfile"
            $foundFiles += $dockerfile
          }
        }
        
        if ($foundFiles.Count -eq 0) {
          Write-Host "⚠️ 依存関係ファイルが見つかりませんでした"
          Write-Host "Snyk解析の対象となるファイルがない可能性があります"
        } else {
          Write-Host "📊 検出されたファイル数: $($foundFiles.Count)"
        }
        
        # 結果を変数に保存
        echo "##vso[task.setvariable variable=FoundDependencyFiles]$($foundFiles.Count)"
    
  # モック脆弱性スキャン（実際のSnyk解析なし）
  - task: PowerShell@2
    displayName: 'モック脆弱性スキャン'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "モック脆弱性スキャンを実行中..."
        
        # テスト用の模擬結果を生成
        # 実際のSnyk解析は行わない
        
        # ランダムな結果を生成（テスト目的）
        $random = Get-Random -Minimum 0 -Maximum 100
        
        # 80%の確率で脆弱性なし、20%の確率で脆弱性ありをシミュレート
        if ($random -lt 80) {
          # 脆弱性なしのケース
          $criticalVulns = 0
          $highVulns = Get-Random -Minimum 0 -Maximum 3
          $mediumVulns = Get-Random -Minimum 0 -Maximum 5
          $lowVulns = Get-Random -Minimum 0 -Maximum 8
          $exitCode = 0
        } else {
          # 脆弱性ありのケース
          $criticalVulns = Get-Random -Minimum 1 -Maximum 3
          $highVulns = Get-Random -Minimum 0 -Maximum 5
          $mediumVulns = Get-Random -Minimum 2 -Maximum 8
          $lowVulns = Get-Random -Minimum 1 -Maximum 10
          $exitCode = 1
        }
        
        Write-Host "==================== モック結果 ===================="
        Write-Host "Critical脆弱性: $criticalVulns 件"
        Write-Host "High脆弱性: $highVulns 件"
        Write-Host "Medium脆弱性: $mediumVulns 件"
        Write-Host "Low脆弱性: $lowVulns 件"
        Write-Host "終了コード: $exitCode"
        Write-Host "=================================================="
        
        # 模擬結果ファイルを作成
        @{
          criticalVulns = $criticalVulns
          highVulns = $highVulns
          mediumVulns = $mediumVulns
          lowVulns = $lowVulns
          exitCode = $exitCode
          isTest = $true
          timestamp = (Get-Date).ToString("yyyy-MM-ddTHH:mm:ssZ")
        } | ConvertTo-Json | Out-File "mock-snyk-results.json"
        
        # 終了コードを保存
        $exitCode | Out-File "snyk-exit-code.txt"
        
        # パイプライン変数に設定
        echo "##vso[task.setvariable variable=CriticalVulns]$criticalVulns"
        echo "##vso[task.setvariable variable=HighVulns]$highVulns"
        echo "##vso[task.setvariable variable=MediumVulns]$mediumVulns"
        echo "##vso[task.setvariable variable=LowVulns]$lowVulns"
        echo "##vso[task.setvariable variable=ScanExitCode]$exitCode"
        
        Write-Host "✅ モックスキャン完了"
    
  # モック結果の評価とPRへのコメント投稿
  - task: PowerShell@2
    displayName: 'モック結果評価・PRコメント投稿'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "モック結果の評価を開始..."
        
        # モック結果を取得
        $criticalVulns = [int]"$(CriticalVulns)"
        $highVulns = [int]"$(HighVulns)"
        $mediumVulns = [int]"$(MediumVulns)"
        $lowVulns = [int]"$(LowVulns)"
        $exitCode = [int]"$(ScanExitCode)"
        $foundFiles = [int]"$(FoundDependencyFiles)"
        
        Write-Host "脆弱性カウント - Critical: $criticalVulns, High: $highVulns, Medium: $mediumVulns, Low: $lowVulns"
        
        # PRコメント用のメッセージを作成
        $statusIcon = if ($criticalVulns -gt 0) { "❌" } else { "✅" }
        $statusMessage = if ($criticalVulns -gt 0) { 
          "**[テスト] Critical脆弱性が検出されました。マージをブロックします。**" 
        } else { 
          "[テスト] Critical脆弱性は検出されませんでした。" 
        }
        
        $comment = @"
        ## 🧪 セキュリティ脆弱性解析結果（テストモード）
        
        $statusIcon $statusMessage
        
        ### 脆弱性サマリー（モック結果）
        - **Critical:** $criticalVulns 件
        - **High:** $highVulns 件
        - **Medium:** $mediumVulns 件
        - **Low:** $lowVulns 件
        
        ### スキャン詳細
        - 検出された依存関係ファイル: $foundFiles 個
        - 依存関係スキャン: $(if ($exitCode -eq 0) { "✅ 正常完了" } else { "⚠️ 脆弱性検出" })
        - コンテナスキャン: 🧪 テストモード（未実行）
        
        ### ⚠️ 重要な注意事項
        **これはテスト実行です。実際のSnyk解析は行われていません。**
        - 上記の脆弱性数はランダムに生成されたモック結果です
        - パイプライン構成とPRコメント機能のテストが目的です
        - 実際のセキュリティ判定には使用しないでください
        
        ---
        *テスト実行日時: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss UTC')*  
        *この結果は自動生成されました。*
        "@
        
        Write-Host "PRコメントを投稿中..."
        
        # Azure DevOps REST APIでPRにコメントを投稿
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
        
        # API URL構築（★要変更: 実際の組織名に置き換え）
        $orgUrl = "https://dev.azure.com/your-org"
        $uri = "$orgUrl/${{ parameters.originalProject }}/_apis/git/repositories/${{ parameters.originalRepo }}/pullRequests/${{ parameters.originalPRNumber }}/threads?api-version=6.0"
        
        Write-Host "API URI: $uri"
        
        try {
          Invoke-RestMethod -Uri $uri -Method Post -Headers $headers -Body $body
          Write-Host "✅ PRコメントの投稿が成功しました"
        } catch {
          Write-Warning "❌ PRコメントの投稿に失敗しました: $_"
          Write-Host "Status Code: $($_.Exception.Response.StatusCode)"
          Write-Host "Response: $($_.Exception.Response)"
          
          # デバッグ用に詳細情報を出力
          Write-Host "Request Headers:"
          $headers | ConvertTo-Json | Write-Host
          Write-Host "Request Body:"
          $body | Write-Host
        }
        
        # テストモードでは常に成功とする（Critical脆弱性があってもマージをブロックしない）
        Write-Host "🧪 テストモード: マージブロックは行いません"
        
        if ($criticalVulns -gt 0) {
          Write-Host "⚠️ 本番モードであれば、Critical脆弱性 $criticalVulns 件によりマージがブロックされます"
        }
        
        Write-Host "✅ テスト用セキュリティ解析が完了しました"
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)

### 1. 初期セットアップ（変更なし）

前の手順1〜4はそのまま実行してください。

### 2. テスト用初期化パイプラインの修正

# azure-pipelines-setup-mirror.yml
# セキュリティミラーリポジトリを完全にクリーンな状態で初期化するパイプライン
# 一回限りの実行で、パイプラインユーザーのみの履歴でリポジトリを初期化

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
- job: CompleteCleanSetup
  displayName: '完全クリーンセットアップ'
  pool:
    vmImage: 'ubuntu-latest'
  
  steps:
  # 製品コードをチェックアウト
  - checkout: self
    displayName: '製品コードチェックアウト'
    persistCredentials: true
    
  # セキュリティミラーリポジトリの完全クリーン初期化
  - task: PowerShell@2
    displayName: '完全クリーンな履歴でセキュリティリポジトリ初期化'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "完全クリーンなセキュリティリポジトリ初期化を開始..."
        
        # Git設定（パイプラインユーザー）
        Write-Host "Gitユーザー設定中..."
        git config --global user.email "$(COMMIT_USER_EMAIL)"
        git config --global user.name "$(COMMIT_USER_NAME)"
        Write-Host "設定完了 - Email: $(COMMIT_USER_EMAIL), Name: $(COMMIT_USER_NAME)"
        
        # セキュリティリポジトリのURL
        $securityRepoUrl = "$(securityProjectUrl)/_git/$(mirrorRepoName)"
        Write-Host "セキュリティリポジトリURL: $securityRepoUrl"
        
        try {
          # 一時ディレクトリでクリーンなリポジトリを作成
          $tempDir = "clean-security-repo"
          Write-Host "一時ディレクトリでクリーンリポジトリを作成: $tempDir"
          
          # 既存の一時ディレクトリをクリーンアップ
          if (Test-Path $tempDir) {
            Remove-Item $tempDir -Recurse -Force
          }
          
          New-Item -ItemType Directory -Path $tempDir -Force
          Set-Location $tempDir
          
          # 新しいGitリポジトリを初期化
          Write-Host "新しいGitリポジトリとして初期化中..."
          git init
          git config user.email "$(COMMIT_USER_EMAIL)"
          git config user.name "$(COMMIT_USER_NAME)"
          
          # セキュリティプロジェクト用のパイプラインファイルを作成
          Write-Host "セキュリティ解析用パイプラインファイルを作成中..."
          
          # テスト用パイプラインファイルをコピー
          $sourceDir = "$(Build.SourcesDirectory)"
          $testPipelineFile = "$sourceDir/snyk-security-analysis-test.yml"
          $prodPipelineFile = "$sourceDir/snyk-security-analysis.yml"
          
          if (Test-Path $testPipelineFile) {
            Copy-Item $testPipelineFile "azure-pipelines.yml"
            Write-Host "✅ テスト用Snyk解析パイプラインファイルをコピー完了"
          } elseif (Test-Path $prodPipelineFile) {
            Copy-Item $prodPipelineFile "azure-pipelines.yml"
            Write-Host "✅ 本番用Snyk解析パイプラインファイルをコピー完了"
          } else {
            Write-Warning "snyk-security-analysis.ymlが見つかりません。最小限のパイプラインファイルを作成します"
            # 最小限のパイプラインファイルを作成
            @"
# Minimal pipeline for security repository
# snyk-security-analysis.yml should be copied from product repository
trigger: none
pool:
  vmImage: 'ubuntu-latest'
steps:
- script: echo "Security repository initialized - Please update azure-pipelines.yml with snyk-security-analysis.yml content"
  displayName: 'Security Repository Ready'
"@ | Out-File "azure-pipelines.yml" -Encoding UTF8
            Write-Host "最小限のパイプラインファイルを作成完了"
          }
          
          # 製品コードをコピー（Gitディレクトリと一時ディレクトリを除く）
          Write-Host "製品コードをコピー中..."
          $excludeDirs = @(".git", $tempDir, "node_modules", "bin", "obj", ".vs", ".vscode")
          
          Get-ChildItem $sourceDir | Where-Object { 
            $_.Name -notin $excludeDirs 
          } | Copy-Item -Destination . -Recurse -Force
          
          Write-Host "ファイルコピー完了"
          
          # .gitignoreファイルを作成（必要に応じて）
          if (-not (Test-Path ".gitignore")) {
            @"
# Security repository .gitignore
node_modules/
*.log
.env
.DS_Store
bin/
obj/
"@ | Out-File ".gitignore" -Encoding UTF8
            Write-Host ".gitignoreファイルを作成"
          }
          
          # 初期コミット（パイプラインユーザーのみの履歴開始）
          Write-Host "初期コミットを作成中..."
          git add .
          
          # コミットメッセージを作成
          $commitMessage = @"
Initial security repository setup

- Source: $(Build.Repository.Name)
- Branch: $(Build.SourceBranch)
- Build: $(Build.BuildNumber)
- Date: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss UTC')

This repository is used for security analysis only.
All commits are made by the pipeline user for security compliance.
"@
          
          git commit -m $commitMessage
          Write-Host "初期コミット作成完了"
          
          # リモートを追加してプッシュ（既存履歴を完全に上書き）
          Write-Host "セキュリティリポジトリに完全上書きプッシュ中..."
          git remote add origin $securityRepoUrl
          
          # 強制プッシュで既存の履歴を完全に置き換え
          git push origin main --force
          
          Write-Host "✅ 完全クリーンなセキュリティリポジトリ初期化完了"
          
          # 履歴確認
          Write-Host ""
          Write-Host "=== 初期化後のコミット履歴確認 ==="
          git log --oneline --format="%h %an <%ae> %s" -n 3
          Write-Host "================================"
          
          # 統計情報
          $fileCount = (Get-ChildItem -Recurse -File | Measure-Object).Count
          Write-Host "📊 初期化統計:"
          Write-Host "  - 総ファイル数: $fileCount"
          Write-Host "  - コミッター: $(COMMIT_USER_NAME) <$(COMMIT_USER_EMAIL)>"
          Write-Host "  - リポジトリ: $(mirrorRepoName)"
          
        } catch {
          Write-Error "❌ 完全クリーン初期化でエラーが発生: $_"
          throw
        } finally {
          # クリーンアップ
          Write-Host "一時ディレクトリのクリーンアップ中..."
          Set-Location "$(Build.SourcesDirectory)"
          if (Test-Path $tempDir) {
            Remove-Item $tempDir -Recurse -Force -ErrorAction SilentlyContinue
          }
        }
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
      
  # 初期化完了の確認
  - task: PowerShell@2
    displayName: '初期化完了確認'
    inputs:
      targetType: 'inline'
      script: |
        Write-Host "🎉 セキュリティリポジトリの完全クリーン初期化が完了しました"
        Write-Host ""
        Write-Host "次のステップ:"
        Write-Host "1. セキュリティプロジェクトでパイプラインを作成"
        Write-Host "2. 通常のミラーリングパイプラインを設定"
        Write-Host "3. PRセキュリティチェックパイプラインを設定"
        Write-Host ""
        Write-Host "⚠️ 重要: この初期化パイプラインは一度だけ実行してください"

#### ステップ1: テスト用ファイルの準備
1. 製品リポジトリに `snyk-security-analysis-test.yml` を作成
2. 他のパイプラインファイルも作成（変更なし）

#### ステップ2: 初期セットアップ実行
1. 「Setup Security Mirror」パイプラインを実行
2. セキュリティリポジトリにテスト用パイプラインが配置されることを確認

#### ステップ3: テスト実行
1. 製品リポジトリでテスト用PRを作成
2. 「PR Security Check」パイプラインが実行されることを確認
3. セキュリティプロジェクトの「snyk-security-analysis」パイプラインが実行されることを確認
4. PRにテスト用コメントが投稿されることを確認

### 4. テスト段階で確認すべき項目

#### ✅ チェックリスト

**パイプライン構成**:
- [ ] セキュリティリポジトリの履歴が単一ユーザー（パイプラインユーザー）のみ
- [ ] ミラーリングが正常に動作
- [ ] PRブランチのミラーリングが正常に動作
- [ ] パイプライン間の連携が正常に動作

**PRコメント機能**:
- [ ] PRにコメントが正常に投稿される
- [ ] コメント内容が期待通り（テストモード表示あり）
- [ ] 脆弱性カウントが表示される
- [ ] 依存関係ファイルの検出が動作

**セキュリティ**:
- [ ] セキュリティリポジトリのコミット履歴にパイプラインユーザー以外が含まれない
- [ ] Variable Groupの設定が正常に読み込まれる
- [ ] 認証周りが正常に動作

#### 期待するPRコメントの例

```
## 🧪 セキュリティ脆弱性解析結果（テストモード）

✅ [テスト] Critical脆弱性は検出されませんでした。

### 脆弱性サマリー（モック結果）
- **Critical:** 0 件
- **High:** 2 件
- **Medium:** 3 件
- **Low:** 5 件

### スキャン詳細
- 検出された依存関係ファイル: 3 個
- 依存関係スキャン: ✅ 正常完了
- コンテナスキャン: 🧪 テストモード（未実行）

### ⚠️ 重要な注意事項
**これはテスト実行です。実際のSnyk解析は行われていません。**
```

### 5. 本番移行手順

テストが成功したら、以下の手順で本番環境に移行：

#### ステップ1: Snyk APIトークンの設定
Variable Groupに実際のSnyk APIトークンを設定

#### ステップ2: 本番用パイプラインの配置
1. セキュリティプロジェクトのパイプライン設定を編集
2. `azure-pipelines.yml` の内容を `snyk-security-analysis.yml` の内容に置き換え

#### ステップ3: Critical脆弱性時のマージブロック有効化
本番用パイプラインではCritical脆弱性検出時にパイプラインが失敗し、マージがブロックされます

### 6. トラブルシューティング（テスト段階）

#### よくある問題

**問題1**: PRコメントが投稿されない
- Azure DevOps組織名の確認
- `System.AccessToken` の権限確認
- APIエンドポイントURLの確認

**問題2**: パイプライン間の連携が動作しない
- パイプライン名の確認
- プロジェクト名の確認
- 認証設定の確認

**問題3**: セキュリティリポジトリに他のユーザーが混入
- 初期セットアップの再実行
- 手動でのファイル編集を避ける
- パイプライン経由でのみ更新

### 7. テスト完了後の確認

すべてのテストが完了したら、以下を確認：

```bash
# セキュリティリポジトリで実行
git log --all --format="%an <%ae>" | sort | uniq

# 期待する結果:
# Azure DevOps Pipeline <pipeline@azuredevops.com>
```

この段階的なアプローチにより、安全にシステムの動作を確認してから実際のSnyk解析を有効化できます。