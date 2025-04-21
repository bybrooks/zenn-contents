---
title: "Flutter WebアプリをVercelにデプロイ（環境変数込）"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Flutter, Vercel, Tech]
published: true
---

# はじめに

Flutter で作成したアプリを Web にデプロイする方法を紹介する。
特に、環境変数を利用したアプリのデプロイ方法はあまり情報がなかったため本記事にて共有する。

# デプロイ手順

基本的な手順としては、public で公開した GitHub Repository を Vercel に Import することでデプロイする。
Vercel での Project 作成と Import の仕方は以下の記事を参照いただきたい。

https://zenn.dev/lnida/articles/6178f3f09b9ced

---

以降では、Flutter Web にて Secrets な環境変数を利用したい場合について説明する。

### 1. vercel.json ファイルの作成

Vercel のコンソールから環境変数を設定できるが、その環境変数をコード上で読み込むには、vercel.json を利用してデプロイする必要がある。
(他の方法を知っている方いらっしゃいましたら教えてください。)

デプロイする Repository に vercel.json を作成し、以下のように明記する。
本サンプルでは、SUPABASE_URL と SUPABASE_ANON_KEY という Secrets な環境変数を利用している。

```
{
    "env": {
        "SUPABASE_URL": "${SUPABASE_URL}",
        "SUPABASE_ANON_KEY": "${SUPABASE_ANON_KEY}"
    },
    "buildCommand": "flutter/bin/flutter build web --release --dart-define=SUPABASE_URL=${SUPABASE_URL} --dart-define=SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}",
    "outputDirectory": "build/web",
    "installCommand": "if cd flutter; then git pull && cd .. ; else git clone https://github.com/flutter/flutter.git; fi && ls && flutter/bin/flutter doctor && flutter/bin/flutter clean && flutter/bin/flutter config --enable-web"
}

```

### 2. Repository の Import

Import ボタンを押下すると、Build 時の設定画面に移る。
Environment Variables にて vercel.json で明記した環境変数を登録する。
本サンプルでは、SUPABASE_URL と SUPABASE_ANON_KEY の 2 つを設定する。
![](/images/flutter-web-deploy-with-env/deploy.png)

設定後、Depoly ボタンを押下すると環境変数を読み込んだ Flutter Web アプリがデプロイされます。

# 終わりに

本記事では、Flutter Web アプリを Vercel にデプロイする際に、環境変数を利用する方法を紹介しました。
vercel.json を利用して環境変数を定義し、Vercel のコンソールで環境変数を設定することで、Secrets な情報を安全に管理しながら Flutter Web アプリをデプロイできます。

最後まで読んでいただきありがとうございます。；）
