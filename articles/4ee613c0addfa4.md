---
title: "Nuxt3を静的ファイルとしてビルドし、.NET Coreに表示する"
emoji: "😀"
type: "tech"
topics: [.net, nuxt3]
published: true
---
Nuxt3をViewとして.NET Coreにデプロイした作業備忘録。
# モチベーション
## Nuxt3をMVCモデルのViewとして使いたい
.NET CoreやdjangoのMVCフレームワークを用いるとフロント側はHTML, CSS, JavaScriptで書くことになる。
しかし、個人的にはライブラリが揃っておりディレクトリ構造がある程度統一されているフレームワークを使ったほうが実装効率が高い。
特に、ルーティング機能やコンポーネント分割、プラグイン機能、TypeScriptなど簡単に扱えるNuxt3・Vueが使い勝手が良かった。そのため、Nuxt3をフロント、.NET CoreやFast APIをバックエンドとして主に開発をしている。

## Azure（クラウド）リソースの節約
Nuxt3のデプロイとして、buildによるAzure Web AppなどのNodeサーバーへのデプロイか、generateによるAzure Static Web Appsなどの静的コンテンツホスティングによるデプロイがある。
現在、Nuxt3をフロント、.NET Coreを認証サーバーなどのバックエンドとして使用しており、buildだとNodeサーバーと.NET Coreサーバーの計2つのサーバーが必要になる。
一方、静的ファイルにgenerateした場合は、.NET Coreが静的ファイルサーバーとして兼任できるためサーバーが一つで済む。これにより少なからずリソースが節約でき、運用・保守費が抑えられるため実践した。

# Nuxt3の静的ファイルのビルド（generate）
## Nuxt3プロジェクトの立ち上げ
今回はnpmを使ってNuxt3のプロジェクトを立ち上げるため、あらかじめLTSのNode.jsをインストールする。
Nuxt3のホームページからNuxtプロジェクトを立ち上げるコマンドを参照する。
```console
npx nuxi@latest init <project-name>
```
https://nuxt.com/docs/getting-started/installation

Nuxt3が開発モードで立ち上がるか確認する。
```console
npm run dev
```
## 静的ファイルの作成
Nuxt3は通常SSR（サーバーサイドレンダリング）であるが、今回は静的ファイルとしてビルドするためSPA（シングルページアプリケーション）として運用する。
nuxt.config.tsの設定をSPAに変更する。
```diff js:nuxt.config.ts
export default defineNuxtConfig({
+  ssr: false,
  devtools: { enabled: true }
})
```
設定後、generateより静的ファイルを生成する。.outputディレクトリが作成されその中に静的ファイルが生成される。これらをこの後.NET Coreにコピーする。
```console
npm run generate
```

# .NET Coreのプロジェクト立ち上げ
## バックエンドWeb APIの作成
続いて、.NET Coreでバックエンドのプロジェクトを立ち上げる。あらかじめ.NET Coreをインストールした状態でVisual StudioからWeb APIを選択するか、以下のコマンドから作成する。
```console
dotnet new webapi -n <project-name>
```

同様に開発モードで立ち上がるか確認する。コマンドラインでは以下のコマンドからホットリロードとして立ち上げる。
```console
dotnet watch run
```
## Projecct.csの設定
静的ファイルを読み込む設定を行う。
今回はwwwrootディレクトリを作成し、その中にビルドした静的ファイルを格納するため、デフォルトの設定をProject.csに設定する。
追加として、index.htmlを認識するUseDefaultFilesと静的ファイルサーバーとするUseStaticFilesを追加する。

```diff cs:Projecct.cs
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseHttpsRedirection();
}

+ app.UseDefaultFiles();
+ app.UseStaticFiles();

app.Run();
```

# .NET Coreへ静的ファイルをコピー
## wwwrootディレクトリ作成・静的ファイルコピー
Nuxt3で生成したファイルをwwwrootにコピーする。
生成した内容は.output/public内のすべてのファイルであり、以下のようなファイルがコピーされる。
```diff
.NET Coreプロジェクト
+  |--wwwroot
+  |     |--_nuxt
+  |     |--200.html
+  |     |--404.html
+  |     |--favicon.ico
+  |     |--index.html
+  |     |--その他生成ファイル群
   |--bin
   |--obj
   |--Properties
   |--Program.cs
```

## 動作確認
### 開発時
開発モードでNuxt3のページが立ち上がるか確認する。コマンドラインでは以下のコマンドからホットリロードとして立ち上げる。
```console
dotnet watch run
```
### デプロイ時
デプロイする場合はさきにpublishコマンドよりプロジェクトをビルドする。
```console
dotnet publish
```
その後、Releaseディレクトリに作成されたexeファイルを起動する。
```console
.\bin\Release\net8.0\publish\test2.exe
```
