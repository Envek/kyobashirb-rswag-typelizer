---
theme: envek
highlighter: shiki
lineNumbers: false
title: Generating OpenAPI schema from serializers throughout the Rails stack
titleTemplate: '%s - Kyobashi.rb #5'
drawings:
  persist: false
download: true
mdc: true
talkDurationMinutes: 10
progressBarStartSlide: 2

layout: cover
class: text-left
---

# Generating OpenAPI schema from serializers <small>throughout the Rails stack</small>

シリアライザーからRailsスタック全体を参考してOpenAPIスキーマの生成方法

<div class="mt-8">
Andrey Novikov, Evil Martians

Kyobashi.rb #5, 2025-02-26
</div>
---
class: annotated-list
layout: image-right
image: /images/andrey-cub.jpg
---

# About me

自己紹介

- Andrey Novikov

  ノヴィコフ・アンドレイ

- Back-end engineer at Evil Martians

  イービルマーシャンズのバックエンドエンジニア

- Ruby, Go, PostgreSQL, Docker, k8s…

- Living in Suita, Osaka for 2 years

  2年間以上、大阪府吹田市に住んでいます

- Love to ride mopeds, motorcycles, and cars over Japan

  原付も、大型バイクも、車で日本を旅するのが好き

---
class: annotated-list
---

# If you have an API you need the schema!

APIがある場合はスキーマが必要です！

<br class="mb-8" />

- For SPA frontends, mobile apps, etc. 

  SPAフロントエンド、モバイルアプリなど

- For documentation, validation, and code generation

  ドキュメント、リクエストとレスポンスの検証、コード生成のため

- Even if you don't have external clients 

  外部クライアントがいなくても


---
class: annotated-list
---

# Different approaches / 異なるアプローチ

## Schema First / スキーマファースト

- Write OpenAPI spec first

  最初にOpenAPIスキーマを書く

- Ensure that implementation matches the spec

  実装がスキーマに一致していることを確認

- More organized, requires more upfront planning

  より整理されているが、事前の計画が必要

