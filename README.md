
<a href="https://speakerdeck.com/pistatium/sutukirifen-karu-python-falserogu"><img src="https://github.com/pistatium/about_python_logging/blob/master/etc/slide_thumb.png?raw=true" width=300></a>


# すっきり分かる Python のログ

Python のロガー周りを理解するための資料です。


## Loggerを使う


```python
from logging import getLogger

# Setup logger
logger = getLogger(__name__)
```

python の Logger はこうやって初期化するのが一般的です。
あとで解説するので `__name__` とかはおまじないだと思ってください。  

早速このloggerを使ってみましょう。


```python
from logging import getLogger

logger = getLogger(__name__)

def main():
    logger.debug('This is a Debug message')
    logger.info('This is a Info message')
    logger.warning('This is a Warning message')
    logger.error('This is a Error message')
    logger.critical('This is a Critical message')
    
if __name__ == '__main__':
    main()
```

他の言語にもあるように Python でも `debug` `info` `warning` `error` `critical` の5種類のログレベルが使用できます。
特に難しいことはないですね。

実行結果も見てみましょう。

>This is a Warning message  
>This is a Error message  
>This is a Critical message

あれ`debug`,` info` はどこへ…。


```python
from logging import getLogger, DEBUG

logger = getLogger(__name__)

# これかな?
logger.setLevel(DEBUG)


def main():
    logger.debug('This is a Debug message')
    logger.info('This is a Info message')
    logger.warning('This is a Warning message')
    logger.error('This is a Error message')
    logger.critical('This is a Critical message')
    
if __name__ == '__main__':
    main()
```

`logger` には `setLevel` という、どのレベル以上のものを表示するかという設定があります。
これで `info` や `debug` が無視されていたに違いないので、早速 `DEBUG` 以上をセットしてみましょう。

>This is a Warning message  
>This is a Error message  
>This is a Critical message

ぐぬぬ…


```python
import sys
from logging import getLogger, DEBUG, StreamHandler

logger = getLogger(__name__)
logger.setLevel(DEBUG)

#  これも？
handler = StreamHandler(sys.stderr)
handler.setLevel(DEBUG)
logger.addHandler(handler)


def main():
    logger.debug('This is a Debug message')
    logger.info('This is a Info message')
    logger.warning('This is a Warning message')
    logger.error('This is a Error message')
    logger.critical('This is a Critical message')
    
if __name__ == '__main__':
    main()
```

Python の Logger には Handler という概念があります。
Handler というのは書き込まれたログをどう扱うかを決めるためのものです。
デフォルトだと標準エラー出力にログが表示されるだけですが、これをカスタマイズすることで、ファイルに書き込むようにしたり、出力のフォーマットや要素を好きなように変更することが出来ます。
この Handler も ログレベルのしきい値を持っているので、こっちのレベルも変更してみましょう。

>This is a Debug message  
>This is a Info message  
>This is a Warning message  
>This is a Error message  
>This is a Critical message  

👏  
無事全部のログが出せましたね。

## Logger について


```python
logger = getLogger(__name__)
```

とは一体何なのか


まず引数の中身から。

`__name__`

この Python 組み込みの変数には現在のモジュール名が入っています。

```
* myapp/
     * __init__.py
     * views.py
     * models/
          * __init__.py
          * items.py
```

みたいなパッケージ構成の場合、 

* `myapp/views.py` での `__name__` は `myapp.views`
* `myapp/models/items.py` での `__name__` は `myapp.models.items`

のようになります。

(但し `if __name__ == '__main__':` のイディオムからも分かるように、対象のファイルを直接実行した場合は、モジュール名ではなく `__main__` という固定値が入るようになっています。今回はややこしくなるので、モジュール名が常に入っているものとして進めさせてください。)

つまり getLogger の1文は、 `logger = getLogger('myapp.views')`  のようにモジュール名を入れて呼び出している事と等価です。
実はこの引数、文字列であればなんでもよかったりします。ロガーに名前をつけてるだけですね。


### なぜ __name__ で呼び出すか 
では、なぜ慣例的に `logger = getLogger(__name__)` のように呼び出すかというと、2つのメリットがあるからです。

1つ目はシンプルで直感的だからという点です。利用する際は毎回この定型文を挿入してあげるだけですし、正しくファイルを分割していれば、それがそのまま適切なロガー名になります。ログからどのファイルでエラーが発生したかも簡単に辿ることができます。他のパッケージのログと混同することもありません。

2つ目の理由は、**ロガーが階層関係を持てる** という点です。ロガーの名前をドットで区切るとそれが階層になります。なので、`myapp.views` と `myapp.models.items` の2つのロガーは共通の `myapp` というロガーを親に持っています。また名前無しで初期化したロガーはルートロガーと呼ばれ、全てのロガーの親となります。

```
* (ルート)
    * myapp
        * views
        * models
            * items
```

上のパッケージ構成でそれぞれが `__name__` で初期化したロガーをもつ場合、こんな感じの階層が自動的に出来上がります。

この階層関係、何が嬉しいのかというと Handler で扱う時に非常に便利になります。
自然にパッケージごとにロガーがグループ分けされますし、後述する Handler の機能を利用すればログの出し分けも自在にできるようになります。
基本のログは標準出力に吐き出すけど、このパッケージの Error 以上のログはファイルに書き出すみたいなことも可能です。

## Handler について

Handler の仕事は、渡ってきたログを適切に処理することです。ログを受取り、必要に応じてフィルターし、指定されたログ出力先に指定されたフォーマットで書き込みます。

代表的な Handler は 標準出力などのストリームに書き込む `StreamHandler` と、ファイルに書き込む `FileHandler` です。その他にも `NullHandler` `SocketHandler` `SMTPHandler` などなどあります。

