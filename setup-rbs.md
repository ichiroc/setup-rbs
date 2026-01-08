# Rails RBS/Steep セットアップガイド

## 目的

Rails アプリに **RBS / Steep** を導入する。
**最初の対象は `app/models` のみ**とし、LSP が安定して動作する状態を作る。

## 前提条件・制約

- 既存の RBS / sig / gem 同梱 RBS と **競合する定義を書かない**
- `Cannot find type (RBS::UnknownTypeName)` が **1件でもある状態は禁止**
- `RBS::DuplicatedDeclarationError` が出る状態は禁止
- **`sig/generated/` 配下のファイルは変更しない**
- namespace は以下のルールに従う
  - `module Admins` が **事前に定義されていれば** `class Admins::Hoge` は可
  - module が未定義の状態で `class Admins::Hoge` を書くのは禁止
- インスタンス変数は型推論されないため **必ず明示的に型定義する**
- `.gem_rbs_collection/` は **git 管理しない**

## 手順

### 1. Gem の追加

```ruby
# Gemfile (development グループに追加)
gem 'rbs', require: false
gem 'rbs_rails', require: false
gem 'rbs-inline', require: false
gem 'steep', require: false
```

```sh
bundle install
```

### 2. Steep 初期化（models のみ対象）

```sh
bundle exec steep init
```

```ruby
# Steepfile
target :app do
  check "app/models"
  signature "sig"
end
```

### 3. rbs collection の導入

```sh
bundle exec rbs collection init
```

生成された `rbs_collection.yaml` を確認（デフォルトで正しい設定）：

```yaml
# rbs_collection.yaml
sources:
  - type: git
    name: ruby/gem_rbs_collection
    remote: https://github.com/ruby/gem_rbs_collection.git
    revision: main
    repo_dir: gems

path: .gem_rbs_collection
```

`.gitignore` に追加：

```sh
echo ".gem_rbs_collection" >> .gitignore
```

**重要:** 最初の install は **gem の sig 衝突を解決してから**実行する（次のステップ）。

### 4. gem の sig 衝突の解決（最重要）

#### 4.1 衝突する gem の特定

以下のコマンドで、gem 自体に sig ディレクトリが含まれている gem を特定：

```ruby
ruby -e "
require 'bundler'
Bundler.load.specs.each do |spec|
  sig_path = File.join(spec.full_gem_path, 'sig')
  if File.directory?(sig_path)
    rbs_files = Dir.glob(File.join(sig_path, '**', '*.rbs'))
    total_lines = rbs_files.sum { |f| File.readlines(f).size rescue 0 }
    puts \"#{spec.name}: #{rbs_files.size} files, #{total_lines} lines\"
  end
end
"
```

#### 4.2 ignore すべき gem の判断基準

**ignore する必要があるのは:**
- gem 自体に sig ディレクトリが含まれており
- かつ、意味のある型定義がある gem（100行以上が目安）

**ignore しない gem:**
- スカスカの型定義（10行以下）の gem → yaml から除外
- gem 自体に sig がない gem → yaml から除外（rbs_collection から取得）

#### 4.3 rbs_collection.yaml に ignore を追加

例:

```yaml
gems:
  - name: meta-tags
    ignore: true
  - name: mutex_m
    ignore: true
  - name: prism
    ignore: true
```

#### 4.4 rbs collection install の実行

**重要:** `rbs_collection.yaml` を修正したら **必ず** install を実行：

```sh
bundle exec rbs collection install
```

### 5. Steep の健全性チェック（必須）

以下 **すべて成功している状態**でなければ次へ進まない。

#### 5.1 steep stats の確認

```sh
bundle exec steep stats
```

- **全ファイルの Status が `success` であること**
- **`error` が 1 行も表示されないこと**

もし `error` が表示される場合は、gem の sig 衝突が残っている可能性がある。
問題の gem を `rbs_collection.yaml` に ignore として追加して再実行。

#### 5.2 steep check の確認

```sh
bundle exec steep check
```

- `Cannot find type (Diagnostic ID: RBS::UnknownTypeName)` が **ゼロ**
- `RBS::DuplicatedDeclarationError` が **ゼロ**

不足している型がある場合は、**`sig/` 直下に新しい `.rbs` ファイルを作成**して補完する：

```ruby
# sig/search_cop.rbs
module SearchCop
end

# sig/aasm.rbs
module AASM
end
```

### 6. rbs_rails によるモデル定義生成

#### 6.1 Association検証Rakeタスクの作成（必須・最重要）

**rbs_rails を実行する前に必ずこのタスクを作成し、実行する。**

```ruby
# lib/tasks/check_associations.rake
namespace :app do
  desc "Check all model associations for errors"
  task check_associations: :environment do
    Rails.application.eager_load!

    ActiveRecord::Base.descendants.each do |model|
      next if model.name.blank? || model.abstract_class?

      model.reflect_on_all_associations.each do |assoc|
        assoc.klass
      rescue => e
        puts "#{model}##{assoc.name}: #{e.message}"
      end
    end
  end
end
```

**このタスクの重要性:**
- rbs_rails が失敗する原因のほとんどはassociation定義の問題
- タイプミス1つでrbs_rails全体が動かなくなる
- このタスクで事前に全てのエラーを検出できる

#### 6.2 Associationチェックの実行

