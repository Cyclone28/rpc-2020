---
title: "人工知能は下準備が肝心"
date: 2020-09-13T07:28:04+09:00
description: "Nonnoプロジェクト"
draft: false
weight: 20
---

<div align="right">80th NIIInonno; 新生华</div>

## 去年

かねてからAI(人工知能)を作ってみたいと思っていたので、2019年の春ぐらいに完全自作でAI作った。

__が、全く精度が揮わない。__

原因は、データ不足である。

いくらネットが普及して、様々なデータが収集できる時代になっても、自身の能力が及ばなかったせいで、画像32万枚しかデータ収集できなかった。

しかも、一枚一枚の"質"が悪いので、AIは上手く学習してくれないのである。

## というわけで

Youtubeとかから連番の静止画を入手できる。
旅先で撮った映像もその場で活用できる。
自分のPCがスペック不足なら他のPCに作業を委託できる。
遠い将来にも利用できるレベルのデータを集めたい。
ノートPCでAI動かしたい。
電子工作にAI組み込んでみたい。などと夢が膨らんで、

### まとめると、

* いろんな種類のデータを、
* いろんなサイトやコミュニティから、
* いつでも、
* (ハードウェア的に)どこでも、
* (物理的に)どこでも、
* 高速、
* 安全、
* かつ快適に収集できる。

そんなソフトが欲しくなったのだ。

## 汎用的すぎるライブラリ

上記の条件を満たすには言語は何で、どのように、なにで組めばいいのか。

迷いに迷った末、以下のことが決まった

### 言語:C\#

C言語、C++は環境依存過ぎて、条件「どこでも」を満たさない。

Pythonはライブラリが充実していていいけれども、「高速」ではないし、「いろんな種類のデータ」に含まれる、カスタマイズ性の高さという点では劣る。

一方C#は、

* 最近マイクロソフトが本格的に機械学習とかサポートを充実させていっている。
* 技術的に安定。
* Python、Rubyと互換性的なものがある。
* 速度はそこそこ出る。
* 近いうち(2020/11)にクロスプラットフォーム化がより進む。
* 簡単にUnityでゲーム化できる。

などなど、結構いいことばかりだった。

Javaはどっちでもよかった。

### 基本構造:`Sample`構造体と`IData<T>`インターフェース

```C#
public struct Sample
{
    private float _value;
    private float _strength;
    public Sample(float value, float strength)
    {
        _value = value;
        _strength = strength;
    }
    public static Sample operator +(Sample a, Sample b) => 
        new Sample(
            (a._value * a._strength + b._value * b._strength) / a._strength + b._strength, 
            a._strength + b._strength);
}

public interface IData<T>
{
    public Sample this[T pos] { get; }
}
```

これを基本構造とする。

これだけ。

## 実用性やいかに

先に掲載したコードの`IData`インターフェースを実装するものは、すべて扱える。

「このインターフェースで一体何がわかるんだ」と言いたくもなるだろうが、実は、画像、動画、音声、立体、値など、さまざまなものに対して使える。

例えば、拡張メソッド`Bitmap ToBitmap(this IData<(float, float, float, float)> @this)`を用いて、`IData<(float, float, float, float)>`型のデータは __すぐに画像に変換__ できる。

拡張メソッド`Bitmap Project(this IData<(float, float, float, float, float, float, float)> @this, Expression<Func<(float, float, float, float, float, float, float), (int, int)>> projecter)`を用いて、`IData<(float, float, float, float, float, float, float)>`型で表せる __3Dデータも、すぐに画像に変換__ できる。

詳しくは調整中のため割愛するが、要はいろいろできる。

まだ全然分からないと思うが、

### 動画⇔画像⇔3D⇔CAD⇔センサ⇔オシロ画像

などとできる予定である。

将来的には物理部で扱うデータ全てに対応させられないだろうか。

## 乞うご期待

じつは、まだいろいろ未完成で、今回の文化祭で公表できる状態にはない。

自分でも、どんなものになるか想像がつかないが、現状3D⇔点変換など着々進んでいるので、悪い結果にはならないと思う。

残念だが、この記事はここまでだ。こんな中途半端な記事を読んでくれてどうもありがとう。

来年、と言わずとも冬休みまでには、この記事は充実して、読みやすくもなっていると思う。その折にまた読んでいただけると嬉しい。
