# 作業メモ
ここからは勉強過程で残した作業用メモです。参考になるかもしれないし、ならないかもしれません。
##  必要なモジュールを含めてLAMMPSをビルドする
ここからはWindows Terminalの話になります。まず開いた直後の状態から上のタブの下矢印(v)をクリックしてください。そしてUbuntu 22.04 LTSをクリックしてください。
![初期画面](./img/2022-06-26_16h29_53.png) 
そうするとUbuntuのターミナル画面が表示されると思います。(PowerShellの方は使わないので×で消してもらって大丈夫です。)
![Ubuntuの画面](./img/2022-06-26_16h24_15.png) 
今回のPCのUbuntuはCUI(キャラクタユーザインターフェース)なので、全部キーボードの入力で操作をしていきます。具体的には&#36;マークの続きにコマンド(指令)を入力し、その結果を見つつさらに新しいコマンドを打つ形になります。まずこの画面で`ls`と打ちエンターを押してみてください。ちなみに`ls`はlist segmentsの略です(&#36;と`ls`の間にあるスペースは自動で入っているものなので気にしないでください)。
そうすると次の文字列が出力されたと思います。

```bash
get-pip.py  lammps-23Jun2022  lammps-stable.tar.gz
```

![lsの結果](./img/2022-06-26_16h40_16.png) 
このlsというコマンドは、今いるフォルダの中にあるファイルとフォルダをすべて表示する
というコマンドです。今回の場合get-pip.py、lammps-23Jun2022、lammps-stable.tar.gzの3つのファイルorフォルダが存在するため、このように表示されています。ちなみに文字色には意味があり、以下の通りです。

- 青：ディレクトリ
- 緑：実行可能または認識されたデータファイル
- スカイブルー：シンボリックリンクファイル
- 背景が黒の黄色：デバイス
- ピンク：グラフィック画像ファイル
- 赤：アーカイブファイル
- 背景が黒の赤：リンク切れ

次に出てきた入力欄に`cd ./lammps-23Jun2022/`と入力してみてください。ちなみに`cd ./l`まで打った後にtabキーを押すと、残りの文字列が自動で入力されます。(この機能は該当する候補が一つのみの時にしかできません)

これをすると&#36;の前の部分に`/lammps-23Jun2022`が加わったと思います。`cd`コマンドはchange directoryの略で、ディレクトリ(フォルダ)を移動できるコマンドです。つまり`cd ./lammps-23Jun2022/`コマンドによって、一つ下の階層のフォルダに移ることができるのです。

また一つ上の階層に戻りたいときは`cd ../`と打ちエンターキーを押すとできます。さらに続けて次の階層に降りたいときは、`cd ./lammps-23Jun2022/`と入力し **エンターキーを押さずに**そのままtabキーを押すと`lammps-23Jun2022`というフォルダの中にあるフォルダを一覧で表示できます。

さてここまで練習した機能を使って、`~/lammps-23Jun2022/build`のフォルダまで移動してください。するとこのような画面になると思います。

![buildまで移動した結果](./img/2022-06-26_17h10_00.png) 

ここからLAMMPSのビルドという作業に移ります。ビルドは簡単に言うとプログラミング言語で
書かれたソースコードをもとにして、実行可能ファイルを作成することです。Windowsだとファイル名の一番後ろの拡張子が.exeとなったファイルをダウンロードすることがあるかと思います。ビルドとは人間が読める状態のソースコードから、exeファイルを作成する作業だと考えてください。

今回はcmakeというC言語あるいはC++プロジェクト向けのビルドツールを使います。ただそんなに難しい作業ではありません。コマンドラインに次の文字列を入力してください。

```bash
cmake -D PKG_GPU=yes -D GPU_API=cuda../cmake
cmake -D PKG_GRANULAR=yes ../cmake
cmake -D BUILD_MPI=yes ../cmake
```

`cmake`はこれからcmakeというビルド作業をするよ、という合図です。もう少し正確に言うと、`../cmake`にあるcmake用のスクリプトを読んできて、ビルドの構成を確認するというコマンドになります。

