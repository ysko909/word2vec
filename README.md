# Word2Vecを使って類語検索可能なモデルを作る

基本的には[ここ](https://qiita.com/kenta1984/items/93b64768494f971edf86) 参照。

## 事前準備

Python環境とMecabがすでにインストールされていることが前提です。PythonはAnacondaでも構わないです。

### コーパスの用意

日本語のWikipediaのデータは[ここ](https://dumps.wikimedia.org/jawiki/latest/)にダンプ化されたものが用意されています。

ダウンロードには下記のコマンドを実行します。あ、PowerShellで実行するのを想定してます。

	$ curl https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2 -o jawiki-latest-pages-articles.xml.bz2

真昼間にダウンロードし始めたところ、大体1時間弱で落としました。

そして、XMLファイルをクレンジングして本文だけテキストとして取り出します。それには[WikiExtractor](https://github.com/attardi/wikiextractor)を使用します。と言っても、やることはクローンしてインストールして落としてきたXMLファイルに対してコマンドを発行するだけです。

	$ python setup.py install
	$ python WikiExtractor.py jawiki-latest-pages-articles.xml.bz2

処理対象のファイルが大きいので、実行にも時間がかかります。一応実績値としては40分かからないくらいでした。

結果は生成されたtextディレクトリ配下に

	text/AA/wiki_00
	text/AA/wiki_01
	...

の形で分割されて出力されます（拡張子はなし）。PowerShellで下記のコマンドを発行してファイル結合します。

	Get-Content .\*\* -Encoding utf8 | Out-File test.txt -Encoding utf8
	Get-Content .\test.txt -Encoding Default | Out-File wiki_default.txt -Encoding default

なお、`-Encoding`オプションを使用していますが、あんまり意味なかったかも知れないです。とりあえず後述の`Mecab`に読み込ませるときエンコード指定しないと文字化けしちゃうので、オプションをつけていました。どうも単純に出力すると入力ファイルの文字コードに関わらず、出力ファイルの文字コードがS-JISになっちゃうっぽい。ちなみに「`> hoge.txt`」みたいな形で出力する場合、出力形式を設定できないので余計おかしな文字コードに勝手に変換してしまうっぽい。これは`chcp`で文字コードを変えても一緒でした。

## コーパスの分かち書き

モデルの作成にはgensimというライブラリを利用しました。pipを使えばすぐにインストールできます。が、今回は`conda`を使用しました。特に意味はありません。クセです。

	$ conda install gensim

gensimを使ってモデルを作成する場合、コーパスを分かち書きする必要があるのですが、これはMeCabを使って行います。なお、MeCabは[オフィシャル](http://taku910.github.io/mecab/)のWindows用インストーラーを使用しました。

	$ mecab -b 81920 -Owakati .\wiki_default.txt | Out-File wiki_wakati_utf8.txt -Encoding utf8

オプションの-bはバッファの調整、-Owakatiは文章を単語で分かち表現するのに用いるオプションです。パイプで繋いだ後者は、utf8でファイル出力するよーというコマンドレット。PowerShellで実行する前提です。コマンドプロンプトじゃこのコマンドレットが使えないはず。

PowerShellやコマンドプロンプト上でのmecabの入出力はS-JISだと思います（たぶん）。少なくともUTF-8のデータをそのまま突っ込むと文字化けして出力されました（出力だけが化けている可能性もある）。なので、コーパス生成時に「`-Encoding default`」でファイルを生成したわけです。たぶんだけど、WindowsのMecab、というよりはPowerShellとかコマンドプロンプト経由でMeCabにデータ入力するってのが、そもそもUTF8にちゃんと対応してないんじゃない？ファイル出力にエンコード指定する分にはとりあえず平気っぽいですが。

ちなみにバッファ指定しなかったところ、エラーメッセージが表示されました。

	input-buffer overflow. The line is split. use -b #SIZE option.

インプットに対してバッファ指定が足りねぇよ！（デフォルトは8192）って言われたので、とりあえず10倍してやった。

## モデルの作成

分かち書きが終わったらモデルを作成します。作成するには下記のコードを実行するだけ。ちなみにこのコードは分かち書きしたデータと同じフォルダに保存してください。

~~~python
from gensim.models import word2vec
import logging

logging.basicConfig(format='%(asctime)s : %(levelname)s : %(message)s', level=logging.INFO)
sentences = word2vec.Text8Corpus('./wiki_wakati_utf8.txt')

model = word2vec.Word2Vec(sentences, size=200, min_count=20, window=15)
model.save("./wiki.model")
~~~

実行するだけ、とは言っても１時間強くらいはかかったと思います。

wiki.modelとそれに付随するファイルが生成されれば、モデルの作成は完了です。

## 実行

下記のPythonコードを実行してみます。ちなみにこのコードはモデルデータと同じフォルダに保存してください。類義語や関連性の高い言葉が返ってくるはずです。

~~~python
from gensim.models import word2vec

model = word2vec.Word2Vec.load("./wiki.model")
results = model.wv.most_similar(positive=['ティーガー'])
for result in results:
	print(result)
~~~

実行結果

~~~

('戦車', 0.7086217403411865)
('パンター', 0.707148551940918)
('ホニ', 0.6609956622123718)
('マルダー', 0.6537086963653564)
('ヴィットマン', 0.6248199939727783)
('トゥラーン', 0.6038566827774048)
('ヤークトティーガー', 0.5888940691947937)
('ヤークトパンター', 0.5750963091850281)
('ズリーニィ', 0.5719200968742371)
('装甲', 0.5595574378967285)

~~~

別の言葉で検索してみる。

~~~python
from gensim.models import word2vec

model = word2vec.Word2Vec.load("./wiki.model")
results = model.wv.most_similar(positive=['南極'])
for result in results:
    print(result)
~~~

実行結果

~~~

('南極大陸', 0.811050534248352)
('北極', 0.7787120938301086)
('極地', 0.7432698607444763)
('ロス海', 0.7250320315361023)
('北極圏', 0.6782439947128296)
('ドローニング・モード・ランド', 0.6576575040817261)
('ウェッデル海', 0.6385233998298645)
('極点', 0.6266143918037415)
('グリーンランド', 0.6156957149505615)
('アムンセン', 0.6097243428230286)

~~~


おお、いい感じ。

