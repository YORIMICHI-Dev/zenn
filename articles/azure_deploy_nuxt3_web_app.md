---
title: "Github ActionsでAzure Web App ServiceにNuxt3をデプロイする"
emoji: "⛹️‍♀️"
type: "tech"
topics: [azure, nuxt3, github]
published: true
---
CI/CDの勉強中に、業務でよく使用しているAzure Web App ServiceにNuxt3をデプロイした作業の自動化を試してみる。

# モチベーション
## Github Actionsを使用してAzure Web App Serviceへのデプロイを自動化したい
Github Actionsを使ってCI/CDについて勉強している中で、実際の業務に応用するためAzure Web App Serviceへのデプロイ方法をいろいろ調べた中で、
Nuxt3のに対してGithub Actionsのデプロイの記述を調べた。

## Azure App ServiceにNuxt3をデプロイしたい
普段はNuxt3をフロントとしたアプリをデプロイするときにはnpm run generateにて静的ファイルを作成した後、
.NETのwwwrootに静的ファイルを移して、.NETを使用したデプロイを行っている。

単純にNuxt3だけをデプロイすることがなかったため、デプロイ手順を調べた。
特に、Static Web AppsにNuxt3をデプロイする方法はいくつかあるが、GIP制御やVNet統合など使いたい機能が不足していたため、
Web App Serviceにてできる方法を調べた。

# Nuxt3をWeb App Serviceにデプロイする準備
## Nuxt3の準備
### Nuxt3プロジェクトの立ち上げ
Nuxt3のホームページからNuxtプロジェクトを立ち上げるコマンドを参照する。
```console
npx nuxi@latest init <project-name>
```
https://nuxt.com/docs/getting-started/installation

Nuxt3が開発モードで立ち上がるか確認する。
```console
npm run dev
```
### Nuxt3のビルド
Nuxt3は通常SSR（サーバーサイドレンダリング）であるが、今回はSPA（シングルページアプリケーション）として運用するため、
nuxt.config.tsの設定をSPAに変更する。
```diff js:nuxt.config.ts
export default defineNuxtConfig({
+  ssr: false,
  devtools: { enabled: true }
})
```
設定後、ビルドより.outputディレクトリが作成されその中にファイルが生成される。
```console
npm run build
```
ビルドしたファイルを確認する。
node .output/server/index.mjs

### package.jsonに起動コマンドを追加
package.jsonにNuxt3起動コマンドを追加する。

```diff json:package.json
{
  "name": "nuxt-app",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare",
+    "start": "node .output/server/index.mjs"
  },
  "dependencies": {
    "nuxt": "^3.13.0",
    "vue": "latest",
    "vue-router": "latest"
  }
}
```

## Azure Web App Serviceの準備
### Azure Web App Serviceの作成
新規にAzure Web App Serviceを作成する。
作成に必要な設定として、ランタイムストックとデプロイについて指定する。
ランタイムストックは特に指定がなければNode 20 LTSを指定する。
![](/images/azure_deploy_nuxt3_web_app/スライド1.JPG)

続いてデプロイについて、継続的デプロイを選択するとGithub Actionsに使用するymlファイルが作成されるが、
この後Github Actionsから設定するため、ここでは継続的デプロイは無効化する。
また認証方法は発行プロファイルを使用するため基本認証にする。
![](/images/azure_deploy_nuxt3_web_app/スライド2.JPG)

### Azure Web App Serviceの確認
作成が完了したら、URLからAzure Web App Serviceにアクセスできるか確認する。

### スタートアップコマンドを設定
Web App ServiceにNuxt3をデプロイするときに、起動するコマンドを設定する。
さきほどpackage.jsonに設定したstartを使いたいため以下のコマンドを設定する。
```console
npm run start
```
![](/images/azure_deploy_nuxt3_web_app/スライド3.JPG)

# Github Actions
## Github Actionsの設定
Github Actionsにworkflowsを追加する。
```diff
Nuxtプロジェクト
+  |--.github
+  |     |--workflows
+  |         |--deployment.html
   |--public
   |--server
   |--.gitignore
   |--app.vue
   |--nuxt.config.tx
   |--package.json
   |--tsconfig.json
```

deployment.ymlは以下を記述する。
```diff yaml
name: Deoployment

on:
  workflow_dispatch:
  push:
    branches:
      - main

env:
  AZURE_WEBAPP_NAME: 'web-app-name'
  AZURE_WEBAPP_PACKAGE_PATH: '.'

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: package.json nstall
        run: npm install

      - name: Build
        run: npm run build

      - name: Deploy
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
```

重要な設定はenvとjobsのDeployになる。
```diff yaml
env:
  AZURE_WEBAPP_NAME: 'web-app-name'
  AZURE_WEBAPP_PACKAGE_PATH: '.'
```
AZURE_WEBAPP_NAMEにはWeb App Serviceにデプロイしたアプリ名を指定する。
AZURE_WEBAPP_PACKAGE_PATHにはWeb App Serviceに登録するディレクトリを指定する。
ここでは、Nuxt3全体をデプロイするため'.'を指定する。

```diff yaml
      - name: Deploy
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
```
jobsのDeployではazure/webapps-deploy@v3を指定する。
このとき詳細をwithに記述する。
app-nameはenvに指定したAZURE_WEBAPP_NAMEを使用。
publish-profileはWeb App Serviceの発行プロファイル情報を使用する。シークレット情報のため、この後説明するGithub ActionsのSecretに保存する。
packageはenvに指定したAZURE_WEBAPP_PACKAGE_PATHを使用。

## Github Actionsに発行プロファイルをSecretとして登録
はじめにGithubから今回のプロジェクトのレポジトリを作成する。
作成後、Settings -> Secrets and variables -> Actionsに移動し、New Repository SecretsからSecretsを作成する。

発行プロファイルはWeb App Serviceの発行プロファイルのダウンロードから入手して貼り付ける。
![](/images/azure_deploy_nuxt3_web_app/スライド4.JPG)
![](/images/azure_deploy_nuxt3_web_app/スライド5.JPG)

## GithubにNuxt3をpush
設定が終わったらNuxt3をGithubのレポジトリにデプロイする。
```diff
git init
git add .
git commit
git branch -M main
git remote add origin <Repository URL>
git push -u origin main
```
Github Actionsが実行されるため、完了するまで待機。
![](/images/azure_deploy_nuxt3_web_app/スライド6.JPG)

完了したらWeb App ServiceのURLからNuxt3のページが表示されるか確認する。
![](/images/azure_deploy_nuxt3_web_app/スライド7.JPG)