また`-D`オプションは、cmake用のスクリプトにおいて変数を変更(上書き)するためのコマンドです。そして`PKG_GPU=yes`の部分で、通常時には無効化されたGraphical User Interface(GPU)に関するパッケージを有効化してくださいという指令を書いています。同様に`PKG_GRANULAR=yes`では粒状体用のパッケージを有効化するという意味になります。`BUILD_MPI=yes`は並列処理によってビルドを行っていくださいという指令です。

## GPUあるいはマルチスレッドの検証
GPGPUが本当にどれぐらい効くのか、またマルチスレッドはどのぐらい効くのか、それを検証してみます。

### 実行可能ファイルのファイル名の変更方法
- 結論：よくわからなかった。cmakeのコマンドをいじれば実行可能ファイル名を変えられるはずなんだけど、CMakeLists.txtの中にもそれらしきものは見つからず...
- LAMMPS_BINARYやOUTPUT_NAMEに実行可能ファイル名が格納されているかもしれない。
- [ここ](https://docs.lammps.org/Build_basics.html#build-the-lammps-executable-and-library)に書いてあった。`-D LAMMPS_MACHINE=name`のビルドオプションを付けるとsuffixがつくらしい。(lmp_nameみたいに)
  
### KOKKOSインストール手順
基本は[ここ](https://docs.lammps.org/Build_extras.html#kokkos)に書いてあるので参照するべし。
- 利用可能なアーキテクチャ一覧がある([こちら](https://docs.lammps.org/Build_extras.html#available-architecture-settings)から参照)
  - 今回の場合はRyzen 5950XとNVIDIA GeForce GTX 1650 or RTX 3090なのでArch-IDの中でも`ZEN3`と`TURING75`か`AMPERE86`が利用可能
    - `CC`はCompute Capabilityの略で[ここ](https://developer.nvidia.com/cuda-gpus)から参照可能
      - `AMPERE80`は過去の遺物?
  - `cmake`のビルド時に以下のコードの入力する必要がある
    - 連続して入力する場合には`cmake -D PKG_KOKKOS=yes -D Kokkos_ARCH_ZEN3=yes ../cmake`のようにする
    - 共通
      ```bash
      -D PKG_KOKKOS=yes
      ```
    - Zen3 CPU向け
      ```bash
      -D Kokkos_ARCH_HOSTARCH=yes  # HOSTARCHはここではZEN3
      -D Kokkos_ENABLE_OPENMP=yes
      -D BUILD_OMP=yes
      ```
    - NVIDIA CPU向け
      ```bash
      -D Kokkos_ARCH_GPUARCH=yes    # GPUARCHはここではTURING75かAMPERE86
      -D Kokkos_ENABLE_CUDA=yes
      -D CMAKE_CXX_COMPILER=${HOME}/lammps/lib/kokkos/bin/nvcc_wrapper
      ```
   
 - コンパイルする
   ```bash
   make -j 4
   ```
   - なんかいろいろ警告が出るけど大きく分けてこの2つ
     - `nvcc warning : The 'compute_35', ~~~`: nvccのコンパイラに`-Xcompiler "/wd 4819"`を渡せば解決するらしいが`cmake`からどのように設定するのか不明
       - 多分CMakeLists.txtを書き換えればいいのかな...
     - `warning #1675-D: unrecognized GCC pragma`: これもnvccのコンパイラに`-Xcudafe "--diag_suppress=unrecognized_gcc_pragma"`を通せばいいみたい

### `mpirun -np 16`とGPUを変更した際の実行結果

結果の前に実行した際の疑問点を以下に...
- OpenMPのスレッドはどのようにして増やす？？
- GPUが使われているようには思えないんだけど...
  - `mpirun -np 16 $LAMMPS_BUILD_DIR/lmp -sf gpu -pk gpu 1 -in in.pour.flatwall`の`-sf gpu -pk gpu 1`が肝心
  - ただメモリ使用量が増えるだけでCUDAは使われないようだ...
    - granularモジュールはCUDAを使えない??
    - 正解。各`pair_style`のコマンドの中に使用できるAcceleratorが書かれている(たとえば[granのページ](https://docs.lammps.org/pair_gran.html))
      - `granular`コマンドはいずれのAcceleratorも使えない
      - `gran/hooke`と`pair_style gran/hertz/history`コマンドは`omp`(OpenMP)のみ
      - `gran/hooke/history`コマンドは`omp`と`kk`(KOKKOS)のみ
      - でもそれぞれのコマンドの解説ページには以下のように書かれてる...
        - >Styles with a gpu, intel, kk, omp, or opt suffix are functionally the same as the corresponding style without the suffix.
    - じゃあKOKKOSってなによ...
      - どういう仕組みではやくなっているのかはよくわからない
      - ただインストール方法は[ここ](https://docs.lammps.org/Build_extras.html#kokkos)に書いてある
      - 長くなりそうなので[上](#kokkosインストール手順KOKKOSインストール手順)に記載
    - じゃあ`gran/hooke/history`とそれ以外のコマンドって何が違うのよ...
      - `history`はせん断応力が履歴をもたらすという意味らしい
        - $F_{hk}=\left(k_n\delta\boldsymbol{n}_{ij}-m_{eff}\gamma_n\boldsymbol{v}_n\right) - \left(k_t\Delta\boldsymbol{s}_t+m_{eff}\gamma_t\boldsymbol{v}_t\right)$のうちの$k_t\Delta\boldsymbol{s}_t$の部分
          - ここで$\Delta\boldsymbol{s}_t$は2粒子間の接線変位ベクトル(降伏関数を満たすように切り捨てたもの←ここ重要！)
          - 粘性によるエネルギー損失だけでなく塑性変形によるエネルギー損失が考慮できる！
            - 参考文献は[Brilliantov et al., 1996](https://journals.aps.org/pre/abstract/10.1103/PhysRevE.53.5382)か[瀬戸内, 2015](https://www.jstage.jst.go.jp/article/jsidre/83/4/83_I_133/_pdf/-char/ja)とか
          - しかし計算コストのかかりそうな`gran/hooke/history`のみがKOKKOSに対応しているのはなぜだ...
      - そもそも`granular`と`gran`の違いって何よ...
        - (まだ調べてない...)
- `mpirun -np 32`と設定するとエラーが出るんだけどなんで？？
  - `mpirun -np 16`が限界だったから物理コアに依存？
  - [ここ](https://www.open-mpi.org/doc/v4.0/man1/mpirun.1.php)を見ると`processing element`は何もオプションを指定しない場合はプロセッサーのコアになるらしい。
    - >By default, Open MPI defines that a "processing element" is a processor core. However, if --use-hwthread-cpus is specified on the mpirun command line, then a "processing element" is a hardware thread.

結果のまとめは以下の通り
- CPUのコア数は物理コア数まではほぼ律速
- `comm`にすごく時間がかかる場合がある
  - RyzenだからMCMの関係かも?(←自信はありません...)
- GPUは`PKG_GPU=on`にしてもメモリを使用するだけで全然速くならない
1. `100.0% CPU use with 1 MPI tasks x 1 OpenMP threads`

```bash
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 4.4758     | 4.4758     | 4.4758     |   0.0 | 73.16
Neigh   | 0.8622     | 0.8622     | 0.8622     |   0.0 | 14.09
Comm    | 0.032106   | 0.032106   | 0.032106   |   0.0 |  0.52
Output  | 0.0011555  | 0.0011555  | 0.0011555  |   0.0 |  0.02
Modify  | 0.7174     | 0.7174     | 0.7174     |   0.0 | 11.73
Other   |            | 0.02894    |            |       |  0.47

Nlocal:           3000 ave        3000 max        3000 min
Histogram: 1 0 0 0 0 0 0 0 0 0
Nghost:            458 ave         458 max         458 min
Histogram: 1 0 0 0 0 0 0 0 0 0
Neighs:          16800 ave       16800 max       16800 min
Histogram: 1 0 0 0 0 0 0 0 0 0

Total # of neighbors = 16800
Ave neighs/atom = 5.6
Neighbor list builds = 1168
Dangerous builds = 0
Total wall time: 0:00:06
```

2. `100.0% CPU use with 2 MPI tasks x 1 OpenMP threads`

```bash
MPI task timing breakdown:
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 1.0778     | 2.2797     | 3.4817     |  79.6 | 48.86
Neigh   | 0.33048    | 0.43536    | 0.54025    |  15.9 |  9.33
Comm    | 0.074782   | 1.4048     | 2.7348     | 112.2 | 30.11
Output  | 0.00052933 | 0.00078022 | 0.0010311  |   0.0 |  0.02
Modify  | 0.47923    | 0.51266    | 0.5461     |   4.7 | 10.99
Other   |            | 0.03222    |            |       |  0.69

Nlocal:           1500 ave        2037 max         963 min
Histogram: 1 0 0 0 0 0 0 0 0 1
Nghost:          461.5 ave         481 max         442 min
Histogram: 1 0 0 0 0 0 0 0 0 1
Neighs:           8459 ave       12697 max        4221 min
Histogram: 1 0 0 0 0 0 0 0 0 1

Total # of neighbors = 16918
Ave neighs/atom = 5.6393333
Neighbor list builds = 1170
Dangerous builds = 0
Total wall time: 0:00:04
```

3. `98.9% CPU use with 4 MPI tasks x 1 OpenMP threads`

```bash
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 0.5379     | 1.1592     | 1.8082     |  56.4 | 44.26
Neigh   | 0.1653     | 0.21779    | 0.27634    |  10.9 |  8.32
Comm    | 0.11505    | 0.83398    | 1.5122     |  73.0 | 31.84
Output  | 0.00033634 | 0.00054757 | 0.0010141  |   0.0 |  0.02
Modify  | 0.36899    | 0.386      | 0.40115    |   2.2 | 14.74
Other   |            | 0.02164    |            |       |  0.83

Nlocal:            750 ave        1031 max         474 min
Histogram: 2 0 0 0 0 0 0 0 0 2
Nghost:         421.75 ave         467 max         378 min
Histogram: 2 0 0 0 0 0 0 0 0 2
Neighs:        4213.25 ave        6457 max        2040 min
Histogram: 2 0 0 0 0 0 0 0 0 2

Total # of neighbors = 16853
Ave neighs/atom = 5.6176667
Neighbor list builds = 1163
Dangerous builds = 0
Total wall time: 0:00:02
```

4. `98.9% CPU use with 8 MPI tasks x 1 OpenMP threads`

```bash
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 0.14694    | 0.63576    | 1.0896     |  45.1 | 34.28
Neigh   | 0.061086   | 0.11576    | 0.15578    |  10.6 |  6.24
Comm    | 0.21373    | 0.69223    | 1.2642     |  49.1 | 37.33
Output  | 0.00044284 | 0.00065848 | 0.0012293  |   0.0 |  0.04
Modify  | 0.32495    | 0.37083    | 0.45551    |   8.2 | 20.00
Other   |            | 0.03922    |            |       |  2.12

Nlocal:            375 ave         559 max         197 min
Histogram: 2 1 1 0 0 0 1 1 0 2
Nghost:            323 ave         447 max         206 min
Histogram: 2 0 1 1 0 0 2 0 0 2
Neighs:        2118.12 ave        3673 max         731 min
Histogram: 2 0 2 0 0 0 1 1 0 2

Total # of neighbors = 16945
Ave neighs/atom = 5.6483333
Neighbor list builds = 1166
Dangerous builds = 0
Total wall time: 0:00:01
```

5. `99.0% CPU use with 16 MPI tasks x 1 OpenMP threads`

```bash
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 0.039149   | 0.38002    | 1.494      |  73.3 | 14.98
Neigh   | 0.014347   | 0.072461   | 0.20876    |  23.5 |  2.86
Comm    | 0.18892    | 1.4073     | 1.8879     |  40.8 | 55.47
Output  | 0.00052081 | 0.0018748  | 0.0069923  |   4.7 |  0.07
Modify  | 0.39832    | 0.5047     | 0.6206     |   7.9 | 19.89
Other   |            | 0.1706     |            |       |  6.72

Nlocal:          187.5 ave         451 max          91 min
Histogram: 9 1 2 0 0 0 2 0 0 2
Nghost:        205.375 ave         384 max         102 min
Histogram: 4 2 2 2 1 1 2 0 0 2
Neighs:        1055.75 ave        3250 max         323 min
Histogram: 10 0 2 0 0 0 2 0 0 2

Total # of neighbors = 16892
Ave neighs/atom = 5.6306667
Neighbor list builds = 1162
Dangerous builds = 0
Total wall time: 0:00:02
```

6. `99.0% CPU use with 16 MPI tasks x 1 OpenMP threads` + GPU

```bash
Section |  min time  |  avg time  |  max time  |%varavg| %total
---------------------------------------------------------------
Pair    | 0.046393   | 0.41233    | 1.5131     |  76.5 | 15.28
Neigh   | 0.020617   | 0.11141    | 0.2955     |  28.9 |  4.13
Comm    | 0.35372    | 1.538      | 2.123      |  44.5 | 57.00
Output  | 0.00063928 | 0.0019649  | 0.0076661  |   5.0 |  0.07
Modify  | 0.34558    | 0.45745    | 0.55578    |   9.1 | 16.95
Other   |            | 0.1773     |            |       |  6.57

Nlocal:          187.5 ave         447 max         102 min
Histogram: 10 1 1 0 0 0 2 0 0 2
Nghost:        212.688 ave         402 max         111 min
Histogram: 5 1 2 2 2 0 2 0 0 2
Neighs:        1199.44 ave        3667 max         412 min
Histogram: 10 1 1 0 0 0 2 0 0 2

Total # of neighbors = 19191
Ave neighs/atom = 6.397
Neighbor list builds = 1155
Dangerous builds = 0
Total wall time: 0:00:04
```

## 研究室内の自分PCにwslをインストール使用としたがうまくいかない...
- `ファイル システムの 1 つをマウント中にエラーが発生しました。詳細については、「dmesg」を実行してください。`
  - [ここ](https://github.com/microsoft/WSL/issues/5680)を見たけど`wsl --update`でも改善しない...
  - `dmesg | grep -i error`(dmesgはLinuxカーネルが起動時に出力したメッセージを表示するコマンド)をやってみた
    ```bash
    [    2.678154] EXT4-fs (sdc): mounted filesystem with ordered data mode. Opts: discard,errors=remount-ro,data=ordered
    [    3.307626] init: (1) ERROR: MountPlan9WithRetry:285: mount drvfs on /mnt/f (cache=mmap,noatime,msize=262144,trans=virtio,aname=drvfs;path=F:\;uid=0;gid=0;symlinkroot=/mnt/
    [    3.967180] init: (1) ERROR: MountPlan9WithRetry:285: mount drvfs on /mnt/f (cache=mmap,noatime,msize=262144,trans=virtio,aname=drvfs;path=F:\;uid=0;gid=0;symlinkroot=/mnt/
    ```
  - なんか`path=F:\`っておかしくね...
  - Fドライブはフォーマットされていない8TBのHDDドライブ
  - 多分以前何かを入れておいて初期化したままになっていたみたい...
  - フォーマットしてバックアップドライブとして指定したら解決！(多分空き容量か何かで見てる？？)

## bin2cエラー
```bash
CMake Error at Modules/Packages/GPU.cmake:44 (message):
  Could not find bin2c, use -DBIN2C=/path/to/bin2c to help cmake finding it.
```
このエラーが無くならない
- 再インストールをしてもだめ
- bin2cへのパスを記入せよと言われているが、そのbin2cのファイルが見当たらない
  - everythingで探していたからだった...
    - everythingのオプション→検索データ→フォルダ→すべてをすぐに更新
    - データベースが更新され`\\wsl.localhost\Ubuntu\usr\local\cuda-11.7\bin\bin2c`にあることが分かった

## 別PC(5950XとRTX3070)でセットアップした際のエラー
- `make -j 16`コンパイル中に以下のようなエラー
  ```bash
  cc1plus: error: bad value (‘znver3’) for ‘-march=’ switch
  cc1plus: note: valid arguments to ‘-march=’ switch are: nocona core2 nehalem corei7 westmere sandybridge corei7-avx ivybridge core-avx-i haswell core-avx2 broadwell skylake skylake-avx512 cannonlake icelake-client icelake-server cascadelake tigerlake bonnell atom silvermont slm goldmont goldmont-plus tremont knl knm x86-64 eden-x2 nano nano-1000 nano-2000 nano-3000 nano-x2 eden-x4 nano-x4 k8 k8-sse3 opteron opteron-sse3 athlon64 athlon64-sse3 athlon-fx amdfam10 barcelona bdver1 bdver2 bdver3 bdver4 znver1 znver2 btver1 btver2 native; did you mean ‘znver1’?
  ```
- gccのバージョンが低いことが理由
  - 下記のコードで解決できるらしいですが、理由までは調べ切れていません。
    ```console
    sudo apt install gcc-10 g++-10
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 100 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
    sudo update-alternatives --config gcc
    ```

## Ghost Atomについて
- [このページ](https://docs.lammps.org/Developer_par_comm.html#communication)に書いてあった。


## computeとthermoの違いが分からない...
- どのように変数を計算させるのか、どのようにそれらをログとして出力するのか
  - 3種類の物理量: `global`、`per-atom`、`local`がある
    - `global`: `thermo_style custom`で出力可能
    - `per-atom`: `dump custom`で出力可能
    - `local`: `compute reduce`で出力可能
  - では杭壁面に作用する応力はどの変数に当たるのか?
    - たとえば[compute pressure](https://docs.lammps.org/compute_pressure.html)というコマンドがある
      - ここで計算される圧力あるいは応力テンソルは次のような形をしている
        - $P=\frac{Nk_BT}{V}+\frac{\sum_i^{N'}r_i \cdot f_i}{dV}$
          - 右辺第1項は状態方程式による項、第2項はビリアル定理による項
          - ビリアル定理は「系全体の運動エネルギー$K$の時間平均は、系全体のポテンシャルエネルギー$V$の時間平均の$1/2$に等しい」ことを示す
      - 何か計算したいものと違う気がする...
        - 粒状体では温度は考慮しないから、第2項しか存在しないはず
        - 各粒子に作用する力は重力と隣接する粒子からの接触力に起因するもの(?)
        - 多分このコマンドは複数の領域に分割されるようなケースは想定していないのでは...?
          - 簡単なコードで解析してみたい
    - また`reduce`というコマンドがある
      - localあるいはper-atomの出力された物理量をまとめられる
    - [これ(compute temp/sphere)](https://docs.lammps.org/compute_temp_sphere.html)かもしれない
          - 
- おそらくthermoから見た方がいいかもしれない

## 別のマニュアル(Rawiwan, 2021)から得られたこと
- 以下のソフトをインストール
  - VMware Workstation
  - Ubuntu desktop
  - Paraview v5.9: 粒子の動きを描画するソフト
  - Matlab: 応力状態や間隙比などを計算するためのプログラム
-  `bash LAMMPS_Build_Script_Serial.sh`が実行されている。
   -  詳しいことは読み取れていないが、やっていることはgranularモジュールを含んだ形でのシリアル(直列)処理とMPI用実行可能ファイルの作成？
   - ```bash
      rm -f lmp_serial # 
      cd src/STUBS
      rm -f libmpi_stubs.a
      rm -f mpi.o
      make -s
      cd ..
      make yes-GRANULAR
      echo 'Compiling mpi...'
      make serial >&Lammps_Output_File
      echo 'Cleaning up files...'
      %make clean-all
      mv lmp_serial ..
      echo 'Done!'
      cd ..
      echo `date` >>Build_Dates.txt
      sleep 2
      ```
- 解析するのに用いているファイルは`.in`ファイル、`.lj`ファイル、実行可能ファイル(`lmp_mpi`か`lmp_serial`)
  - `.in`ファイルには載荷条件が書かれている
    - デフォルトにない設定項目が`in.isotropic`内には存在する→恐らく新たに追加されたコマンド
      - `pair_modify trace_energy`: 接触点ごとのエネルギー項をシミュレーション中に出力するコマンドらしい
        - 一種のフラグとして機能している(`if (trac_energy)`のように)
        - このフラグによって計算されるかどうかが左右されるのは`shear`という変数
          - `gitLAMMPS_2019_Rigid_mpi_Ver8`だと`shear[3]`から`shear[8]`、`shear[26]`から`shear[29]`が更新されている
          - この`shear`が定義されているコードは発見できなかった
        - なぜパラメータ計算に関するコマンドが`pair_modify`で定義されているのか？`compute`じゃないのか？
      - 他にも`run_style verlet`や`timestep auto`、`fix ID GROUP-ID multistress`などがある
      - `fix ID GROUP-ID multistress`: 複数の応力条件や等体積条件を満たすように境界領域をコントロールする。
        - ```script
            fix ID group-ID multistress N M R keyword
              ID, group-ID are documented in "fix"_fix.html command
              multistress = style name of this fix command
              N = the percentage error in the target stress. See note 1.
              M = the number of steps to run within N% of the target stress before displaying that the target stress has been attained. See note 1.
              R = optional but highly recommended - this is the maximum engineering strain rate permissible for stress-controlled boundaries.
              one or more keyword/value pairs must be appended
          ```
      - ここに境界領域の応力の求め方が書いてあるはず...(詳しく見たいので[別項で](#fix-id-group-id-multistressをもう少し詳しく見る))
  - `.lj`ファイルには粒子の初期座標が書かれている
    - 生のテキストファイルとして読めるけどファイル構造は結構簡単そう
      - ヘッダーに粒子の種類数、粒子数、存在範囲
      - そのあとに各々の粒子の情報(種類ID、分からん、分からん、x, y ,z)
      - [ここ](https://docs.lammps.org/read_data.html#format-of-the-body-of-a-data-file)に書いてあった。atom typeがsphereの場合は次の通り→`atom-ID atom-type diameter density x y z`

## `fix ID GROUP-ID multistress`をもう少し詳しく見る
- 全体的な流れとしては6成分の応力を計算した後、if文で応力増分が一定の値に落ち着くまでひずみを最急降下法とかで最適化しているはず
  - 等体積条件での制御も同じ
- 参照したファイルは`fix_multistress.cpp`
- `constantpq_loop`
  - `?`は三項演算子
  - `therates`というのに上限と下限を設けている
  - `firstid`と`secondid`が何か
- `eval_fix_deform_params_constantpq`で`constantpq_loop`は使用されている。
  - `therates`は`temprates`: テンプレートではなくtemporal strain rateの略
  - `temprates`の計算方法は二つ
    1. 目標応力と現在応力の差分に体積弾性率を掛けたもの
      - なんでパラメータ名が`tallymeans`なの？何の略？
      - 
    2. `multistress`以降のパラメータとして与えられたもの
  - 
    - 
  
## TODO
1. 既往文献をあたる(国内でDEMをやられている先生はかなりいる、筑波大松島先生、名工大前田先生、土研大坪先生)
   1. マイクロパラメータ(粒子間剛性や粘性、使用している弾塑性モデル)がどのように設定されているのか
   2. 杭の挿入の際にどのような境界条件を変化させているのか
      1. 粉体工学のミルの設計に関する研究などが約に立つかもしれない
2. 適切な粒子数の設定
   1. DEMでは粒子数が多くなると解析に必要な時間が増える(解析時間と詳細度のトレードオフ)
   2. GPGPU(要はグラボ)を導入すると爆発的に速くなるらしいけど、詳細は不明。お金はあるので効果があるようだったら早急に購入したい。
3. 初期密度や間隙比をどのように調整するのか
4. 粒子に適切な重さを与えることができるのか
5. 境界面に作用する応力をデータとして吐き出すことができるのか
6. このREADMEの情報追加
   1. WSL2のUbuntuセットアップ手順
   2. LAMMPSのインストール手順


   