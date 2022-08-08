# ビリアル応力の説明
## ビリアルの定義と時間微分
まず天下りですが、以下のような物理量 $G$ を定義します。

$$
G = \sum_i \boldsymbol{p}_i \cdot \boldsymbol{q}_i
$$

ここで $\boldsymbol{p}_i$ は $i$ 番目の粒子の運動量、 $\boldsymbol{q}_i$ は $i$ 番目の粒子の位置座標を指します。ここで両辺を時間微分してみます。

$$
\frac{\partial G}{\partial t} = \sum_i \frac{\partial \boldsymbol{p}_i}{\partial t} \cdot \boldsymbol{q}_i + \sum_i \boldsymbol{p}_i \cdot \frac{\partial \boldsymbol{q}_i}{\partial t} \tag{1}
$$

この式の右辺について詳しく見ていきます。

## $\frac{\partial \boldsymbol{p}_i}{\partial t}$ について

まず右辺第1項はニュートンの第2法則より

$$
\sum_i \frac{\partial \boldsymbol{p}_i}{\partial t} = \sum_i m_i \frac{\partial \boldsymbol{v}_i}{\partial t} = \sum_i \boldsymbol{f}_i \tag{2}
$$

となります。ここで $m$ は粒子の質量、 $\boldsymbol{f}_i$ は粒子 $i$ に作用する力です。
次にこの $\boldsymbol{f}_i$ を内力と外力に分けていきます。
また外力を $\boldsymbol{f}_i^\mathrm{ext}$ とします。すると $\boldsymbol{f}_i$ は次のように分解できます。

$$
\boldsymbol{f}_i = \sum_{i \neq j} \boldsymbol{f}_{ij} + \boldsymbol{f}_i^\mathrm{ext} \tag{3}
$$

よって式 $\mathrm{(3)}$ はつぎのようになる。