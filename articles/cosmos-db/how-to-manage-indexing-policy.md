---
title: Azure Cosmos DB でインデックス作成ポリシーを管理する
description: Azure Cosmos DB でインデックス作成ポリシーを管理する方法について説明します
author: ThomasWeiss
ms.service: cosmos-db
ms.topic: sample
ms.date: 04/08/2019
ms.author: thweiss
ms.openlocfilehash: 76275420e1e6ed7fdec8309da9e11a272f08fee0
ms.sourcegitcommit: bf509e05e4b1dc5553b4483dfcc2221055fa80f2
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/22/2019
ms.locfileid: "60005590"
---
# <a name="manage-indexing-policies-in-azure-cosmos-db"></a>Azure Cosmos DB でインデックス作成ポリシーを管理する

Azure Cosmos DB では、コンテナーごとに定義された[インデックス作成ポリシー](index-policy.md)に従ってデータのインデックスが作成されます。 新しく作成したコンテナーの既定のインデックス作成ポリシーでは、文字列または数値には範囲インデックス、Point 型の GeoJSON オブジェクトには空間インデックスが適用されます。 このポリシーは、次の方法でオーバーライドできます。

- Azure portal を使用する
- Azure CLI を使用する
- SDK のいずれかを使用する

[インデックス作成ポリシーの更新](index-policy.md#modifying-the-indexing-policy)により、インデックスの変換がトリガーされます。 この変換の進行状況は、SDK から追跡することもできます。

## <a name="use-the-azure-portal"></a>Azure ポータルの使用

Azure Cosmos のコンテナーには、そのインデックス作成ポリシーが Azure portal で直接編集できる JSON ドキュメントとして格納されます。

1. [Azure Portal](https://portal.azure.com/) にサインインします。

1. 新しい Azure Cosmos アカウントを作成するか、既存のアカウントを選択します。

1. **[データ エクスプローラー]** ウィンドウを開いて、操作の対象となるコンテナーを選択します。

1. **[Scale & Settings]\(スケールと設定\)** をクリックします。

1. インデックス作成ポリシーの JSON ドキュメントに変更を加えます ([以下](#indexing-policy-examples)の例を参照)。

1. 完了したら、**[保存]** をクリックします。

![Azure portal を使用してインデックス作成を管理する](./media/how-to-manage-indexing-policy/indexing-policy-portal.png)

## <a name="use-the-azure-cli"></a>Azure CLI の使用

Azure CLI で [az cosmosdb collection update](/cli/azure/cosmosdb/collection#az-cosmosdb-collection-update) を使用すると、コンテナーのインデックス作成ポリシーの JSON 定義を置き換えることができます。

```azurecli-interactive
az cosmosdb collection update \
    --resource-group-name $resourceGroupName \
    --name $accountName \
    --db-name $databaseName \
    --collection-name $containerName \
    --indexing-policy "{\"indexingMode\": \"consistent\", \"includedPaths\": [{ \"path\": \"/*\", \"indexes\": [{ \"dataType\": \"String\", \"kind\": \"Range\" }] }], \"excludedPaths\": [{ \"path\": \"/headquarters/employees/?\" } ]}"
```

## <a name="use-the-net-sdk-v2"></a>.NET SDK V2 の使用

[.NET SDK v2](https://www.nuget.org/packages/Microsoft.Azure.DocumentDB/) の `DocumentCollection` オブジェクト (その使用法については[こちらのクイック スタート](create-sql-api-dotnet.md)を参照) では `IndexingPolicy` プロパティを公開しています。これを使用すると、`IndexingMode` を変更したり `IncludedPaths` や `ExcludedPaths` を追加または削除したりすることができます。

```csharp
// retrieve the container's details
ResourceResponse<DocumentCollection> containerResponse = await client.ReadDocumentCollectionAsync(UriFactory.CreateDocumentCollectionUri("database", "container"));
// set the indexing mode to Consistent
containerResponse.Resource.IndexingPolicy.IndexingMode = IndexingMode.Consistent;
// add an excluded path
containerResponse.Resource.IndexingPolicy.ExcludedPaths.Add(new ExcludedPath { Path = "/headquarters/employees/?" });
// update the container with our changes
await client.ReplaceDocumentCollectionAsync(containerResponse.Resource);
```

インデックス変換の進行状況を追跡するには、`PopulateQuotaInfo` プロパティを `true` に設定する `RequestOptions` オブジェクトを渡します。

```csharp
// retrieve the container's details
ResourceResponse<DocumentCollection> container = await client.ReadDocumentCollectionAsync(UriFactory.CreateDocumentCollectionUri("database", "container"), new RequestOptions { PopulateQuotaInfo = true });
// retrieve the index transformation progress from the result
long indexTransformationProgress = container.IndexTransformationProgress;
```

## <a name="use-the-java-sdk"></a>Java SDK の使用

[Java SDK](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb) の `DocumentCollection` オブジェクト (その使用法については[こちらのクイック スタート](create-sql-api-java.md)を参照) では、`getIndexingPolicy()` および `setIndexingPolicy()` メソッドを公開しています。 これらによって操作される `IndexingPolicy` オブジェクトを使用すると、インデックス作成モードを変更したり、対象のパスと対象外のパスを追加または削除したりすることができます。

```java
// retrieve the container's details
Observable<ResourceResponse<DocumentCollection>> containerResponse = client.readCollection(String.format("/dbs/%s/colls/%s", "database", "container"), null);
containerResponse.subscribe(result -> {
    DocumentCollection container = result.getResource();
    IndexingPolicy indexingPolicy = container.getIndexingPolicy();
    // set the indexing mode to Consistent
    indexingPolicy.setIndexingMode(IndexingMode.Consistent);
    Collection<ExcludedPath> excludedPaths = new ArrayList<>();
    ExcludedPath excludedPath = new ExcludedPath();
    excludedPath.setPath("/*");
    // add an excluded path
    excludedPaths.add(excludedPath);
    indexingPolicy.setExcludedPaths(excludedPaths);
    // update the container with our changes
    client.replaceCollection(container, null);
});
```

コンテナーに対するインデックス変換の進行状況を追跡するには、クォータ情報の読み込みを要求する `RequestOptions` オブジェクトを渡したうえで、その値を `x-ms-documentdb-collection-index-transformation-progress` 応答ヘッダーから取得します。

```java
// set the RequestOptions object
RequestOptions requestOptions = new RequestOptions();
requestOptions.setPopulateQuotaInfo(true);
// retrieve the container's details
Observable<ResourceResponse<DocumentCollection>> containerResponse = client.readCollection(String.format("/dbs/%s/colls/%s", "database", "container"), requestOptions);
containerResponse.subscribe(result -> {
    // retrieve the index transformation progress from the response headers
    String indexTransformationProgress = result.getResponseHeaders().get("x-ms-documentdb-collection-index-transformation-progress");
});
```

## <a name="use-the-nodejs-sdk"></a>Node.js SDK の使用

[Node.js SDK](https://www.npmjs.com/package/@azure/cosmos) の `ContainerDefinition` インターフェイス (その使用法については[こちらのクイック スタート](create-sql-api-nodejs.md)を参照) では `indexingPolicy` プロパティを公開しています。これを使用すると、`indexingMode` を変更したり `includedPaths` や `excludedPaths` を追加または削除したりすることができます。

```javascript
// retrieve the container's details
const containerResponse = await client.database('database').container('container').read();
// set the indexing mode to Consistent
containerResponse.body.indexingPolicy.indexingMode = "consistent";
// add an excluded path
containerResponse.body.indexingPolicy.excludedPaths.push({ path: '/headquarters/employees/?' });
// update the container with our changes
const replaceResponse = await client.database('database').container('container').replace(containerResponse.body);
```

コンテナーに対するインデックス変換の進行状況を追跡するには、`populateQuotaInfo` プロパティを `true` に設定する `RequestOptions` オブジェクトを渡したうえで、その値を `x-ms-documentdb-collection-index-transformation-progress` 応答ヘッダーから取得します。

```javascript
// retrieve the container's details
const containerResponse = await client.database('database').container('container').read({
    populateQuotaInfo: true
});
// retrieve the index transformation progress from the response headers
const indexTransformationProgress = replaceResponse.headers['x-ms-documentdb-collection-index-transformation-progress'];
```

## <a name="use-the-python-sdk"></a>Python SDK の使用

[Python SDK](https://pypi.org/project/azure-cosmos/) (その使用法については[こちらのクイック スタート](create-sql-api-python.md)を参照) を使用する場合、コンテナーの構成がディクショナリとして管理されます。 このディクショナリから、インデックス作成ポリシーとそのすべての属性にアクセスすることができます。

```python
containerPath = 'dbs/database/colls/collection'
# retrieve the container's details
container = client.ReadContainer(containerPath)
# set the indexing mode to Consistent
container['indexingPolicy']['indexingMode'] = 'consistent'
# add an excluded path
container['indexingPolicy']['excludedPaths'] = [{"path" : "/headquarters/employees/?"}]
# update the container with our changes
response = client.ReplaceContainer(containerPath, container)
```

## <a name="indexing-policy-examples"></a>インデックス作成ポリシーの例

JSON 形式で示されたインデックス作成ポリシーの例をいくつか紹介します。Azure portal では、これらがこの形式で公開されます。 同じパラメーターは、Azure CLI のほか、任意の SDK で設定することができます。

### <a name="opt-out-policy-to-selectively-exclude-some-property-paths"></a>一部のプロパティ パスを選択的に除外するオプトアウト ポリシー

    {
        "indexingPolicy": "consistent",
        "includedPaths": [
            {
                "path": "/*",
                "indexes": [
                    {
                        "kind": "Range",
                        "dataType": "Number"
                    },
                    {
                        "kind": "Range",
                        "dataType": "String"
                    },
                    {
                        "kind": "Spatial",
                        "dataType": "Point"
                    }
                ]
            }
        ],
        "excludedPaths": [
            {
                "path": "/path/to/single/excluded/property/?"
            },
            {
                "path": "/path/to/root/of/multiple/excluded/properties/*"
            }
        ]
    }

### <a name="opt-in-policy-to-selectively-include-some-property-paths"></a>一部のプロパティ パスを選択的に包含するオプトイン ポリシー

    {
        "indexingPolicy": "consistent",
        "includedPaths": [
            {
                "path": "/path/to/included/property/?",
                "indexes": [
                    {
                        "kind": "Range",
                        "dataType": "Number"
                    }
                ]
            },
            {
                "path": "/path/to/root/of/multiple/included/properties/*",
                "indexes": [
                    {
                        "kind": "Range",
                        "dataType": "Number"
                    }
                ]
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            }
        ]
    }

注:一般的に、モデルに追加される新しいプロパティのインデックスを Azure Cosmos DB がプロアクティブに作成できるよう、**オプトアウト** インデックス作成ポリシーの使用をお勧めします。

### <a name="using-a-spatial-index-on-a-specific-property-path-only"></a>特定のプロパティ パスに対してのみ空間インデックスを使用する

    {
        "indexingPolicy": "consistent",
        "includedPaths": [
            {
                "path": "/*",
                "indexes": [
                    {
                        "kind": "Range",
                        "dataType": "Number"
                    },
                    {
                        "kind": "Range",
                        "dataType": "String"
                    }
                ]
            },
            {
                "path": "/path/to/geojson/property/?",
                "indexes": [
                    {
                        "kind": "Spatial",
                        "dataType": "Point"
                    }
                ]
            }
        ],
        "excludedPaths": []
    }

### <a name="excluding-all-property-paths-but-keeping-indexing-active"></a>インデックス作成をアクティブ状態に保ちながらすべてのプロパティ パスを除外する

このポリシーは、[Time-to-Live (TTL) 機能](time-to-live.md)がアクティブであるものの、(Azure Cosmos DB を純粋なキー/値ストアとして使用するための) セカンダリ インデックスは不要である状況で使用できます。

    {
        "indexingMode": "consistent",
        "includedPaths": [],
        "excludedPaths": [{
            "path": "/*"
        }]
    }

### <a name="no-indexing"></a>インデックス作成なし

    {
        "indexingPolicy": "none"
    }

## <a name="next-steps"></a>次の手順

以下の記事で、インデックス作成についての詳細を参照してください。

- [インデックス作成の概要](index-overview.md)
- [インデックス作成ポリシー](index-policy.md)
