---
title: "【Golang】Azure OpenAI で Embedding したベクトルを使って、自前検索エンジンを作ろう"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postgresql", "openai", "go", "docker", "azure"]
published: false
publication_name: "microsoft"
---

この記事は、[Azure Advent Calendar 2023](https://qiita.com/advent-calendar/2023/microsoft-azure-tech) の 10 日目の記事です。

# はじめに

Azure OpenAI Service では、 `text-embedding-ada-002` というモデルを使って、文章を 1536 次元のベクトルに Embedding できます。
また、PostgreSQL では、 `pgvector` という拡張機能を使って、ベクトルを保存・検索機能を導入できます。

今回は、これらを組み合わせて、Azure OpenAI で Embedding したベクトルを Golang のアプリケーションから PostgreSQL に保存し、類似度検索する方法を紹介します。

https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/models#embeddings-models

# pgvector とは

pgvector は、PostgreSQL にベクトルデータを保存・検索する機能を追加する拡張機能です。
ベクトルデータを保存するためのデータ型と、ベクトルデータを検索するための演算子を追加します。

## 拡張機能のインストール

以下の Dockerfile を利用することで、簡単に PostgreSQL に pgvector を導入できます。
PostgresSQL のバージョンなどは、適宜変更してください。

```Dockerfile
FROM postgres:15-bullseye

# insatll git make
RUN apt-get update && apt-get install -y git make gcc postgresql-server-dev-15

# install pgvector
WORKDIR /tmp

RUN git clone --branch v0.5.1 https://github.com/pgvector/pgvector.git

WORKDIR /tmp/pgvector

RUN make
RUN make install
```

## ベクトルデータの保存・検索

スキーマの定義では、以下のように `vector` 型を使って、ベクトルデータを保存できます。

```sql
--   3次元ベクトルを保存するテーブル
CREATE TABLE vectors (
  id int,
  vector vector(3)
);
```

ベクトルの保存には、以下のように `vector` 型のリテラルを使って、ベクトルデータを保存できます。

```sql
--   ベクトルデータを保存
INSERT INTO vectors VALUES (1, '[1, 2, 3]');
```

また、ベクトルデータを検索するためには、以下の演算子を使用して類似度検索を実装できます。

| 演算子 | 検索方法 |
| --- | --- |
| `<=>` | コサイン類似度 |
| `<->` | L2 ノルムによる距離 |
| `<#>` | 内積による類似度 |

```sql
--   コサイン類似度による類似度検索
SELECT * FROM vectors WHERE vector <=> '[1, 2, 3]';
```

詳細な使い方は、以下のリポジトリを参照してください。

https://github.com/pgvector/pgvector


# Golang アプリでの実装編

## スキーマの定義

Azure OpenAI で Embedding したベクトルは、1536 次元のベクトルになります。
したがって、以下のように `vector` 型を使って、ベクトルデータを保存します。

```sql
--   1536次元ベクトルを保存するテーブル
CREATE TABLE vectors (
  id text,
  plain_text text,
  vector vector(1536)
);
```

## Golang アプリケーションからpostgresqlに接続する

Golang から PostgreSQL に接続するためには、以下のように `database/sql` と `github.com/lib/pq` を利用します。

`config` は、環境変数から PostgreSQL への接続情報を取得するために利用しています。
環境変数が取れなかったときのエラーハンドリングだったり、オレオレ実装を組み込んでいるものなので、需要があれば別途記事にします。

```go
package database

import (
	"database/sql"
	"fmt"

	"github.com/kazuki-komori/sample/config"
	_ "github.com/lib/pq"
)

// Postgresとの接続を確立する
func NewPostgres() (*sql.DB, error) {
	dsn := fmt.Sprintf(
		"postgres://%s:%s@%s:%s/%s?sslmode=%s",
		config.POSTGRES_USER(),
		config.POSTGRES_PASSWORD(),
		config.POSTGRES_HOST(),
		config.POSTGRES_PORT(),
		config.POSTGRES_DB(),
		config.POSTGRES_SSLMODE(),
	)

	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, err
	}

	return db, nil
}
```

## ベクトルデータの保存

Azure OpenAI で Embedding したベクトルは、1536 次元のベクトルになります。
今回は ORM を使わずに、 `database/sql` を使って、ベクトルデータを保存します。

```go
package database

import (
	"database/sql"

	"github.com/kazuki-komori/sample/helper"
)


func (r *ContentRepository) Save(ID string, plainText string, vector []float64) error {
	query := `
		INSERT INTO vectors (id, plain_text, vector)
		VALUES ($1, $2, $3)
		`

	resp := r.db.QueryRow(query, ID, plainText, helper.Vector(vector))

	if err := resp.Err(); err != nil {
		return err
	}

	return nil
}
```

`Save` メソッドは、以下のように `[]float64` を `vector` 型に変換して、ベクトルデータを保存します。
引数として与えられる `vector` に、Azure OpenAI で Embedding したベクトルをそのまま渡せば、ベクトルデータを保存できます。

:::message alert
`github.com/lib/pq` には、 `pg.Array` という `[]float64` を `[]string` に変換する機能がありますが、実際に変換すると、 `{0.1, 0.2, 0.3}` という形式になります。
しかし `pgvector` では、 `[0.1, 0.2, 0.3]` という形式でベクトルデータを保存する必要があり、 `helper.Vector` はそのためのヘルパー関数です。
:::

ヘルパー関数は以下のように、スライスを受け取って、 `[0.1, 0.2, 0.3]` という形式の文字列に変換します。


```go
package helper

import (
	"fmt"
	"reflect"
	"strings"
)

func Vector(array interface{}) string {
	sliceValue := reflect.ValueOf(array)

	if sliceValue.Kind() != reflect.Slice {
		return "Not a slice"
	}

	strSlice := make([]string, sliceValue.Len())
	for i := 0; i < sliceValue.Len(); i++ {
		element := sliceValue.Index(i)
		strSlice[i] = fmt.Sprintf("%v", element.Interface())
	}

	result := "[" + strings.Join(strSlice, ", ") + "]"
	return result
}
```

以下にテストコードを書いておきましたので、利用例の参考にしてください。

:::details テストコード
```go
package helper_test

import (
	"testing"

	"github.com/kazuki-komori/sample/helper"
)

func TestVector(t *testing.T) {
	tests := []struct {
		name     string
		array    interface{}
		expected string
	}{
		{
			name:     "int",
			array:    []int{1, 2, 3},
			expected: "[1, 2, 3]",
		},
		{
			name:     "float64",
			array:    []float64{1.5, 2.5, 3.5},
			expected: "[1.5, 2.5, 3.5]",
		},
		{
			name:     "string",
			array:    []string{"a", "b", "c"},
			expected: "[a, b, c]",
		},
	}

	for _, test := range tests {
		t.Run(test.name, func(t *testing.T) {
			actual := helper.Vector(test.array)
			if actual != test.expected {
				t.Errorf("got %v\nwant %v", actual, test.expected)
			}
		})
	}
}
```
:::

## ベクトルデータの検索

より実用的な実装をしてみました。
`Similarity` は、コサイン類似度で上位 N 件の類似度のものを取得するメソッドです。
これにより、ベクトルデータを保存したテーブルから、類似度の高いものを取得できます。

```go
package database

import (
	"database/sql"

	"github.com/kazuki-komori/sample/domain/model"
	"github.com/kazuki-komori/sample/helper"
)

// コサイン類似度で上位N件の類似度のものを取得する
func (r *ContentRepository) Similarity(vector []float64, topN int) (*[]model.Vector, error) {
	query := `
		SELECT id, plain_text
		FROM vectors
		ORDER BY vector <=> $1
		LIMIT $2
		`

	rows, err := r.db.Query(query, helper.Vector(vector), topN)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var vectors []model.Vector
	for rows.Next() {
		var content model.Vector
		err := rows.Scan(&content.ID, &content.PlainText)
		if err != nil {
			return nil, err
		}
		vectors = append(vectors, content)
	}

	return &vectors, nil
}
```

# まとめ

今回は、Azure OpenAI で Embedding したベクトルを Golang のアプリケーションから PostgreSQL に保存し、類似度検索する方法を紹介しました。

Azure OpenAI で Embedding したベクトルを PostgreSQL に保存することで、ベクトルデータを保存・検索する機能を簡単に導入できます。

ベクトルを保存する際にヘルパー関数を使うなど、少しトリッキーな実装が必要になりますが、検索エンジンを自前で実装するには、十分な機能を提供してくれると思います。
今回紹介したような実装を利用することで、文書検索など様々なシナリオで活用できると思います。

# 参考文献

https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/models#embeddings-models

https://github.com/pgvector/pgvector
