---
layout: page
title: 關於我
description:  我就是我，不一樣的煙火！
keywords: 關於, 聯繫
comments: false
sharebar: false
menu: 關於
permalink: /about/
---

<center>
    <p><img src="{{ site.url }}/Reion.png" align="center"></p>
    <p> 瘋瘋又癲癲，古古且怪怪。</p>
    <p> 身居桃花庵，安作桃花仙。</p>
</center>

## 悟道

> 昨夜西風凋碧樹。獨上高樓，望盡天涯路。<br>
> 衣帶漸寬終不悔，為伊消得人憔悴。<br>
> 眾裡尋他千百度，驀然回首，那人卻在燈火闌珊處。<br>
> 人生終歸經歷此三境，失意不變形，得意不忘形。<br>
> 無論如何被生活摧殘，依然相信只要自己認認真真、勤勤懇懇，<br>
> 這個世界就會為此變得美好一點點……<br>

## 聯繫

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

