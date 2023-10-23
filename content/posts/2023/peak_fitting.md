--- 
title: "ピークフィッティングまとめ"
date: 2023-10-18T16:33:52+09:00
draft: false
disableComments: false
description: "pythonとROOTを使ったピークサーチ、ピークフィッティングのまとめ"
slug: ""
tags: ["analysis", "python","ROOT","peak_fitting","peak_search"]
categories: []
externalLink: ""
series: []
math: true
---

普段[ROOT](https://root.cern/)を使って解析していますが、物理解析に特化しており一般的ではないと思います。
特に有用なデータ処理に対してpythonを用いて同じことができれば良いなということで、特にピークフィットについて調べました。

## 環境について
テストデータを用いた実験は以下のマシンで行いました。
```shell
> sw_vers
ProductName:		macOS
ProductVersion:		13.6
BuildVersion:		22G120
> uname -v
Darwin Kernel Version 22.6.0: Fri Sep 15 13:41:30 PDT 2023; root:xnu-8796.141.3.700.8~1/RELEASE_ARM64_T8103
```
ROOTのバージョンは、
```shell
> root --version
ROOT Version: 6.28/06
Built for macosxarm64 on Aug 28 2023, 11:29:15
From tags/v6-28-06@v6-28-06
```
python 3.11.6を用い、パッケージのバージョンはgithubにある[requirements.txt](https://github.com/okawak/example/blob/main/peak_fitting/requirements.txt)を参照してください。

今回の記事ではデータの例として、Ge検出器で$^{133}$Baのガンマ線を測定したデータを用いようと思います。
データ構造としては最も単純と考えられる、テキストファイルが与えられているとします。
例えば、0chから順に、
```txt
1
10
1000
233
0
```
のような一列になっているファイルを考えます。

## ROOTでの解析
fit.Cという名前のマクロを用いて解析します。

### データの読み込み
ROOTのTH1にはテキストファイルから直接TH1オブジェクトを生成するコンストラクタは無さそう(あったら教えてください...)なので、
一行ずつ読み込ませてヒストグラムにセットしていきます。
また、4096チャンネル(12ビット)を仮定しています。

まず、単純にヒストグラムを表示させるマクロは以下の通りに書けます。
```cpp
void fit(TString text_data=""){
  ifstream fin(text_data);
  if(!fin){
    std::cerr << "Error: not found " << text_data << std::endl;
    return;
  }

  Int_t max_bin = 4096;
  TH1I *h = new TH1I("h", "h", max_bin, 0, max_bin - 1);

  string buf;
  Int_t ch = 0;
  while(getline(fin, buf)){
    h->SetBinContent(ch, stoi(buf));
    ch++;
  }

  if(max_bin != ch){
    std::cerr << "max channel number is different: setting value = " << max_bin
              <<", but data have " << ch << " channels" << std::endl;
    return;
  }

  TCanvas *c1 = new TCanvas("c", "c", 600, 600);
  h->Draw();
  c1->SaveAs("raw_data.png"); // save
}
```
次のようにテキストファイルの名前を引数に入れて実行すれば、結果として次のような絵が得られるはずです。
テキストファイルとマクロファイルは同じディレクトリにあると仮定しています。
```shell
root 'fit.C("Ba.txt")'
```
{{< figure src="raw_data.png" width=400 >}}

今後はこのfit.Cを拡張していってマクロを完成させます。

### ピークサーチ
フィッティングを収束させるためには、初期値をうまく設定する必要があります。
この生データを見ながら大体ここら辺にピークがあるとして、一つ一つピークフィッティングしても良いですが、
何回も行ったり、何個もピークがあると単純作業で大変です。
そこで、ピークを自動で見つけてくれる`TSpectrum`というオブジェクトを使います。

TSpectrumのコンストラクタは以下の通りです。
```c++
TSpectrum (Int_t maxpositions, Double_t resolution=1)
```
- maxposition: 見つけるピークの数
- resolution: 1は隣り合うピークが3sigma以上離れたものを探すことに対応する。数字が大きくなるとより近いピークを探すようになる。

これを用いて$^{133}$Baのピークを8つ探します。(簡単のため80 keV付近の2つの重なったピークは一つとします。)
マクロの引数に探したいピークの数を追加して以下のように書き換えます。

```c++
void fit(TString text_data="", Int_t peak_num=0){
  -- snip --

  h->SetAxisRange(80, 1100); // size
  TSpectrum *spec = new TSpectrum(peak_num, 1.5); // 1st: peak number, 2nd: resolution
  Int_t nfound = spec->Search(h, 1, "", 0.0045); // 1st: histogram, 2nd: sigma, 3rd: option, 4th: threshold

  h->Draw();
  gPad->SetLogy();
  c1->SaveAs("tspectrum.png"); // save

  if(nfound != peak_num){
    std::cerr << "Warning: tried to find " << peak_num << " peaks, but found " << nfound << " peaks" << std::endl;
    return;
  }
}
```
ここで、`h->SetAxisRange()`はピークを探す範囲を限定するために用いています。(ペデスタルを除いたり、自然放射線のピークを避ける。)
デフォルトの値ではうまくピークサーチすることが難しいと思うので、TSpectrumのコンスラクタの引数や、
Searchメソッドの引数を適当に設定することでうまくサーチすることができると思います。

これを実行すると、次の絵が得られます。
```shell
root 'fit.C("Ba.txt", 8)'
```

{{< figure src="tspectrum.png" width=400 >}}

### フィッティング
ピークサーチではこれらピークの中心値を得ることができます。しかし、これはビン幅の精度なのでこれを元にガウシアンフィッティングして、
ピークの情報を得ます。

これ以降は通常のフィッティングの要領で行えると思うのでコードのみ紹介します。
フィット関数はガウシアン＋一次関数のバックグラウンドを仮定しています。
また注意点として、80 keV付近の重なったピークの処理は行っていないです。
状況に応じて、個々のピークに対する対応は行ってください。

```cpp
  -- snip --

  gStyle->SetOptFit(1111);
  Double_t *xpeaks = spec->GetPositionX();
  std::sort(xpeaks, xpeaks + peak_num);

  TF1 *f[peak_num];
  for(Int_t i=0; i<peak_num; i++){
    f[i] = new TF1(Form("f[%d]", i), "gausn(0) + pol1(3)");
  }

  for(Int_t i=0; i<peak_num; i++){
    f[i]->SetRange(xpeaks[i] - 15.0, xpeaks[i] + 15.0);
    f[i]->SetParameters(10000, xpeaks[i], 1.0, 10.0, -0.001); // initial value
    f[i]->SetParLimits(2, 0., 100.);
    h->SetAxisRange(xpeaks[i] - 15.0, xpeaks[i] + 15.0);
    h->Fit(f[i], "r");
    f[i]->Draw("same");
    c1->SaveAs(Form("fit%d_result.png", i)); // save
  }

  h->SetAxisRange(80, 1100); // size
  c1->SaveAs("fit_all.png"); // save
}
```
ここで`std::sort`しているのは、Searchして得られる配列がchが小さい順に並んでいないためです。(今のバージョンではそんなことない？)
従って、ピークの高さも`spec->GetPositionY()`で得られますが、sortすることで順番が混じってしまうので、簡単のためchの値のみ取得しています。
うまく処理すれば、ピークの高さも用いてうまい初期値を設定することができると思います。(pythonの解析を参照)

初期値の値は中心以外は適当です。`h->Fit(f[i], "r")`のオプションの部分に`"rq"`とqを追加すると結果がターミナル上に表示されなくなります。
表示された値を用いて他の処理を行いたい場合は、`f[i]->GetParameter(0)`などのメソッドを用いてアクセスすることができます。

このマクロでは各ピークと全体の絵を保存するようにしました。全体の図はこのようになります。
統計ボックスに表示されているパラメータの値は最後のフィット、つまり一番エネルギーが高いピークに対してであることに注意してください。

```shell
root 'fit.C("Ba.txt", 8)'
```

{{< figure src="fit_all.png" width=400 >}}

ここで紹介したマクロは、コメントアウトを残した行を変更することで、より良いものにリファクタリングできると思います。

## pythonでの解析

Jupyter Notebookを使ってfit.ipynbという名前のファイルで解析します。
vscodeなどでNotebookを使う環境を整えれば、簡単に解析することができると思います。
まず次のようにimportします。
```python
import numpy as np
import matplotlib.pyplot as plt
from scipy.optimize import curve_fit
from scipy.signal import find_peaks
```

### ピークサーチ
ファイルからデータを読み出す際は、通常通りnumpy配列として読み込むだけで問題ないと思います。
また、ヒストグラムのオブジェクトを作成する方が適切かと思いますが、簡単のためグラフとして読み込みたいと思います。
(ROOTではTH1ではなく、TGraphとして読み込む)

まず、生データを簡単に確認するコードの例はこのようになります。
```python
# input the text file name
file_name = "Ba.txt"

raw_data = np.loadtxt(file_name)
ch = np.arange(4096)

plt.plot(ch, raw_data)
plt.savefig("py_raw_data.png") # save
plt.show()
```

{{< figure src="py_raw_data.png" width=400 >}}

ピークサーチに用いられる関数は様々あるようですが、ここではよく使われていると考えられるscipyの`find_peaks`を用いて見つけていこうと思います。
この関数の主な引数は、
- height: ピークを探すときの最小の高さを設定する
- threshold: ピークの点とその隣の点の最小距離を指定する
- distance: 隣り合うピークの最小距離を設定する
- prominence: どのくらい顕著なピークを見つけるかを表す。値が大きいほど顕著なピークのみを探すようになる。

であり、主に`prominence`（おそらくROOTのTSpectrumのresolutionに対応？）を変えて結果を見ていくのが良いと思います。
また、次の解析の都合上(peak heightを得るために)heightの引数が必須なので、適当に小さな値を入れておくと良いです。

この関数で得られる位置の配列は小さい順になっているはずなので、ピークの高さも次のように同時に得ておきます。
```python
# 範囲の指定
data_range = (80, 1100)

data = raw_data[data_range[0]:data_range[1]]
ch = np.arange(data_range[0], data_range[1])

# parameters: height, threshold, distance, prominence
peaks, results = find_peaks(data, height=10, prominence=50)
pos_peaks = peaks + data_range[0]
height_peaks = results["peak_heights"]

plt.plot(ch, data, label="data")
plt.plot(ch[peaks], data[peaks], "x", label="find peaks")
plt.yscale("log")
plt.legend()
plt.savefig("py_find_peaks.png") # save
plt.show()
```

{{< figure src="py_find_peaks.png" width=400 >}}

このようにして、ROOTのTSpectrumと同じような処理を行うことができました。

### フィッティング
フィッティングはscipyの`curve_fit`を用いて行います。
他にもleastsqなどの方法もありますが、一番直感的なものはこの関数だと思います。

まず、フィットする関数を定義します。
```python
def fit_func(x, count, mu, sigma, a, b):
    return count * np.exp(- (x - mu)**2 / (2 * sigma**2)) / (np.sqrt(2. * np.pi) * sigma) + a + b * x
```

最終的にフィットするコードの例はこちらになります。
```python
x_list = []
y_list = []

for i in range(len(peaks)):
    fine_range = (pos_peaks[i] - 15, pos_peaks[i] + 15)
    fine_data = raw_data[fine_range[0]:fine_range[1]]
    fine_ch = np.arange(fine_range[0], fine_range[1])

    # initial parameters
    par, cov = curve_fit(fit_func, fine_ch, fine_data, sigma=np.sqrt(fine_data), p0=[height_peaks[i]*2.0, pos_peaks[i], 1.0, 10.0, -0.001])
    chi2 = np.sum(((fit_func(fine_ch, par[0], par[1], par[2], par[3], par[4]) - fine_data)/np.sqrt(fine_data))**2)
    ndf = len(fine_ch) - 5

    plt.plot(fine_ch, fine_data, 'o', label="data")
    x = np.arange(fine_range[0], fine_range[1], 0.01)
    x_list.append(x)
    y_list.append(fit_func(x, par[0], par[1], par[2], par[3], par[4]))
    plt.plot(x, fit_func(x, par[0], par[1], par[2], par[3], par[4]), label="fitting")
    plt.yscale("log")
    plt.legend()
    plt.grid()
    plt.savefig("py_fit" + str(i) + "_result.png") # save
    plt.show()

    print("chi2/ndf = {:7.3f}/{}".format(chi2, ndf))
    print("p0 : {:10.5f} +- {:10.5f}".format(par[0], np.sqrt(cov[0,0]/chi2*ndf)))
    print("p1 : {:10.5f} +- {:10.5f}".format(par[1], np.sqrt(cov[1,1]/chi2*ndf)))
    print("p2 : {:10.5f} +- {:10.5f}".format(par[2], np.sqrt(cov[2,2]/chi2*ndf)))
    print("p3 : {:10.5f} +- {:10.5f}".format(par[3], np.sqrt(cov[3,3]/chi2*ndf)))
    print("p4 : {:10.5f} +- {:10.5f}".format(par[4], np.sqrt(cov[4,4]/chi2*ndf)))

plt.plot(ch, data)
for i in range(len(x_list)):
    plt.plot(x_list[i], y_list[i])
plt.yscale("log")
plt.savefig("py_fit_all.png") # save
plt.show()
```

{{< figure src="py_fit_all.png" width=400 >}}

curve_fitにおけるエラーは、デフォルトの値だと正確ではないので修正を加える必要があります。
具体的には、共分散行列の対角成分にカイ二乗の値を用いて少し修正する必要があるみたいです。
ROOTを用いれば修正の必要がないそうなので、余力があれば、ROOTの結果とpythonを使ったフィットの結果の比較を記載したいと思います。
- 参考：[フィッティングプログラムの比較](https://qiita.com/niikura/items/79dc6837f017c05afaa7)

## まとめ

やはり、ROOTは物理解析に特化していることから、解析コードは単純に書けるなと感じました。(ROOTに慣れているだけ？)
特にcurve_fitを使う際はフィッティングエラーに補正が必要らしいので、これがちょっと面倒な部分だと思います。
ただ、いつもROOTを使っていましたが、pythonでもピークフィットを自動化できることを知って、やっぱりpythonってすごいなと改めて思いました。

また、今回作成したマクロ、pythonスクリプト、データ例(Ba.txt)はgithubの[レポジトリ](https://github.com/okawak/example/tree/main/peak_fitting)に置いたので興味がある方は触ってみてください。
