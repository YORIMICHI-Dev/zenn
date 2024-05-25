---
title: "Nuxt3(node module)でよく使うライブラリ"
emoji: "👾"
type: "tech"
topics: [nuxt3]
published: true
---
Nuxt3でよく使っているライブラリのメモ。
つどつど追加
# 状態管理
## Pinia
Vue・Nuxt3を使う場合はほぼ必須
Nuxt3からインストールする場合
https://pinia.vuejs.org/ssr/nuxt.html

Piniaのセッション管理を行う場合
https://nuxt.com/modules/pinia-plugin-persistedstate

# UI・コンポーネント作成
## Vuetify
Vue・Nuxt3でUIを作成するライブラリ1
https://vuetifyjs.com/en/

## Quasar
Vue・Nuxt3でUIを作成するライブラリ2
Vuetifyと比較してコンポーネントが充実している印象
https://quasar.dev/

## Perfect Scroll bar
スクロールバーのスタイル変更
個人的に少し使いづらい
https://perfectscrollbar.com/

# Use Hook系
## VueUse
Use Hook系のライブラリ群
Event系を実装する場合に重宝する
https://vueuse.org/

# グラフ作成
## ApexCharts
グラフを作成する場合に使用する
Vue3用のvu3-apexchartsを使用
https://apexcharts.com/
https://apexcharts.com/docs/vue-charts/

# Validation
## yup
入力フォームのバリデーション
https://github.com/jquense/yup?tab=readme-ov-file

## VeeValidate
yupのバリデーションの状態管理に使う
https://vee-validate.logaretm.com/v3/

# データ変換
## Lodash
データ変換のライブラリ群
https://lodash.com/

## date-fns
Dateを臨機応変に取り扱う
https://date-fns.org/

# 認証・認可
## @azure/msal-browser
Azure Entra IDによる認証
https://www.npmjs.com/package/@azure/msal-browser
https://github.com/AzureAD/microsoft-authentication-library-for-js

# マップ表示
## mapbox
マップの表示・操作が容易にできる
緯度・経度上に3Dモデルを表示できる
https://docs.mapbox.com/mapbox-gl-js/guides
https://www.npmjs.com/package/mapbox-gl