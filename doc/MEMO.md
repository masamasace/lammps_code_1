# 作業メモ
ここからは勉強過程で残した作業用メモです。参考になるかもしれないし、ならないかもしれません。

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
      - `meaneffectivestress = (tallymeans[0] + tallymeans[1] + tallymeans[2]) / 3.0`という書き方を見ると応力テンソルの対角成分のことを意味するらしい
      - なぜdiagonal componentとかではないのか...？
      - headerファイルに`The total mean stresses from all processors`と記載されていた
      - `fix_multistress.cpp`内には`tallymeans`は計算されていない
        - 1次元配列で要素数は6個、上三角行列の各成分に対応
      - どこで計算されているんだ...？
    2. `multistress`以降のパラメータとして`maxrate`
  - `MPI_Allreduce(&means[0], &tallymeans[0], 6, MPI_DOUBLE, MPI_SUM, world);`この式はどういう意味？
    - `MPI_ALLREDUC`とは`MPI_REDUCE`の結果を全プロセスに渡すという意味
    - `MPI_REDUCE`とは`MPI_GATHER`と演算を組み合わせたもの．例えば，各々のプロセスのデータの和，積，最大値，最小値などを求め，1つのプロセスに渡す．
    - そもそもMPIとは...
      - Message Passing Interfaceの略。
      - スレッドとメモリをひとまとめにしたものをプロセスと呼ぶ
      - MPIはプロセス間のメモリ情報のやり取りを規定したもの
    - `MPI_GATHER`とは各々のプロセスのデータを連結して1つのプロセスに集める
    - 引数は次の通り→`MPI_Allreduce(sendbuf, recvbuf, count, datatype, op, comm, ierror)`
      - countとはデータの個数のこと
      - commはどのプロセス間での通信を許可するかの範囲を決めたもの(理解が曖昧)。デフォルトで`MPI_COMM_WORLD`が用意されている。今回の`world`もその派生？
    - (なるほど！)`sendbuf`と`recvbuf`はそれぞれ送信するデータと受信するデータの**先頭のアドレス**を引数とする
      - だから`means[i]`みたいな書き方にはならない
      - ちなみに`means`は「現在のタイムステップにおける平均応力」
    - なのでこのコードでやっていることを日本語で記述すると「各プロセス内の`means`に関して総和を取ったものを各プロセス内の`tallymeans`にブロードキャスト(同じ値を代入)する」
  - `means`はどうやって計算されている？
    - `fix_mutilstress.h`だと` double *means; //~ The mean stresses on the current timestep`と書かれている
    - ヘッダーファイルの変数に付くアスタリスクってどういう意味だっけ？
      - ポインタだ
    - 例えばこんな書き方になっている→`double *means = stressatom->array_export();`
    - アロー演算子(`->`)はポインタからでも構造体や共用体の要素にアクセスできる。
    - (特定の粒子あるいは粒子全体に作用する応力の総和によって、領域に作用する圧力を計算しているとすれば、杭に作用する応力ってどうやって計算するんだ...？)
    - `stressatom`はcomputeクラスで`compute_stress_atom.cpp`と`compute_stress_atom.h`に定義されている
    - 詳しい計算は`array_export()`メンバ内に書かれている
  - `array_export()`内の計算内容
    - (これかもしれない...) `if (mask[i] & groupbit){...}`という構文がある
      - `...`では各粒子の体積を次元を考慮して計算した後、`stress`と体積を掛け算して、`stress`の一次元目に対して総和を取っている
        - `stress[i][j]`のうち`i`は粒子総数、`j`は応力テンソルの6成分
      - `stress[i][j]`は`vatom[i][j]`の総和...?
        - i番目の粒子に作用する力jによって生じる6成分の応力成分だったとたら`vatom[i][j][6]`じゃないとおかしくない...？
      - `vatom[i][j]`の定義はどうなっている
        - vはおそらくvirial(ビリアル)の略...ビリアル応力
        - とりあえず[lammpsの公式サイト](https://docs.lammps.org/compute_stress_atom.html)を読
        - あるいは[こちらのサイト](https://github.com/kaityo256/md2019/blob/main/pressure/README.md)も参考になるかもしれない
          - [こちら](VIRIAL.md)に勉強内容を書いた
        - 完全には理解できていないが、「ビリアル定理によって熱平衡状態にある多粒子系の応力テンソルは粒子間相互作用と粒子それぞれの運動エネルギーから定義できる」ことを指す
        - `vatom[i][j]`のうち`[i]`は粒子のインデックス、`[j]`は応力テンソルの6成分。(おそらくドキュメントに書かれている順番からxx, yy, zz, xy, xz, yzのはず)
        - `vatom[i][j]`は粒子間相互作用以外にも結合力(bond)や結合角(angle)、二面角(dihedral), improper, kspaceからも計算される
    - `compute_stress_atom.cpp`内の`if (mask[i] & groupbit){...}`の`mask[i]`と`groupbit`とはそれぞれなに？
      - `mask[i]`は`nlocal`個の配列長さを持つ
      - `nlocal`は`atom.h`内にプロセッサーが有する粒子というコメントがある
        - lammpsでは解析する空間を区切っているがそれに関する名前が複数あって違いが良く分からない
        - Region, Processor, Rank, Mask, Group, Neiborlist
      - `mask[i] = (int) ubuf(buf[m++]).i;`というコードがよく見かけられる
        -  `ubuf`は共用体として定義されていて、型変換を行っている
           -  コンパイラによる警告を避けるためと書かれているけど、あまりよくない書き方なのでは...？あるいはbit数の異なるOSそれぞれに対応するための折衷案？
           -  半分ぐらいは`unpack_border`の中に書かれている
              -  引数が`n, first, *buf`→`buf`はポインタ渡し
           -  `m++`はインクリメントなので、bufの中身が重要
       -  条件付きの再帰関数になっている
          -  再帰関数ではなくコールバック関数？
          - ```cpp
              if (atom->nextra_border)
                for (int iextra = 0; iextra < atom->nextra_border; iextra++)
                  m += modify->fix[atom->extra_border[iextra]]->
                    unpack_border(n,first,&buf[m]);
            ```
          - このコードが良く分からない...
          - `nextra_boarder`はおそらくnumber of particle to be extracted around borderの略
          - `buf`は文字通り仮の変数で色々な者が入ってくる
            - 例えば粒子の位置であったり、速度であったり
          - `pack_border`は粒子情報をバッファに保存し、近傍で再構築する際に通信するコマンド
          - `unpack_border`バッファから粒子情報を取り出す
          - `mask[i] = (int) ubuf(buf[m++]).i;`というコードはバッファからのやり取りで本質的なものではない
       - `mask[i]`で境界領域の粒子の判別をしている？ではそのフラグはどのようにして切り替わっている？
         - 調べても情報は出てこなかった
         - `fix_pour.cpp`内では`atom->mask[n] = 1 | groupbit;`と書かれている。
         - `|`はbitでのOR演算子
         - `groupbit`は`compute_stress_atom.cpp`内で`int groupbit = bitmask[igroup];`と書かれている
         - `bitmask`は`group.cpp`内で`for (int i = 0; i < MAX_GROUP; i++) bitmask[i] = 1 << i;`と書かれている
           - MAX_GROUPは32と定義されている
         - これだとこのコードの後は'[11111111111111111111111111111111]'となっているはず
         - `bitmask`の値を変更するようなコードは見つからない...
     - `bitmask`は最大32個のグループを識別するもの
       - ということは`mask[i]`も`int`型でないといけない
     - ここで打ち止め、もう少しlammpsのコードを触ってからこちらに戻ってくる。

## `lmp: error while loading shared libraries: libcuda.so.1`の解決策について
- このエラーに行きつく前のエラー達
  - `cmake -D CMAKE_CXX_COMPILER=${HOME}/lammps-stable_23Jun2022/lib/kokkos/bin/nvcc_wrapper ../cmake`のコマンドだけは別に実行しないと行けないらしい...
    - 理由は不明。とりあえずこのひとつ前のコードと分けて実行すると`Enable Package`のリストが正しく更新されることを確認済み。`make -j 4`の最初に表示されるビルド情報をしっかりと見ること
  - そもそも`make install`をすると新しくビルドをしてもう一度`make install`をしても更新されない現象がある
    - 解決策については以下に記載予定(現在編集中)
      - `lammps`配下の`build`フォルダを一旦削除
      - `~/.local/bin`内にある`lmp`を削除
      - `~/.profile`の`PATH=$PATH:$HOME/.local/bin`の行も一旦削除
    - 現在調査中
- `libcuda.so.1`から`libcuda.so`へのシンボリックリンクが作成されていないことが原因
  - ソースコードを修正するとしたら、ビルドの過程でシンボリックリンクを作成する必要があるがどうやるの？
  - `libcuda.so`自体は`usr\local\cuda-11.7\targets\x86_64-linux\lib\stubs\libcuda.so`に存在するが、`libcuda.so.1`は存在しない
  - ただ`libcuda.so.1`がどの環境変数によって参照されているのかが分からない
  - `ubuntu20.04_gpu`では次のような記述がある
    - ```bash
        # add missing symlink 
        ln -s /usr/local/cuda-${CUDA_PKG_VERSION}/lib64/stubs/libcuda.so /usr/local/cuda-${CUDA_PKG_VERSION}/lib64/stubs/libcuda.so.1
      ```
    - `libcuda.so.1`から`libcuda.so`へのシンボリックリンクが作成するコマンド
      - ただ`/usr/local/cuda-${CUDA_PKG_VERSION}/lib64/stubs/`内には`libcuda.so.1`は存在していない
        - cudaが複数あるけどこれはよいのか...(`cuda/      cuda-11/   cuda-11.7/`)
      - ちなみに
        ```bash
        /usr/local/cuda-11.7$ ls -n
                              lrwxrwxrwx 1 0 0    24 Jun  9 10:10 lib64 -> targets/x86_64-linux/lib
        ```
        となっているので`targets/x86_64-linux/lib`配下のファイルはすべて`lib64`配下からシンボリックリンクとして作成されている
  - ひとまず`sudo ln -s libcuda.so libcuda.so.1`コマンドを`usr\local\cuda-11.7\targets\x86_64-linux\lib\stubs`で実行
    - ただ相変わらず同じエラーが出る
    - もう一回ビルドをやってみる
    - `mkdir build`をした段階ではもちろんフォルダは空
    - 次のコードを行うと`build`フォルダの中にファイルが生成
      ```bash
      cmake -D PKG_GPU=yes -D GPU_API=cuda -D GPU_ARCH=sm_61 -D PKG_GRANULAR=yes -D PKG_KOKKOS=yes -D Kokkos_ARCH_SKX=yes -D Kokkos_ENABLE_OPENMP=yes -D BUILD_OMP=yes -D Kokkos_ARCH_PASCAL61=yes -D Kokkos_ENABLE_CUDA=yes ../cmake
      ```
    - 次のコードを行うと`CMakeCXXCompiler.cmake`内の最初の行が`set(CMAKE_CXX_COMPILER "/home/masa/lammps-stable_23Jun2022/lib/kokkos/bin/nvcc_wrapper")`に置き換わる
      ```bash
      cmake -D CMAKE_CXX_COMPILER=${HOME}/lammps-stable_23Jun2022/lib/kokkos/bin/nvcc_wrapper ../cmake
      ```
    - `make`を走らせる(走らせるときには`./lmp -in in.foobar`としないとダメ)
      - やっぱりパッケージが全部入っていない...
    - `..cmake`を常に書き込みしていた...
    - 正しい手順は
      ```bash
      mkdir build
      cd build
      cmake ../cmake # ベースとなる構成ファイルをcmakeから取ってくる
      cmake -D PKG_GPU=yes -D GPU_API=cuda -D GPU_ARCH=sm_61 -D PKG_GRANULAR=yes -D PKG_KOKKOS=yes -D Kokkos_ARCH_SKX=yes -D Kokkos_ENABLE_OPENMP=yes -D BUILD_OMP=yes -D Kokkos_ARCH_PASCAL61=yes -D Kokkos_ENABLE_CUDA=yes -D CMAKE_CXX_COMPILER=${HOME}/lammps-stable_23Jun2022/lib/kokkos/bin/nvcc_wrapper . # 最後は.(ドット)のみ!
      ```
    - 結果は変わらない
  - `LD_LIBRARY_PATH`に`usr\local\cuda-11.7\targets\x86_64-linux\lib\stubs`のパスを追加してもダメだった
    - ちなみに`PATH=$PATH:/usr/local/cuda-11.7/bin`の中で`$PATH`が付いているのは、既存の環境変数を上書きしないようにするため(環境変数は`:`で複数設定できる)
    - 環境変数に追加してもだめ、シンボリックリンクを複数作成してもダメ
  - ビルドオプションで`-D CMAKE_CXX_COMPILER=${HOME}/lammps-stable_23Jun2022/lib/kokkos/bin/nvcc_wrapper`の部分を外してみた。
    - `Illegal instruction`の出力が出た
      - そもそもなぜこのオプションを追加したのか記録が残っていない...
    - `ldd ./lmp`で見ると`libcuda.so.1`の問題は解決している
      - あれそうしたら、nvcc_wrapperの部分が問題なのでは？？
    - もう一度nvccも含めてビルドしなおしたら`Illegal instruction`のエラーが出るようになった
  - 明日に向けての可能性の整理
    - CUDAのインストールがおかしい
    - KOKKOSのオプションの設定がおかしい
    - cmakeでデバッグオプションみたいなものってつけれれないの？
  - 回った...！
    - 正解はKOKKOSのオプションでした...
      - 今回使っているCPUはIntel Core i9-9900Kで世代としてはCoffee Lakeに相当
      - [KOKKOSのオプション](https://docs.lammps.org/stable/Build_extras.html#kokkos)では以下のように書かれていて、Coffee Lake世代はSkylake世代よりも後だからArch-IDをSKXに設定していた
        ```
        Arch-ID | Description
        BDW     | Intel Broadwell Xeon E-class CPU (AVX 2 + transactional mem)
        SKX     | Intel Sky Lake Xeon E-class HPC CPU (AVX512 + transactional mem)
        ```
      - ただ[Intelのサイト](https://www.intel.co.jp/content/www/jp/ja/products/sku/186605/intel-core-i99900k-processor-16m-cache-up-to-5-00-ghz/specifications.html)にもあるように9900KはAVX2までしか対応をしていない。そのためArch-IDはBDWに設定するのが正解だった。
  - マジで長かった...

## `Cuda driver error 34 in call at file '/home/ms/lammps-stable_23Jun2022/lib/gpu/geryon/nvd_device.h' in line 326.`に関する解決策
- 今度は別のエラー
- GPUを使ったサンプルコードを回す際に`mpirun -np 1 ./lmp -sf gpu -pk gpu 0 -in ../code/sample2.in`を入力
- ```bash
    Cuda driver error 34 in call at file '/home/ms/lammps-stable_23Jun2022/lib/gpu/geryon/nvd_device.h' in line 326.
  ```
  というエラーが出る。
  - このエラーコードは[Nvidiaのサイト](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__TYPES.html#:~:text=cudaErrorStubLibrary)によると`cudaErrorStubLibrary`に該当するものらしい
  - 内容としてはdebug用に使うはずのスタブライブラリを使用しているため、発生するもの
  - [このサイト](https://forums.developer.nvidia.com/t/checkmacros-cpp-272-error-code-1-cuda-runtime-cuda-driver-is-a-stub-library/202911/11)が参考になりそう
    - おそらくパスをしっかりと設定していないことが原因か？



## TODO
1. 既往文献をあたる(国内でDEMをやられている先生はかなりいる、筑波大松島先生、土研大坪先生)
   1. マイクロパラメータ(粒子間剛性や粘性、使用している弾塑性モデル)がどのように設定されているのか
   2. 杭の挿入の際にどのような境界条件を変化させているのか
      1. 粉体工学のミルの設計に関する研究などが約に立つかもしれない
2. 適切な粒子数の設定
   1. DEMでは粒子数が多くなると解析に必要な時間が増える(解析時間と詳細度のトレードオフ)
   2. GPGPU(要はグラボ)を導入すると爆発的に速くなるらしいけど、詳細は不明。お金はあるので効果があるようだったら早急に購入したい。
3. 初期密度や間隙比をどのように調整するのか
4. 粒子に適切な重さを与えることができるのか
5. 境界面に作用する応力をデータとして吐き出すことができるのか


   