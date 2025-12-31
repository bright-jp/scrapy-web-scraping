# Scrapy を使った Webスクレイピング

[![Bright Data Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/)

このガイドでは、Webスクレイピングの実用的な適用例として、保護者によくある課題である「学校から送られてくる情報の収集と整理」に取り組みます。具体的には、宿題の詳細と給食（食事）メニューの収集に焦点を当てます。

以下は、構築するものの全体アーキテクチャを示す図です。

![Diagram of the final project](https://github.com/luminati-io/scrapy-web-scraping/blob/main/images/diagram-of-the-final-project.png)

計画は次のとおりです。

- [プロジェクトのセットアップ](#setting-up-the-project)
- [宿題 Spider の開発](#developing-the-homework-spider)
- [食事 Spider の開発](#developing-the-meal-spider)
- [データの整形](#data-formatting)
- [Webスクレイピングの課題への対応](#handling-web-scraping-challenges)

## 要件

このチュートリアルを開始する前に、次のものが揃っていることを確認してください。

- [Python 3.10+](https://www.python.org/downloads/) がインストールされていること
- [有効化された Python 仮想環境](https://docs.python.org/3/library/venv.html)
- pip 経由で [Scrapy 2.11.1+](https://docs.scrapy.org/en/latest/intro/install.html#intro-install) がインストールされていること
- コードエディタ（[VS Code](https://code.visualstudio.com/) や [PyCharm](https://www.jetbrains.com/pycharm/) など）

プライバシーの観点から、このデモ用学校 Web サイトを使用します: [https://systemcraftsman.github.io/scrapy-demo/website/](https://systemcraftsman.github.io/scrapy-demo/website/)

## Setting Up the Project

まず、プロジェクトディレクトリを作成します。

```sh
mkdir school-scraper
```

このディレクトリへ移動し、新しい Scrapy プロジェクトを初期化します。

```sh
cd school-scraper & \
scrapy startproject school_scraper
```

これにより、次のようなフォルダ構成が生成されます。

```
school-scraper
└── school_scraper
    ├── school_scraper
    │   ├── __init__.py
    │   ├── items.py
    │   ├── middlewares.py
    │   ├── pipelines.py
    │   ├── settings.py
    │   └── spiders
    │       └── __init__.py
    └── scrapy.cfg
```

このコマンドは、`school_scraper` ディレクトリを 2 重に作成します。内側のディレクトリでは、Scrapy がいくつかの重要ファイルを生成します。
- Scrapy の [middlewares](https://docs.scrapy.org/en/latest/topics/spider-middleware.html) を定義するための `middlewares.py`
- カスタムデータ処理パイプラインを作成するための `pipelines.py`
- スクレイピングアプリケーションを設定するための `settings.py`
- Spider クラスを配置する `spiders` フォルダ

現在、`spiders` フォルダは空ですが、次にここへ追加していきます。

## Developing the Homework Spider

宿題情報を収集するには、学校システムへログインし、宿題課題ページへ移動する Spider が必要です。

![diagram showing the process of creating a spider and navigating to scrape the data](https://github.com/luminati-io/scrapy-web-scraping/blob/main/images/diagram-showing-the-process-of-creating-a-spider-and-navigating-to-scrape-the-data.png)

Scrapy CLI を使って Spider を生成します。`school-scraper/school_scraper` ディレクトリから次を実行してください。

```sh
scrapy genspider homework_spider systemcraftsman.github.io/scrapy-demo/website/index.html
```

> **REMINDER:** すべての Python および Scrapy コマンドは、有効化した仮想環境内で実行してください。

次のような出力が表示されるはずです。

```
Created spider 'homework_spider' using template 'basic' in module:
  school_scraper.spiders.homework_spider
```

これにより、`school_scraper/spiders` ディレクトリに `homework_spider.py` が作成され、内容は次のようになります。

```python
class HomeworkSpiderSpider(scrapy.Spider):
    name = "homework_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]

    def parse(self, response):
        pass
```

クラス名を `HomeworkSpider` に変更して、名前に含まれる冗長な `Spider` を削除してください。`parse` メソッドはスクレイピングのエントリポイントで、ログインに使用します。

> **NOTE:**
> 
> `https://systemcraftsman.github.io/scrapy-demo/index.html` のログインフォームは JavaScript でシミュレートされています。静的な HTML ページであるため、HTTP GET リクエストを使用してログインプロセスをシミュレートします。

`parse` メソッドを次のように更新します。

```python
...code omitted...
    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)
```

これにより、ログインするためのフォームリクエストが送信され、`after_login` コールバックを伴って `welcome_page_url` へリダイレクトされ、スクレイピングを継続します。

クラス変数に `welcome_page_url` を追加します。

```python
...code omitted...
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
...code omitted...
```

次に、`parse` メソッドの後に `after_login` メソッドを追加します。

```python
...code omitted...
    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.homework_page_url,
                           callback=self.parse_homework_page
                           )
...code omitted...
```

このメソッドはレスポンスが成功（ステータス 200）かどうかを確認し、その後宿題ページへ移動して `parse_homework_page` メソッドを呼び出します。

クラス変数に `homework_page_url` を追加します。

```python
...code omitted...
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
...code omitted...
```

次に、`after_login` の後に `parse_homework_page` メソッドを追加します。

```python
...code omitted...
    def parse_homework_page(self, response):
        if response.status == 200:
            data = {}
            rows = response.xpath('//*[@class="table table-file"]//tbody//tr')
            for row in rows:
                if self._get_item(row, 4) == self.date_str:
                    if self._get_item(row, 2) not in data:
                        data[self._get_item(row, 2)] = self._get_item(row, 3)
                    else:
                        data[self._get_item(row, 2) + "-2"] = self._get_item(row, 3)
            return data
...code omitted...
```

このメソッドはレスポンスステータスが 200 であることを確認し、HTML テーブルから宿題データを抽出します。XPath を使用して行を取得し、ヘルパーメソッド `_get_item` で特定のデータを抽出します。

クラスに `_get_item` ヘルパーメソッドを追加します。

```python
...code omitted...
    def _get_item(self, row, col_number):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for content in contents:
            item_str = item_str + content
        return item_str
```

このヘルパーメソッドは、行と列のインデックスを用いた XPath でセル内容を抽出します。セル内に複数のテキストノードがある場合は、それらを連結します。

また、`parse_homework_page` メソッドには `date_str` 変数が必要です。静的なデモサイトのデータに合わせて `12.03.2024` に設定します。

> **NOTE:**
> 
> 実際のアプリケーションでは、この日付を動的に生成するのが一般的です。

クラス変数に `date_str` を追加します。

```python
...code omitted...
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
    date_str = "12.03.2024"
...code omitted...
```

これで、完成した `homework_spider.py` は次のようになります。

```python
import scrapy

from scrapy import FormRequest, Request


class HomeworkSpider(scrapy.Spider):
    name = "homework_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
    date_str = "12.03.2024"

    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)

    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.homework_page_url,
                           callback=self.parse_homework_page
                           )

    def parse_homework_page(self, response):
        if response.status == 200:
            data = {}
            rows = response.xpath('//*[@class="table table-file"]//tbody//tr')
            for row in rows:
                if self._get_item(row, 4) == self.date_str:
                    if self._get_item(row, 2) not in data:
                        data[self._get_item(row, 2)] = self._get_item(row, 3)
                    else:
                        data[self._get_item(row, 2) + "-2"] = self._get_item(row, 3)
            return data

    def _get_item(self, row, col_number):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for content in contents:
            item_str = item_str + content
        return item_str
```

次を実行して Spider をテストします。

```
scrapy crawl homework_spider
```

次のような出力が表示されるはずです。

```
...output omitted...
2024-03-20 01:36:05 [scrapy.core.scraper] DEBUG: Scraped from <200 https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html>
{'MATHS': "Matematik Konu Anlatımlı Çalışma Defteri-6 sayfa 13'ü yapınız.\n", 'ENGLISH': 'Read the story "Manny and His Monster Manners" on pages 100-107 in your Reading Log and complete the activities on pages 108 and 109 according to the story.\n\nReading Log kitabınızın 100-107 sayfalarındaki "Manny and His Monster Manners" isimli hikayeyi okuyunuz ve 108 ve 109\'uncu sayfalarındaki aktiviteleri hikayeye göre tamamlayınız.\n'}
2024-03-20 01:36:05 [scrapy.core.engine] INFO: Closing spider (finished)
...output omitted...
```

## Developing the Meal Spider

食事リストページ用に別の Spider を作成します。

```sh
scrapy genspider meal_spider systemcraftsman.github.io/scrapy-demo/website/index.html
```

これにより、次の内容の新しいファイルが生成されます。

```python
class MealSpiderSpider(scrapy.Spider):
    name = "meal_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]

    def parse(self, response):
        pass
```

食事 Spider は宿題 Spider と同様のパターンに従いますが、主な違いは HTML のパースロジックです。

`meal_spider.py` の内容を次に置き換えてください。

```python
import scrapy

from datetime import datetime

from scrapy import FormRequest, Request


class MealSpider(scrapy.Spider):
    name = "meal_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    meal_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/meal-list.html"
    date_str = "13.03.2024"

    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)

    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.meal_page_url,
                           callback=self.parse_meal_page
                           )

    def parse_meal_page(self, response):
        if response.status == 200:
            data = {"BREAKFAST": "", "LUNCH": "", "SALAD/DESSERT": "", "FRUIT TIME": ""}
            week_no = datetime.strptime(self.date_str, '%d.%m.%Y').isoweekday()
            rows = response.xpath('//*[@class="table table-condensed table-yemek-listesi"]//tr')
            key = ""
            try:
                for row in rows[1:]:
                    if self._get_item(row, week_no) in data.keys():
                        key = self._get_item(row, week_no)
                    else:
                        data[key] = self._get_item(row, week_no, "\n")
            finally:
                return data

    def _get_item(self, row, col_number, seperator=""):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for i, content in enumerate(contents):
            item_str = item_str + content + seperator
        return item_str
```

`parse` と `after_login` メソッドは宿題 Spider のものとほぼ同一である点に注意してください。主な違いは `parse_meal_page` メソッドにあり、異なる XPath 式とデータ処理ロジックを使用して食事情報を抽出します。

次で食事 Spider をテストします。

```sh
scrapy crawl meal_spider
```

次のように表示されるはずです。

```
...output omitted...
2024-03-20 02:44:42 [scrapy.core.scraper] DEBUG: Scraped from <200 https://systemcraftsman.github.io/scrapy-demo/website/meal-list.html>
{'BREAKFAST': 'PANCAKE \n KREM PEYNİR \n SÜZME PEYNİR \n\nKAKAOLU FINDIK KREMASI \n SÜT\n', 'LUNCH': 'TARHANA ÇORBA\nEKŞİLİ KÖFTE\nERİŞTE\n', 'SALAD/DESSERT': 'AYRAN\nKIRMIZILAHANA SALATA\nROKALI GÖBEK SALATA\n', 'FRUIT TIME': 'FINDIK& KURU ÜZÜM\n'}
2024-03-20 02:44:42 [scrapy.core.engine] INFO: Closing spider (finished)
...output omitted...
```

## Data Formatting

Spider で JSON 形式のデータを抽出できるようになったので、Spider をプログラムから実行して、より見やすく整形したくなるかもしれません。

通常、Scrapy プロジェクトでは `main.py` のようなエントリポイントは不要です（Scrapy CLI が Spider 実行の枠組みを提供するため）が、出力データを整形するために作成します。

`school-scraper` プロジェクトのルートディレクトリに `main.py` という名前のファイルを作成し、次を記述します。

```python
import sys

from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

from school_scraper.school_scraper.spiders.homework_spider import HomeworkSpider
from school_scraper.school_scraper.spiders.meal_spider import MealSpider

results = []


class ResultsPipeline(object):
    def process_item(self, item, spider):
        results.append(item)


def _prepare_message(title, data_dict):
    if len(data_dict.items()) == 0:
        return None

    message = f"===={title}====\n----------------\n"
    for key, value in data_dict.items():
        message = message + f"==={key}===\n{value}\n----------------\n"

    return message


def main(args=None):
    if args is None:
        args = sys.argv
    settings = get_project_settings()
    settings.set("ITEM_PIPELINES", {'__main__.ResultsPipeline': 1})
    process = CrawlerProcess(settings)

    if args[1] == "homework":
        process.crawl(HomeworkSpider)
        process.start()
        print(_prepare_message("HOMEWORK ASSIGNMENTS", results[0]))
    elif args[1] == "meal":
        process.crawl(MealSpider)
        process.start()
        print(_prepare_message("MEAL LIST", results[0]))


if __name__ == "__main__":
    main()
```

このスクリプトはアプリケーションのエントリポイントを提供します。`main` 関数はコマンドライン引数を処理して、どの Spider を実行するかを決定します。

Spider の出力を `results` 配列に収集するカスタム `ResultsPipeline` を設定します。`_prepare_message` ヘルパー関数は、表示用にこのデータを整形します。

渡された引数（"homework" または "meal"）に応じて、スクリプトは適切な Spider を実行し、その出力を整形します。

宿題課題を取得するには次を実行します。

```sh
python main.py homework
```

次のように表示されるはずです。

```
...output omitted...
====HOMEWORK ASSIGNMENTS====
----------------
===MATHS===
Matematik Konu Anlatımlı Çalışma Defteri-6 sayfa 13'ü yapınız.

----------------
===ENGLISH===
Read the story "Manny and His Monster Manners" on pages 100-107 in your Reading Log and complete the activities on pages 108 and 109 according to the story.

Reading Log kitabınızın 100-107 sayfalarındaki "Manny and His Monster Manners" isimli hikayeyi okuyunuz ve 108 ve 109'uncu sayfalarındaki aktiviteleri hikayeye göre tamamlayınız.

----------------
...output omitted...
```

食事リストの場合は次です。

```sh
python main.py meal
```

出力:

```
...output omitted...
====MEAL LIST====
----------------
===BREAKFAST===
PANCAKE 
 KREM PEYNİR 
 SÜZME PEYNİR 

KAKAOLU FINDIK KREMASI 
 SÜT

----------------
===LUNCH===
TARHANA ÇORBA
EKŞİLİ KÖFTE
ERİŞTE

----------------
===SALAD/DESSERT===
AYRAN
KIRMIZILAHANA SALATA
ROKALI GÖBEK SALATA

----------------
===FRUIT TIME===
FINDIK& KURU ÜZÜM

----------------

...output omitted...
```

## Handling Web Scraping Challenges

Scrapy により Webスクレイピングは比較的簡単になりますが、実運用のアプリケーションではさまざまな課題に直面することがあります。ここでは、一般的な障害に対処するためのヒントを紹介します。

### Dynamic Websites

Dynamic な Web サイトは、場所、デバイス、または設定などの要因に基づいて、ユーザーに異なるコンテンツを提供します。Scrapy でも Dynamic な Web サイトを扱えますが、この目的に特化して設計されているわけではありません。

Dynamic なコンテンツの場合は、Scrapy Spider を定期的に実行するようスケジューリングし、時間経過に伴う変更を追跡する仕組みを実装することを検討してください。場合によっては、更新頻度が低い Dynamic なコンテンツを、実質的に静的コンテンツとして扱えることもあります。

### CAPTCHAs

[CAPTCHAs](https://brightdata.jp/blog/web-data/what-is-a-captcha) は、歪んだテキストや画像を表示し、人間が識別して進む必要があるセキュリティ機構です。自動化されたスクレイピングツールをブロックするために特別に設計されています。

この例では CAPTCHA は使用していませんが、実運用では遭遇する可能性があります。1 つのアプローチとして、CAPTCHA 画像をダウンロードし、OCR（Optical Character Recognition）ライブラリでテキストに変換する Scrapy middleware を作成する方法があります。

### Session and Cookie Management

Web アプリケーションは、ユーザーがサイト内を移動する際の状態情報を維持するためにセッションを使用します。Cookie はユーザーの端末に情報を保存し、Web サイトが後続の訪問時に参照できるようにします。

スクレイピング中にセッション状態を維持したり、Cookie を操作したりする必要がある場合があります。Scrapy には Cookie 処理とセッション管理のための組み込みサポートがあり、サードパーティライブラリを通じて追加機能も利用できます。

### IP Blocking

多くの Web サイトは、単一の送信元からの過剰なリクエストを防ぐために [IP blocking](https://brightdata.jp/blog/proxy-101/how-to-bypass-an-ip-ban) を実装しています。サイトが（スクレイピングボットのような）異常なリクエストパターンを検知すると、IPアドレスを一時的または恒久的にブロックすることがあります。

IP blocking に遭遇した場合は、複数の IP にリクエストを分散するために [rotating IP addresses](https://brightdata.jp/solutions/rotating-proxies) の利用を検討してください。

## Summary

Webスクレイピング能力を強化したい方に向けて、Bright Data は公開 Web データ収集のために特別に設計されたソリューションを提供しています。同社の [Scrapy integration](https://brightdata.jp//integration/scrapy) は、プロキシサービスを利用して IP ban の回避を支援し、Scrapy の機能を拡張します。

本日サインアップして、スクレイピングのニーズについて同社のデータエキスパートにご相談ください。