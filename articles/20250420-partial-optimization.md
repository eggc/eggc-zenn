---
title: "部分最適化と全体最適化、それからマイクロサービスアーキテクチャ"
emoji: "❇️️"
type: "tech"
topics:
  - "アーキテクチャ"
published: true
published_at: "2025-04-20 11:00"
hugo_date: "2025-04-20T11:00:00+09:00"
---

## はじめに

チームビルディングの文脈で「部分最適化」「全体最適化」という言葉を耳にする機会が増えてきたように思います。そこにマイクロサービスアーキテクチャの話を絡めて、最近私が思っていることをまとめてみました。

## 部分最適化と全体最適化

一般に、一部分に集中しすぎると、全体のバランスが崩れてうまくいかないことが多いと言われています。たとえば囲碁や将棋の初心者は、石や駒がぶつかっている局所戦ばかりに気を取られ、他のエリアを相手に支配されてしまいます。その結果、部分的には勝っていても試合には負けてしまうことになります。だからこそ、大局観と呼ばれるように、盤面全体を見て手を打つことが望ましいとされています。

高校美術のデッサンの授業でも、同様のことが言われています。いきなり細部を描き込もうとすると、バランスが取れず失敗します。少し引いた目線で全体のシルエットを描き、その後で細部を埋めていくのがよいそうです。

この考え方は、プログラミングにもそのまま当てはまります。あるアプリケーションに対して新しい部品を作るとき、初心者はその部品の仕事だけを考えてコードを書いてしまいがちです。自分にとって書きやすく、使いやすいコードを優先してしまいます。しかし、そうして書かれたコードは、他の部品と調和せず、他の人にとっては使いづらいものになってしまいます。

既存のコードを読みながらバランスを取ることが重要です。再利用できる部品は活用し、新しい部品を作る際には、既存の命名規則や設計方針に合わせることで、わかりやすさが向上します。また、部品同士をどのようにつなぎ合わせるかも重要な要素です。

ところが、アプリケーションに含まれるコードの量が一定以上になると、全体最適は非常に困難になります。たとえば、1万ファイルを超えるような大規模なアプリケーションがあるとします。このような場合、新しい部品を作る前にすべてのファイルを確認するのは現実的ではありません。そのため、ディレクトリ構成を見たり、特によく使われる部品や中心的な機能を支える部品だけを確認したりします。必要に応じて、過去のコードを整えたり、一貫性を与えるためのルールを導入したりすることもあります。

しかし、アプリケーションがさらに大規模化し、プログラマの数やファイルの数が増えていくと、主要な部品でさえ把握が難しくなってきます。大局観を持ったプログラマがいたとしても、その人にかかる負担は大きくなり、その人が退職してしまえば、誰も全体を把握できなくなってしまいます。そうなると、もはや全体最適は不可能になります。この課題に対して、ソフトウェア業界ではマイクロサービスアーキテクチャという考え方で突破口を見出そうとしています。

## マイクロサービスアーキテクチャ

マイクロサービスアーキテクチャでは、アプリケーション全体を小さなサービス単位に分割します。そして、サービスごとに担当者を決め、お互いのサービスのコードはなるべく意識しないようにします。一つのサービスに含まれるコードは限られているため、従来の手法でも最適化が可能です。このようにして、各サービス単位での部分最適化を行い、最後にそれらを組み合わせる段階で担当者同士が話し合い、全体最適化を図ります。

とはいえ、マイクロサービスアーキテクチャの実践は容易ではありません。大規模なアプリケーションを複数のサービスに分解する際には、どのコードをどのサービスに含めるかを慎重に考える必要があります。この設計に失敗すると、サービス間に深い依存関係が生まれ、本来は独立して動くはずのサービスが、お互いを深く理解しなければ成り立たなくなってしまいます。

もう一つの課題は、コストの増大です。サービスの数だけ担当者が必要となるため、人員の確保がこれまで以上に重要になります。また、お互いのサービスを意識しないことで、コードの再利用が難しくなる場合もあります。さらに、部分最適化と全体最適化の間を行き来する必要があり、ほとんどの場合、従来のアーキテクチャに比べて開発に時間がかかります。サービスごとのデプロイやモニタリングの仕組みが必要となり、運用コストも増大するのが普通です。

## 組織とアーキテクチャの類似性

ところで、このようなプログラマの集まりを俯瞰してみると、資本主義と社会主義の運営と対比できる側面があるように思います。全体最適化を目指す場合、リードプログラマが中央集権的に全体の情報を集約し、必要な部品の設計を計画し、下流のプログラマに指示を出します。下流のプログラマは提案や創意工夫のインセンティブが乏しく、上からの指示に従ってコードを書くことになります。リードプログラマが優秀であり続ける限り整然と効率的に開発が進む一方で、誤った判断をすると、アプリケーション全体が混乱に陥るリスクがあります。これは社会主義的ですね。

一方、部分最適化では、全体を気にしないことで、自由に各プログラマが自らのアイデア実践できます。流行の技術を取り入れたり、好みの設計手法をとることもできるでしょう。しかし全体で見れば無駄が生まれやすく、統一感にも欠けます。最終的には、それぞれの関係者の思惑や工夫によって、危ういながらもバランスが取れていきます。この自由さ、発展の速さ、無秩序さが資本主義的なように思います。

しばしば、チームごとの独立した部分最適化はサイロ化として批判されます。確かに、チームが自分たちの範囲に閉じこもり、情報共有や助け合いが行われない状態では、全体の効率は下がってしまうでしょう。マイクロサービスアーキテクチャにも、そのような側面があることは否定できません。

この段階では一時的な非効率を受け入れつつも、各チームの専門性を高めることになります。しかし、アプリケーション完成までには、いずれマイクロサービスの統合フェーズ、すなわち全体最適化のフェーズがやってきます。ここでは、関係者同士が情報交換や協力が絶対に必要です。開発の初期段階から定期的にコミュニケーションの場を設けておくことで、最終的なサイロ化の問題を大きく緩和できるでしょう。

## おわりに

この記事では、プログラミングにおける部分最適化と全体最適化というテーマに焦点を当てましたが、マイクロサービスアーキテクチャに関する議論は、これだけでは十分に掘り下げられていない部分が多くあります。特に、アーキテクチャの設計においてはインフラの構成やサービス間の依存関係といった要素まで含めて考慮する必要があることを念頭に置くべきです。さらに、今回は主にソフトウェア開発者の観点で進めましたが、実際には運用やデプロイメントの観点も考慮した議論が不可欠です。

また、全体最適化が目指すコードの品質がどれだけのビジネス上の利益を産むのか、といった観点も重要です。この観点を考慮すると、状況によっては「適度な部分最適化」が実用的な解決策となる場合もあるでしょう。
