+++
title = '制作物の紹介'
weight = 20
draft = false
+++
# 制作物の紹介

ここでは、私が制作した次の3つの成果物について紹介いたします。

- [週7英単語](#週7英単語)
- [Pdf2Ppt](#pdf2ppt)
- [Notion2QuizPdf](#notion2quizpdf-quizpdf)

## 週7英単語

![Pdf2Pptのプレビュー](/portfolio/img/preview_sh7e.png)

週7英単語は、毎日の英単語学習を目的としたウェブサイトです。

- ウェブサイト: [https://shu7-eitango.com](https://shu7-eitango.com)

- GitHubリポジトリ: [https://github.com/kyasuda516/shu7-eitango](https://github.com/kyasuda516/shu7-eitango)

- サーバ構築手順書: [週7英単語 サーバ構築手順書](/portfolio/docs/construction-manual/)

### 使用技術

使用している技術を表にまとめると次のとおりです。

| 分類 | 技術スタック |
| ----------------- | ------------------------------ |
| Frontend          | Bootstrap                      |
| Backend           | Nginx, uWSGI, Flask, Python    |
| Infrastructure    | Amazon EC2, Cloudflare         |
| Database          | MySQL, Redis, Grafana Loki     |
| Monitoring        | Grafana                        |
| Environment setup | Docker Compose                 |
| CI/CD             | GitHub Actions                 |
| Design            | PowerPoint                     |
| etc.              | Promtail, Certbot, geoipupdate |

これらの具体的な使用方法につきましては、[週7英単語 サーバ構築手順書](/portfolio/docs/construction-manual/)をご参照いただけると幸いです。

### 制作の背景

週7英単語は、ウェブ開発技術（特にインフラ関連の技術）の学習の成果物として紹介する目的で制作を始めました。

実はこれに先行し、高度な日本語語彙を身につけるための[週7日本語](https://javocabflushcards.com)というウェブサイトを開発していました。ただ、そのサイトは個人的な学習ニーズに特化しており、紹介には適していないと感じました。

そこで、よりたくさんの人々の需要にかなう英単語学習サイトとして、改めてこの「週7英単語」を立ち上げました。本サイトでは、目にすることが少ない英単語を毎日ピックアップしていて、英単語カードのように学習できるようになっています。これまで知らなかった英単語に出会い、新たな学びを得ることができる、発見型学習ウェブサイトとしてより多くのニーズに応えられるよう、本プロジェクトを開発しています。

### アピールポイント

次の点がアピールポイントです。

  - セキュリティ対策

      - 各コンテナ内ではrootではなく一般ユーザとして実行しています。

      - コンテナデータの永続化にはバインドマウントではなくボリュームを利用しています。

      - Grafanaにログインがあった場合はメールで通知するようにしています。

  - 権利への注意

    - 英単語データのソースについて、権利侵害がないように注意しています。
  
  - パフォーマンス考慮

    - コンテナイメージをコンパクトにしています。

    - ディスクがいっぱいにならないよう、cron、logrotateを用いてログローテーションしています。

  - 費用削減

      - フリーAPI、フリーデータベースを利用することで、インフラ部分以外の出費を抑えています。

<br>

## Pdf2Ppt

![Pdf2Pptのプレビュー](/portfolio/img/preview_pdf2ppt.png)

Pdf2PptはPDFをPowerPointプレゼンテーションに変換するプログラムです。

- GitHubリポジトリ: [https://github.com/kyasuda516/pdf2ppt](https://github.com/kyasuda516/pdf2ppt)

### 制作の背景

コロナ禍以降、大学の講義資料は紙ではなく電子ファイルとして配布されるのが普通になりました。  
大抵はPCでファイルを見ながら講義を受けることになりますが、時折りメモや穴埋めなどでファイル中への書き込みが必要になります。

しかし、PowerPointプレゼンテーションファイルならともかく、PDFファイルで配布された場合、柔軟に、直感的に書き込みを行うことが困難になります。  
そうした状況でふと友人が「PDFをPowerPointにできたら楽なのに」と嘆いているのを聞き、PDFをPowerPointファイルにするプログラムを作ることに思い至りました。

### アピールポイント

アピールしたいのは以下の2点です。

- PDFのビューを崩さずにPowerPointファイルに変換していること。

    - PDFの各ページを画像に変換し、その画像をスライド背景にするという処理をおこなっています。そのことでPDFのビューを保ったまま、さらにデバイス間で見た目が変わらないようになっています。

- コンピュータに不案内な友人でも簡単に導入できるようにしていること。

<br>

## Notion2QuizPdf, QuizPdf

![QuizPdfのプレビュー](/portfolio/img/preview_quizpdf.png)

Notion2QuizPdfは、Notion記事を穴埋め問題ふうのPDFにするためのプログラムです。

- GitHubリポジトリ: [https://github.com/kyasuda516/notion2quiz-pdf](https://github.com/kyasuda516/notion2quiz-pdf)

そしてQuizPdfは、このプログラムによって書き出したPDFの集合です。

- GitHubリポジトリ: [https://github.com/kyasuda516/quiz-pdf](https://github.com/kyasuda516/quiz-pdf)

### 制作の背景

私は技術知識の備忘録としてメモアプリNotionを使っています。

ある時、メモしたものを必要になるたびに参照するのではなく、まるっきり覚えてしまいたいと思いました。

そこで、記事の覚えたい部分を隠して、穴埋め問題のようにしたものを作ろうと考え、問題化して書き出すプログラムとしてNotion2QuizPdfを、書き出したものとしてQuizPdfを制作しました。

### アピールポイント

率直に言って、Notion2QuizPdfは一般的なニーズがあるものではありません。  
ただ、私の作ったPythonアプリケーションの中で最も複雑なものであり、自らのPythonスキルを示すために紹介させていただきます。

アピールすべき点としては、単なる手続き型のプログラムにはせず、プログラムを構成する主要な4ステップそれぞれでカプセル化することで、実行内容を理解しやすくしているところです。

それからQuizPdfにつきましては、私が何を学習してきたかを端的に示すのによい材料であると思い、併せてご紹介しています。

Pythonに関するページは特に充実しており、それだけ長くPythonを勉強していることを表しています。  
また最近は、Dockerをはじめとするインフラ技術に関するページにも追記を進めています。  
そのほか、Linux、MySQL、Rubyのページが比較的充実した内容になっています。