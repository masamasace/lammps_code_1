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

## $\frac{\partial \bm{p}_i}{\partial t}$について

まず右辺第1項はニュートンの第2法則より
$$
\sum_i \frac{\partial \bm{p}_i}{\partial t} = \sum_i m_i \ 
    \frac{\partial \bm{v}_i}{\partial t} = \sum_i \bm{f}_i \tag{2}
$$
となります。ここで$m$は粒子の質量、$\bm{f}_i$は粒子$i$に作用する力です。次にこの$\bm{f}_i$を内力と外力に分けていきます。ここで内力(粒子$j$から粒子$i$に作用する力)を$\bm{f}_{ij}$とします。また外力を$\bm{f}_i^\mathrm{ext}$とします。すると$\bm{f}_i$は次のように分解できます。

$$
\bm{f}_i = \sum_{i \neq j} \bm{f}_{ij} + \bm{f}_i^\mathrm{ext} \tag{3}
$$

よって$(\refeq{2})$