- Recommended in [our blog post](https://evilmartians.com/chronicles/let-there-be-docs-a-documentation-first-approach-to-rails-api-development)

  当社のブログ記事で推奨

![](https://evilmartians.com/social-cards/chronicles/let-there-be-docs-a-documentation-first-approach-to-rails-api-development.jpg){class="h-36 absolute right-220px bottom-54px"}
<qr-code url="https://evilmartians.com/chronicles/let-there-be-docs-a-documentation-first-approach-to-rails-api-development" caption="Blog post" class="w-36 absolute bottom-48px right-60px" />


---
class: annotated-list
---

# Different approaches / 異なるアプローチ

## Implementation First / 実装ファースト

- Ensures that spec matches the implementation

  スキーマが実装に一致していることを確認

- Simpler for small teams 

  小規模チームに適している

- Faster iteration

  より速い反復開発

- Today's topic

  今日の話題

---
class: annotated-list
---

# Typical Rails API stack

典型的なRails APIスタック

- Database: stores data and has types

  データを保存し、データ型の情報を持つ

- Models: defines relationships and introspects database

  モデルは関係を定義し、データベースを調査

- Controllers: handle requests and responses

  コントローラーはリクエストとレスポンスを処理

- Serializers: generates JSON for API responses from models

  シリアライザーはモデルからAPIレスポンスのJSONを生成


---
layout: two-cols-header
class: text-sm annotated-list
---

# Existing Tools Review

既存ツールのレビュー

::right::

## RSwag

- ✅ Actively maintained

  アクティブに保守されている

- ✅ OpenAPI 3 support

  OpenAPI 3をサポート



::left::

## Swagger Blocks
- ❌ Abandoned

  開発が停止
- ⚠️ Limited OpenAPI 3.0 support

  OpenAPI 3.0の限定的なサポート

## Apipie

- ❌ OpenAPI 2.0 only

  OpenAPI 2.0のみ

- ❌ Abandoned

  開発が停止

<!--
オプションが複数があるみたいんですけど、RSwagしか何もないんです。
-->

---
layout: two-cols-header
class: annotated-list
---

# RSwag: Pros and Cons

RSwag：長所と短所

::left::

## Pros / 長所

- ✅ Works well

  正常に動作

- ✅ Maintained

  メンテナンスされている

- ✅ Good integration

  良好な統合

::right::

## Cons / 短所

- ❌ No separate type definitions

  個別の型定義がない

- ❌ Manual template maintenance

  テンプレートの手動管理が必要

- ❌ $ref support limitations

  $refサポートの制限

---

# Typical RSwag test

典型的なRSwagテスト

```ruby {all|8-15}{at:1,class:'!children:text-sm'}
# spec/requests/api/v1/foos_spec.rb
RSpec.describe "/api/v1/foos", openapi_spec: "v1/schema.yml" do
  path "/v1/foos/{id}" do
    get "Get a Foo" do
      parameter name: :page, in: :query, schema: { type: :integer, default: 1 }, required: false, description: "The page number to retrieve"

      response "200", "A successful response" do
        schema type: :object,
               properties: {
                 id: { type: :integer },
                 full_name: { type: :string },
                 bar: { type: :object, properties: { … }}
               },
               required: [ 'id', 'full_name' ]

        run_test!
      end
    end
  end
end
```

---
class: annotated-list
---

# ❌ No separate type definitions

個別の型定義がない

 - What if we could get type info from serializers…

   シリアライザーからデータ型情報を取得できたらいいな…

```ruby {all}{at:1,class:'!children:text-sm'}
class FooSerializer < ActiveModel::Serializer

  attribute :id # Type can be inferred from the model Foo

  attribute :full_name do # Need to declare somehow
    first_name + ' ' + last_name
  end

  has_one :bar, serializer: BarSerializer

end
```


---
class: annotated-list
---

# Introducing Typelizer gem

Typelizer gemの紹介

- Generates TypeScript type definitions

  TypeScriptの型定義を生成

- Works with several serializer libraries

  いくつかのシリアライザーライブラリと連携

  - ActiveModelSerializer
  - Alba

- Made by a martian Svyatoslav [@skryukov](https://github.com/skryukov) 

  火星人のスヴャトスラフさんが作ったgem

- But how it can help us with OpenAPI schema?

  しかし、OpenAPIスキーマにどのように役立つのか？

![skryukov/typelizer repository card](https://opengraph.githubassets.com/49b49450898dcd99f2da5afe1ebcd6b88d89a8533afa0afe4156e141d5c790e5/skryukov/typelizer){class="h-36 absolute right-60px bottom-200px"}
<qr-code url="https://github.com/skryukov/typelizer" caption="Typelizer gem" class="w-36 absolute bottom-48px right-60px" />


---
layout: center
class: annotated-list
---

# Let's hack around and find out!

ハックをやってみて、どうなってしまうか見てみよう！

<br class="mb-8" />

- Can we extract and re-use type information without generating TypeScript defs?

  TypeScriptの定義を生成せずに、型情報を抽出して再利用する方法できるのか？

---

# Step 1: Add annotations to serializers

スキーマにアノテーションを追加

```ruby
class FooSerializer < ActiveModel::Serializer
  include Typelizer::DSL

  attribute :id # Will be inferred from the model Foo

  typelize :string
  attribute :full_name do
    first_name + ' ' + last_name
  end

  has_one :bar, serializer: BarSerializer
end
```

---

# Step 2: Define RSwag schema template

RSwag用のスキーマテンプレートを定義

```ruby
# spec/swagger_helper.rb
RSpec.configure do |config|
  config.openapi_specs = {
    "schema.yml" => {
      openapi: "3.1.0",
      paths: {}, # RSwag will fill this in
      components: {
        schemas: {
          Typelizer::Generator.new.interfaces.to_h do |interface|
            [
              interface.name,
              # Magic is here
            ]
          end
        }
      }
    }
  }
end
```

---

# Step 3: Convert typelizer data to OpenAPI

TypelizerのデータをOpenAPIスキーマに変換

```ruby {all}{at:1,class:'!children:text-xs'}
{
  type: :object,
  properties: interface.properties.to_h do |property|
    definition = case property.type
                  when Typelizer::Interface
                    { :$ref => "#/components/schemas/#{property.type.name}" }
                  else
                    { type: property.type.to_s }
                  end
    
    definition[:nullable] = true if property.nullable
    definition[:description] = property.comment if property.comment
    definition[:enum] = property.enum if property.enum    
    definition = { type: :array, items: definition } if property.multi
    [
      property.name,
      definition
    ]
  end,
  required: interface.properties.reject(&:optional).map(&:name)
}
```

---

# Step 4: Write RSwag specs as usual

通常通りRSwagスペックを記述

```ruby {all|13}{at:1,class:'!children:text-sm'}
# spec/requests/api/v1/foos_spec.rb
require "swagger_helper"

RSpec.describe "/api/v1/foos", openapi_spec: "v1/schema.yml" do
  path "/v1/foos" do
    get "List Foos" do
      produces "application/json"
      description "Returns a collection of foos"

      parameter name: :page, in: :query, schema: { type: :integer, default: 1 }, required: false, description: "The page number to retrieve"

      response "200", "A successful response" do
        schema type: :array, items: { "$ref" => "#/components/schemas/Foo" }

        run_test!
      end
    end
  end
end
```

---
class: annotated-list
---

# Hint: AI can rewrite specs to RSwag

ヒント：AIはスペックをRSwagにうまく書き換えます

- If there are already controller or request RSpec tests

  すでにコントローラーまたはリクエストRSpecテストがある場合

- Claude AI can rewrite them to RSwag specs

  Claude AIはそれらをRSwagスペックに書き換えます

- Though amount of manual work for re-checking is qute high

  ただし、再確認のための手作業がかなり多い


---
layout: two-cols-header
---

# And voila!

そして、できあがり！

```sh
$ bundle exec rails rswag
```

::left::

```yaml {all}{at:1,class:'!children:text-sm'}
paths:
  /v1/foos:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Foo'
```

::right::

```yaml  {all}{at:1,class:'!children:text-sm'}
components:
  schemas:
    Foo:
      type: object
      properties:
        id:
          type: integer
        full_name:
          type: string
        bar:
          $ref: '#/components/schemas/Bar'
```

<!--
以上です。できました！
`rails rswag`というコマンドを実行すると、OpenAPIスキーマを生成してくれます！
-->

---
class: annotated-list
---

# The recipe is

レシピは

- Generate schema definitions from serializers

  シリアライザーからTypeScriptの型を生成

- Describe endpoint schemas as RSwag specs

  エンドポイントスキーマをRSwagスペックとして記述

- Reference schema definitions using `$ref` in specs

  スペックで`$ref`を使用してスキーマ定義を参照

---
class: annotated-list
---

# Validating generated schema

生成されたスキーマの検証

- Use [Spectral](https://stoplight.io/open-source/spectral) to validate OpenAPI schema

  OpenAPIスキーマを検証するためにSpectralを使用

- Add following check to your Github Actions

  Github Actionsに以下のチェックを追加

  ```yaml
  - uses: stoplightio/spectral-action@latest
    with:
      file_glob: 'openapi/**/schema.yaml'
      spectral_ruleset: 'openapi/.spectral.yml'
  ```

<qr-code url="https://stoplight.io/open-source/spectral" caption="Spectral" class="w-36 absolute bottom-48px right-60px" />

---
class: annotated-list
---

# Ensuring schema is re-generated

スキーマが再生成されることを確認

- Add following check to your Github Actions using plain git commands

  次のチェックを、プレーンなgitコマンドを使用してGithub Actionsに追加

  ```yaml
  - name: Re-generate OpenAPI spec and check if it is up-to-date
    run: |
      bundle exec rails rswag
      if [[ -n $(git status --porcelain openapi/) ]]; then
        echo "::error::OpenAPI documentation is out of date. Please run `rails rswag` locally and commit the changes."
        git status
        git diff
        exit 1
      fi
  ```

---

# Detect breaking changes

- Use [oasdiff] to get a diff between two OpenAPI schemas

  [oasdiff]を使用して2つのOpenAPIスキーマの差分を取得

  ```sh
  docker run --rm -t -v $(pwd):/specs:ro tufin/oasdiff changelog \
    old_schema.yml new_schema.yml -f html > oas_diff.html
  ```

<qr-code url="https://github.com/Tufin/oasdiff" caption="oasdiff" class="w-36 absolute bottom-48px right-60px" />

[oasdiff]: https://github.com/Tufin/oasdiff "OpenAPI Diff and Breaking Changes"

---

# Is it used in production?

production環境で使われているのか？

Yes, at [Whop.com](https://whop.com)!

Whop is rapidly growing social commerce platform, development speed is their top priority, and ability to generate OpenAPI schema from serializers is a major pain relief.

<LightOrDark>
  <template #dark>
    <img alt="Pull request with OpenAPI schema generation for Whop.com: stats header" src="/images/whop-openapi-pr-stats-header-dark.png" />
  </template>
  <template #light>
    <img alt="Pull request with OpenAPI schema generation for Whop.com: stats header" src="/images/whop-openapi-pr-stats-header-light.png" />
  </template>
</LightOrDark>

---
layout: center
---

> Thanks @envek, this setup is pretty sweet, much better than manually editing a massive text file.{.text-xl}

— Diego Figueroa, staff engineer at Whop.com 

---
layout: cover
---

# Thank you!

ご清聴ありがとうございました！

<div class="grid grid-cols-[8rem_3fr_4fr] mt-6 gap-2">

<div class="justify-self-start">
<img alt="Andrey Novikov" src="https://secure.gravatar.com/avatar/d0e95abdd0aed671ebd0920c16d393d4?s=512" class="w-32 h-32 object-contain" />
</div>

<ul class="list-none">
<li><a href="https://github.com/Envek"><logos-github-icon class="dark:invert" /> @Envek</a></li>
<li><a href="https://twitter.com/Envek"><logos-twitter /> @Envek</a></li>
<li><a href="https://mastodon.social/@Envek"><logos-mastodon-icon /> @Envek<span class="text-sm tracking-tight">@mastodon.social</span></a></li>
<li><a href="https://bsky.app/profile/envek.bsky.social"><logos-bluesky />  @envek<span class="text-sm tracking-tight">.bsky.social</span></a></li>
</ul>

<div>
<qr-code url="https://github.com/Envek" caption="github.com/Envek" class="w-32 mt-2" />
</div>

<div class="justify-self-start">
<a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="w-32 h-32 object-contain block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="w-32 h-32 object-contain hidden dark:block" /></a>
</div>

<div>

- <logos-github-icon class="dark:invert" /> [@evilmartians](https://github.com/evilmartians?utm_source=kyobashirb&utm_medium=slides&utm_campaign=rswag-typelizer)
- <logos-twitter /> [@evilmartians](https://twitter.com/evilmartians/?utm_source=kyobashirb&utm_medium=slides&utm_campaign=rswag-typelizer)
- <logos-mastodon-icon /> [@evilmartians<span class="text-sm tracking-tight">@mastodon.social</span>](https://mastodon.social/@evilmartians?utm_source=kyobashirb&utm_medium=slides&utm_campaign=rswag-typelizer)
- <logos-bluesky /> [@evilmartians.com](https://bsky.app/profile/evilmartians.com?utm_source=kyobashirb&utm_medium=slides&utm_campaign=rswag-typelizer)
</div>

<div>
<qr-code url="https://evilmartians.com/" caption="evilmartians.com" class="w-32 my-2" />
</div>

<div class="col-span-3">

Our awesome blog: <span v-mark.orange class="font-bold">[evilmartians.com/chronicles](https://evilmartians.com/chronicles/?utm_source=kyobashirb&utm_medium=slides&utm_campaign=rswag-typelizer)</span>!

</div>
</div>

<style>
  h1 { margin-bottom: 4px; }
  ul a { border-bottom: none !important; }
  ul { list-style-type: none !important; }
  ul li { margin-left: 0; padding-left: 0; }
</style>
