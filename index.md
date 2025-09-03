---
title: Azure 開発者向けの演習
permalink: index.html
layout: home
---

## 概要

次の演習は、ソリューションを構築し、Microsoft Azure にデプロイするときに開発者が行う一般的なタスクについて詳しく知る、実践的な学習エクスペリエンスを提供するように設計されています。

> **注**: これらの演習を完了するには、必要な Azure リソースをプロビジョニングするのに十分なアクセス許可とクォータがある Azure サブスクリプションが必要です。 まだお持ちでない場合は、[Azure アカウント](https://azure.microsoft.com/free)にサインアップできます。 

一部の演習には、追加または特定の要件が含まれる場合があります。 これらには、その演習に固有の「**開始する前に**」セクションが含まれます。

## 対象分野
{% assign exercises = site.pages | where_exp:"page", "page.url contains '/instructions'" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %}

<ul>
{% for group in grouped_exercises %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in grouped_exercises %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }} 

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">トップに戻る</a> {% endfor %}

