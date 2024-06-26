---
title: "Nuxt3 + .NET Core + Azureで作るログイン機能サンプル：3. Nuxt3からログインする"
emoji: "🫠"
type: "tech"
topics: [dotnet, nuxt3, azure]
published: false
---
Azure環境でNuxt3をフロント、.NET Coreをバックエンドとしてログイン機能を作成する作業備忘録。
# モチベーション
## Nuxt3をViewとしたときに.NET Coreでログイン機能を実装したい
.NET CoreのMVCの場合はEntity Frame CoreとIdentity.EntityFrameworkCoreがあればログイン機能を簡単に実装することができる。
一方、Nuxt3などViewにフレームワークを使うとログイン機能連携に少し工数が増えるため、実際のデプロイ環境をAzure（App Service, SQL Server）を用いて構築する。

# ログイン機能の作成（.NET Core）
## .NET Coreプロジェクトの立ち上げ
今回はnpmを使ってNuxt3のプロジェクトを立ち上げるため、あらかじめLTSのNode.jsをインストールする。
Nuxt3のホームページからNuxtプロジェクトを立ち上げるコマンドを参照する。
```console
npx nuxi@latest init <project-name>
```
https://nuxt.com/docs/getting-started/installation