```sh
bundle exec rake app:check_associations
```

**エラーが1件も出力されない場合のみ次へ進む。**

エラーが出た場合の対処法:
1. エラーメッセージから`Model#association_name`を確認
2. 該当モデルファイルを開く
3. タイプミスや存在しないクラスの参照を修正
4. 再度 `rake app:check_associations` を実行

**よくあるエラー例:**
```ruby
# 誤り（タイプミス）
has_many :schedule_taggins, through: :schedules  # 末尾のgが1つ足りない

# 正しい
has_many :schedule_taggings, through: :schedules
```

#### 6.3 rbs_rails のインストールと実行

associationチェックが成功したら:

```sh
bin/rails g rbs_rails:install
```

```sh
rails rbs_rails:all
```

- `sig/rbs_rails/` 配下に **DB 定義に基づく model の RBS** が生成される
- 生成されたファイルは **手で編集しない**

**もしエラーが出たら:**
`rake app:check_associations` を再実行して見落としがないか確認。

### 7. rbs-inline（モデル限定）

```sh
bundle exec rbs-inline app/models/**/*.rb --output --opt-out
```

- `sig/generated/` 配下に RBS が生成される
- 生成されたファイルは **内容を編集しない**
- 追記・修正が必要な場合は **別の sig ファイルを作成する**
- 対象はモデルだけにして **Controller の型定義は追加しない** ようにする

### 8. namespace 定義の自動生成（重要）

以下の Ruby を実行し、出力を `sig/namespaces.rbs` として保存する：

```sh
bundle exec rails runner "
puts Rails.autoloaders.main.all_expected_cpaths
       .filter { _1.include?(Rails.root.to_s) }
       .filter { _2.include?('::') }
       .map { _1.include?('concerns') ? _2.split('::') : _2.split('::')[..-2] }
       .uniq
       .sort
       .map { _1.join('::') }
       .map { \"module #{_1}\nend\" }
       .join(\"\n\n\")
" > sig/namespaces.rbs
```

### 9. 最終確認

もう一度 `steep stats` と `steep check` を実行：

```sh
bundle exec rbs collection install  # 念のため再実行
bundle exec steep stats
bundle exec steep check
```

## 完了条件（絶対）

- [ ] `bundle exec rake app:check_associations` が成功（エラー出力なし）
- [ ] `bundle exec steep stats` で全ファイルが **`success`**
- [ ] `bundle exec steep check` で `UnknownTypeName` が **ゼロ**
- [ ] `bundle exec steep check` で `RBS::DuplicatedDeclarationError` が **ゼロ**
- [ ] `sig/generated/` を一切編集していない
- [ ] `.gem_rbs_collection/` を git ignore に追加済み
- [ ] 作成した lib/tasks/check_associations.rake を削除すること
- [ ] LSP が正常に動作すること

## トラブルシューティング

### エラー修正の効率化

steep check でエラーが出たら、**最初に全エラーを一括収集**してから対処する：

```bash
# 重複エラーを全て抽出
bundle exec steep check 2>&1 | grep "DuplicatedDeclaration" | grep -oP "::[\w:]+" | sort -u > /tmp/duplicates.txt

# 不足型を全て抽出
bundle exec steep check 2>&1 | grep "Cannot find type" | awk -F'`' '{print $2}' | sort -u > /tmp/missing_types.txt
```

これにより、エラー修正の往復を 大幅に削減 できます。

### rbs_rails が `undefined method 'class_name' for nil` で失敗する

**これが最も多いエラーです。**

**原因:** モデルのassociation定義にタイプミスや不整合がある

**解決手順:**
1. `bundle exec rake app:check_associations` を実行
2. エラー出力から問題のモデルとassociationを特定
3. 該当モデルファイルを開いて修正
4. 再度 `rake app:check_associations` でチェック
5. エラーがなくなったら `rails rbs_rails:all` を実行

**重要:** このエラーはassociation定義の問題であり、rbs_railsの問題ではない。

### UnknownTypeName エラーが出る

**原因:** 使用している gem やモジュールの型定義がない

**解決方法:**
`sig/` 直下に `.rbs` ファイルを作成して型定義を追加：

```ruby
# sig/gem_name.rbs
module GemName
end
```

### DuplicatedDeclarationError が出る

**原因:** 複数の場所で同じ型が定義されている

**解決方法:**
1. エラーメッセージから重複している型名を確認
2. `rg -n "TypeName" sig .gem_rbs_collection` で検索
3. 重複している gem を `rbs_collection.yaml` で ignore

## 重要な注意事項

1. **rbs_rails を実行する前に必ず `rake app:check_associations` でチェック**
2. **`rbs_collection.yaml` を修正したら必ず `bundle exec rbs collection install` を実行**
3. **gem 自体に sig が含まれている gem のみを ignore する**
4. **スカスカの型定義（10行以下）の gem は ignore する価値がない**
5. **`sig/generated/` 配下は絶対に編集しない**
6. **型定義の追加は `sig/` 直下に新しいファイルを作成**

## まとめ

**最重要ポイント:**
- rbs_railsのエラーの99%はassociation定義の問題
- `rake app:check_associations` で事前に全て検出できる
- タイプミス1つで全体が動かなくなる
- 10行のRakeタスクがrbs_railsを成功させる鍵