HandlerがどのようにLogを処理するかはこの図をみると分かりやすいです。

<img src='https://docs.python.jp/3/_images/logging_flow.png'>
([Logging HOWTO - Python 3.6.3 ドキュメント](https://docs.python.jp/3/howto/logging.html#useful-handlers)より)

この図のポイントは 左側の "Set current logger to parent" という部分ですね。 

Logger は階層関係になっていると言いましたが、この部分で再帰的に親の Handler を呼び出しています。なので `myapp.models` 以下のエラーをファイルに書き込み、かつ `myapp` 以下のエラーを標準出力に流したい場合は、 `myapp.models` に `FileHandler` を、 `myapp` に `StreamHandler` を設定してあげれば実現可能です。

また Loggerは `propagate`(伝播) という属性を持っていて、これを `False` にセットすれば親のログをたどることを止めることが出来ます。上の例で言うと、 `myapp.models` のログをファイルだけの書き込みだけに制限できます。

### Formatter
Handler には ログをどう出力するかという Formatter を指定できます。Handler ごとに持っているので、出力先に応じた適切なフォーマットでログを返せます。
https://docs.python.jp/3/library/logging.html#logrecord-attributes
使える変数はこちら。

## スッキリ書く



```python
import sys
from logging import getLogger, DEBUG, ERROR, StreamHandler, FileHandler, NullHandler, Formatter

# フォーマッターを定義
simple_formatter = Formatter('%(asctime)s: %(message)s')

# 標準エラー出力に出すためのハンドラを定義
console_handler = StreamHandler(sys.stderr)
console_handler.setLevel(ERROR)
console_handler.setFormatter(simple_formatter)

# 自作アプリ用に全てのログをファイルに書き出すようハンドラを定義
myapp_file_handler = FileHandler('myapp.log')
myapp_file_handler.setLevel(DEBUG)

# ルートロガーにセット
root_logger = getLogger()
root_logger.addHandler(console_handler)

# 自作アプリのパッケージのロガーにセット
myapp_logger = getLogger('myapp')
myapp_logger.setLevel(DEBUG)
myapp_logger.addHandler(myapp_file_handler)

# hoge パッケージのログは全て出さない
hoge_handler = getLogger('hoge')
hoge_handler.addHandler(NullHandler)
hoge_handler.propergate = False
```

ログの設定を書き連ねると結構長くなってしまいます。

ログはエントリポイントなど最初の方で設定してあげる必要があるので、これらをコードで挿入すると見通しが悪くなってしまいます。
別ファイルにコードをまとめて import する方法もありますが、もうちょっとスマートなやり方があります。

### 設定ファイルでログの設定をする

```
{
    "version": 1,
    "formatters": {
        "simple": {
            "format": "%(asctime)s: %(message)s"
        }
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "ERROR",
            "formatter": "simple",
            "stream": "ext://sys.stderr"
        },
        "myapp_file": {
            "class": "logging.FileHandler",
            "level": "DEBUG",
            "filename": "myapp.log"
        },
        "blackhole": {
            "class": "logging.NullHandler"
        }
    },
    "loggers": {
        "myapp": {
            "level": "DEBUG",
            "handlers": [
                "myapp_file"
            ]
        },
        "hoge": {
            "handlers": [
                "blackhole"
            ],
            "propagate": false
        }
    },
    "root": {
        "handlers": [
            "console"
        ]
    }
}
 ```
 
 このような JSON ファイルを用意して、


```python
import json
from logging.config import dictConfig

# 設定ファイルから読み込む
with open('logging.json') as f:
    dictConfig(json.load(f))
```

のようにすれば上の Python のコードと同じ設定が可能です。(ルートのロガーだけ、`loggers` の外で定義している点に注意してください)

JSON だと改行や綴じカッコの分長くなってしまいますが、コードと設定を分離するという意味ではこちらの形式を利用したほうが好ましいです。

ちなみに dictConfig の引数は Python の辞書オブジェクトなので、JSON以外の形式でも定義可能です。
公式ドキュメントでは YAML の例が使われていますね。 Python のようにインデントで制御しているので、かなり簡潔に書く事ができます。
ただ YAML のパーサーは Python 標準で付属していないため、 pip 等からインストールする必要があります。

`fileConfig` という confファイル形式で定義する方法も用意されていますが、dictConfig の方が新しく細かい設定も出来るため、こちらのほうが推奨されているみたいです。


```python
import json
from logging.config import dictConfig

with open('logging.json') as f:
    dictConfig(json.load(f))


# 動作確認
from logging import getLogger

# RootLogger: error のみが標準エラーに
root_logger = getLogger()
root_logger.debug('RootLogger: debug')
root_logger.error('RootLogger: error')

# myapp: debug は指定したファイルに
# myapp: error は指定したファイルと標準エラーに
myapp_logger = getLogger('myapp.test')
myapp_logger.debug('myapp: debug')
myapp_logger.error('myapp: error')

# これはすべて無視される
hoge_logger = getLogger('hoge.fuga.piyo')
hoge_logger.debug('hoge: debug')
hoge_logger.error('hoge: error')
```

    2017-10-25 15:29:39,322: RootLogger: error
    2017-10-25 15:29:39,324: myapp: error


最後に動作確認例を載せておきます。

この挙動を理解できれば思い通りのログ処理ができるようになるのではないでしょうか。



# 参考

 Logging HOWTO - Python3 公式ドキュメント
https://docs.python.jp/3/howto/logging.html  

 
ログ出力のための print と import logging はやめてほしい - Qiita
https://qiita.com/amedama/items/b856b2f30c2f38665701 
