---
title: "Biep を使って Azure App Service を Private Endpoint にデプロイする方法"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "bicep", "appservice"]
published: false
---

# はじめに

Bicep は、宣言型の構文を使用して Azure リソースを定義するためのドメイン固有言語（DSL）です。Bicep を使うと、IaC (Infrastructure as Code) により一貫性のあるデプロイが可能になります。
今回は、Bicep を使って Azure App Service を閉域網として Private Endpoint にデプロイする方法を紹介します。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/overview

# 事前準備

最新の [Azure Developer CLI (azd)](https://learn.microsoft.com/ja-jp/azure/developer/azure-developer-cli/reference) をインストールしておきましょう。
また、VSCode 用の [Bicep 拡張機能](https://marketplace.visualstudio.com/items?WT.mc_id=AZ-MVP-5003113&itemName=ms-azuretools.vscode-bicep)を使うと、補完や構文チェックができるので便利です。

```bash
$ azd version
> azd version 1.2.0
```

# プロジェクトの作成

`azd init` コマンドを使って、`Starter - Bicep` を選択すると既定の Bicep プロジェクトを作成してくれます。

```bash
$ azd init

? Select a project template:  [Use arrows to move, type to filter]
  Minimal
> Starter - Bicep
  Starter - Terraform
  React Web App with C# API and MongoDB
  React Web App with C# API and SQL Database
  React Web App with Java API and MongoDB
  React Web App with Node.js API and MongoDB
```

初期化すると以下のようなディレクトリ構成でプロジェクトが作成されます。
```bash
$ tree bicep-project

bicep-project/
├── README.md
├── azure.yaml
└── infra
    ├── abbreviations.json
    ├── main.bicep
    └── main.parameters.json
```

# リソースの定義

Bicep は、各リソースをモジュールとして定義することで、リソースの再利用性を高めることができます。
今回は、`./infra/resource` ディレクトリに各リソースを定義し、`main.bicep` で参照するようにします。

# 仮想ネットワークの作成

まずは、仮想ネットワークを作成します。`./infra/resource/vnet.bicep` を作成し、以下のように定義します。パラメータで `addressPrefix` としてアドレス空間を受け取り、再利用性を高めています。

```bicep:./infra/resource/vnet.bicep
param location string = resourceGroup().location
param name string
// 作成するvnetのアドレス空間
param addressPrefix string

resource vnet 'Microsoft.Network/virtualNetworks@2021-02-01' = {
  name: name
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [addressPrefix]
    }
  }
}

output name string = vnet.name
output id string = vnet.id
```

# サブネットの作成

次に、先ほど作成した VNet に紐づくサブネットを作成するモジュールを定義します。
VNet に対して `existing` キーワードを指定し、既存の VNet の名前を指定することで、既存のリソースを参照できます。
また、参照先の subnet で `parent` プロパティに親リソースの VNet を指定することで、親子関係を定義できます。

```bicep:./infra/resource/subnet.bicep
param name string
param addressPrefix string
param existingVirtualNetworkName string

// 既に存在する仮想ネットワークを参照
resource existingVirtualNetwork 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  scope: resourceGroup()
  name: existingVirtualNetworkName
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  parent: existingVirtualNetwork
  name: name
  properties: {
    addressPrefix: addressPrefix
  }
}

output id string = subnet.id
output name string = subnet.name
```

# App Service Plan の作成

App Service Plan は、App Service の実行環境を定義するリソースです。
実行するアプリケーションの規模に応じて、サイズやインスタンス数を変更できます。

```bicep:./infra/resource/app-service-plan.bicep
param name string
param location string = resourceGroup().location
param sku object
param kind string = ''

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: name
  location: location
  sku: sku
  kind: kind
}

output id string = appServicePlan.id
output name string = appServicePlan.name
```

# App Service の作成

App Service は、Web アプリケーションや API をホストするための PaaS サービスです。
App Service は、App Service Plan に紐づいているため、App Service Plan の名前を指定することで、App Service Plan を参照できます。
また、`publicNetworkAccess` に `Enabled` または `Disabled` を指定することで、インターネットからのアクセスを許可するかどうかを指定できます。

```bicep:./infra/resource/app-service.bicep
param name string
param location string = resourceGroup().location
param existingAppServicePlanName string
@allowed(['Enabled', 'Disabled'])
param publicNetworkAccess string

resource existingAppServicePlan 'Microsoft.Web/serverfarms@2022-09-01' existing = {
  scope: resourceGroup()
  name: existingAppServicePlanName
}

resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: name
  location: location
  properties: {
    serverFarmId: existingAppServicePlan.id
    publicNetworkAccess: publicNetworkAccess
  }
}

output id string = appService.id
output name string = appService.name
```

# Private Endpoint と private DNS Zone の作成

Private Endpoint は、Azure リソースにプライベート IP アドレスを割り当てることができるリソースです。
Private DNS Zone は、Private Endpoint で割り当てたプライベート IP アドレスを仮想ネットワーク内で名前解決するための DNS ゾーンです。
また、Azure ポータルでは Private Endpoint を作成すると、Private DNS Zone が自動的に作成されますが、Bicep では Private DNS Zone を定義する必要があります。

ここで、DNS Zone 名は `privatelink.{リソースがデプロイされるURL}` を指定します。
つまり、App Service の場合、デプロイ先の URL は `azurewebsites.net` となるため、DNS Zone 名は `privatelink.azurewebsites.net` を指定します。

```bicep:./infra/resource/private-endpoint.bicep
param location string = resourceGroup().location
param dnsZoneName string
param linkVnetId string
param name string
param subnetId string
param privateLinkServiceId string
param privateLinkServiceGroupIds array

// private DNS zoneの作成
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.${dnsZoneName}'
  location: 'global'
}

// private DNS zoneを仮想ネットワークに紐付ける
resource virtualNetworkLinks 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  name: 'vnet-link-${name}'
  location: 'global'
  parent: privateDnsZone
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: linkVnetId
    }
  }
}

// private DNS zoneを入れるためのgroup
resource privateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-04-01' = {
  parent: privateEndpoint
  name: privateDnsZone.name
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'private-link-${name}'
        properties: {
          privateDnsZoneId: privateDnsZone.id
        }
      }
    ]
  }
}

// private endpointの作成
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-02-01' = {
  name: '${name}-endpoint'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${name}-connection}'
        properties: {
          privateLinkServiceId: privateLinkServiceId
          groupIds: privateLinkServiceGroupIds
        }
      }
    ]
  }
}

output privateEndpointId string = privateEndpoint.id
output privateEndpointName string = privateEndpoint.name
```

プライベートエンドポイントを作成する上で、`privateLinkServiceConnections[0].groupIds` を指定する必要がありますが、これはプライベートエンドポイントとしてデプロイしたい対象のリソースごとに固有の値が定められており、以下のドキュメントから確認できます。

https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/private-link/private-endpoint-overview.md

# main.bicep に各モジュールを読み込む

ここまでで、各リソースの Bicep ファイルを作成できたので、main.bicep に各モジュールを読み込みます。

```bicep:./infra/main.bicep
targetScope = 'subscription'

param resourceGroupName string = 'rg-bicep-private-endpoint-sample'
param location string = 'japaneast'

resource resourceGroup 'Microsoft.Resources/resourceGroups@2022-09-01' = {
  name: resourceGroupName
  location: location
}

module vnet 'resource/vnet.bicep' = {
  name: 'vnet'
  scope: resourceGroup
  params: {
    name: 'vnet-private-endpoint'
    addressPrefix: '10.0.0.0/16'
    location: location
  }
}

module appServiceSubnet 'resource/subnet.bicep' = {
  name: 'sub-bicep-private-endpoint-sample'
  scope: resourceGroup
  params: {
    name: 'subnet-app-service'
    addressPrefix: '10.0.0.0/24'
    existingVirtualNetworkName: vnet.outputs.name
  }
}

module appServicePlan 'resource/app-service-plan.bicep' = {
  name: 'ase-bicep-private-endpoint-sample'
  scope: resourceGroup
  params: {
    name: 'ase-bicep-private-endpoint-sample'
    location: location
    sku: {
      name: 'F1'
      capacity: 1
    }
    kind: 'linux'
  }
}

module appService 'resource/app-service.bicep' = {
  name: 'app-bicep-private-endpoint-sample'
  scope: resourceGroup
  params: {
    name: 'app-bicep-private-endpoint-sample'
    location: location
    existingAppServicePlanName: appServicePlan.outputs.name
    publicNetworkAccess: 'Disabled'
  }
}

module appServicePrivateEndpoint 'resource/private-endpoint.bicep' = {
  name: 'app-service-private-endpoint'
  scope: resourceGroup
  params: {
    name: appService.outputs.name
    location: location
    subnetId: appServiceSubnet.outputs.id
    privateLinkServiceId: appService.outputs.id
    privateLinkServiceGroupIds: ['sites']
    dnsZoneName: 'azurewebsites.net'
    linkVnetId: vnet.outputs.id
  }
}
```

# デプロイ

`azd up` を実行して、リソースをデプロイします。

```bash
$ azd up
```


デプロイ後、App Service の URL にブラウザからアクセスすると、403 エラーを確認できるでしょう。

# リソースの削除

`azd down` を実行して、全てのリソースを削除します。
```bash
$ azd down
```

# おまけ：仮想マシンを作成して、App Service にアクセスする


