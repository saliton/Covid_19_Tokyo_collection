# 東京都のページのPDFファイルから市区町村別の新型コロナ感染者数を抽出してみた

データ解析の初手はデータの収集です。この記事では具体的なデータの収集手段をひとつ提示します。

昨今（2021年6月現在）の新型コロナウィルスの流行により、東京都の感染者数が市区町村別に毎日発表されています。都道府県別の感染者数は報道もされますが、東京都の市区町村別の集計はこの発表でしかお目にかかれません。そこで、この情報を取得して加工可能なデータとして抽出します。感染者数はPDFファイルとして提示されているので、PDFファイルからデータを取り出します。

まず、「[東京都新型コロナウイルス感染症対策本部報](https://www.bousai.metro.tokyo.lg.jp/taisaku/saigai/1010035/index.html)」のページを取得してBeautifulSoupで解析します。


```
from urllib import request
from bs4 import BeautifulSoup

url_base = 'https://www.bousai.metro.tokyo.lg.jp/taisaku/saigai/1010035/'
with request.urlopen(url_base + 'index.html') as response:
    soup = BeautifulSoup(response, 'html.parser')
```

このページの配下のページで「患者の発生」が含まれるリンクの先のページに感染者数が載っています。最初のリンクを表示してみましょう。


```
import os.path

for base_link in soup.select('a'):
    if 'href' in base_link.attrs and base_link.attrs['href'].find('../../../taisaku/saigai/1010035') == 0:
        with request.urlopen(url_base + base_link.attrs['href']) as response:
            for page_link in BeautifulSoup(response, 'html.parser').select('a'):
                if page_link.text.find('患者の発生') >= 0:
                    page_link = os.path.join(os.path.dirname(base_link.attrs['href']), page_link.attrs['href'])
                    print(page_link)
                    break
            else:
                continue
            break
```

    ../../../taisaku/saigai/1010035/1013932/../../../../taisaku/saigai/1010035/1013932/1014017.html


このリンク先のページを取得してみます。


```
with request.urlopen(url_base + page_link) as response:
    page = BeautifulSoup(response)
```

このページのリリース日時が\<div class="releasedate"\>の中の\<p\>に書いてあります。


```
release = page.select_one('div[class=releasedate]').select_one('p').text
print(release)
```

    令和3年6月15日 16時45分


この文字列を解析するのにparseモジュールを使ってみましょう。


```
!pip install parse
```

    Collecting parse
      Downloading https://files.pythonhosted.org/packages/89/a1/82ce536be577ba09d4dcee45db58423a180873ad38a2d014d26ab7b7cb8a/parse-1.19.0.tar.gz
    Building wheels for collected packages: parse
      Building wheel for parse (setup.py) ... [?25l[?25hdone
      Created wheel for parse: filename=parse-1.19.0-cp37-none-any.whl size=24592 sha256=5939de41da0df73fc818b581bb1a3934d621028a05038ef3abdca9f3c21fd79b
      Stored in directory: /root/.cache/pip/wheels/c0/39/ea/e2fd678bd130953f5438470b8dfa529f00787e9b8b92b27467
    Successfully built parse
    Installing collected packages: parse
    Successfully installed parse-1.19.0



```
from parse import parse
from datetime import datetime

parsed = parse('令和{:d}年{:d}月{:d}日 {:d}時{:d}分', release)
print(parsed)
release_date = datetime(parsed[0] + 2018, parsed[1], parsed[2])
print(release_date)
```

    <Result (3, 6, 15, 16, 45) {}>
    2021-06-15 00:00:00


リンク先がPDFファイルのリンクを探します。


```
import os

for pdf_link in page.select('a'):
    if 'href' in pdf_link.attrs and pdf_link.attrs['href'].find('.pdf') >= 0:
        break

pdf_href = os.path.join(os.path.dirname(page_link), pdf_link.attrs['href'])
print(pdf_href)
```

    ../../../taisaku/saigai/1010035/1013932/../../../../taisaku/saigai/1010035/1013932/../../../../_res/projects/default_project/_page_/001/014/017/20210615aa.pdf


このPDFファイルの中からテーブルを読むのにcamelotモジュールを使います。まず、必要なモジュールをインストールしましょう。


```
!pip install camelot-py
!apt install ghostscript
```

    Collecting camelot-py
    [?25l  Downloading https://files.pythonhosted.org/packages/f5/0f/6e0425aa3aa0411b6e384ec4f92b729880d9a4832497aa0f309b37318d11/camelot_py-0.9.0-py3-none-any.whl (43kB)
    [K     |████████████████████████████████| 51kB 5.9MB/s 
    [?25hRequirement already satisfied: chardet>=3.0.4 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (3.0.4)
    Requirement already satisfied: click>=6.7 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (7.1.2)
    Collecting pdfminer.six>=20200726
    [?25l  Downloading https://files.pythonhosted.org/packages/93/f3/4fec7dabe8802ebec46141345bf714cd1fc7d93cb74ddde917e4b6d97d88/pdfminer.six-20201018-py3-none-any.whl (5.6MB)
    [K     |████████████████████████████████| 5.6MB 24.7MB/s 
    [?25hRequirement already satisfied: pandas>=0.23.4 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (1.1.5)
    Requirement already satisfied: openpyxl>=2.5.8 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (2.5.9)
    Requirement already satisfied: numpy>=1.13.3 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (1.19.5)
    Collecting PyPDF2>=1.26.0
    [?25l  Downloading https://files.pythonhosted.org/packages/b4/01/68fcc0d43daf4c6bdbc6b33cc3f77bda531c86b174cac56ef0ffdb96faab/PyPDF2-1.26.0.tar.gz (77kB)
    [K     |████████████████████████████████| 81kB 8.2MB/s 
    [?25hCollecting cryptography
    [?25l  Downloading https://files.pythonhosted.org/packages/b2/26/7af637e6a7e87258b963f1731c5982fb31cd507f0d90d91836e446955d02/cryptography-3.4.7-cp36-abi3-manylinux2014_x86_64.whl (3.2MB)
    [K     |████████████████████████████████| 3.2MB 32.9MB/s 
    [?25hRequirement already satisfied: sortedcontainers in /usr/local/lib/python3.7/dist-packages (from pdfminer.six>=20200726->camelot-py) (2.4.0)
    Requirement already satisfied: python-dateutil>=2.7.3 in /usr/local/lib/python3.7/dist-packages (from pandas>=0.23.4->camelot-py) (2.8.1)
    Requirement already satisfied: pytz>=2017.2 in /usr/local/lib/python3.7/dist-packages (from pandas>=0.23.4->camelot-py) (2018.9)
    Requirement already satisfied: jdcal in /usr/local/lib/python3.7/dist-packages (from openpyxl>=2.5.8->camelot-py) (1.4.1)
    Requirement already satisfied: et-xmlfile in /usr/local/lib/python3.7/dist-packages (from openpyxl>=2.5.8->camelot-py) (1.1.0)
    Requirement already satisfied: cffi>=1.12 in /usr/local/lib/python3.7/dist-packages (from cryptography->pdfminer.six>=20200726->camelot-py) (1.14.5)
    Requirement already satisfied: six>=1.5 in /usr/local/lib/python3.7/dist-packages (from python-dateutil>=2.7.3->pandas>=0.23.4->camelot-py) (1.15.0)
    Requirement already satisfied: pycparser in /usr/local/lib/python3.7/dist-packages (from cffi>=1.12->cryptography->pdfminer.six>=20200726->camelot-py) (2.20)
    Building wheels for collected packages: PyPDF2
      Building wheel for PyPDF2 (setup.py) ... [?25l[?25hdone
      Created wheel for PyPDF2: filename=PyPDF2-1.26.0-cp37-none-any.whl size=61102 sha256=0e19f2f5ba764da31e0b6ecefb369e9838cec1776d4c7cc8c459d3125cbdca88
      Stored in directory: /root/.cache/pip/wheels/53/84/19/35bc977c8bf5f0c23a8a011aa958acd4da4bbd7a229315c1b7
    Successfully built PyPDF2
    Installing collected packages: cryptography, pdfminer.six, PyPDF2, camelot-py
    Successfully installed PyPDF2-1.26.0 camelot-py-0.9.0 cryptography-3.4.7 pdfminer.six-20201018
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    The following additional packages will be installed:
      fonts-droid-fallback fonts-noto-mono gsfonts libcupsfilters1 libcupsimage2
      libgs9 libgs9-common libijs-0.35 libjbig2dec0 poppler-data
    Suggested packages:
      fonts-noto ghostscript-x poppler-utils fonts-japanese-mincho
      | fonts-ipafont-mincho fonts-japanese-gothic | fonts-ipafont-gothic
      fonts-arphic-ukai fonts-arphic-uming fonts-nanum
    The following NEW packages will be installed:
      fonts-droid-fallback fonts-noto-mono ghostscript gsfonts libcupsfilters1
      libcupsimage2 libgs9 libgs9-common libijs-0.35 libjbig2dec0 poppler-data
    0 upgraded, 11 newly installed, 0 to remove and 39 not upgraded.
    Need to get 14.1 MB of archives.
    After this operation, 49.9 MB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu bionic/main amd64 fonts-droid-fallback all 1:6.0.1r16-1.1 [1,805 kB]
    Get:2 http://archive.ubuntu.com/ubuntu bionic/main amd64 poppler-data all 0.4.8-2 [1,479 kB]
    Get:3 http://archive.ubuntu.com/ubuntu bionic/main amd64 fonts-noto-mono all 20171026-2 [75.5 kB]
    Get:4 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 libcupsimage2 amd64 2.2.7-1ubuntu2.8 [18.6 kB]
    Get:5 http://archive.ubuntu.com/ubuntu bionic/main amd64 libijs-0.35 amd64 0.35-13 [15.5 kB]
    Get:6 http://archive.ubuntu.com/ubuntu bionic/main amd64 libjbig2dec0 amd64 0.13-6 [55.9 kB]
    Get:7 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 libgs9-common all 9.26~dfsg+0-0ubuntu0.18.04.14 [5,092 kB]
    Get:8 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 libgs9 amd64 9.26~dfsg+0-0ubuntu0.18.04.14 [2,265 kB]
    Get:9 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 ghostscript amd64 9.26~dfsg+0-0ubuntu0.18.04.14 [51.3 kB]
    Get:10 http://archive.ubuntu.com/ubuntu bionic/main amd64 gsfonts all 1:8.11+urwcyr1.0.7~pre44-4.4 [3,120 kB]
    Get:11 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 libcupsfilters1 amd64 1.20.2-0ubuntu3.1 [108 kB]
    Fetched 14.1 MB in 1s (12.1 MB/s)
    Selecting previously unselected package fonts-droid-fallback.
    (Reading database ... 160772 files and directories currently installed.)
    Preparing to unpack .../00-fonts-droid-fallback_1%3a6.0.1r16-1.1_all.deb ...
    Unpacking fonts-droid-fallback (1:6.0.1r16-1.1) ...
    Selecting previously unselected package poppler-data.
    Preparing to unpack .../01-poppler-data_0.4.8-2_all.deb ...
    Unpacking poppler-data (0.4.8-2) ...
    Selecting previously unselected package fonts-noto-mono.
    Preparing to unpack .../02-fonts-noto-mono_20171026-2_all.deb ...
    Unpacking fonts-noto-mono (20171026-2) ...
    Selecting previously unselected package libcupsimage2:amd64.
    Preparing to unpack .../03-libcupsimage2_2.2.7-1ubuntu2.8_amd64.deb ...
    Unpacking libcupsimage2:amd64 (2.2.7-1ubuntu2.8) ...
    Selecting previously unselected package libijs-0.35:amd64.
    Preparing to unpack .../04-libijs-0.35_0.35-13_amd64.deb ...
    Unpacking libijs-0.35:amd64 (0.35-13) ...
    Selecting previously unselected package libjbig2dec0:amd64.
    Preparing to unpack .../05-libjbig2dec0_0.13-6_amd64.deb ...
    Unpacking libjbig2dec0:amd64 (0.13-6) ...
    Selecting previously unselected package libgs9-common.
    Preparing to unpack .../06-libgs9-common_9.26~dfsg+0-0ubuntu0.18.04.14_all.deb ...
    Unpacking libgs9-common (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Selecting previously unselected package libgs9:amd64.
    Preparing to unpack .../07-libgs9_9.26~dfsg+0-0ubuntu0.18.04.14_amd64.deb ...
    Unpacking libgs9:amd64 (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Selecting previously unselected package ghostscript.
    Preparing to unpack .../08-ghostscript_9.26~dfsg+0-0ubuntu0.18.04.14_amd64.deb ...
    Unpacking ghostscript (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Selecting previously unselected package gsfonts.
    Preparing to unpack .../09-gsfonts_1%3a8.11+urwcyr1.0.7~pre44-4.4_all.deb ...
    Unpacking gsfonts (1:8.11+urwcyr1.0.7~pre44-4.4) ...
    Selecting previously unselected package libcupsfilters1:amd64.
    Preparing to unpack .../10-libcupsfilters1_1.20.2-0ubuntu3.1_amd64.deb ...
    Unpacking libcupsfilters1:amd64 (1.20.2-0ubuntu3.1) ...
    Setting up libgs9-common (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Setting up fonts-droid-fallback (1:6.0.1r16-1.1) ...
    Setting up gsfonts (1:8.11+urwcyr1.0.7~pre44-4.4) ...
    Setting up poppler-data (0.4.8-2) ...
    Setting up fonts-noto-mono (20171026-2) ...
    Setting up libcupsfilters1:amd64 (1.20.2-0ubuntu3.1) ...
    Setting up libcupsimage2:amd64 (2.2.7-1ubuntu2.8) ...
    Setting up libjbig2dec0:amd64 (0.13-6) ...
    Setting up libijs-0.35:amd64 (0.35-13) ...
    Setting up libgs9:amd64 (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Setting up ghostscript (9.26~dfsg+0-0ubuntu0.18.04.14) ...
    Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
    Processing triggers for fontconfig (2.12.6-0ubuntu2) ...
    Processing triggers for libc-bin (2.27-3ubuntu1.2) ...
    /sbin/ldconfig.real: /usr/local/lib/python3.7/dist-packages/ideep4py/lib/libmkldnn.so.0 is not a symbolic link
    


PDFファイルを読み込みます。複数のテーブルが読み込まれた場合、最後のテーブルが市区町村別感染者数です。


```
from camelot import read_pdf
from pprint import pprint
tables = read_pdf(url_base + pdf_href)
table = tables[-1]
pprint(table.data)
```

    [['千代田', '中央', '港', '新宿', '文京', '台東', '墨田', '江東', '品川', '目黒', '大田'],
     ['870 (841)',
      '(2634)\n2698',
      '(5361)\n5510',
      '(8902)\n9042',
      '(2379)\n2477',
      '(2959)\n3030',
      '(3220)\n3335',
      '(5306)\n5495',
      '(4713)\n4872',
      '(4439)\n4549',
      '(7874)\n8080'],
     ['世田谷', '渋谷', '中野', '杉並', '豊島', '北', '荒川', '板橋', '練馬', '足立', '葛飾'],
     ['(11606)\n12081',
      '(4653)\n4789',
      '(5244)\n5382',
      '(6452)\n6599',
      '(4263)\n4401',
      '(3637)\n3746',
      '(2609)\n2682',
      '(6044)\n6189',
      '(7142)\n7305',
      '(7422)\n7607',
      '(5484)\n5602'],
     ['江戸川', '八王子', '立川', '武蔵野', '三鷹', '青梅', '府中', '昭島', '調布', '町田', '小金井'],
     ['(6895)\n7044',
      '(3961)\n4043',
      '(1304)\n1366',
      '(1269)\n1309',
      '1555',
      '(1499) 799 (787)',
      '1914',
      '(1830) 804 (783)',
      '(1881)\n1958',
      '2693',
      '(2622) 959 (928)'],
     ['小平', '日野', '東村山', '国分寺', '国立', '福生', '狛江', '東大和', '清瀬', '東久留米', '武蔵村山'],
     ['(1117)\n1141',
      '1168',
      '(1154) 842 (806) 830 (795) 446 (419) 485 (470) 603 (580) 464 (446) 395 '
      '(384) 662 (647) 372 (363)',
      '',
      '',
      '',
      '',
      '',
      '',
      '',
      ''],
     ['多摩', '稲城', '羽村', 'あきる野 西東京', '', '瑞穂', '日の出', '檜原', '奥多摩', '大島', '利島'],
     ['',
      '924 (894) 550 (538) 348 (345) 558 (525)',
      '',
      '',
      '1618',
      '(1543) 180 (178) 106 (101) 10 (10) 25 (25) 23 (23)',
      '',
      '',
      '',
      '',
      '(0)\n0'],
     ['新島', '神津島', '三宅', '御蔵島', '八丈', '青ヶ島', '小笠原', '都外', '調査中', '', ''],
     ['(1)\n1',
      '(1)\n1',
      '(4)\n4',
      '(2)\n2',
      '(9)\n9',
      '(0)\n0',
      '(4)\n4',
      '13946',
      '(13863) 76 (76)',
      '',
      '']]


うまく取れませんでした。どうやら文字間隔の調整が必要なようです。


```
tables = read_pdf(url_base + pdf_href, layout_kwargs = { 'char_margin': 0.01 })
table = tables[-1]
pprint(table.data)
```

    [['千代田', '中央', '港', '新宿', '文京', '台東', '墨田', '江東', '品川', '目黒', '大田'],
     ['(841)\n870',
      '(2634)\n2698',
      '(5361)\n5510',
      '(8902)\n9042',
      '(2379)\n2477',
      '(2959)\n3030',
      '(3220)\n3335',
      '(5306)\n5495',
      '(4713)\n4872',
      '(\n4439)\n4549',
      '(7874)\n8080'],
     ['世田谷', '渋谷', '中野', '杉並', '豊島', '北', '荒川', '板橋', '練馬', '足立', '葛飾'],
     ['(11606)\n12081',
      '(4653)\n4789',
      '(5244)\n5382',
      '(6452)\n6599',
      '(4263)\n4401',
      '(3637)\n3746',
      '(2609)\n2682',
      '(6044)\n6189',
      '(7142)\n7305',
      '(\n7422)\n7607',
      '(5484)\n5602'],
     ['江戸川', '八王子', '立川', '武蔵野', '三鷹', '青梅', '府中', '昭島', '調布', '町田', '小金井'],
     ['(6895)\n7044',
      '(3961)\n4043',
      '(1304)\n1366',
      '(1269)\n1309',
      '(1499)\n1555',
      '(787)\n799',
      '(1830)\n1914',
      '(783)\n804',
      '(1881)\n1958',
      '(\n2622)\n2693',
      '(928)\n959'],
     ['小平', '日野', '東村山', '国分寺', '国立', '福生', '狛江', '東大和', '清瀬', '東久留米', '武蔵村山'],
     ['(1117)\n1141',
      '(1154)\n1168',
      '(806)\n842',
      '(795)\n830',
      '(419)\n446',
      '(470)\n485',
      '(580)\n603',
      '(446)\n464',
      '(384)\n395',
      '(647)\n662',
      '(363)\n372'],
     ['多摩', '稲城', '羽村', 'あきる野', '西東京', '瑞穂', '日の出', '檜原', '奥多摩', '大島', '利島'],
     ['(894)\n924',
      '(538)\n550',
      '(345)\n348',
      '(525)\n558',
      '(1543)\n1618',
      '(178)\n180',
      '(101)\n106',
      '(10)\n10',
      '(25)\n25',
      '(\n23)\n23',
      '(0)\n0'],
     ['新島', '神津島', '三宅', '御蔵島', '八丈', '青ヶ島', '小笠原', '都外', '調査中', '', ''],
     ['(1)\n1',
      '(1)\n1',
      '(4)\n4',
      '(2)\n2',
      '(9)\n9',
      '(0)\n0',
      '(4)\n4',
      '(13863)\n13946',
      '(76)\n76',
      '',
      '']]


今度はうまく取れました。さらに、タイトルと数値が一行おきになっているので、それを勘案して読み込みます。また、括弧で囲われた数値は退院した人数なので、削除しましょう。


```
import re
for i in range(0, len(table.data), 2):
    for key, num in zip(table.data[i], table.data[i+1]):
        num = re.sub(r'\([^\)]*\)', '', num).strip()
        print(key, num)
```

    千代田 870
    中央 2698
    港 5510
    新宿 9042
    文京 2477
    台東 3030
    墨田 3335
    江東 5495
    品川 4872
    目黒 4549
    大田 8080
    世田谷 12081
    渋谷 4789
    中野 5382
    杉並 6599
    豊島 4401
    北 3746
    荒川 2682
    板橋 6189
    練馬 7305
    足立 7607
    葛飾 5602
    江戸川 7044
    八王子 4043
    立川 1366
    武蔵野 1309
    三鷹 1555
    青梅 799
    府中 1914
    昭島 804
    調布 1958
    町田 2693
    小金井 959
    小平 1141
    日野 1168
    東村山 842
    国分寺 830
    国立 446
    福生 485
    狛江 603
    東大和 464
    清瀬 395
    東久留米 662
    武蔵村山 372
    多摩 924
    稲城 550
    羽村 348
    あきる野 558
    西東京 1618
    瑞穂 180
    日の出 106
    檜原 10
    奥多摩 25
    大島 23
    利島 0
    新島 1
    神津島 1
    三宅 4
    御蔵島 2
    八丈 9
    青ヶ島 0
    小笠原 4
    都外 13946
    調査中 76
     
     


実はこれで一件落着ではありません。2020年5月18日の情報を取得してみましょう。


```
tables = read_pdf('https://www.bousai.metro.tokyo.lg.jp/_res/projects/default_project/_page_/001/010/322/2020051801.pdf', layout_kwargs = { 'char_margin': 0.01 })
table = tables[-1]
pprint(table.data)
```

    [['千代田', '中央', '港', '新宿', '文京', '台東', '墨田', '江東', '品川', '目黒', '大田'],
     ['43', '111', '308', '398', '92', '172', '146', '219', '185', '163', '240'],
     ['世田谷', '渋谷', '中野', '杉並', '豊島', '北', '荒川', '板橋', '練馬', '足立', '葛飾'],
     ['455', '176', '223', '248', '142', '97', '79', '139', '243', '151', '132'],
     ['江戸川', '八王子', '立川', '武蔵野', '三鷹', '青梅', '府中', '昭島', '調布', '町田', '小金井'],
     ['140', '42', '14', '17', '28', '6', '70', '9', '35', '53', '17'],
     ['小平', '日野', '東村山', '国分寺', '国立', '福生', '狛江', '東大和', '清瀬', '', '東久留米武蔵村山'],
     ['18', '20', '13', '12', '6', '1', '22', '6', '15', '15', '2'],
     ['多摩', '稲城', '羽村', 'あきる野', '西東京', '瑞穂', '日の出', '檜原', '奥多摩', '大島', '利島'],
     ['37', '11', '5', '7', '46', '1', '0', '0', '0', '0', '0'],
     ['新島', '神津島', '三宅', '御蔵島', '八丈', '青ヶ島', '小笠原', '都外', '調査中', '', ''],
     ['0', '0', '0', '1', '0', '0', '0', '200', '24', '', '']]


残念ながら東久留米と武蔵村山が一つになっています。これを分けようとしてchar_marginをもっと小さくすると他の文字列が1文字ずつに別れてしまいます。仕方がないので対症療法にしましょう。これが起こると直前の文字列が空になります。また4文字が連なる場合だけです。そこで、直前の文字列が空の場合に、現在の文字列を半分に分けた前半を直前の文字列に振り分けます。処理は以下になります。


```
import re
data = []
for i in range(0, len(table.data), 2):
    for key, num in zip(table.data[i], table.data[i+1]):
        key = re.sub(r'\s', '', key)
        if len(num) > 0:
            if len(data) > 0 and len(data[-1][0]) == 0:
                data[-1][0] = key[:len(key)//2]
                data.append([key[len(key)//2:], int(num)])
            else:
                data.append([key, int(num)])
pprint(data)
```

    [['千代田', 43],
     ['中央', 111],
     ['港', 308],
     ['新宿', 398],
     ['文京', 92],
     ['台東', 172],
     ['墨田', 146],
     ['江東', 219],
     ['品川', 185],
     ['目黒', 163],
     ['大田', 240],
     ['世田谷', 455],
     ['渋谷', 176],
     ['中野', 223],
     ['杉並', 248],
     ['豊島', 142],
     ['北', 97],
     ['荒川', 79],
     ['板橋', 139],
     ['練馬', 243],
     ['足立', 151],
     ['葛飾', 132],
     ['江戸川', 140],
     ['八王子', 42],
     ['立川', 14],
     ['武蔵野', 17],
     ['三鷹', 28],
     ['青梅', 6],
     ['府中', 70],
     ['昭島', 9],
     ['調布', 35],
     ['町田', 53],
     ['小金井', 17],
     ['小平', 18],
     ['日野', 20],
     ['東村山', 13],
     ['国分寺', 12],
     ['国立', 6],
     ['福生', 1],
     ['狛江', 22],
     ['東大和', 6],
     ['清瀬', 15],
     ['東久留米', 15],
     ['武蔵村山', 2],
     ['多摩', 37],
     ['稲城', 11],
     ['羽村', 5],
     ['あきる野', 7],
     ['西東京', 46],
     ['瑞穂', 1],
     ['日の出', 0],
     ['檜原', 0],
     ['奥多摩', 0],
     ['大島', 0],
     ['利島', 0],
     ['新島', 0],
     ['神津島', 0],
     ['三宅', 0],
     ['御蔵島', 1],
     ['八丈', 0],
     ['青ヶ島', 0],
     ['小笠原', 0],
     ['都外', 200],
     ['調査中', 24]]


東久留米と武蔵村山がうまく別れました。

それでは、上記をまとめてコーディングします。結果はGoogle Driveに保存しましょう。まずは下記を実行してGoogle Driveをマウントしてください。




```
from google.colab import drive
drive.mount('/content/drive')
!mkdir -p /content/drive/MyDrive/coronavirus
```

    Mounted at /content/drive


過去にデータ取得を行っていた場合、新規のデータのみを追加します。その場合の既存のデータの読み込みとその中の最新の日付を取得します。


```
import os
import json
import datetime
if os.path.exists('/content/drive/MyDrive/coronavirus/tokyo.json'):
    with open('/content/drive/MyDrive/coronavirus/tokyo.json') as fin:
        result = json.load(fin)
    stop_date = datetime.datetime.strptime(result[-1][0], '%Y-%m-%d %H:%M:%S')
else:
    result = []
    stop_date = None
```

取得するデータの中のリリース日時は患者の発生データから1日遅れなので、 target_date = release_date - timedelta(days=1)として日付をずらしています。既存のデータがある場合はその最新日付で読み込みを打ち切ります。そうでない場合、2020年3月31日より前は市区町村別のデータが掲載されていませんので、この日付で打ち切りにします。
データ取得には時間がかかるので、今回は2021年5月1日以降のデータを取得してみます。


```
!pip install bs4 camelot-py parse
!apt install ghostscript
```

    Requirement already satisfied: bs4 in /usr/local/lib/python3.7/dist-packages (0.0.1)
    Requirement already satisfied: camelot-py in /usr/local/lib/python3.7/dist-packages (0.9.0)
    Requirement already satisfied: parse in /usr/local/lib/python3.7/dist-packages (1.19.0)
    Requirement already satisfied: beautifulsoup4 in /usr/local/lib/python3.7/dist-packages (from bs4) (4.6.3)
    Requirement already satisfied: openpyxl>=2.5.8 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (2.5.9)
    Requirement already satisfied: pdfminer.six>=20200726 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (20201018)
    Requirement already satisfied: numpy>=1.13.3 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (1.19.5)
    Requirement already satisfied: pandas>=0.23.4 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (1.1.5)
    Requirement already satisfied: click>=6.7 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (7.1.2)
    Requirement already satisfied: PyPDF2>=1.26.0 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (1.26.0)
    Requirement already satisfied: chardet>=3.0.4 in /usr/local/lib/python3.7/dist-packages (from camelot-py) (3.0.4)
    Requirement already satisfied: jdcal in /usr/local/lib/python3.7/dist-packages (from openpyxl>=2.5.8->camelot-py) (1.4.1)
    Requirement already satisfied: et-xmlfile in /usr/local/lib/python3.7/dist-packages (from openpyxl>=2.5.8->camelot-py) (1.1.0)
    Requirement already satisfied: sortedcontainers in /usr/local/lib/python3.7/dist-packages (from pdfminer.six>=20200726->camelot-py) (2.4.0)
    Requirement already satisfied: cryptography in /usr/local/lib/python3.7/dist-packages (from pdfminer.six>=20200726->camelot-py) (3.4.7)
    Requirement already satisfied: pytz>=2017.2 in /usr/local/lib/python3.7/dist-packages (from pandas>=0.23.4->camelot-py) (2018.9)
    Requirement already satisfied: python-dateutil>=2.7.3 in /usr/local/lib/python3.7/dist-packages (from pandas>=0.23.4->camelot-py) (2.8.1)
    Requirement already satisfied: cffi>=1.12 in /usr/local/lib/python3.7/dist-packages (from cryptography->pdfminer.six>=20200726->camelot-py) (1.14.5)
    Requirement already satisfied: six>=1.5 in /usr/local/lib/python3.7/dist-packages (from python-dateutil>=2.7.3->pandas>=0.23.4->camelot-py) (1.15.0)
    Requirement already satisfied: pycparser in /usr/local/lib/python3.7/dist-packages (from cffi>=1.12->cryptography->pdfminer.six>=20200726->camelot-py) (2.20)
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    ghostscript is already the newest version (9.26~dfsg+0-0ubuntu0.18.04.14).
    0 upgraded, 0 newly installed, 0 to remove and 39 not upgraded.



```
from urllib import request
from bs4 import BeautifulSoup
from camelot import read_pdf
from parse import parse
from datetime import datetime, timedelta
import re
import os

url_base = 'https://www.bousai.metro.tokyo.lg.jp/taisaku/saigai/1010035/'
with request.urlopen(url_base + 'index.html') as response:
    soup = BeautifulSoup(response)

for base_link in soup.select('a'):
    if 'href' in base_link.attrs and base_link.attrs['href'].find('../../../taisaku/saigai/1010035') == 0:
        with request.urlopen(url_base + base_link.attrs['href']) as response:
            for page_link in BeautifulSoup(response, 'html.parser').select('a'):
                if page_link.text.find('患者の発生') >= 0:
                    page_link = os.path.join(os.path.dirname(base_link.attrs['href']), page_link.attrs['href'])
                    with request.urlopen(url_base + page_link) as response:
                        page = BeautifulSoup(response)
                        release = parse('令和{:d}年{:d}月{:d}日 {:d}時{:d}分',
                                        page.select_one('div[class=releasedate]').select_one('p').text)
                        release_date = datetime(release[0] + 2018, release[1], release[2])
                        target_date = release_date - timedelta(days=1)
                        if target_date == stop_date:
                            break
                        for link in page.select('a'):
                            if 'href' in link.attrs:
                                href = link.attrs['href']
                                if href.find('.pdf') >= 0:
                                    href = os.path.join(os.path.dirname(page_link), href)
                                    tables = read_pdf(url_base + href, layout_kwargs = { 'char_margin': 0.01 })
                                    table = tables[-1]
                                    data = []
                                    for i in range(0, len(table.data), 2):
                                        for key, num in zip(table.data[i], table.data[i+1]):
                                            key = re.sub(r'\s', '', key)
                                            num = re.sub(r'\s', '', num)
                                            if len(num) > 0:
                                                num = re.sub(r'\([^\)]*\)', '', num).strip()
                                                if len(data) > 0 and len(data[-1][0]) == 0:
                                                    data[-1][0] = key[:len(key)//2]
                                                    data.append([key[len(key)//2:], int(num)])
                                                else:
                                                    data.append([key, int(num)])
                                    print(target_date.strftime('%Y/%m/%d'), data)
                                    result.append([datetime.strftime(target_date, '%Y-%m-%d %H:%M:%S'), data])
                                    break

#                    if target_date == datetime(2020, 3, 31):　# 全期間のデータを取得する場合
                    if target_date == datetime(2021, 5, 1): # 今回取得する範囲
                        break
            else:
                continue
            break
result = sorted(result)
```

    2021/06/14 [['千代田', 870], ['中央', 2698], ['港', 5510], ['新宿', 9042], ['文京', 2477], ['台東', 3030], ['墨田', 3335], ['江東', 5495], ['品川', 4872], ['目黒', 4549], ['大田', 8080], ['世田谷', 12081], ['渋谷', 4789], ['中野', 5382], ['杉並', 6599], ['豊島', 4401], ['北', 3746], ['荒川', 2682], ['板橋', 6189], ['練馬', 7305], ['足立', 7607], ['葛飾', 5602], ['江戸川', 7044], ['八王子', 4043], ['立川', 1366], ['武蔵野', 1309], ['三鷹', 1555], ['青梅', 799], ['府中', 1914], ['昭島', 804], ['調布', 1958], ['町田', 2693], ['小金井', 959], ['小平', 1141], ['日野', 1168], ['東村山', 842], ['国分寺', 830], ['国立', 446], ['福生', 485], ['狛江', 603], ['東大和', 464], ['清瀬', 395], ['東久留米', 662], ['武蔵村山', 372], ['多摩', 924], ['稲城', 550], ['羽村', 348], ['あきる野', 558], ['西東京', 1618], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13946], ['調査中', 76]]
    2021/06/13 [['千代田', 870], ['中央', 2694], ['港', 5504], ['新宿', 9032], ['文京', 2474], ['台東', 3027], ['墨田', 3327], ['江東', 5489], ['品川', 4864], ['目黒', 4541], ['大田', 8071], ['世田谷', 12064], ['渋谷', 4779], ['中野', 5375], ['杉並', 6593], ['豊島', 4395], ['北', 3739], ['荒川', 2680], ['板橋', 6181], ['練馬', 7288], ['足立', 7604], ['葛飾', 5597], ['江戸川', 7040], ['八王子', 4039], ['立川', 1365], ['武蔵野', 1308], ['三鷹', 1553], ['青梅', 799], ['府中', 1913], ['昭島', 802], ['調布', 1956], ['町田', 2691], ['小金井', 958], ['小平', 1140], ['日野', 1167], ['東村山', 842], ['国分寺', 829], ['国立', 446], ['福生', 485], ['狛江', 603], ['東大和', 464], ['清瀬', 394], ['東久留米', 662], ['武蔵村山', 372], ['多摩', 923], ['稲城', 550], ['羽村', 348], ['あきる野', 557], ['西東京', 1612], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13922], ['調査中', 76]]
    2021/06/12 [['千代田', 867], ['中央', 2688], ['港', 5500], ['新宿', 9021], ['文京', 2468], ['台東', 3024], ['墨田', 3316], ['江東', 5473], ['品川', 4852], ['目黒', 4533], ['大田', 8054], ['世田谷', 12033], ['渋谷', 4772], ['中野', 5369], ['杉並', 6580], ['豊島', 4383], ['北', 3731], ['荒川', 2672], ['板橋', 6173], ['練馬', 7279], ['足立', 7578], ['葛飾', 5591], ['江戸川', 7028], ['八王子', 4035], ['立川', 1362], ['武蔵野', 1306], ['三鷹', 1549], ['青梅', 799], ['府中', 1907], ['昭島', 802], ['調布', 1955], ['町田', 2687], ['小金井', 956], ['小平', 1137], ['日野', 1167], ['東村山', 841], ['国分寺', 829], ['国立', 446], ['福生', 485], ['狛江', 603], ['東大和', 461], ['清瀬', 394], ['東久留米', 662], ['武蔵村山', 372], ['多摩', 921], ['稲城', 549], ['羽村', 348], ['あきる野', 557], ['西東京', 1606], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13903], ['調査中', 76]]
    2021/06/11 [['千代田', 866], ['中央', 2676], ['港', 5475], ['新宿', 8996], ['文京', 2459], ['台東', 3019], ['墨田', 3306], ['江東', 5450], ['品川', 4836], ['目黒', 4523], ['大田', 8018], ['世田谷', 12005], ['渋谷', 4762], ['中野', 5350], ['杉並', 6563], ['豊島', 4370], ['北', 3720], ['荒川', 2664], ['板橋', 6160], ['練馬', 7270], ['足立', 7562], ['葛飾', 5581], ['江戸川', 7012], ['八王子', 4026], ['立川', 1359], ['武蔵野', 1306], ['三鷹', 1543], ['青梅', 797], ['府中', 1901], ['昭島', 802], ['調布', 1949], ['町田', 2675], ['小金井', 955], ['小平', 1136], ['日野', 1164], ['東村山', 839], ['国分寺', 825], ['国立', 446], ['福生', 482], ['狛江', 601], ['東大和', 459], ['清瀬', 394], ['東久留米', 662], ['武蔵村山', 372], ['多摩', 916], ['稲城', 548], ['羽村', 348], ['あきる野', 555], ['西東京', 1596], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13858], ['調査中', 76]]
    2021/06/10 [['千代田', 864], ['中央', 2664], ['港', 5465], ['新宿', 8977], ['文京', 2456], ['台東', 3017], ['墨田', 3290], ['江東', 5430], ['品川', 4820], ['目黒', 4507], ['大田', 7999], ['世田谷', 11971], ['渋谷', 4751], ['中野', 5328], ['杉並', 6543], ['豊島', 4356], ['北', 3709], ['荒川', 2657], ['板橋', 6141], ['練馬', 7255], ['足立', 7542], ['葛飾', 5570], ['江戸川', 6994], ['八王子', 4022], ['立川', 1354], ['武蔵野', 1299], ['三鷹', 1540], ['青梅', 797], ['府中', 1893], ['昭島', 801], ['調布', 1943], ['町田', 2671], ['小金井', 949], ['小平', 1134], ['日野', 1160], ['東村山', 837], ['国分寺', 824], ['国立', 444], ['福生', 482], ['狛江', 598], ['東大和', 459], ['清瀬', 394], ['東久留米', 662], ['武蔵村山', 370], ['多摩', 910], ['稲城', 547], ['羽村', 348], ['あきる野', 554], ['西東京', 1587], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13837], ['調査中', 76]]
    2021/06/09 [['千代田', 864], ['中央', 2653], ['港', 5444], ['新宿', 8946], ['文京', 2448], ['台東', 3008], ['墨田', 3283], ['江東', 5401], ['品川', 4806], ['目黒', 4495], ['大田', 7987], ['世田谷', 11948], ['渋谷', 4743], ['中野', 5318], ['杉並', 6531], ['豊島', 4346], ['北', 3703], ['荒川', 2652], ['板橋', 6127], ['練馬', 7243], ['足立', 7523], ['葛飾', 5555], ['江戸川', 6980], ['八王子', 4005], ['立川', 1349], ['武蔵野', 1296], ['三鷹', 1535], ['青梅', 796], ['府中', 1878], ['昭島', 801], ['調布', 1939], ['町田', 2667], ['小金井', 948], ['小平', 1131], ['日野', 1158], ['東村山', 832], ['国分寺', 821], ['国立', 437], ['福生', 480], ['狛江', 597], ['東大和', 459], ['清瀬', 393], ['東久留米', 661], ['武蔵村山', 370], ['多摩', 907], ['稲城', 547], ['羽村', 348], ['あきる野', 554], ['西東京', 1578], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13792], ['調査中', 76]]
    2021/06/08 [['千代田', 860], ['中央', 2645], ['港', 5428], ['新宿', 8921], ['文京', 2439], ['台東', 3001], ['墨田', 3271], ['江東', 5386], ['品川', 4796], ['目黒', 4479], ['大田', 7964], ['世田谷', 11906], ['渋谷', 4727], ['中野', 5301], ['杉並', 6508], ['豊島', 4340], ['北', 3696], ['荒川', 2647], ['板橋', 6112], ['練馬', 7225], ['足立', 7514], ['葛飾', 5548], ['江戸川', 6958], ['八王子', 3993], ['立川', 1344], ['武蔵野', 1292], ['三鷹', 1532], ['青梅', 795], ['府中', 1873], ['昭島', 798], ['調布', 1931], ['町田', 2660], ['小金井', 945], ['小平', 1130], ['日野', 1156], ['東村山', 830], ['国分寺', 820], ['国立', 436], ['福生', 480], ['狛江', 592], ['東大和', 459], ['清瀬', 393], ['東久留米', 660], ['武蔵村山', 370], ['多摩', 903], ['稲城', 545], ['羽村', 347], ['あきる野', 554], ['西東京', 1572], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13761], ['調査中', 76]]
    2021/06/07 [['千代田', 859], ['中央', 2640], ['港', 5404], ['新宿', 8895], ['文京', 2431], ['台東', 2995], ['墨田', 3256], ['江東', 5367], ['品川', 4783], ['目黒', 4468], ['大田', 7941], ['世田谷', 11884], ['渋谷', 4714], ['中野', 5292], ['杉並', 6503], ['豊島', 4332], ['北', 3686], ['荒川', 2639], ['板橋', 6103], ['練馬', 7214], ['足立', 7498], ['葛飾', 5546], ['江戸川', 6948], ['八王子', 3989], ['立川', 1342], ['武蔵野', 1291], ['三鷹', 1529], ['青梅', 794], ['府中', 1871], ['昭島', 794], ['調布', 1927], ['町田', 2656], ['小金井', 943], ['小平', 1128], ['日野', 1155], ['東村山', 825], ['国分寺', 819], ['国立', 434], ['福生', 479], ['狛江', 592], ['東大和', 456], ['清瀬', 393], ['東久留米', 657], ['武蔵村山', 369], ['多摩', 898], ['稲城', 544], ['羽村', 347], ['あきる野', 552], ['西東京', 1566], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13726], ['調査中', 76]]
    2021/06/06 [['千代田', 854], ['中央', 2637], ['港', 5394], ['新宿', 8878], ['文京', 2428], ['台東', 2993], ['墨田', 3252], ['江東', 5362], ['品川', 4769], ['目黒', 4461], ['大田', 7936], ['世田谷', 11868], ['渋谷', 4708], ['中野', 5287], ['杉並', 6493], ['豊島', 4321], ['北', 3682], ['荒川', 2637], ['板橋', 6099], ['練馬', 7207], ['足立', 7489], ['葛飾', 5539], ['江戸川', 6941], ['八王子', 3983], ['立川', 1340], ['武蔵野', 1291], ['三鷹', 1527], ['青梅', 794], ['府中', 1869], ['昭島', 794], ['調布', 1923], ['町田', 2655], ['小金井', 941], ['小平', 1126], ['日野', 1155], ['東村山', 823], ['国分寺', 817], ['国立', 434], ['福生', 479], ['狛江', 592], ['東大和', 456], ['清瀬', 392], ['東久留米', 655], ['武蔵村山', 369], ['多摩', 897], ['稲城', 544], ['羽村', 347], ['あきる野', 551], ['西東京', 1563], ['瑞穂', 180], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13687], ['調査中', 76]]
    2021/06/05 [['千代田', 854], ['中央', 2629], ['港', 5376], ['新宿', 8860], ['文京', 2420], ['台東', 2990], ['墨田', 3243], ['江東', 5353], ['品川', 4752], ['目黒', 4455], ['大田', 7927], ['世田谷', 11830], ['渋谷', 4698], ['中野', 5271], ['杉並', 6488], ['豊島', 4311], ['北', 3676], ['荒川', 2629], ['板橋', 6097], ['練馬', 7201], ['足立', 7474], ['葛飾', 5532], ['江戸川', 6929], ['八王子', 3971], ['立川', 1338], ['武蔵野', 1288], ['三鷹', 1524], ['青梅', 793], ['府中', 1862], ['昭島', 793], ['調布', 1914], ['町田', 2645], ['小金井', 939], ['小平', 1122], ['日野', 1155], ['東村山', 818], ['国分寺', 816], ['国立', 432], ['福生', 479], ['狛江', 592], ['東大和', 456], ['清瀬', 392], ['東久留米', 654], ['武蔵村山', 369], ['多摩', 897], ['稲城', 544], ['羽村', 347], ['あきる野', 549], ['西東京', 1553], ['瑞穂', 179], ['日の出', 106], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13652], ['調査中', 76]]
    2021/06/04 [['千代田', 852], ['中央', 2626], ['港', 5355], ['新宿', 8842], ['文京', 2410], ['台東', 2982], ['墨田', 3241], ['江東', 5338], ['品川', 4738], ['目黒', 4443], ['大田', 7901], ['世田谷', 11791], ['渋谷', 4692], ['中野', 5256], ['杉並', 6474], ['豊島', 4290], ['北', 3663], ['荒川', 2622], ['板橋', 6090], ['練馬', 7185], ['足立', 7460], ['葛飾', 5527], ['江戸川', 6906], ['八王子', 3963], ['立川', 1335], ['武蔵野', 1286], ['三鷹', 1520], ['青梅', 792], ['府中', 1859], ['昭島', 792], ['調布', 1911], ['町田', 2634], ['小金井', 937], ['小平', 1118], ['日野', 1153], ['東村山', 817], ['国分寺', 812], ['国立', 431], ['福生', 476], ['狛江', 592], ['東大和', 456], ['清瀬', 391], ['東久留米', 654], ['武蔵村山', 369], ['多摩', 895], ['稲城', 540], ['羽村', 346], ['あきる野', 549], ['西東京', 1550], ['瑞穂', 179], ['日の出', 105], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13592], ['調査中', 76]]
    2021/06/03 [['千代田', 851], ['中央', 2616], ['港', 5318], ['新宿', 8817], ['文京', 2405], ['台東', 2977], ['墨田', 3230], ['江東', 5326], ['品川', 4723], ['目黒', 4421], ['大田', 7885], ['世田谷', 11752], ['渋谷', 4681], ['中野', 5237], ['杉並', 6451], ['豊島', 4277], ['北', 3656], ['荒川', 2615], ['板橋', 6079], ['練馬', 7170], ['足立', 7445], ['葛飾', 5514], ['江戸川', 6888], ['八王子', 3952], ['立川', 1329], ['武蔵野', 1280], ['三鷹', 1517], ['青梅', 791], ['府中', 1852], ['昭島', 789], ['調布', 1906], ['町田', 2626], ['小金井', 933], ['小平', 1117], ['日野', 1152], ['東村山', 817], ['国分寺', 809], ['国立', 431], ['福生', 475], ['狛江', 592], ['東大和', 454], ['清瀬', 391], ['東久留米', 653], ['武蔵村山', 368], ['多摩', 892], ['稲城', 539], ['羽村', 344], ['あきる野', 547], ['西東京', 1549], ['瑞穂', 179], ['日の出', 105], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13543], ['調査中', 76]]
    2021/06/02 [['千代田', 851], ['中央', 2612], ['港', 5301], ['新宿', 8783], ['文京', 2401], ['台東', 2969], ['墨田', 3224], ['江東', 5307], ['品川', 4710], ['目黒', 4404], ['大田', 7854], ['世田谷', 11712], ['渋谷', 4666], ['中野', 5221], ['杉並', 6430], ['豊島', 4260], ['北', 3648], ['荒川', 2604], ['板橋', 6064], ['練馬', 7148], ['足立', 7425], ['葛飾', 5496], ['江戸川', 6861], ['八王子', 3943], ['立川', 1326], ['武蔵野', 1278], ['三鷹', 1512], ['青梅', 791], ['府中', 1844], ['昭島', 789], ['調布', 1899], ['町田', 2622], ['小金井', 930], ['小平', 1113], ['日野', 1148], ['東村山', 815], ['国分寺', 807], ['国立', 426], ['福生', 475], ['狛江', 592], ['東大和', 454], ['清瀬', 390], ['東久留米', 651], ['武蔵村山', 367], ['多摩', 889], ['稲城', 537], ['羽村', 343], ['あきる野', 545], ['西東京', 1541], ['瑞穂', 179], ['日の出', 105], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13496], ['調査中', 76]]
    2021/06/01 [['千代田', 846], ['中央', 2605], ['港', 5286], ['新宿', 8761], ['文京', 2385], ['台東', 2957], ['墨田', 3215], ['江東', 5282], ['品川', 4697], ['目黒', 4386], ['大田', 7834], ['世田谷', 11658], ['渋谷', 4647], ['中野', 5211], ['杉並', 6423], ['豊島', 4247], ['北', 3638], ['荒川', 2589], ['板橋', 6048], ['練馬', 7136], ['足立', 7407], ['葛飾', 5486], ['江戸川', 6830], ['八王子', 3939], ['立川', 1321], ['武蔵野', 1274], ['三鷹', 1506], ['青梅', 790], ['府中', 1839], ['昭島', 786], ['調布', 1893], ['町田', 2617], ['小金井', 927], ['小平', 1112], ['日野', 1147], ['東村山', 812], ['国分寺', 802], ['国立', 425], ['福生', 473], ['狛江', 591], ['東大和', 453], ['清瀬', 390], ['東久留米', 649], ['武蔵村山', 367], ['多摩', 889], ['稲城', 536], ['羽村', 343], ['あきる野', 542], ['西東京', 1539], ['瑞穂', 177], ['日の出', 105], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13453], ['調査中', 76]]
    2021/05/31 [['千代田', 844], ['中央', 2598], ['港', 5268], ['新宿', 8733], ['文京', 2377], ['台東', 2950], ['墨田', 3211], ['江東', 5255], ['品川', 4684], ['目黒', 4367], ['大田', 7814], ['世田谷', 11629], ['渋谷', 4630], ['中野', 5200], ['杉並', 6401], ['豊島', 4232], ['北', 3632], ['荒川', 2583], ['板橋', 6022], ['練馬', 7103], ['足立', 7397], ['葛飾', 5477], ['江戸川', 6819], ['八王子', 3931], ['立川', 1315], ['武蔵野', 1271], ['三鷹', 1503], ['青梅', 790], ['府中', 1835], ['昭島', 782], ['調布', 1889], ['町田', 2615], ['小金井', 924], ['小平', 1106], ['日野', 1146], ['東村山', 804], ['国分寺', 801], ['国立', 424], ['福生', 469], ['狛江', 589], ['東大和', 451], ['清瀬', 389], ['東久留米', 648], ['武蔵村山', 367], ['多摩', 888], ['稲城', 535], ['羽村', 343], ['あきる野', 542], ['西東京', 1533], ['瑞穂', 177], ['日の出', 105], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13402], ['調査中', 76]]
    2021/05/30 [['千代田', 841], ['中央', 2596], ['港', 5260], ['新宿', 8724], ['文京', 2370], ['台東', 2946], ['墨田', 3210], ['江東', 5245], ['品川', 4676], ['目黒', 4359], ['大田', 7804], ['世田谷', 11608], ['渋谷', 4623], ['中野', 5180], ['杉並', 6391], ['豊島', 4217], ['北', 3628], ['荒川', 2579], ['板橋', 6006], ['練馬', 7095], ['足立', 7385], ['葛飾', 5471], ['江戸川', 6804], ['八王子', 3926], ['立川', 1312], ['武蔵野', 1268], ['三鷹', 1501], ['青梅', 789], ['府中', 1835], ['昭島', 781], ['調布', 1888], ['町田', 2614], ['小金井', 924], ['小平', 1105], ['日野', 1145], ['東村山', 804], ['国分寺', 800], ['国立', 423], ['福生', 469], ['狛江', 589], ['東大和', 451], ['清瀬', 387], ['東久留米', 646], ['武蔵村山', 366], ['多摩', 887], ['稲城', 535], ['羽村', 343], ['あきる野', 542], ['西東京', 1531], ['瑞穂', 176], ['日の出', 104], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 9], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13381], ['調査中', 76]]
    2021/05/29 [['千代田', 840], ['中央', 2593], ['港', 5252], ['新宿', 8700], ['文京', 2370], ['台東', 2936], ['墨田', 3203], ['江東', 5232], ['品川', 4666], ['目黒', 4345], ['大田', 7783], ['世田谷', 11570], ['渋谷', 4617], ['中野', 5168], ['杉並', 6373], ['豊島', 4202], ['北', 3611], ['荒川', 2574], ['板橋', 5965], ['練馬', 7078], ['足立', 7375], ['葛飾', 5453], ['江戸川', 6793], ['八王子', 3918], ['立川', 1308], ['武蔵野', 1267], ['三鷹', 1499], ['青梅', 789], ['府中', 1829], ['昭島', 780], ['調布', 1886], ['町田', 2605], ['小金井', 921], ['小平', 1099], ['日野', 1145], ['東村山', 801], ['国分寺', 799], ['国立', 420], ['福生', 468], ['狛江', 586], ['東大和', 448], ['清瀬', 386], ['東久留米', 643], ['武蔵村山', 364], ['多摩', 886], ['稲城', 535], ['羽村', 342], ['あきる野', 539], ['西東京', 1526], ['瑞穂', 176], ['日の出', 104], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13326], ['調査中', 76]]
    2021/05/28 [['千代田', 838], ['中央', 2584], ['港', 5234], ['新宿', 8669], ['文京', 2363], ['台東', 2925], ['墨田', 3196], ['江東', 5216], ['品川', 4650], ['目黒', 4334], ['大田', 7760], ['世田谷', 11533], ['渋谷', 4603], ['中野', 5147], ['杉並', 6350], ['豊島', 4186], ['北', 3595], ['荒川', 2554], ['板橋', 5944], ['練馬', 7061], ['足立', 7362], ['葛飾', 5440], ['江戸川', 6769], ['八王子', 3910], ['立川', 1305], ['武蔵野', 1264], ['三鷹', 1491], ['青梅', 789], ['府中', 1824], ['昭島', 779], ['調布', 1881], ['町田', 2598], ['小金井', 916], ['小平', 1091], ['日野', 1143], ['東村山', 796], ['国分寺', 794], ['国立', 416], ['福生', 464], ['狛江', 585], ['東大和', 448], ['清瀬', 386], ['東久留米', 642], ['武蔵村山', 364], ['多摩', 885], ['稲城', 533], ['羽村', 342], ['あきる野', 537], ['西東京', 1519], ['瑞穂', 176], ['日の出', 104], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13260], ['調査中', 76]]
    2021/05/27 [['千代田', 833], ['中央', 2577], ['港', 5221], ['新宿', 8650], ['文京', 2358], ['台東', 2915], ['墨田', 3184], ['江東', 5199], ['品川', 4630], ['目黒', 4317], ['大田', 7740], ['世田谷', 11487], ['渋谷', 4577], ['中野', 5124], ['杉並', 6320], ['豊島', 4165], ['北', 3578], ['荒川', 2529], ['板橋', 5918], ['練馬', 7036], ['足立', 7345], ['葛飾', 5422], ['江戸川', 6750], ['八王子', 3896], ['立川', 1300], ['武蔵野', 1257], ['三鷹', 1485], ['青梅', 788], ['府中', 1819], ['昭島', 777], ['調布', 1873], ['町田', 2588], ['小金井', 911], ['小平', 1090], ['日野', 1140], ['東村山', 789], ['国分寺', 789], ['国立', 416], ['福生', 464], ['狛江', 581], ['東大和', 447], ['清瀬', 385], ['東久留米', 639], ['武蔵村山', 364], ['多摩', 879], ['稲城', 532], ['羽村', 339], ['あきる野', 528], ['西東京', 1515], ['瑞穂', 176], ['日の出', 104], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13195], ['調査中', 76]]
    2021/05/26 [['千代田', 831], ['中央', 2562], ['港', 5194], ['新宿', 8613], ['文京', 2352], ['台東', 2910], ['墨田', 3168], ['江東', 5175], ['品川', 4604], ['目黒', 4298], ['大田', 7723], ['世田谷', 11432], ['渋谷', 4555], ['中野', 5103], ['杉並', 6298], ['豊島', 4146], ['北', 3560], ['荒川', 2519], ['板橋', 5899], ['練馬', 7010], ['足立', 7327], ['葛飾', 5406], ['江戸川', 6711], ['八王子', 3876], ['立川', 1296], ['武蔵野', 1255], ['三鷹', 1478], ['青梅', 786], ['府中', 1808], ['昭島', 773], ['調布', 1870], ['町田', 2575], ['小金井', 908], ['小平', 1084], ['日野', 1133], ['東村山', 783], ['国分寺', 782], ['国立', 414], ['福生', 461], ['狛江', 579], ['東大和', 445], ['清瀬', 385], ['東久留米', 636], ['武蔵村山', 362], ['多摩', 875], ['稲城', 529], ['羽村', 339], ['あきる野', 523], ['西東京', 1510], ['瑞穂', 174], ['日の出', 103], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13119], ['調査中', 76]]
    2021/05/25 [['千代田', 824], ['中央', 2547], ['港', 5175], ['新宿', 8576], ['文京', 2348], ['台東', 2897], ['墨田', 3158], ['江東', 5152], ['品川', 4568], ['目黒', 4275], ['大田', 7695], ['世田谷', 11370], ['渋谷', 4534], ['中野', 5086], ['杉並', 6270], ['豊島', 4128], ['北', 3540], ['荒川', 2500], ['板橋', 5868], ['練馬', 6975], ['足立', 7298], ['葛飾', 5390], ['江戸川', 6680], ['八王子', 3863], ['立川', 1281], ['武蔵野', 1248], ['三鷹', 1467], ['青梅', 785], ['府中', 1801], ['昭島', 772], ['調布', 1860], ['町田', 2557], ['小金井', 904], ['小平', 1079], ['日野', 1131], ['東村山', 778], ['国分寺', 778], ['国立', 413], ['福生', 458], ['狛江', 577], ['東大和', 443], ['清瀬', 383], ['東久留米', 634], ['武蔵村山', 361], ['多摩', 871], ['稲城', 527], ['羽村', 336], ['あきる野', 512], ['西東京', 1506], ['瑞穂', 173], ['日の出', 102], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 13060], ['調査中', 76]]
    2021/05/24 [['千代田', 821], ['中央', 2534], ['港', 5147], ['新宿', 8555], ['文京', 2339], ['台東', 2885], ['墨田', 3147], ['江東', 5134], ['品川', 4548], ['目黒', 4262], ['大田', 7674], ['世田谷', 11340], ['渋谷', 4517], ['中野', 5066], ['杉並', 6247], ['豊島', 4118], ['北', 3531], ['荒川', 2495], ['板橋', 5850], ['練馬', 6949], ['足立', 7271], ['葛飾', 5380], ['江戸川', 6655], ['八王子', 3856], ['立川', 1275], ['武蔵野', 1246], ['三鷹', 1464], ['青梅', 785], ['府中', 1788], ['昭島', 770], ['調布', 1855], ['町田', 2548], ['小金井', 901], ['小平', 1075], ['日野', 1124], ['東村山', 776], ['国分寺', 775], ['国立', 413], ['福生', 453], ['狛江', 575], ['東大和', 442], ['清瀬', 380], ['東久留米', 632], ['武蔵村山', 360], ['多摩', 870], ['稲城', 527], ['羽村', 336], ['あきる野', 506], ['西東京', 1506], ['瑞穂', 172], ['日の出', 99], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12998], ['調査中', 76]]
    2021/05/23 [['千代田', 819], ['中央', 2531], ['港', 5140], ['新宿', 8543], ['文京', 2337], ['台東', 2882], ['墨田', 3139], ['江東', 5126], ['品川', 4541], ['目黒', 4255], ['大田', 7658], ['世田谷', 11328], ['渋谷', 4507], ['中野', 5054], ['杉並', 6221], ['豊島', 4111], ['北', 3525], ['荒川', 2488], ['板橋', 5840], ['練馬', 6937], ['足立', 7264], ['葛飾', 5374], ['江戸川', 6646], ['八王子', 3844], ['立川', 1271], ['武蔵野', 1244], ['三鷹', 1461], ['青梅', 785], ['府中', 1782], ['昭島', 769], ['調布', 1854], ['町田', 2532], ['小金井', 900], ['小平', 1069], ['日野', 1122], ['東村山', 774], ['国分寺', 773], ['国立', 411], ['福生', 453], ['狛江', 572], ['東大和', 441], ['清瀬', 379], ['東久留米', 632], ['武蔵村山', 360], ['多摩', 869], ['稲城', 526], ['羽村', 336], ['あきる野', 503], ['西東京', 1502], ['瑞穂', 170], ['日の出', 99], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12933], ['調査中', 76]]
    2021/05/22 [['千代田', 818], ['中央', 2526], ['港', 5126], ['新宿', 8525], ['文京', 2335], ['台東', 2872], ['墨田', 3126], ['江東', 5111], ['品川', 4532], ['目黒', 4240], ['大田', 7639], ['世田谷', 11279], ['渋谷', 4484], ['中野', 5045], ['杉並', 6202], ['豊島', 4102], ['北', 3511], ['荒川', 2481], ['板橋', 5812], ['練馬', 6915], ['足立', 7245], ['葛飾', 5357], ['江戸川', 6622], ['八王子', 3822], ['立川', 1266], ['武蔵野', 1237], ['三鷹', 1460], ['青梅', 784], ['府中', 1769], ['昭島', 769], ['調布', 1847], ['町田', 2520], ['小金井', 895], ['小平', 1061], ['日野', 1118], ['東村山', 770], ['国分寺', 769], ['国立', 411], ['福生', 450], ['狛江', 570], ['東大和', 441], ['清瀬', 375], ['東久留米', 632], ['武蔵村山', 360], ['多摩', 868], ['稲城', 526], ['羽村', 336], ['あきる野', 502], ['西東京', 1495], ['瑞穂', 170], ['日の出', 96], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12873], ['調査中', 76]]
    2021/05/21 [['千代田', 816], ['中央', 2519], ['港', 5093], ['新宿', 8501], ['文京', 2329], ['台東', 2855], ['墨田', 3116], ['江東', 5090], ['品川', 4511], ['目黒', 4222], ['大田', 7628], ['世田谷', 11231], ['渋谷', 4470], ['中野', 5022], ['杉並', 6169], ['豊島', 4082], ['北', 3494], ['荒川', 2465], ['板橋', 5789], ['練馬', 6895], ['足立', 7230], ['葛飾', 5342], ['江戸川', 6590], ['八王子', 3805], ['立川', 1258], ['武蔵野', 1236], ['三鷹', 1452], ['青梅', 780], ['府中', 1763], ['昭島', 766], ['調布', 1843], ['町田', 2513], ['小金井', 893], ['小平', 1058], ['日野', 1111], ['東村山', 768], ['国分寺', 767], ['国立', 410], ['福生', 450], ['狛江', 570], ['東大和', 441], ['清瀬', 372], ['東久留米', 628], ['武蔵村山', 359], ['多摩', 865], ['稲城', 521], ['羽村', 334], ['あきる野', 497], ['西東京', 1492], ['瑞穂', 170], ['日の出', 96], ['檜原', 10], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12818], ['調査中', 76]]
    2021/05/20 [['千代田', 812], ['中央', 2512], ['港', 5061], ['新宿', 8471], ['文京', 2316], ['台東', 2842], ['墨田', 3103], ['江東', 5072], ['品川', 4477], ['目黒', 4208], ['大田', 7606], ['世田谷', 11181], ['渋谷', 4455], ['中野', 5005], ['杉並', 6149], ['豊島', 4070], ['北', 3463], ['荒川', 2453], ['板橋', 5769], ['練馬', 6858], ['足立', 7218], ['葛飾', 5329], ['江戸川', 6558], ['八王子', 3790], ['立川', 1252], ['武蔵野', 1232], ['三鷹', 1448], ['青梅', 777], ['府中', 1753], ['昭島', 763], ['調布', 1836], ['町田', 2506], ['小金井', 889], ['小平', 1058], ['日野', 1108], ['東村山', 765], ['国分寺', 763], ['国立', 407], ['福生', 448], ['狛江', 566], ['東大和', 439], ['清瀬', 371], ['東久留米', 627], ['武蔵村山', 354], ['多摩', 865], ['稲城', 521], ['羽村', 334], ['あきる野', 495], ['西東京', 1487], ['瑞穂', 170], ['日の出', 96], ['檜原', 9], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12739], ['調査中', 76]]
    2021/05/19 [['千代田', 811], ['中央', 2491], ['港', 5032], ['新宿', 8411], ['文京', 2308], ['台東', 2831], ['墨田', 3093], ['江東', 5052], ['品川', 4446], ['目黒', 4191], ['大田', 7571], ['世田谷', 11120], ['渋谷', 4423], ['中野', 4988], ['杉並', 6115], ['豊島', 4046], ['北', 3431], ['荒川', 2439], ['板橋', 5726], ['練馬', 6832], ['足立', 7184], ['葛飾', 5322], ['江戸川', 6520], ['八王子', 3778], ['立川', 1249], ['武蔵野', 1226], ['三鷹', 1437], ['青梅', 772], ['府中', 1733], ['昭島', 762], ['調布', 1824], ['町田', 2497], ['小金井', 882], ['小平', 1048], ['日野', 1102], ['東村山', 763], ['国分寺', 756], ['国立', 403], ['福生', 443], ['狛江', 565], ['東大和', 435], ['清瀬', 370], ['東久留米', 624], ['武蔵村山', 351], ['多摩', 856], ['稲城', 519], ['羽村', 334], ['あきる野', 494], ['西東京', 1477], ['瑞穂', 170], ['日の出', 94], ['檜原', 9], ['奥多摩', 25], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12657], ['調査中', 76]]
    2021/05/18 [['千代田', 807], ['中央', 2487], ['港', 5005], ['新宿', 8394], ['文京', 2293], ['台東', 2808], ['墨田', 3071], ['江東', 5021], ['品川', 4429], ['目黒', 4160], ['大田', 7545], ['世田谷', 11055], ['渋谷', 4403], ['中野', 4967], ['杉並', 6080], ['豊島', 4018], ['北', 3409], ['荒川', 2427], ['板橋', 5697], ['練馬', 6792], ['足立', 7167], ['葛飾', 5301], ['江戸川', 6486], ['八王子', 3764], ['立川', 1241], ['武蔵野', 1219], ['三鷹', 1432], ['青梅', 769], ['府中', 1715], ['昭島', 760], ['調布', 1818], ['町田', 2487], ['小金井', 872], ['小平', 1042], ['日野', 1098], ['東村山', 762], ['国分寺', 748], ['国立', 402], ['福生', 441], ['狛江', 562], ['東大和', 430], ['清瀬', 367], ['東久留米', 621], ['武蔵村山', 349], ['多摩', 854], ['稲城', 515], ['羽村', 334], ['あきる野', 489], ['西東京', 1470], ['瑞穂', 170], ['日の出', 94], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12592], ['調査中', 76]]
    2021/05/17 [['千代田', 804], ['中央', 2471], ['港', 4977], ['新宿', 8351], ['文京', 2286], ['台東', 2802], ['墨田', 3067], ['江東', 4997], ['品川', 4382], ['目黒', 4135], ['大田', 7525], ['世田谷', 11021], ['渋谷', 4382], ['中野', 4938], ['杉並', 6039], ['豊島', 3992], ['北', 3383], ['荒川', 2419], ['板橋', 5662], ['練馬', 6769], ['足立', 7139], ['葛飾', 5286], ['江戸川', 6457], ['八王子', 3752], ['立川', 1233], ['武蔵野', 1213], ['三鷹', 1430], ['青梅', 767], ['府中', 1701], ['昭島', 760], ['調布', 1814], ['町田', 2478], ['小金井', 871], ['小平', 1041], ['日野', 1096], ['東村山', 760], ['国分寺', 742], ['国立', 402], ['福生', 440], ['狛江', 560], ['東大和', 425], ['清瀬', 365], ['東久留米', 620], ['武蔵村山', 346], ['多摩', 850], ['稲城', 515], ['羽村', 333], ['あきる野', 486], ['西東京', 1466], ['瑞穂', 170], ['日の出', 93], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12494], ['調査中', 76]]
    2021/05/16 [['千代田', 802], ['中央', 2467], ['港', 4969], ['新宿', 8332], ['文京', 2285], ['台東', 2791], ['墨田', 3062], ['江東', 4992], ['品川', 4377], ['目黒', 4123], ['大田', 7504], ['世田谷', 10995], ['渋谷', 4373], ['中野', 4924], ['杉並', 6021], ['豊島', 3975], ['北', 3373], ['荒川', 2414], ['板橋', 5636], ['練馬', 6750], ['足立', 7125], ['葛飾', 5277], ['江戸川', 6445], ['八王子', 3740], ['立川', 1229], ['武蔵野', 1207], ['三鷹', 1425], ['青梅', 762], ['府中', 1697], ['昭島', 760], ['調布', 1807], ['町田', 2474], ['小金井', 869], ['小平', 1040], ['日野', 1093], ['東村山', 757], ['国分寺', 736], ['国立', 402], ['福生', 438], ['狛江', 556], ['東大和', 423], ['清瀬', 364], ['東久留米', 620], ['武蔵村山', 346], ['多摩', 847], ['稲城', 510], ['羽村', 331], ['あきる野', 485], ['西東京', 1464], ['瑞穂', 170], ['日の出', 91], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12433], ['調査中', 76]]
    2021/05/15 [['千代田', 799], ['中央', 2457], ['港', 4956], ['新宿', 8315], ['文京', 2279], ['台東', 2787], ['墨田', 3052], ['江東', 4967], ['品川', 4361], ['目黒', 4109], ['大田', 7491], ['世田谷', 10952], ['渋谷', 4354], ['中野', 4910], ['杉並', 6004], ['豊島', 3966], ['北', 3355], ['荒川', 2405], ['板橋', 5616], ['練馬', 6734], ['足立', 7101], ['葛飾', 5256], ['江戸川', 6407], ['八王子', 3721], ['立川', 1223], ['武蔵野', 1203], ['三鷹', 1424], ['青梅', 754], ['府中', 1688], ['昭島', 756], ['調布', 1802], ['町田', 2462], ['小金井', 862], ['小平', 1034], ['日野', 1091], ['東村山', 757], ['国分寺', 734], ['国立', 400], ['福生', 435], ['狛江', 556], ['東大和', 423], ['清瀬', 362], ['東久留米', 616], ['武蔵村山', 344], ['多摩', 844], ['稲城', 507], ['羽村', 329], ['あきる野', 483], ['西東京', 1461], ['瑞穂', 169], ['日の出', 91], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12382], ['調査中', 76]]
    2021/05/14 [['千代田', 798], ['中央', 2444], ['港', 4925], ['新宿', 8280], ['文京', 2270], ['台東', 2769], ['墨田', 3036], ['江東', 4936], ['品川', 4328], ['目黒', 4091], ['大田', 7461], ['世田谷', 10872], ['渋谷', 4333], ['中野', 4898], ['杉並', 5978], ['豊島', 3948], ['北', 3331], ['荒川', 2394], ['板橋', 5596], ['練馬', 6708], ['足立', 7073], ['葛飾', 5243], ['江戸川', 6370], ['八王子', 3686], ['立川', 1214], ['武蔵野', 1194], ['三鷹', 1421], ['青梅', 752], ['府中', 1675], ['昭島', 754], ['調布', 1798], ['町田', 2454], ['小金井', 856], ['小平', 1026], ['日野', 1085], ['東村山', 755], ['国分寺', 733], ['国立', 397], ['福生', 435], ['狛江', 553], ['東大和', 423], ['清瀬', 360], ['東久留米', 613], ['武蔵村山', 341], ['多摩', 838], ['稲城', 501], ['羽村', 327], ['あきる野', 482], ['西東京', 1456], ['瑞穂', 169], ['日の出', 90], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12304], ['調査中', 76]]
    2021/05/13 [['千代田', 791], ['中央', 2431], ['港', 4897], ['新宿', 8252], ['文京', 2264], ['台東', 2756], ['墨田', 3026], ['江東', 4913], ['品川', 4308], ['目黒', 4062], ['大田', 7431], ['世田谷', 10806], ['渋谷', 4299], ['中野', 4869], ['杉並', 5938], ['豊島', 3917], ['北', 3303], ['荒川', 2372], ['板橋', 5564], ['練馬', 6673], ['足立', 7036], ['葛飾', 5214], ['江戸川', 6346], ['八王子', 3668], ['立川', 1207], ['武蔵野', 1190], ['三鷹', 1410], ['青梅', 751], ['府中', 1656], ['昭島', 751], ['調布', 1791], ['町田', 2440], ['小金井', 848], ['小平', 1017], ['日野', 1080], ['東村山', 746], ['国分寺', 732], ['国立', 395], ['福生', 434], ['狛江', 552], ['東大和', 421], ['清瀬', 359], ['東久留米', 607], ['武蔵村山', 337], ['多摩', 834], ['稲城', 499], ['羽村', 327], ['あきる野', 480], ['西東京', 1450], ['瑞穂', 169], ['日の出', 90], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12212], ['調査中', 75]]
    2021/05/12 [['千代田', 782], ['中央', 2415], ['港', 4851], ['新宿', 8183], ['文京', 2251], ['台東', 2744], ['墨田', 3003], ['江東', 4877], ['品川', 4284], ['目黒', 4037], ['大田', 7383], ['世田谷', 10738], ['渋谷', 4277], ['中野', 4836], ['杉並', 5887], ['豊島', 3896], ['北', 3281], ['荒川', 2364], ['板橋', 5536], ['練馬', 6631], ['足立', 7002], ['葛飾', 5188], ['江戸川', 6307], ['八王子', 3633], ['立川', 1201], ['武蔵野', 1182], ['三鷹', 1403], ['青梅', 750], ['府中', 1632], ['昭島', 749], ['調布', 1779], ['町田', 2426], ['小金井', 846], ['小平', 1009], ['日野', 1076], ['東村山', 742], ['国分寺', 720], ['国立', 393], ['福生', 433], ['狛江', 547], ['東大和', 418], ['清瀬', 355], ['東久留米', 605], ['武蔵村山', 336], ['多摩', 829], ['稲城', 496], ['羽村', 325], ['あきる野', 478], ['西東京', 1439], ['瑞穂', 169], ['日の出', 90], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 12097], ['調査中', 75]]
    2021/05/11 [['千代田', 782], ['中央', 2406], ['港', 4827], ['新宿', 8160], ['文京', 2236], ['台東', 2719], ['墨田', 2984], ['江東', 4850], ['品川', 4250], ['目黒', 4002], ['大田', 7356], ['世田谷', 10659], ['渋谷', 4234], ['中野', 4807], ['杉並', 5841], ['豊島', 3874], ['北', 3252], ['荒川', 2345], ['板橋', 5508], ['練馬', 6591], ['足立', 6961], ['葛飾', 5163], ['江戸川', 6267], ['八王子', 3612], ['立川', 1188], ['武蔵野', 1171], ['三鷹', 1395], ['青梅', 750], ['府中', 1619], ['昭島', 740], ['調布', 1769], ['町田', 2404], ['小金井', 843], ['小平', 997], ['日野', 1068], ['東村山', 735], ['国分寺', 717], ['国立', 390], ['福生', 433], ['狛江', 544], ['東大和', 416], ['清瀬', 350], ['東久留米', 601], ['武蔵村山', 335], ['多摩', 825], ['稲城', 486], ['羽村', 324], ['あきる野', 477], ['西東京', 1434], ['瑞穂', 169], ['日の出', 90], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11986], ['調査中', 75]]
    2021/05/10 [['千代田', 779], ['中央', 2389], ['港', 4800], ['新宿', 8112], ['文京', 2216], ['台東', 2713], ['墨田', 2973], ['江東', 4820], ['品川', 4224], ['目黒', 3982], ['大田', 7310], ['世田谷', 10618], ['渋谷', 4196], ['中野', 4775], ['杉並', 5798], ['豊島', 3853], ['北', 3229], ['荒川', 2334], ['板橋', 5479], ['練馬', 6556], ['足立', 6918], ['葛飾', 5140], ['江戸川', 6222], ['八王子', 3576], ['立川', 1185], ['武蔵野', 1164], ['三鷹', 1388], ['青梅', 748], ['府中', 1604], ['昭島', 733], ['調布', 1760], ['町田', 2392], ['小金井', 836], ['小平', 991], ['日野', 1059], ['東村山', 731], ['国分寺', 711], ['国立', 388], ['福生', 431], ['狛江', 543], ['東大和', 414], ['清瀬', 347], ['東久留米', 598], ['武蔵村山', 334], ['多摩', 816], ['稲城', 478], ['羽村', 323], ['あきる野', 476], ['西東京', 1429], ['瑞穂', 169], ['日の出', 88], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11867], ['調査中', 77]]
    2021/05/09 [['千代田', 776], ['中央', 2385], ['港', 4789], ['新宿', 8082], ['文京', 2210], ['台東', 2707], ['墨田', 2957], ['江東', 4806], ['品川', 4204], ['目黒', 3964], ['大田', 7290], ['世田谷', 10587], ['渋谷', 4186], ['中野', 4749], ['杉並', 5786], ['豊島', 3835], ['北', 3223], ['荒川', 2330], ['板橋', 5456], ['練馬', 6536], ['足立', 6900], ['葛飾', 5121], ['江戸川', 6200], ['八王子', 3553], ['立川', 1174], ['武蔵野', 1152], ['三鷹', 1384], ['青梅', 747], ['府中', 1600], ['昭島', 731], ['調布', 1755], ['町田', 2388], ['小金井', 833], ['小平', 986], ['日野', 1058], ['東村山', 728], ['国分寺', 707], ['国立', 379], ['福生', 430], ['狛江', 543], ['東大和', 414], ['清瀬', 343], ['東久留米', 593], ['武蔵村山', 333], ['多摩', 803], ['稲城', 473], ['羽村', 319], ['あきる野', 477], ['西東京', 1425], ['瑞穂', 169], ['日の出', 87], ['檜原', 9], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11778], ['調査中', 78]]
    2021/05/08 [['千代田', 772], ['中央', 2367], ['港', 4763], ['新宿', 8017], ['文京', 2192], ['台東', 2690], ['墨田', 2940], ['江東', 4773], ['品川', 4187], ['目黒', 3935], ['大田', 7259], ['世田谷', 10517], ['渋谷', 4159], ['中野', 4721], ['杉並', 5740], ['豊島', 3802], ['北', 3203], ['荒川', 2315], ['板橋', 5420], ['練馬', 6484], ['足立', 6846], ['葛飾', 5093], ['江戸川', 6141], ['八王子', 3523], ['立川', 1162], ['武蔵野', 1143], ['三鷹', 1382], ['青梅', 746], ['府中', 1582], ['昭島', 726], ['調布', 1743], ['町田', 2362], ['小金井', 829], ['小平', 977], ['日野', 1053], ['東村山', 717], ['国分寺', 702], ['国立', 374], ['福生', 429], ['狛江', 538], ['東大和', 412], ['清瀬', 340], ['東久留米', 586], ['武蔵村山', 332], ['多摩', 793], ['稲城', 471], ['羽村', 316], ['あきる野', 476], ['西東京', 1417], ['瑞穂', 169], ['日の出', 87], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11690], ['調査中', 77]]
    2021/05/07 [['千代田', 765], ['中央', 2348], ['港', 4728], ['新宿', 7974], ['文京', 2178], ['台東', 2664], ['墨田', 2917], ['江東', 4722], ['品川', 4146], ['目黒', 3905], ['大田', 7220], ['世田谷', 10441], ['渋谷', 4145], ['中野', 4694], ['杉並', 5704], ['豊島', 3779], ['北', 3180], ['荒川', 2288], ['板橋', 5381], ['練馬', 6432], ['足立', 6783], ['葛飾', 5067], ['江戸川', 6096], ['八王子', 3491], ['立川', 1150], ['武蔵野', 1131], ['三鷹', 1367], ['青梅', 746], ['府中', 1563], ['昭島', 723], ['調布', 1728], ['町田', 2338], ['小金井', 819], ['小平', 968], ['日野', 1040], ['東村山', 709], ['国分寺', 692], ['国立', 369], ['福生', 428], ['狛江', 533], ['東大和', 410], ['清瀬', 334], ['東久留米', 583], ['武蔵村山', 330], ['多摩', 784], ['稲城', 470], ['羽村', 315], ['あきる野', 473], ['西東京', 1408], ['瑞穂', 169], ['日の出', 87], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11578], ['調査中', 76]]
    2021/05/06 [['千代田', 762], ['中央', 2332], ['港', 4698], ['新宿', 7908], ['文京', 2160], ['台東', 2648], ['墨田', 2897], ['江東', 4682], ['品川', 4117], ['目黒', 3879], ['大田', 7192], ['世田谷', 10381], ['渋谷', 4122], ['中野', 4678], ['杉並', 5677], ['豊島', 3764], ['北', 3159], ['荒川', 2276], ['板橋', 5347], ['練馬', 6404], ['足立', 6741], ['葛飾', 5045], ['江戸川', 6067], ['八王子', 3468], ['立川', 1135], ['武蔵野', 1124], ['三鷹', 1358], ['青梅', 743], ['府中', 1550], ['昭島', 720], ['調布', 1706], ['町田', 2330], ['小金井', 814], ['小平', 960], ['日野', 1033], ['東村山', 703], ['国分寺', 683], ['国立', 366], ['福生', 427], ['狛江', 529], ['東大和', 407], ['清瀬', 328], ['東久留米', 581], ['武蔵村山', 330], ['多摩', 779], ['稲城', 465], ['羽村', 312], ['あきる野', 471], ['西東京', 1397], ['瑞穂', 169], ['日の出', 85], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11478], ['調査中', 75]]
    2021/05/05 [['千代田', 758], ['中央', 2317], ['港', 4679], ['新宿', 7882], ['文京', 2159], ['台東', 2628], ['墨田', 2892], ['江東', 4652], ['品川', 4108], ['目黒', 3855], ['大田', 7172], ['世田谷', 10332], ['渋谷', 4105], ['中野', 4659], ['杉並', 5658], ['豊島', 3753], ['北', 3149], ['荒川', 2268], ['板橋', 5320], ['練馬', 6363], ['足立', 6720], ['葛飾', 5038], ['江戸川', 6045], ['八王子', 3453], ['立川', 1127], ['武蔵野', 1121], ['三鷹', 1353], ['青梅', 742], ['府中', 1548], ['昭島', 718], ['調布', 1703], ['町田', 2329], ['小金井', 808], ['小平', 956], ['日野', 1029], ['東村山', 695], ['国分寺', 681], ['国立', 364], ['福生', 427], ['狛江', 528], ['東大和', 405], ['清瀬', 327], ['東久留米', 576], ['武蔵村山', 329], ['多摩', 778], ['稲城', 463], ['羽村', 312], ['あきる野', 468], ['西東京', 1391], ['瑞穂', 168], ['日の出', 84], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 1], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11402], ['調査中', 74]]
    2021/05/04 [['千代田', 753], ['中央', 2305], ['港', 4648], ['新宿', 7869], ['文京', 2159], ['台東', 2623], ['墨田', 2877], ['江東', 4637], ['品川', 4087], ['目黒', 3842], ['大田', 7147], ['世田谷', 10262], ['渋谷', 4092], ['中野', 4640], ['杉並', 5633], ['豊島', 3732], ['北', 3136], ['荒川', 2263], ['板橋', 5297], ['練馬', 6338], ['足立', 6701], ['葛飾', 5020], ['江戸川', 6024], ['八王子', 3437], ['立川', 1119], ['武蔵野', 1115], ['三鷹', 1346], ['青梅', 742], ['府中', 1547], ['昭島', 713], ['調布', 1695], ['町田', 2304], ['小金井', 807], ['小平', 955], ['日野', 1028], ['東村山', 694], ['国分寺', 678], ['国立', 364], ['福生', 426], ['狛江', 524], ['東大和', 401], ['清瀬', 323], ['東久留米', 576], ['武蔵村山', 329], ['多摩', 776], ['稲城', 460], ['羽村', 312], ['あきる野', 468], ['西東京', 1387], ['瑞穂', 168], ['日の出', 80], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 0], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11318], ['調査中', 74]]
    2021/05/03 [['千代田', 752], ['中央', 2294], ['港', 4631], ['新宿', 7853], ['文京', 2157], ['台東', 2619], ['墨田', 2865], ['江東', 4599], ['品川', 4069], ['目黒', 3827], ['大田', 7118], ['世田谷', 10219], ['渋谷', 4070], ['中野', 4625], ['杉並', 5608], ['豊島', 3712], ['北', 3119], ['荒川', 2259], ['板橋', 5283], ['練馬', 6301], ['足立', 6670], ['葛飾', 5001], ['江戸川', 6004], ['八王子', 3425], ['立川', 1115], ['武蔵野', 1108], ['三鷹', 1338], ['青梅', 741], ['府中', 1544], ['昭島', 713], ['調布', 1684], ['町田', 2293], ['小金井', 801], ['小平', 946], ['日野', 1021], ['東村山', 690], ['国分寺', 671], ['国立', 362], ['福生', 425], ['狛江', 520], ['東大和', 401], ['清瀬', 319], ['東久留米', 572], ['武蔵村山', 327], ['多摩', 771], ['稲城', 459], ['羽村', 311], ['あきる野', 465], ['西東京', 1380], ['瑞穂', 168], ['日の出', 80], ['檜原', 6], ['奥多摩', 24], ['大島', 23], ['利島', 0], ['新島', 0], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11263], ['調査中', 74]]
    2021/05/02 [['千代田', 751], ['中央', 2281], ['港', 4613], ['新宿', 7822], ['文京', 2138], ['台東', 2614], ['墨田', 2845], ['江東', 4580], ['品川', 4053], ['目黒', 3804], ['大田', 7106], ['世田谷', 10171], ['渋谷', 4053], ['中野', 4586], ['杉並', 5588], ['豊島', 3688], ['北', 3108], ['荒川', 2253], ['板橋', 5257], ['練馬', 6272], ['足立', 6648], ['葛飾', 4980], ['江戸川', 5975], ['八王子', 3381], ['立川', 1108], ['武蔵野', 1102], ['三鷹', 1335], ['青梅', 741], ['府中', 1539], ['昭島', 710], ['調布', 1679], ['町田', 2285], ['小金井', 798], ['小平', 941], ['日野', 1014], ['東村山', 687], ['国分寺', 665], ['国立', 359], ['福生', 423], ['狛江', 519], ['東大和', 397], ['清瀬', 318], ['東久留米', 562], ['武蔵村山', 326], ['多摩', 765], ['稲城', 456], ['羽村', 309], ['あきる野', 459], ['西東京', 1374], ['瑞穂', 168], ['日の出', 80], ['檜原', 6], ['奥多摩', 23], ['大島', 23], ['利島', 0], ['新島', 0], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11175], ['調査中', 74]]
    2021/05/01 [['千代田', 748], ['中央', 2269], ['港', 4585], ['新宿', 7802], ['文京', 2125], ['台東', 2604], ['墨田', 2829], ['江東', 4543], ['品川', 4031], ['目黒', 3785], ['大田', 7075], ['世田谷', 10090], ['渋谷', 4034], ['中野', 4559], ['杉並', 5558], ['豊島', 3672], ['北', 3090], ['荒川', 2237], ['板橋', 5228], ['練馬', 6220], ['足立', 6610], ['葛飾', 4955], ['江戸川', 5909], ['八王子', 3357], ['立川', 1096], ['武蔵野', 1091], ['三鷹', 1329], ['青梅', 737], ['府中', 1528], ['昭島', 708], ['調布', 1662], ['町田', 2266], ['小金井', 792], ['小平', 936], ['日野', 1001], ['東村山', 682], ['国分寺', 657], ['国立', 359], ['福生', 422], ['狛江', 512], ['東大和', 397], ['清瀬', 315], ['東久留米', 559], ['武蔵村山', 323], ['多摩', 760], ['稲城', 456], ['羽村', 306], ['あきる野', 457], ['西東京', 1366], ['瑞穂', 166], ['日の出', 80], ['檜原', 6], ['奥多摩', 23], ['大島', 23], ['利島', 0], ['新島', 0], ['神津島', 1], ['三宅', 4], ['御蔵島', 2], ['八丈', 7], ['青ヶ島', 0], ['小笠原', 4], ['都外', 11104], ['調査中', 74]]


次を実行するとデータが/content/drive/MyDrive/coronavirus/tokyo.jsonに保存されます。


```
import json
!mkdir -p /content/drive/MyDrive/coronavirus
with open('/content/drive/MyDrive/coronavirus/tokyo.json', 'w') as fout:
    json.dump(result, fout, default=str)
```

最後にエクセルなどでcsvとして貼り付けるのに便利なように出力しておきます。


```
print('Date', ','.join([item[0] for item in result[0][1]]), sep=',')
for r in result:
    print(','.join([r[0][:10].replace('-', '/')] + [str(item[1]) for item in r[1]]))
```

    Date,千代田,中央,港,新宿,文京,台東,墨田,江東,品川,目黒,大田,世田谷,渋谷,中野,杉並,豊島,北,荒川,板橋,練馬,足立,葛飾,江戸川,八王子,立川,武蔵野,三鷹,青梅,府中,昭島,調布,町田,小金井,小平,日野,東村山,国分寺,国立,福生,狛江,東大和,清瀬,東久留米,武蔵村山,多摩,稲城,羽村,あきる野,西東京,瑞穂,日の出,檜原,奥多摩,大島,利島,新島,神津島,三宅,御蔵島,八丈,青ヶ島,小笠原,都外,調査中※
    2020/03/31,3,19,39,22,4,15,5,10,24,21,15,44,18,15,28,9,4,2,4,20,8,6,8,4,0,3,7,1,1,0,1,6,1,1,3,0,0,0,0,0,1,0,1,0,0,1,3,0,8,0,0,0,0,0,0,0,0,0,0,0,0,0,20,116
    2020/04/01,3,19,40,30,4,19,5,12,24,23,16,54,21,18,30,10,7,2,6,21,10,14,8,4,0,4,7,1,1,0,1,10,2,2,4,0,0,0,0,0,1,0,1,0,0,3,3,0,9,0,0,0,0,0,0,0,0,0,0,0,0,0,20,118
    2020/04/02,4,19,50,36,4,23,7,12,29,29,18,67,24,19,34,13,7,3,9,25,11,14,11,4,0,4,7,1,1,0,1,10,2,2,4,0,0,0,0,0,1,0,1,0,0,3,3,0,9,0,0,0,0,0,0,0,0,0,0,0,0,0,22,141
    2020/04/03,4,20,58,38,6,25,10,13,30,34,19,79,26,24,37,14,7,3,13,34,12,16,11,5,0,4,7,1,1,1,1,10,2,3,4,0,0,0,0,3,1,0,2,0,0,3,3,0,9,0,0,0,0,0,0,0,0,0,0,0,0,0,22,158
    2020/04/04,5,21,67,39,7,27,11,14,38,40,28,93,32,29,44,17,9,5,15,39,20,18,13,6,0,4,8,1,1,1,3,11,2,3,6,1,0,0,0,3,1,1,2,1,0,3,3,0,9,0,0,0,0,0,0,0,0,0,0,0,0,0,27,162
    2020/04/05,6,25,83,60,8,29,11,22,41,42,37,104,37,33,56,20,10,6,20,42,22,19,15,9,2,4,8,1,1,2,4,14,3,3,6,1,0,0,0,3,1,1,2,1,0,3,3,1,10,0,0,0,0,0,0,0,0,0,0,0,0,0,32,170
    2020/04/06,7,28,87,72,8,31,11,22,41,44,44,113,39,41,60,20,10,7,23,46,25,26,16,9,2,5,8,1,1,2,4,16,3,3,8,1,0,0,0,3,2,1,2,1,0,3,3,1,9,0,0,0,0,0,0,0,0,0,0,0,0,0,36,170
    2020/04/07,9,29,89,80,10,32,14,23,49,49,45,118,41,43,67,27,11,7,24,48,27,29,16,10,3,5,9,1,1,3,5,16,4,3,8,1,0,0,0,3,2,1,2,1,0,3,3,1,9,0,0,0,0,0,0,0,0,0,0,0,0,0,39,174
    2020/04/08,9,31,107,84,12,35,14,26,56,51,51,124,49,46,76,31,13,7,30,48,28,29,18,11,3,5,12,1,5,3,6,20,5,5,8,1,2,0,0,5,3,1,2,1,0,3,5,1,10,0,0,0,0,0,0,0,0,0,0,0,0,0,42,203
    2020/04/09,9,31,126,99,16,40,21,39,56,54,52,147,53,50,77,39,16,8,32,52,33,34,21,11,5,5,12,1,6,3,8,20,5,6,9,1,3,2,0,5,3,2,2,1,0,3,5,1,10,0,0,0,0,0,0,0,0,0,0,0,0,0,46,236
    2020/04/10,10,33,143,133,20,42,24,44,70,56,58,173,64,53,85,45,17,10,37,54,35,35,25,13,5,6,12,1,6,3,9,20,8,6,10,1,3,2,0,6,4,3,3,1,0,4,5,1,10,0,0,0,0,0,0,0,0,0,0,0,0,0,48,249
    2020/04/11,10,35,155,139,22,46,25,51,84,58,65,191,69,62,92,49,19,11,38,60,44,44,39,15,8,7,13,2,10,3,12,21,8,6,11,1,3,2,0,7,4,3,3,1,5,4,5,1,11,0,0,0,0,0,0,0,0,0,0,0,0,0,50,278
    2020/04/12,12,39,160,170,29,47,27,60,91,64,65,193,71,70,98,51,20,17,47,66,47,45,43,16,8,7,13,2,10,3,12,22,8,6,11,1,3,2,0,7,4,5,5,1,5,4,5,1,11,0,0,0,0,0,0,0,0,0,0,0,0,0,53,367
    2020/04/13,12,39,163,177,23,47,28,52,94,72,67,205,72,67,97,49,23,13,47,60,46,44,43,19,8,7,14,2,14,3,13,23,8,6,12,3,3,3,0,7,4,6,6,1,7,5,5,1,13,0,0,0,0,0,0,0,0,0,0,0,0,0,58,367
    2020/04/14,12,51,168,180,26,49,35,56,104,76,71,234,81,70,107,62,30,13,50,67,50,46,47,20,8,8,14,2,15,3,15,24,8,6,12,4,3,3,0,7,4,6,7,1,8,6,5,1,13,0,0,0,0,0,0,0,0,0,0,0,0,0,60,371
    2020/04/15,16,53,172,187,28,49,36,57,104,80,77,242,89,77,115,65,30,13,51,80,51,48,52,21,8,10,16,2,20,3,20,25,8,8,12,4,5,3,0,8,4,6,11,1,12,6,5,2,13,0,0,0,0,0,0,0,0,0,0,0,0,0,63,377
    2020/04/16,18,56,179,205,36,53,47,59,111,83,79,247,94,79,120,71,34,13,51,88,57,59,59,22,9,10,18,2,20,3,22,26,8,9,12,3,5,3,0,9,4,6,11,1,13,6,5,2,15,0,0,0,0,0,0,0,0,0,0,0,0,0,67,384
    2020/04/17,19,58,197,219,39,53,48,65,123,92,79,259,105,82,127,79,35,13,57,90,65,64,70,23,13,12,20,3,27,3,23,28,9,9,13,4,6,4,0,9,5,9,11,1,16,6,5,2,18,1,0,0,0,0,0,0,0,0,0,0,0,0,76,400
    2020/04/18,19,65,208,227,41,55,51,68,130,94,90,277,113,83,134,88,44,13,64,100,69,77,77,30,13,13,20,3,28,5,24,30,9,11,13,5,6,4,0,9,5,9,11,1,19,6,5,2,19,1,0,0,0,0,0,0,0,0,0,0,0,0,77,410
    2020/04/19,19,67,212,227,45,59,57,73,132,97,96,293,118,90,145,88,45,13,66,101,70,78,81,31,13,14,20,3,30,6,25,31,10,12,13,5,6,4,0,10,5,10,12,1,19,6,5,3,21,1,0,0,0,0,0,0,0,0,0,0,0,0,82,412
    2020/04/20,19,68,215,255,46,59,60,75,138,101,97,305,119,90,146,88,47,14,69,112,70,80,84,33,13,14,21,3,32,6,26,31,10,13,14,5,6,4,0,11,5,11,12,1,19,6,5,3,25,1,0,0,0,0,0,0,0,0,0,0,0,0,85,412
    2020/04/21,19,70,224,259,50,59,64,81,141,102,100,314,132,93,154,93,51,17,75,114,76,81,87,35,13,14,22,3,33,6,26,34,10,13,16,5,7,4,1,11,5,11,12,1,21,6,5,3,26,1,0,0,0,0,0,0,0,0,0,0,0,0,86,421
    2020/04/22,22,73,238,260,51,59,67,85,145,102,109,317,133,136,158,96,53,18,76,121,77,83,90,35,13,14,23,3,33,6,27,34,10,14,16,5,7,4,1,11,5,11,12,1,21,6,5,3,26,1,0,0,0,0,0,0,0,0,0,0,0,0,100,422
    2020/04/23,22,74,242,267,53,60,74,88,150,103,120,326,138,145,161,104,55,20,77,125,80,87,96,35,13,14,24,3,37,6,29,34,11,14,16,5,10,4,1,11,6,11,12,1,23,6,5,4,27,1,0,0,0,0,0,0,0,0,0,0,0,0,105,437
    2020/04/24,23,79,251,279,54,64,78,93,153,104,138,338,142,147,165,108,60,22,81,150,86,90,100,36,13,14,24,3,40,6,30,36,12,15,16,6,10,4,1,11,6,11,12,1,24,8,5,5,28,1,0,0,0,0,0,0,0,0,0,0,0,0,110,440
    2020/04/25,23,79,251,282,57,65,85,95,155,109,140,349,145,148,173,109,61,22,84,157,90,93,102,38,13,15,25,3,45,7,30,39,13,16,17,6,12,5,1,12,6,11,12,1,28,9,5,6,29,1,0,0,0,0,0,0,0,0,0,0,0,0,115,442
    2020/04/26,23,80,261,289,59,65,86,101,156,109,140,358,147,148,176,109,61,22,85,157,90,93,107,38,13,15,25,3,46,7,31,39,14,16,17,6,11,6,1,12,6,11,12,1,28,9,5,6,30,1,0,0,0,0,0,0,0,0,0,0,0,0,118,459
    2020/04/27,23,84,261,296,59,65,87,101,156,115,140,366,149,151,177,109,61,22,86,157,90,93,107,38,13,15,25,3,46,7,31,40,14,16,17,7,11,6,1,12,6,11,12,1,28,9,5,6,31,1,0,0,0,0,0,0,0,0,0,0,0,0,119,461
    2020/04/28,24,87,264,299,60,66,93,104,161,115,142,377,152,152,182,116,68,22,86,162,95,95,111,39,13,15,25,3,49,7,31,41,14,16,18,7,11,6,1,12,6,11,12,1,28,9,5,7,31,1,0,0,0,0,0,0,0,0,0,0,0,0,120,487
    2020/04/29,24,87,266,299,60,66,95,105,167,115,146,379,153,155,183,116,68,27,89,162,99,96,113,39,13,16,26,3,50,7,32,41,14,16,18,7,11,6,1,12,6,11,12,1,28,9,5,7,33,1,0,0,0,0,0,0,0,0,0,0,0,0,124,487
    2020/04/30,24,87,268,304,60,66,95,106,167,121,148,384,153,155,183,116,68,29,91,168,104,97,113,39,13,16,26,3,54,7,32,41,14,16,18,7,11,6,1,12,6,11,12,1,28,9,5,7,34,1,0,0,0,0,0,0,0,0,0,0,0,0,128,487
    2020/05/01,28,91,273,312,69,74,120,125,170,126,157,394,155,155,198,126,74,46,97,207,109,106,118,39,13,16,26,3,55,8,34,43,16,16,18,8,11,6,1,13,6,11,11,1,28,8,5,7,34,1,1,0,0,0,0,0,0,0,0,0,0,0,141,529
    2020/05/02,28,89,273,323,62,75,116,119,170,126,177,401,158,165,199,124,74,40,95,183,110,109,121,41,13,16,26,3,63,8,34,45,14,16,18,9,11,6,1,14,6,11,12,2,29,10,5,7,34,1,0,0,0,0,0,0,0,0,0,0,0,0,142,542
    2020/05/03,30,89,285,329,63,75,116,126,174,128,191,406,160,166,202,124,75,45,97,187,112,109,121,41,13,16,27,3,66,8,35,45,14,16,18,10,11,6,1,17,6,11,12,2,29,10,5,7,36,1,0,0,0,0,0,0,0,0,0,0,0,0,144,547
    2020/05/04,31,90,289,336,70,75,116,128,177,129,197,415,160,173,215,127,75,45,99,191,114,109,123,41,13,17,27,3,66,9,35,46,14,16,18,10,11,6,1,19,6,12,12,2,29,10,5,7,40,1,0,0,0,0,0,0,0,0,0,0,0,0,147,547
    2020/05/05,36,93,292,338,75,76,121,132,178,130,197,414,163,173,219,128,76,45,99,192,116,110,124,42,13,18,27,3,66,9,35,46,14,16,18,10,12,6,1,19,6,13,12,2,29,10,5,7,41,1,0,0,0,0,0,0,0,0,0,0,0,0,152,550
    2020/05/06,36,93,292,353,76,76,121,135,178,131,197,415,164,173,221,128,76,45,99,198,117,110,124,42,13,18,27,3,66,9,35,47,14,16,18,10,12,6,1,19,6,13,12,2,29,10,5,7,42,1,0,0,0,0,0,0,0,0,0,0,0,0,156,551
    2020/05/07,36,93,292,353,78,76,121,136,178,135,197,417,165,174,221,131,80,45,99,198,118,110,124,42,13,18,27,3,66,9,35,47,14,16,18,10,12,6,1,20,6,13,12,2,29,10,5,7,42,1,0,0,0,0,0,0,0,0,0,0,0,0,159,551
    2020/05/08,36,93,292,358,79,77,125,138,178,135,200,424,167,174,221,131,83,45,99,200,118,110,125,42,13,18,27,4,67,9,35,47,14,16,19,10,12,6,1,20,6,14,12,2,29,10,5,7,42,1,0,0,0,0,0,0,0,0,0,0,0,0,162,552
    2020/05/09,36,93,292,362,79,77,127,138,179,135,202,425,171,176,222,134,86,45,100,201,118,116,125,42,13,18,27,4,67,9,35,48,15,16,19,10,12,6,1,20,6,14,12,2,29,10,5,7,42,1,0,0,0,0,0,0,0,0,1,0,0,0,164,552
    2020/05/10,37,93,294,363,79,78,128,139,179,136,202,430,171,177,223,134,86,45,101,202,119,116,125,42,13,18,27,4,67,9,35,50,15,16,19,10,12,6,1,21,6,14,12,2,29,10,5,7,42,1,0,0,0,0,0,0,0,0,1,0,0,0,165,552
    2020/05/11,41,97,308,375,86,172,142,212,180,162,235,450,172,214,226,136,86,55,133,233,122,121,138,42,13,18,27,4,67,9,35,50,17,16,19,10,12,6,1,21,6,14,12,2,28,10,5,7,42,1,0,0,0,0,0,0,0,0,1,0,0,0,169,199
    2020/05/12,41,110,308,389,90,172,143,212,184,162,238,451,175,214,245,137,86,78,135,237,123,121,139,42,13,18,27,5,67,9,35,52,17,16,19,10,12,6,1,22,6,14,12,2,28,10,5,7,42,1,0,0,0,0,0,0,0,0,1,0,0,0,199,99
    2020/05/13,42,110,308,389,90,172,146,212,184,163,238,451,175,214,245,141,88,78,137,236,123,132,139,42,14,17,27,6,69,9,35,52,17,18,20,13,12,6,1,22,6,15,14,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,199,60
    2020/05/14,43,110,308,394,91,172,146,216,184,163,239,453,175,218,247,142,96,79,138,239,151,132,140,42,14,17,28,6,70,9,35,52,17,18,20,13,12,6,1,22,6,15,15,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,199,24
    2020/05/15,43,110,308,395,91,172,146,216,184,163,239,454,175,221,247,142,97,79,138,241,151,132,140,42,14,17,28,6,70,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,199,24
    2020/05/16,43,111,308,397,92,172,146,219,184,163,239,455,176,221,248,142,97,79,139,243,151,132,140,42,14,17,28,6,70,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,200,24
    2020/05/17,43,111,308,398,92,172,146,219,185,163,240,455,176,223,248,142,97,79,139,243,151,132,140,42,14,17,28,6,70,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,200,24
    2020/05/18,43,112,308,399,93,172,147,220,185,163,240,456,176,223,249,142,97,79,139,244,151,132,140,42,14,17,28,6,70,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,11,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,202,24
    2020/05/19,43,112,309,400,93,172,147,220,185,163,241,457,176,223,249,142,97,79,139,244,151,132,140,42,14,17,28,6,70,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,202,24
    2020/05/20,43,112,309,402,93,172,147,220,186,163,241,458,176,223,250,142,97,79,139,244,151,132,140,41,14,17,28,6,71,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,0,0,0,0,0,0,0,0,1,0,0,0,202,24
    2020/05/21,43,111,309,406,94,171,152,221,187,164,245,458,177,225,249,143,97,80,139,274,154,133,140,42,14,17,28,6,71,9,35,53,17,18,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,206,23
    2020/05/22,43,111,309,407,94,171,152,221,187,164,245,458,177,225,249,143,97,80,139,275,154,133,140,42,14,17,28,6,71,9,35,53,17,19,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,206,23
    2020/05/23,43,111,309,407,94,171,152,221,187,164,245,459,177,225,249,143,97,80,139,275,154,133,140,42,14,17,28,6,71,9,35,53,17,20,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,206,23
    2020/05/24,43,112,309,412,94,171,152,222,188,164,245,459,177,225,250,143,98,80,139,275,154,134,140,42,14,17,29,6,72,9,35,53,17,21,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,206,23
    2020/05/25,43,112,309,412,94,172,152,226,188,164,245,459,177,225,250,143,98,80,139,275,154,134,141,42,14,17,29,6,72,9,36,53,17,21,20,13,12,6,1,22,6,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,207,23
    2020/05/26,43,112,309,412,94,172,152,226,188,165,245,460,178,225,250,145,99,80,139,275,154,134,142,42,14,17,29,6,72,9,37,53,17,21,20,13,12,6,1,22,7,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,207,23
    2020/05/27,43,112,312,413,94,172,152,226,188,165,246,462,178,226,251,145,99,80,139,275,155,134,142,42,14,17,29,6,72,9,37,53,17,21,20,13,12,6,1,22,8,15,15,2,37,12,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,207,23
    2020/05/28,43,112,315,415,94,172,152,226,188,165,246,462,179,227,252,146,99,80,140,275,155,135,142,42,14,17,29,6,72,9,37,53,17,21,20,13,13,6,1,22,8,15,14,2,37,11,5,7,46,1,1,0,0,0,0,0,0,0,1,0,0,0,208,23
    2020/05/29,43,112,317,417,95,173,152,227,188,166,246,462,179,227,255,147,100,80,142,276,155,135,143,42,14,17,29,6,72,9,37,53,18,23,20,13,13,6,1,22,8,15,15,2,37,12,5,7,47,1,1,0,0,0,0,0,0,0,1,0,0,0,211,23
    2020/05/30,43,112,318,418,95,174,152,228,188,166,246,462,179,228,255,148,100,80,143,276,156,135,144,42,14,17,29,6,72,9,37,53,21,23,20,13,13,7,1,22,8,15,15,2,37,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,211,23
    2020/05/31,43,112,318,421,95,174,152,228,188,166,246,463,179,228,255,148,100,80,143,276,156,135,144,43,14,17,29,6,72,9,37,53,21,23,20,13,13,7,1,22,8,15,15,2,37,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,211,23
    2020/06/01,43,112,318,426,95,174,152,228,188,166,246,466,179,229,255,149,100,80,143,276,156,135,144,43,14,18,29,6,72,9,37,53,21,23,20,13,13,7,1,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,212,23
    2020/06/02,43,114,324,427,95,175,152,228,189,167,246,467,182,231,256,149,100,81,143,277,157,135,145,43,14,18,29,6,74,9,37,53,28,23,20,13,13,7,1,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,214,23
    2020/06/03,43,114,325,427,95,175,152,228,190,168,246,469,183,234,256,149,100,81,143,277,157,135,147,43,14,18,29,6,74,9,37,53,28,23,20,13,13,7,1,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,215,23
    2020/06/04,44,114,327,435,95,175,153,228,190,169,246,476,185,234,256,151,100,81,144,278,157,135,147,43,14,18,29,6,74,9,37,53,28,23,20,14,13,7,1,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,216,23
    2020/06/05,44,114,328,441,95,175,153,229,190,170,246,478,185,238,257,152,100,81,145,278,157,135,147,43,15,18,29,6,74,9,37,53,28,23,20,14,13,7,1,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,217,23
    2020/06/06,44,114,331,453,95,175,153,229,190,170,247,481,185,240,258,153,100,82,145,278,157,135,147,43,15,18,29,6,74,9,37,53,28,24,20,14,13,7,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,217,23
    2020/06/07,44,114,331,455,95,175,153,229,190,170,247,487,185,241,259,153,100,82,145,280,157,135,147,43,15,18,29,6,74,9,37,53,28,24,20,15,13,8,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,217,23
    2020/06/08,44,114,332,458,95,175,153,229,190,171,247,487,185,245,261,153,100,82,145,280,157,135,147,43,15,18,29,6,75,9,37,53,28,25,20,15,13,8,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,217,23
    2020/06/09,44,114,333,459,97,175,153,230,190,172,249,487,188,245,261,153,100,82,145,280,157,136,147,43,15,18,29,6,75,9,37,53,28,25,20,15,13,8,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,217,23
    2020/06/10,44,114,336,463,97,175,153,230,190,172,249,491,188,248,262,153,100,82,147,280,157,136,147,43,15,18,29,6,75,9,37,53,28,25,20,15,13,8,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,218,23
    2020/06/11,44,114,337,472,98,175,153,232,190,172,249,492,188,250,262,153,100,82,147,280,157,136,148,44,15,18,29,6,77,9,37,54,29,25,20,15,13,8,2,22,8,15,15,2,38,12,5,7,48,1,1,0,0,0,0,0,0,0,1,0,0,0,218,23
    2020/06/12,44,114,337,484,98,175,153,232,190,172,249,493,189,251,262,155,100,82,148,280,157,136,148,44,15,18,29,6,77,9,37,54,30,25,20,15,13,8,2,22,8,15,15,2,38,12,5,7,49,1,1,0,0,0,0,0,0,0,1,0,0,0,223,23
    2020/06/13,44,115,339,489,98,175,153,233,191,172,249,495,192,252,263,155,100,82,148,281,158,136,148,44,15,18,29,6,77,9,37,55,30,26,20,15,14,8,2,22,8,15,15,2,38,12,5,7,50,1,1,0,0,0,0,0,0,0,1,0,0,0,224,23
    2020/06/14,44,115,340,511,98,175,153,233,192,172,249,502,194,252,263,156,100,83,149,282,158,137,149,45,15,18,30,8,77,9,37,55,31,26,20,15,14,8,2,23,8,15,15,2,38,12,5,7,51,1,1,0,0,0,0,0,0,0,1,0,0,0,225,23
    2020/06/15,44,115,342,531,99,175,153,234,193,173,251,503,194,255,263,161,100,83,151,284,158,138,150,46,16,19,30,8,78,9,37,55,31,26,20,15,14,8,2,23,8,15,15,2,38,12,5,7,51,1,1,0,0,0,0,0,0,0,1,0,0,0,226,23
    2020/06/16,44,115,342,540,99,175,153,234,193,173,253,503,194,259,266,163,100,83,153,285,158,138,152,46,16,19,30,8,78,9,37,55,31,26,20,15,14,8,2,24,8,15,14,2,38,11,5,7,51,1,1,0,0,0,0,0,0,0,1,0,0,0,227,23
    2020/06/17,44,115,343,542,99,175,153,235,194,173,253,506,194,260,266,163,101,83,154,286,158,138,153,46,16,20,30,8,78,9,37,56,31,26,20,15,14,8,2,25,8,15,14,2,38,11,5,7,51,1,1,0,0,0,0,0,0,0,1,0,0,0,227,23
    2020/06/18,44,116,346,551,99,175,154,235,195,175,253,510,194,260,268,166,101,83,158,287,159,139,154,46,16,21,30,9,79,9,37,56,32,26,20,15,14,8,2,25,8,15,14,2,38,11,5,7,51,1,1,0,0,0,0,0,0,0,1,0,0,0,230,23
    2020/06/19,44,117,346,561,99,175,154,235,195,175,254,513,195,261,268,168,101,83,160,290,159,139,155,46,16,22,30,9,79,9,37,56,33,27,20,15,14,8,2,25,8,15,15,2,38,11,5,10,51,1,1,0,0,0,0,0,0,0,1,0,0,0,233,23
    2020/06/20,44,117,346,579,99,176,156,235,196,176,255,514,196,264,266,171,101,83,160,290,162,139,155,47,17,22,30,9,79,9,37,56,33,27,20,15,14,8,2,25,8,15,15,2,38,11,5,10,51,1,1,0,0,0,0,0,0,0,1,0,0,0,234,23
    2020/06/21,44,117,347,594,99,177,156,235,196,176,255,517,196,266,269,173,101,84,164,290,163,139,155,47,17,22,30,9,79,10,37,56,34,28,20,15,14,8,2,25,8,15,15,2,38,11,5,10,51,2,1,0,0,0,0,0,0,0,1,0,0,0,235,23
    2020/06/22,44,121,347,604,99,178,156,235,196,176,255,518,197,271,271,173,101,84,163,290,163,139,155,48,17,22,30,9,79,10,37,56,34,27,20,15,14,8,2,25,8,15,16,2,38,11,5,10,51,2,1,0,0,0,0,0,0,0,1,0,0,0,237,23
    2020/06/23,44,121,349,607,100,178,156,235,196,178,257,518,198,272,272,178,101,85,165,294,165,140,155,49,17,22,30,9,79,10,37,56,34,27,20,15,14,8,2,25,8,15,16,2,38,11,5,10,52,2,1,0,0,0,0,0,0,0,1,0,0,0,238,23
    2020/06/24,45,121,349,617,101,178,157,235,197,179,259,522,200,273,272,181,103,85,168,296,169,141,158,49,18,22,30,10,79,10,37,57,34,27,20,16,14,8,2,26,8,15,16,2,38,11,5,10,52,2,1,0,0,0,0,0,0,0,1,0,0,0,246,23
    2020/06/25,45,123,353,632,102,179,158,236,198,179,259,525,200,273,273,188,104,85,169,298,170,141,158,51,18,22,30,10,79,10,37,58,34,27,20,16,15,8,2,26,8,15,16,2,38,11,5,10,52,2,1,0,0,0,0,0,0,0,1,0,0,0,248,23
    2020/06/26,46,124,353,644,102,179,159,238,199,179,260,529,201,278,277,194,106,85,173,299,174,141,159,51,18,22,30,10,79,10,37,58,34,27,20,16,15,8,2,26,8,15,16,2,38,11,5,10,53,2,1,0,0,0,0,0,0,0,1,0,0,0,250,23
    2020/06/27,46,124,353,658,103,180,161,239,199,179,263,529,204,282,277,198,106,85,179,299,178,141,162,51,19,22,32,10,79,10,38,58,36,27,20,16,15,8,2,26,8,15,17,2,38,11,5,10,54,2,1,0,0,0,0,0,0,0,1,0,0,0,253,23
    2020/06/28,47,124,356,676,104,182,161,243,199,180,263,530,205,284,279,205,107,85,180,304,179,141,163,51,19,22,32,10,79,10,38,59,36,27,20,16,15,8,2,27,8,16,17,2,38,11,5,10,54,2,1,0,0,0,0,0,0,0,1,0,0,0,257,23
    2020/06/29,47,125,359,700,105,182,161,243,199,180,263,531,205,292,280,209,107,90,181,306,179,141,163,51,19,22,34,10,79,10,38,60,36,27,20,16,15,8,2,27,8,16,17,2,38,12,5,10,54,2,1,0,0,0,0,0,0,0,1,0,0,0,260,23
    2020/06/30,47,125,360,702,105,183,163,244,201,180,265,532,209,293,284,210,107,90,186,314,184,142,163,52,20,24,34,10,79,12,38,60,36,27,20,16,15,8,2,27,8,17,17,2,38,12,5,10,55,2,1,0,0,0,0,0,0,0,1,0,0,0,265,23
    2020/07/01,47,125,364,711,105,184,164,244,201,183,268,533,212,297,286,223,110,91,192,314,185,142,166,53,20,24,34,10,80,12,38,60,36,28,20,16,15,8,2,27,8,17,17,2,38,12,5,10,55,2,1,0,0,0,0,0,0,0,1,0,0,0,271,23
    2020/07/02,47,127,366,728,106,185,164,245,203,184,269,539,214,302,292,241,112,97,198,321,188,142,166,54,20,25,34,10,80,13,38,62,37,29,20,18,15,8,2,28,8,17,18,2,38,12,5,10,55,2,1,0,0,0,0,0,0,0,1,0,0,0,278,23
    2020/07/03,47,127,369,770,106,187,165,248,206,185,270,544,218,305,297,248,120,98,201,325,190,145,169,55,20,26,35,10,82,13,38,64,37,29,21,20,15,8,2,28,8,17,18,2,38,12,5,10,56,2,1,0,0,0,0,0,0,0,1,0,0,0,287,23
    2020/07/04,47,129,376,814,107,190,166,249,207,187,271,550,223,319,302,252,123,100,205,327,192,146,170,55,21,26,38,10,82,13,38,64,38,30,21,20,15,8,2,28,8,17,19,2,39,12,5,10,58,2,1,0,0,0,0,0,0,0,1,0,0,0,296,23
    2020/07/05,47,131,379,851,107,193,168,253,207,188,275,555,223,320,308,257,128,108,210,332,192,149,172,56,21,26,38,10,82,13,38,64,38,31,21,21,15,8,2,28,8,18,19,2,39,12,5,10,59,2,1,0,0,0,0,0,0,0,1,0,0,0,301,23
    2020/07/06,47,133,381,888,107,193,168,263,207,189,277,558,224,332,310,262,129,108,214,337,195,149,173,56,24,26,40,10,83,13,38,64,38,31,21,21,15,8,3,28,8,18,19,2,39,12,5,11,59,2,1,0,0,0,0,0,0,0,1,0,0,0,304,23
    2020/07/07,47,135,387,898,110,193,170,266,208,190,281,562,235,337,313,270,130,108,219,344,196,150,176,58,25,27,41,10,84,13,39,65,39,32,22,21,17,8,3,28,8,18,19,2,40,13,5,11,62,2,1,0,0,0,0,0,0,0,1,0,0,0,311,23
    2020/07/08,49,135,390,900,112,195,171,268,208,191,284,576,238,339,314,272,130,108,225,351,197,152,184,60,26,28,42,10,84,13,39,66,39,32,22,21,17,8,3,28,8,18,20,2,40,14,5,11,63,2,1,0,0,0,0,0,0,0,1,0,0,0,313,23
    2020/07/09,50,136,397,964,114,197,175,272,211,194,289,589,250,355,320,283,136,112,231,355,208,154,190,64,27,28,42,10,84,13,42,67,41,33,25,22,19,9,3,28,9,18,20,2,40,14,5,12,64,2,1,0,0,0,0,0,0,0,1,0,0,0,322,23
    2020/07/10,50,137,401,1056,115,203,176,278,216,197,291,596,259,373,326,298,142,119,235,359,213,157,192,67,29,31,44,10,86,13,42,67,41,33,26,24,21,9,3,29,9,18,23,2,40,14,5,13,65,2,1,0,0,0,0,0,0,0,1,0,0,0,335,23
    2020/07/11,50,143,403,1101,125,208,177,285,221,201,294,614,266,389,332,301,146,122,243,372,219,162,196,70,31,33,45,10,87,13,43,67,44,33,27,25,21,9,3,29,9,18,24,2,40,14,5,13,68,2,1,0,0,0,0,0,0,0,1,0,0,0,341,23
    2020/07/12,50,145,409,1146,141,212,184,287,227,205,299,629,267,398,353,306,152,128,248,378,234,162,203,70,31,33,45,10,87,13,45,69,44,33,27,25,21,9,3,29,9,18,25,2,40,14,5,13,70,2,1,0,0,0,0,0,0,0,1,0,0,0,347,23
    2020/07/13,50,147,413,1179,142,213,185,288,229,211,302,638,278,411,355,306,152,128,248,382,242,163,203,70,31,34,45,11,87,13,45,69,45,35,27,25,21,9,3,29,9,18,25,2,40,15,5,13,71,2,1,0,0,0,0,0,0,0,1,0,0,0,357,23
    2020/07/14,50,148,413,1199,148,217,185,291,232,213,306,646,288,420,359,310,153,131,254,385,253,173,206,73,31,34,47,11,90,13,48,70,46,36,28,25,22,9,4,29,9,18,26,3,41,16,5,13,74,2,1,0,0,0,0,0,0,0,1,0,0,0,361,23
    2020/07/15,54,149,425,1207,161,218,186,293,236,218,307,669,295,428,363,314,160,131,263,392,263,176,208,73,32,36,47,11,91,15,48,70,47,36,28,25,22,9,4,29,9,18,26,4,41,16,5,13,78,2,1,0,0,0,0,0,0,0,1,0,0,0,378,23
    2020/07/16,54,152,436,1296,168,221,191,305,239,221,317,681,303,434,374,321,165,132,271,402,276,180,211,76,32,37,51,11,92,15,50,72,48,39,30,26,24,11,4,31,11,19,31,5,43,17,5,13,81,2,1,0,0,0,0,0,0,0,1,0,0,0,389,23
    2020/07/17,57,156,436,1377,176,222,194,317,244,223,322,699,307,456,380,327,165,136,281,411,291,186,221,92,35,39,55,13,98,15,52,75,48,41,30,26,24,11,4,32,11,19,34,6,44,18,5,13,84,2,1,0,0,0,0,0,0,0,1,0,0,0,398,23
    2020/07/18,60,159,453,1423,182,229,195,320,256,233,327,717,319,469,390,342,169,139,289,423,319,189,235,102,36,39,57,13,101,15,54,77,49,43,30,27,25,11,4,32,11,19,35,6,45,18,5,13,84,2,1,0,0,0,0,0,0,0,1,0,0,0,408,23
    2020/07/19,61,162,459,1470,183,231,198,326,261,238,332,737,327,471,395,345,177,141,291,426,328,190,241,108,38,39,57,13,105,15,58,80,50,43,30,27,25,11,5,32,12,20,35,6,45,18,5,13,86,2,1,0,0,0,0,0,0,0,1,0,0,0,418,23
    2020/07/20,61,164,463,1502,186,232,201,327,263,245,333,747,329,487,399,345,178,143,307,434,338,194,245,113,40,39,59,14,107,15,58,80,50,45,30,27,26,11,7,32,12,20,36,6,45,18,5,13,87,2,1,0,0,0,0,0,0,0,1,0,0,0,434,23
    2020/07/21,64,167,466,1527,189,235,204,332,273,249,340,756,334,497,412,356,179,143,313,437,370,202,265,116,42,41,61,14,112,15,60,82,50,47,33,27,26,11,7,32,12,20,36,6,48,18,5,13,90,2,1,0,0,0,0,0,0,0,1,0,0,0,455,23
    2020/07/22,65,171,485,1550,196,238,205,337,296,263,346,775,345,503,419,360,191,146,327,444,382,207,275,117,43,41,64,14,113,17,61,84,50,47,33,27,26,11,8,32,12,20,37,6,48,18,5,14,92,2,1,0,0,0,0,0,0,0,1,0,0,0,461,23
    2020/07/23,65,173,496,1620,202,250,213,347,311,275,357,796,355,512,426,371,202,150,341,456,419,212,293,120,48,43,65,14,115,18,62,86,52,48,37,29,26,11,9,34,12,20,37,6,51,18,5,14,95,2,1,0,0,0,0,0,0,0,1,0,0,0,476,23
    2020/07/24,68,180,501,1670,206,252,218,355,315,288,361,814,365,535,438,376,210,155,360,461,428,214,300,121,49,43,66,14,117,18,63,87,53,52,38,31,27,12,10,35,12,21,40,6,52,18,5,14,96,2,1,0,0,0,0,0,0,0,1,0,0,0,483,23
    2020/07/25,70,182,520,1720,207,260,225,358,318,296,367,831,381,540,446,380,220,158,374,467,457,229,306,125,49,46,67,14,124,18,67,89,53,56,38,33,28,12,10,38,13,22,41,6,53,18,5,14,96,2,1,0,0,0,0,0,0,0,1,0,0,0,501,23
    2020/07/26,72,185,522,1724,208,266,230,372,344,307,371,858,392,547,469,387,225,159,392,470,464,229,318,136,49,46,68,14,125,19,68,91,56,56,39,33,28,12,11,38,14,22,43,6,54,18,6,14,98,2,1,0,0,0,0,0,0,0,1,0,0,0,512,23
    2020/07/27,72,193,528,1733,208,267,230,375,346,314,375,868,393,556,472,398,228,168,396,484,469,232,320,137,49,46,69,14,126,19,69,91,56,58,39,34,28,12,11,39,14,22,44,6,54,18,6,14,99,2,1,0,0,0,0,0,0,0,1,0,0,0,519,23
    2020/07/28,73,202,534,1749,220,275,234,387,360,327,381,882,399,567,484,404,235,175,403,492,483,236,334,138,54,47,70,14,129,21,74,97,57,59,41,36,29,14,11,40,15,22,44,7,55,20,6,14,105,2,1,0,0,0,0,0,0,0,1,0,0,0,529,23
    2020/07/29,74,203,547,1762,223,282,236,391,383,334,389,921,414,571,490,410,244,179,416,499,489,238,343,141,56,49,76,14,129,22,76,103,60,59,41,36,29,14,11,40,15,23,45,7,55,21,6,14,111,2,1,0,0,0,0,0,0,0,1,0,0,0,543,23
    2020/07/30,75,215,562,1773,232,285,237,401,412,345,402,971,433,581,499,421,253,183,435,514,504,247,357,145,56,51,81,14,131,23,82,108,60,63,43,39,29,14,12,41,15,23,46,7,58,21,6,14,120,2,1,0,0,0,0,0,0,0,1,0,0,0,561,23
    2020/07/31,78,230,578,1890,239,291,244,410,427,361,413,997,457,614,509,427,260,187,441,528,524,259,365,148,58,51,85,17,132,23,88,112,62,69,45,39,31,14,13,42,15,23,47,7,63,24,6,14,122,2,1,0,0,0,0,0,0,0,1,0,0,0,585,23
    2020/08/01,85,238,598,1929,250,297,248,433,454,384,429,1038,475,639,525,442,267,196,463,544,539,263,376,159,61,53,88,17,137,24,91,113,66,74,45,40,35,16,13,43,15,24,47,7,65,27,6,14,128,2,1,0,0,0,0,0,0,0,1,0,0,0,616,23
    2020/08/02,85,241,615,1973,253,298,258,446,472,396,450,1044,477,646,548,452,271,198,472,553,561,273,395,159,61,56,91,17,138,24,93,115,66,76,45,40,36,16,14,43,15,24,48,7,66,27,6,14,128,3,1,0,0,0,0,0,0,0,1,0,0,0,625,23
    2020/08/03,85,247,626,2003,253,302,260,457,478,411,459,1099,493,662,553,455,278,205,481,559,562,273,398,160,64,57,93,17,138,24,93,115,71,76,45,40,41,16,14,45,15,24,49,7,66,27,6,14,129,3,1,0,0,0,0,0,0,0,1,0,0,0,640,23
    2020/08/04,89,252,643,2029,270,302,265,467,483,427,473,1118,506,674,567,458,289,211,488,576,581,281,412,165,69,59,100,17,140,24,96,117,73,79,47,40,43,18,14,46,17,24,49,7,68,27,6,14,133,3,1,0,0,0,0,0,0,0,1,0,0,0,641,23
    2020/08/05,90,255,660,2035,271,310,268,469,503,437,484,1167,513,677,572,464,293,212,504,585,605,287,422,166,70,60,104,17,140,25,105,126,73,81,49,41,46,18,15,47,17,24,49,7,69,27,6,14,134,3,1,0,0,0,0,0,0,0,1,0,0,0,644,23
    2020/08/06,91,259,679,2060,281,323,273,485,526,450,489,1211,539,684,577,468,303,218,515,603,615,291,439,169,73,61,108,17,143,26,105,132,72,89,49,41,52,18,15,49,19,24,49,8,70,30,6,14,135,3,1,0,0,0,0,0,0,0,1,0,0,0,662,23
    2020/08/07,96,277,709,2069,290,333,282,496,553,471,497,1261,565,714,582,495,314,220,530,622,625,311,459,176,75,66,110,18,145,27,111,140,74,93,50,42,52,18,15,50,20,27,49,8,71,31,6,14,138,3,1,0,0,0,0,0,0,0,1,0,0,0,682,23
    2020/08/08,96,288,728,2149,292,345,289,505,566,486,500,1305,587,732,602,499,319,226,540,638,642,318,478,179,76,68,114,18,154,27,113,145,74,94,50,43,55,19,15,54,20,27,51,8,72,32,6,14,141,3,1,0,0,0,0,0,0,0,1,0,0,0,709,23
    2020/08/09,100,293,745,2174,294,350,292,514,590,496,520,1340,608,736,625,509,327,230,551,650,658,320,488,182,77,72,115,18,159,27,117,146,74,96,53,43,58,20,16,55,23,27,51,8,73,32,6,14,146,3,1,0,0,0,0,0,0,0,1,0,0,0,721,23
    2020/08/10,101,299,760,2198,294,351,299,520,594,508,525,1355,618,753,629,517,329,232,552,661,664,331,490,184,78,73,117,18,160,28,117,147,74,97,53,44,58,20,16,56,23,28,52,8,73,32,6,14,149,3,1,0,0,0,0,0,0,0,1,0,0,0,731,23
    2020/08/11,102,303,770,2211,295,357,301,520,611,519,530,1366,621,757,631,519,330,241,561,678,672,333,501,186,79,76,118,19,162,28,120,152,75,98,53,44,58,20,17,58,23,28,52,8,75,33,6,14,149,3,1,0,0,0,0,0,0,0,1,0,0,0,744,23
    2020/08/12,103,304,784,2216,302,358,305,526,625,524,539,1386,624,765,658,528,337,243,573,680,678,339,508,186,82,83,121,19,168,28,122,154,77,100,54,44,58,20,17,59,23,28,53,8,75,35,6,14,153,3,1,0,0,0,0,0,0,0,1,0,0,0,754,23
    2020/08/13,104,308,808,2226,306,371,307,536,633,530,546,1401,626,773,674,541,343,248,580,681,681,347,514,186,84,84,122,19,169,28,124,157,77,100,56,45,58,20,17,59,23,28,53,8,76,36,6,14,154,3,1,0,0,0,0,0,0,0,1,0,0,0,765,23
    2020/08/14,112,318,832,2275,315,374,312,543,655,543,564,1450,642,774,687,543,348,254,594,687,696,353,528,190,88,87,128,19,174,29,130,163,81,101,57,47,62,23,18,60,24,29,53,9,78,39,6,14,158,3,1,0,0,0,0,0,0,0,1,0,0,0,775,23
    2020/08/15,112,327,846,2298,321,386,320,560,664,556,583,1470,663,797,710,558,353,260,606,714,711,365,543,195,88,89,130,20,181,29,130,167,81,102,59,48,63,24,18,63,24,29,54,10,80,40,6,14,161,3,1,0,0,0,0,0,0,0,1,0,0,0,798,23
    2020/08/16,112,334,856,2322,323,393,321,573,671,565,601,1496,672,810,726,564,360,262,618,732,721,368,546,195,88,93,132,20,182,29,133,173,83,103,59,49,66,24,18,64,24,29,55,10,80,40,6,14,163,3,1,0,0,0,0,0,0,0,1,0,0,0,808,23
    2020/08/17,112,338,861,2340,325,394,327,578,673,569,619,1504,680,831,735,564,366,264,625,737,726,368,549,196,89,96,133,20,182,29,135,175,84,103,59,49,67,24,18,65,24,29,55,10,80,40,6,15,163,3,1,0,0,0,0,0,0,0,1,0,0,0,816,23
    2020/08/18,114,342,874,2349,329,395,329,589,681,571,637,1516,685,835,749,576,369,265,628,744,738,374,563,201,90,97,133,21,183,31,137,178,86,104,62,49,67,24,18,65,24,29,55,11,80,41,6,15,166,3,1,0,0,0,0,0,0,0,1,0,0,0,829,23
    2020/08/19,114,344,891,2354,336,408,334,596,683,573,651,1527,688,843,753,576,373,267,640,754,750,384,570,203,90,98,134,21,186,32,137,182,88,105,63,50,67,24,18,65,25,29,55,12,80,41,6,15,169,3,1,0,0,0,0,0,0,0,1,0,0,0,836,23
    2020/08/20,115,350,917,2388,342,418,344,615,702,587,662,1542,694,857,776,590,377,274,649,760,759,392,589,206,90,99,137,24,187,34,138,188,90,106,63,51,68,25,18,65,25,30,56,13,81,42,6,19,169,3,1,0,0,0,0,0,0,0,1,0,0,0,850,23
    2020/08/21,118,360,929,2403,346,423,348,632,716,601,675,1560,698,871,794,597,379,276,657,770,766,395,600,213,90,102,141,24,188,36,139,190,93,109,64,51,68,25,18,66,25,30,56,13,81,42,6,21,172,3,1,1,0,0,0,0,0,0,1,0,0,0,859,23
    2020/08/22,118,364,945,2421,350,427,354,648,727,614,681,1565,701,882,812,612,384,279,670,777,777,399,609,215,91,105,142,26,190,38,139,191,94,109,65,51,70,25,18,67,26,32,57,14,84,46,6,21,172,3,1,1,0,0,0,0,0,0,1,0,0,0,882,23
    2020/08/23,118,368,958,2438,353,431,368,656,736,619,688,1570,712,891,825,614,388,282,676,791,786,402,620,221,92,105,143,26,190,38,140,194,95,110,67,51,70,25,18,68,26,32,58,14,85,46,6,24,174,3,1,1,0,0,0,0,0,0,1,0,0,0,896,23
    2020/08/24,118,371,960,2449,354,431,368,658,736,627,689,1571,717,898,827,615,389,283,683,797,790,407,625,229,92,105,146,28,190,38,143,194,95,110,68,51,70,25,18,68,26,32,58,14,85,46,6,25,174,3,1,1,0,0,0,0,0,0,1,0,0,0,900,23
    2020/08/25,119,373,969,2457,362,432,372,661,742,636,702,1591,721,904,837,625,393,289,685,803,796,413,630,232,92,106,147,30,193,38,144,194,95,110,68,51,70,25,18,68,26,33,58,15,87,46,7,28,177,3,1,1,0,0,0,0,0,0,1,0,0,0,911,23
    2020/08/26,122,375,975,2461,370,439,381,671,744,638,706,1633,757,909,847,626,398,291,693,809,803,422,642,234,93,106,150,30,195,38,144,199,96,114,68,52,71,25,18,69,27,33,58,16,88,47,7,28,178,3,1,1,0,0,0,0,0,0,1,0,0,0,921,23
    2020/08/27,122,377,988,2488,375,446,389,677,758,644,715,1656,759,915,859,633,401,295,707,815,817,422,654,241,95,109,151,31,198,39,145,203,96,116,70,52,72,25,18,69,28,35,58,17,89,47,7,29,180,3,1,1,0,0,0,0,0,0,1,0,0,0,935,23
    2020/08/28,122,383,992,2499,377,450,398,689,769,650,729,1672,793,916,865,635,403,295,717,815,830,426,666,256,97,110,153,31,200,39,147,203,97,119,71,52,72,25,18,70,28,35,58,17,90,48,8,29,182,4,1,1,0,0,0,0,0,0,1,0,0,0,946,23
    2020/08/29,122,385,1012,2510,381,453,406,702,777,662,737,1694,799,935,878,638,405,295,729,820,844,436,673,265,97,111,154,31,200,39,147,208,97,122,72,53,72,27,18,71,28,35,59,18,91,50,8,29,184,4,1,1,0,0,0,0,0,0,1,0,0,0,960,23
    2020/08/30,122,389,1014,2525,382,463,409,712,777,666,749,1699,806,941,883,640,410,296,738,828,851,439,679,268,97,112,156,32,202,39,150,209,97,124,72,53,73,27,18,71,28,35,59,18,91,50,8,29,185,4,1,1,0,0,0,0,0,0,1,0,0,0,966,23
    2020/08/31,123,389,1018,2537,382,463,417,716,778,670,755,1703,812,948,886,640,410,296,740,831,856,446,679,270,98,112,158,33,202,40,151,209,98,124,72,53,76,29,18,71,29,35,59,18,91,51,8,29,186,4,1,1,0,0,0,0,0,0,1,0,0,0,972,23
    2020/09/01,127,394,1021,2545,383,468,420,720,781,677,766,1741,814,955,889,649,412,299,744,836,866,449,682,271,99,113,160,33,203,41,153,209,98,125,73,53,78,29,18,71,29,35,59,18,92,51,8,29,188,4,1,1,0,3,0,0,0,0,1,0,0,0,980,23
    2020/09/02,128,395,1021,2548,385,474,426,721,785,683,772,1762,823,959,891,652,415,302,754,843,873,453,687,274,99,114,162,34,203,41,153,213,100,125,73,53,78,29,18,71,29,35,59,18,92,51,10,29,191,5,1,1,0,5,0,0,0,0,1,0,0,0,986,23
    2020/09/03,129,398,1035,2562,390,480,430,733,794,685,788,1781,828,961,897,660,416,306,759,850,885,468,692,280,99,117,164,36,204,42,153,214,100,126,76,55,78,30,18,71,29,35,59,18,93,51,12,29,192,5,1,1,0,5,0,0,0,0,1,0,0,0,995,23
    2020/09/04,132,401,1044,2571,391,481,433,740,798,688,794,1788,837,968,904,664,419,307,762,857,897,472,694,284,99,118,165,36,205,42,153,216,101,126,77,56,79,30,18,72,29,35,59,18,93,52,12,30,192,5,1,1,0,5,0,0,0,0,1,0,0,0,1000,23
    2020/09/05,134,402,1053,2577,397,487,437,745,798,692,798,1812,847,969,910,668,424,310,766,862,908,487,706,287,101,121,168,36,207,43,158,216,104,128,77,56,79,30,18,72,30,35,59,18,94,52,13,30,195,6,1,1,0,5,0,0,0,0,1,0,0,0,1003,23
    2020/09/06,134,408,1060,2582,400,494,445,747,802,695,803,1813,848,971,911,668,426,313,767,868,920,487,706,291,102,122,169,40,210,44,158,217,103,128,77,56,81,30,18,73,30,35,59,18,94,52,12,31,197,7,1,1,0,5,0,0,0,0,1,0,0,0,1016,23
    2020/09/07,134,410,1064,2588,401,494,447,749,806,698,807,1813,861,972,916,670,427,313,769,872,921,487,709,291,102,126,172,40,211,44,159,217,105,128,77,56,83,30,18,73,30,35,59,21,94,52,12,31,197,7,1,1,0,5,0,0,0,0,1,0,0,0,1020,23
    2020/09/08,135,412,1071,2597,406,499,449,757,810,703,810,1842,864,975,920,677,431,317,774,875,937,498,713,294,102,129,173,43,213,44,160,219,105,128,77,56,83,30,18,73,30,35,59,21,94,52,12,31,197,7,1,1,0,5,0,0,0,0,1,0,0,0,1031,23
    2020/09/09,136,413,1074,2605,408,510,451,758,813,710,816,1853,874,980,927,679,436,322,781,877,951,500,722,297,102,129,174,47,214,46,162,220,107,128,78,56,83,30,18,73,31,36,59,21,94,53,13,31,197,7,1,1,0,5,0,0,0,0,1,1,0,0,1034,23
    2020/09/10,138,416,1088,2622,415,523,455,771,821,716,847,1874,882,987,934,684,439,325,783,885,968,515,742,307,102,129,174,49,218,46,165,220,105,133,79,58,83,31,18,73,31,36,60,21,94,53,14,31,196,7,1,1,0,5,0,0,0,0,1,3,0,0,1046,24
    2020/09/11,139,417,1093,2634,417,526,460,773,829,722,858,1895,888,998,943,690,442,326,788,893,975,521,753,313,102,132,175,51,221,46,167,222,105,133,79,58,84,32,18,73,31,38,61,21,95,53,15,32,196,7,1,1,0,5,0,0,0,0,1,3,0,0,1056,24
    2020/09/12,139,420,1098,2645,420,533,470,784,835,729,870,1911,893,1008,951,699,444,331,793,900,987,523,772,330,102,135,180,51,222,46,171,222,107,133,79,58,84,33,18,74,31,40,61,21,95,53,15,33,197,7,1,1,0,5,0,0,0,0,1,3,0,0,1069,24
    2020/09/13,139,422,1100,2655,420,538,472,790,841,734,885,1914,897,1015,957,706,447,335,796,903,990,534,774,344,102,137,180,51,224,46,173,224,107,134,80,58,86,33,18,74,31,40,61,21,95,53,15,34,197,7,1,1,0,5,0,0,0,0,1,3,0,0,1079,24
    2020/09/14,139,427,1100,2663,420,538,476,791,843,735,890,1916,901,1019,958,708,448,335,797,910,992,534,780,350,102,138,181,54,225,46,173,224,107,134,82,58,86,33,18,74,31,41,61,21,95,53,16,34,198,7,1,1,0,5,0,0,0,0,1,4,0,0,1085,24
    2020/09/15,141,433,1111,2677,422,543,478,796,846,741,899,1949,905,1020,961,710,448,339,800,919,1003,540,801,355,102,139,182,54,228,46,174,224,109,135,82,58,86,34,19,74,31,41,61,21,95,53,16,34,200,7,1,1,0,5,0,0,0,0,1,4,0,0,1096,24
    2020/09/16,142,435,1113,2679,427,548,481,801,851,746,912,1958,909,1024,967,710,450,346,811,924,1007,542,810,374,102,139,182,59,232,47,177,226,109,136,83,58,86,36,19,74,31,41,61,21,96,54,17,34,200,7,1,1,0,5,0,0,0,0,1,4,0,0,1106,25
    2020/09/17,143,440,1122,2689,430,551,491,804,858,750,924,1973,916,1025,971,710,456,347,812,931,1011,544,826,387,102,139,182,63,233,47,178,227,111,136,84,59,86,36,19,74,31,41,61,22,97,54,17,34,204,8,2,1,0,5,0,0,0,0,1,4,0,0,1114,25
    2020/09/18,143,445,1126,2696,431,559,496,814,866,755,943,1989,927,1030,977,716,460,347,820,938,1018,551,842,396,103,141,182,66,240,48,183,230,113,137,84,59,87,37,19,74,31,41,62,22,98,55,17,34,205,8,3,1,0,5,0,0,0,0,1,4,0,0,1128,25
    2020/09/19,144,454,1139,2706,432,563,496,816,874,767,953,2012,935,1033,990,717,467,349,826,942,1024,563,869,397,103,143,183,67,244,49,185,231,113,138,85,59,87,37,19,75,31,41,62,22,99,56,17,34,208,8,3,1,0,5,0,0,0,0,1,4,0,2,1141,25
    2020/09/20,144,457,1145,2712,433,568,499,824,878,775,963,2017,935,1035,997,717,470,351,835,950,1032,565,889,405,103,145,184,69,245,49,190,233,112,139,85,60,87,39,19,76,31,41,62,22,99,58,17,34,209,8,3,1,0,5,0,0,0,0,1,4,0,2,1151,25
    2020/09/21,144,459,1150,2714,435,568,503,825,883,784,969,2020,939,1039,998,719,473,353,836,951,1035,571,897,410,103,148,184,70,246,50,191,233,113,140,85,60,88,40,20,76,31,41,63,22,99,58,19,34,209,10,4,1,0,5,0,0,0,0,1,4,0,2,1156,25
    2020/09/22,145,463,1150,2718,437,572,503,825,888,786,971,2040,939,1039,1005,725,475,353,836,953,1036,575,903,412,105,150,185,70,246,50,192,234,113,140,85,60,88,40,20,76,31,41,63,22,100,58,19,34,209,10,4,1,0,5,0,0,0,0,1,4,0,2,1162,25
    2020/09/23,145,463,1154,2719,437,573,503,827,888,786,980,2041,941,1042,1006,727,477,356,837,955,1037,575,908,412,105,151,185,72,248,50,193,234,113,140,85,60,88,40,20,76,31,41,64,22,100,58,20,35,209,11,4,1,1,5,0,0,0,0,1,4,0,2,1170,25
    2020/09/24,145,466,1163,2730,437,576,504,830,890,796,990,2054,942,1045,1014,735,479,360,838,957,1039,580,951,417,107,152,188,73,250,50,197,241,114,140,86,60,88,40,21,76,31,41,64,23,102,58,20,35,211,11,4,1,1,5,0,0,0,0,1,4,0,2,1188,25
    2020/09/25,147,467,1168,2738,438,582,510,843,892,800,1006,2073,950,1049,1020,743,484,366,845,958,1049,588,963,417,107,153,188,75,252,50,199,241,115,141,89,61,88,41,21,76,31,41,65,23,105,61,20,35,212,11,4,1,1,5,0,0,0,0,1,4,0,2,1201,25
    2020/09/26,148,472,1177,2756,442,586,516,853,902,807,1025,2094,961,1056,1029,749,487,368,850,963,1057,603,987,419,109,159,192,76,258,50,202,248,119,142,92,61,88,41,21,78,32,41,66,24,107,61,20,36,214,11,4,1,1,5,0,0,0,0,1,4,0,2,1215,25
    2020/09/27,149,477,1184,2769,444,586,518,857,908,812,1040,2098,970,1057,1038,750,488,371,850,969,1067,611,990,420,109,162,195,78,259,50,202,251,119,143,92,62,88,41,21,78,32,41,66,24,107,61,20,36,216,11,4,1,1,5,0,0,0,0,1,4,0,2,1226,26
    2020/09/28,150,480,1191,2774,445,588,519,859,908,817,1048,2104,970,1060,1038,753,488,372,849,972,1069,612,992,422,109,161,196,79,263,50,204,251,118,143,92,62,89,41,21,78,32,41,67,23,108,61,20,36,216,11,4,1,1,5,0,0,0,0,1,4,0,2,1231,26
    2020/09/29,150,484,1201,2780,448,595,526,863,919,822,1062,2135,982,1065,1046,760,490,373,854,974,1074,621,1015,422,112,162,197,79,264,50,204,251,119,143,93,63,90,43,21,78,32,41,67,23,110,61,20,36,218,11,4,1,1,5,0,0,0,0,1,4,0,2,1248,29
    2020/09/30,152,487,1212,2787,454,603,530,867,927,829,1075,2148,987,1071,1049,762,492,376,860,987,1087,624,1027,422,113,162,198,80,264,50,206,253,118,143,95,63,91,43,21,79,32,41,67,23,117,61,20,36,223,11,4,1,1,5,0,0,0,0,1,4,0,2,1260,29
    2020/10/01,154,491,1216,2805,455,608,532,872,939,836,1098,2162,993,1076,1063,766,495,383,864,1025,1092,630,1042,423,115,165,198,81,265,51,206,254,119,143,96,64,92,43,21,79,32,41,69,23,118,61,20,36,228,11,4,1,1,5,0,0,0,0,1,4,0,2,1275,29
    2020/10/02,154,496,1221,2818,458,610,539,874,948,841,1112,2178,1001,1086,1065,770,497,384,870,1034,1097,635,1062,426,118,167,202,83,265,54,210,255,121,145,96,65,93,43,21,79,32,41,70,23,122,63,20,36,228,11,4,1,1,5,0,0,0,0,1,4,0,2,1283,29
    2020/10/03,154,500,1227,2825,462,616,543,884,956,844,1130,2191,1010,1097,1071,777,499,386,875,1047,1110,638,1069,428,121,168,206,83,266,54,212,259,121,148,97,65,94,43,21,79,32,41,71,23,124,64,20,36,232,11,4,1,1,5,0,0,0,0,1,4,0,2,1297,29
    2020/10/04,154,504,1227,2832,465,622,547,889,963,847,1142,2194,1011,1100,1073,778,500,388,881,1051,1116,638,1074,428,121,169,207,84,267,54,213,259,120,148,97,66,95,44,21,79,32,41,71,23,124,64,20,36,232,11,4,1,1,5,0,0,0,0,1,4,0,2,1304,30
    2020/10/05,154,509,1231,2838,465,622,547,891,967,852,1145,2196,1015,1112,1073,778,500,388,882,1054,1119,638,1078,428,122,170,209,85,268,54,215,259,122,148,97,66,95,45,21,79,32,41,71,23,128,64,20,37,232,11,4,1,1,5,0,0,0,0,1,4,0,2,1306,30
    2020/10/06,154,511,1235,2840,471,624,557,893,978,858,1157,2227,1016,1121,1074,780,502,390,887,1063,1128,646,1085,431,126,173,210,85,269,55,217,260,122,149,99,68,95,45,21,79,32,41,72,23,129,65,20,36,234,11,4,1,1,5,0,0,0,0,1,4,0,2,1310,31
    2020/10/07,155,514,1243,2843,472,624,559,897,989,865,1168,2238,1019,1122,1076,786,509,392,891,1069,1137,651,1093,432,126,174,210,86,270,55,219,265,122,153,99,68,95,45,21,79,32,41,72,24,132,65,20,37,235,11,4,1,1,5,0,0,0,0,1,4,0,2,1320,31
    2020/10/08,157,521,1259,2853,475,631,563,910,1002,879,1191,2250,1027,1125,1082,791,517,399,898,1077,1142,661,1099,432,128,177,212,87,275,55,228,269,122,154,99,68,96,45,21,79,32,44,72,24,133,66,20,37,238,11,4,1,1,5,0,0,0,0,1,4,0,2,1335,31
    2020/10/09,157,527,1267,2858,480,632,565,916,1019,890,1204,2269,1033,1133,1091,794,527,400,903,1088,1153,664,1105,432,130,182,214,88,276,55,228,270,122,155,101,68,96,45,21,79,32,44,74,24,134,66,20,36,239,11,4,1,1,5,0,0,0,0,1,4,0,2,1352,31
    2020/10/10,158,530,1278,2862,485,637,573,923,1024,898,1218,2309,1036,1138,1110,798,532,406,912,1095,1176,672,1117,432,131,184,216,88,278,57,231,272,122,155,101,68,99,45,21,81,32,46,76,24,134,66,20,37,240,11,4,1,1,5,0,0,0,0,1,4,0,2,1366,31
    2020/10/11,158,532,1282,2874,485,642,577,930,1032,903,1228,2311,1037,1145,1114,799,537,408,914,1105,1185,681,1125,432,131,184,217,88,279,57,234,273,122,156,104,69,99,45,21,81,32,49,76,24,136,67,20,39,243,11,4,1,1,5,0,0,0,0,1,4,0,2,1372,32
    2020/10/12,158,534,1285,2876,485,642,578,933,1035,905,1236,2312,1043,1152,1121,799,538,409,917,1109,1188,681,1126,432,131,186,217,88,280,57,240,274,122,156,104,69,102,46,21,81,33,50,76,24,136,67,20,39,243,11,4,1,1,5,0,0,0,0,1,4,0,2,1376,32
    2020/10/13,159,538,1288,2884,487,643,583,938,1042,908,1251,2337,1050,1153,1128,805,539,412,925,1117,1193,687,1134,434,131,191,218,88,281,57,240,274,122,158,105,70,102,46,21,81,33,50,76,24,136,67,20,39,245,11,4,1,1,5,0,0,0,0,1,4,0,2,1388,32
    2020/10/14,159,539,1293,2887,487,648,586,944,1046,915,1260,2353,1053,1156,1133,808,546,418,933,1118,1213,694,1145,436,131,192,220,88,283,57,245,274,122,161,106,72,102,46,22,85,33,51,77,24,136,67,20,39,251,11,4,1,1,5,0,0,0,0,1,4,0,2,1401,32
    2020/10/15,160,547,1307,2906,491,653,589,946,1058,922,1305,2387,1069,1165,1142,816,550,422,942,1127,1227,697,1151,440,132,196,221,88,285,57,247,277,122,162,106,73,102,46,22,87,33,52,77,24,135,68,20,38,251,11,4,1,1,5,0,0,0,0,1,4,0,2,1414,32
    2020/10/16,160,551,1317,2918,495,658,595,948,1066,925,1317,2405,1072,1171,1160,817,554,423,945,1139,1238,700,1157,444,135,197,221,88,285,57,250,280,123,163,108,73,102,46,22,87,33,53,77,25,137,69,20,39,252,11,4,1,1,5,0,0,0,0,1,4,0,2,1426,32
    2020/10/17,162,555,1327,2931,500,660,596,953,1077,933,1347,2425,1077,1179,1163,821,564,428,956,1145,1250,710,1161,444,135,205,227,89,289,58,252,282,123,164,108,74,105,46,22,88,34,53,78,25,137,69,21,39,253,11,4,1,1,5,0,0,0,0,1,4,0,2,1438,32
    2020/10/18,164,559,1335,2942,501,668,600,956,1082,934,1358,2425,1083,1187,1170,822,566,429,958,1158,1255,717,1161,446,136,206,229,89,291,58,253,284,124,164,108,74,105,46,22,88,34,53,78,26,137,69,21,38,255,11,4,1,1,5,0,0,0,0,1,4,0,2,1443,32
    2020/10/19,164,563,1337,2950,501,668,601,959,1084,936,1366,2428,1083,1194,1173,823,566,436,958,1161,1259,717,1164,446,139,207,230,89,291,58,254,284,124,165,108,74,106,46,22,88,35,53,78,26,137,69,21,38,255,11,4,1,1,5,0,0,0,0,1,4,0,2,1451,32
    2020/10/20,166,563,1341,2953,504,669,604,961,1089,939,1377,2452,1086,1202,1180,825,570,436,963,1170,1264,719,1168,449,140,211,233,89,291,58,255,287,125,165,109,74,106,46,22,88,35,53,78,27,137,69,21,38,256,11,4,1,1,5,0,0,0,0,1,4,0,2,1458,32
    2020/10/21,166,564,1349,2954,508,671,610,965,1094,941,1390,2466,1087,1202,1187,833,574,438,968,1185,1271,725,1169,449,141,212,234,89,291,58,256,291,128,165,109,75,106,47,22,90,35,53,80,27,138,69,21,38,260,11,4,1,1,5,0,0,0,0,1,4,0,2,1473,32
    2020/10/22,168,570,1358,2965,510,671,611,970,1107,944,1400,2483,1089,1206,1194,841,578,442,978,1199,1282,729,1175,453,141,213,237,89,294,58,257,292,130,165,110,75,106,48,22,90,35,53,80,27,138,70,22,38,261,11,4,1,1,5,0,0,0,0,1,4,0,2,1484,33
    2020/10/23,171,573,1363,2970,511,673,613,979,1110,944,1415,2502,1093,1217,1201,847,585,447,983,1203,1290,730,1180,453,141,216,237,90,297,58,258,293,130,167,111,75,106,48,22,91,35,54,79,27,137,72,22,38,261,11,4,1,1,5,0,0,0,0,1,4,0,2,1516,33
    2020/10/24,172,576,1372,2981,513,673,618,980,1116,949,1429,2528,1098,1222,1208,849,592,451,990,1206,1297,736,1184,457,142,220,238,91,300,58,261,312,132,170,111,75,107,49,22,94,35,54,82,27,139,72,23,38,268,11,4,1,1,5,0,0,0,0,1,4,0,2,1530,33
    2020/10/25,174,580,1377,2988,522,678,618,980,1120,950,1443,2526,1101,1223,1214,851,597,455,1000,1212,1304,737,1184,459,142,222,239,91,303,58,262,316,131,171,111,75,107,49,22,94,35,54,82,27,138,73,24,38,272,11,4,1,1,5,0,0,0,0,1,4,0,2,1534,33
    2020/10/26,174,582,1380,2994,522,680,620,986,1120,954,1448,2527,1108,1228,1216,853,601,458,1004,1215,1309,739,1184,459,143,224,240,92,303,58,262,316,131,173,111,80,107,49,22,94,35,54,82,27,138,73,24,38,272,11,4,1,1,5,0,0,0,0,1,4,0,2,1556,33
    2020/10/27,176,583,1384,2999,525,681,620,988,1126,956,1462,2563,1113,1230,1223,859,603,462,1011,1221,1312,747,1192,462,144,226,241,92,306,58,262,317,132,173,112,80,107,50,22,94,35,54,83,27,138,73,24,38,273,11,4,1,1,5,0,0,0,0,1,4,0,2,1564,33
    2020/10/28,177,583,1389,3003,527,685,625,990,1127,956,1480,2567,1118,1232,1226,865,608,464,1029,1230,1320,750,1202,465,146,230,241,92,310,58,266,319,132,175,116,80,107,50,22,94,36,54,85,27,138,73,24,38,285,11,4,1,1,5,0,0,0,0,1,4,0,2,1574,33
    2020/10/29,178,585,1404,3017,528,692,633,1000,1134,961,1493,2588,1124,1240,1239,873,610,464,1035,1244,1322,753,1207,467,146,232,242,93,313,59,269,324,133,176,116,80,108,51,22,94,37,55,86,27,138,73,24,38,287,11,4,1,1,5,0,0,0,0,1,4,0,2,1596,33
    2020/10/30,180,590,1416,3026,531,694,634,1002,1137,966,1514,2598,1127,1247,1249,879,618,472,1043,1252,1325,759,1212,471,147,235,243,93,319,59,274,329,136,178,118,83,108,52,22,95,37,55,87,28,139,74,25,38,288,11,4,1,1,5,0,0,0,0,1,4,0,2,1615,33
    2020/10/31,181,593,1424,3040,534,698,639,1009,1144,972,1527,2612,1130,1256,1260,891,624,475,1053,1256,1327,769,1225,476,147,237,245,96,320,60,276,329,137,179,121,83,108,52,23,95,37,57,87,28,139,76,25,38,292,11,4,1,2,5,0,0,0,0,1,4,0,2,1631,33
    2020/11/01,181,594,1427,3046,536,699,640,1009,1147,973,1534,2614,1136,1259,1267,891,631,479,1056,1262,1331,777,1228,476,148,239,249,97,321,61,277,331,136,180,122,86,110,53,23,95,37,58,89,28,139,77,25,38,293,11,4,1,2,5,0,0,0,0,1,4,0,2,1638,33
    2020/11/02,181,599,1431,3057,536,699,642,1011,1150,977,1538,2615,1138,1264,1273,895,632,480,1058,1265,1334,779,1230,480,148,240,249,99,322,62,279,331,136,180,122,86,110,53,23,95,37,58,89,28,139,77,25,39,293,11,4,1,2,5,0,0,0,1,1,4,0,2,1645,33
    2020/11/03,182,602,1434,3064,537,703,643,1021,1156,981,1545,2641,1145,1274,1286,904,637,490,1063,1275,1345,780,1245,488,149,241,250,99,322,63,280,333,138,181,123,88,110,53,23,95,37,61,89,28,139,79,25,39,296,11,5,1,2,5,0,0,0,1,1,4,0,2,1655,33
    2020/11/04,183,602,1440,3070,538,708,647,1026,1158,982,1556,2647,1149,1293,1287,905,637,495,1069,1278,1347,782,1249,490,149,242,251,99,325,63,283,333,139,185,123,88,111,53,23,96,38,61,90,29,139,79,25,39,298,11,5,1,2,5,0,0,0,2,1,4,0,2,1659,33
    2020/11/05,184,608,1453,3099,542,709,647,1035,1165,992,1579,2663,1155,1300,1298,920,645,505,1078,1290,1361,792,1258,493,150,244,252,100,325,64,286,339,142,187,125,88,111,53,23,98,38,61,90,29,139,80,25,40,301,11,5,1,2,5,0,0,0,2,1,4,0,2,1666,33
    2020/11/06,184,611,1470,3103,545,711,654,1045,1178,993,1605,2682,1163,1309,1306,922,654,506,1090,1299,1375,798,1268,497,151,244,255,100,325,66,287,346,143,188,125,90,111,54,25,98,38,62,91,29,140,80,25,43,305,11,5,1,2,5,0,0,0,2,1,4,0,2,1680,33
    2020/11/07,184,621,1486,3124,552,717,654,1059,1187,998,1627,2702,1164,1329,1313,932,658,508,1105,1315,1385,809,1281,503,154,248,258,100,326,67,289,349,144,188,125,90,113,54,26,98,38,63,92,30,140,80,25,43,306,11,5,1,2,5,0,0,0,3,1,4,0,2,1703,33
    2020/11/08,184,630,1493,3137,557,720,660,1063,1198,1000,1636,2706,1167,1335,1321,938,659,509,1115,1336,1398,810,1285,503,154,247,258,101,327,67,290,352,144,189,125,91,113,55,26,99,38,63,93,31,140,82,25,43,309,11,5,1,2,5,0,0,0,3,1,4,0,2,1721,33
    2020/11/09,184,633,1500,3147,557,722,660,1067,1202,1010,1641,2714,1169,1349,1338,939,660,509,1120,1341,1408,822,1290,506,157,248,259,102,327,69,291,353,145,189,125,92,113,55,29,100,38,63,93,31,140,83,26,43,308,11,6,1,2,5,0,0,0,3,1,4,0,2,1732,33
    2020/11/10,189,641,1509,3160,561,725,661,1079,1206,1017,1659,2757,1175,1359,1343,951,666,518,1134,1348,1423,826,1297,509,158,248,260,102,330,73,296,353,145,191,128,92,113,55,30,100,38,63,94,31,141,86,26,44,312,13,6,1,3,5,0,0,0,3,1,4,0,2,1767,33
    2020/11/11,192,646,1524,3167,573,726,664,1086,1211,1027,1678,2767,1185,1363,1355,959,676,525,1142,1359,1435,845,1308,516,160,250,260,102,334,73,301,359,146,195,128,95,114,57,32,101,39,66,95,31,141,89,26,45,315,13,8,1,3,5,0,0,0,3,1,4,0,2,1820,33
    2020/11/12,194,663,1542,3191,582,734,667,1104,1230,1033,1702,2790,1199,1376,1368,976,690,528,1158,1381,1452,855,1322,520,161,252,264,103,337,74,307,360,146,196,128,96,115,59,34,101,43,67,97,32,141,89,27,46,317,13,9,1,3,5,0,0,0,3,1,4,0,2,1847,33
    2020/11/13,197,671,1553,3207,594,745,673,1114,1239,1043,1724,2826,1202,1385,1388,981,701,533,1163,1400,1472,859,1333,527,163,253,269,107,339,76,315,366,150,200,134,98,117,59,36,104,44,69,97,34,144,91,28,46,321,13,9,1,3,5,0,0,0,3,1,4,0,2,1879,34
    2020/11/14,200,687,1563,3226,608,748,676,1125,1254,1051,1752,2858,1212,1403,1393,994,708,537,1175,1409,1480,875,1346,535,164,253,273,107,343,77,319,368,153,201,143,100,118,59,38,104,44,69,98,34,145,93,28,46,321,13,9,2,3,5,0,0,0,3,1,4,0,2,1907,34
    2020/11/15,201,693,1577,3238,615,749,680,1132,1259,1058,1774,2864,1215,1410,1402,997,712,543,1188,1423,1499,877,1360,535,166,256,275,107,345,77,322,373,154,202,145,103,118,59,41,106,46,70,101,34,145,93,28,46,324,13,9,2,3,5,0,0,0,3,1,4,0,2,1938,34
    2020/11/16,202,698,1583,3247,615,753,681,1142,1264,1062,1780,2867,1224,1416,1412,1000,713,545,1192,1430,1509,902,1367,535,166,257,276,108,346,78,324,373,154,202,147,104,118,59,44,107,47,70,102,34,145,93,28,47,325,13,9,2,3,5,0,0,0,3,1,4,0,2,1962,34
    2020/11/17,202,705,1601,3253,624,756,687,1150,1268,1068,1794,2903,1235,1425,1424,1003,716,550,1205,1446,1531,917,1378,537,170,260,279,108,347,80,329,373,154,202,150,105,118,59,45,107,48,71,103,34,145,93,28,49,331,13,9,2,3,6,0,0,0,3,1,4,0,2,1984,34
    2020/11/18,202,721,1626,3263,643,771,693,1168,1276,1079,1815,2922,1252,1431,1439,1008,726,558,1218,1460,1574,936,1397,556,174,260,284,108,349,82,341,375,156,203,168,105,120,60,47,109,48,73,103,34,148,100,28,49,339,15,9,2,3,6,0,0,0,3,1,4,0,2,2035,34
    2020/11/19,210,733,1652,3274,658,778,733,1191,1298,1098,1849,2976,1264,1448,1449,1016,734,566,1240,1486,1599,953,1406,567,178,261,287,109,351,85,350,379,157,207,170,106,122,62,47,111,48,74,104,35,148,101,28,51,342,16,10,2,3,6,0,0,0,3,1,4,0,2,2084,34
    2020/11/20,214,749,1680,3305,666,786,747,1213,1310,1106,1866,3006,1281,1470,1471,1025,743,574,1250,1506,1630,981,1427,586,180,265,287,115,352,88,354,387,161,212,171,106,124,63,49,112,48,77,105,36,151,102,28,52,347,18,10,2,3,6,0,0,0,3,1,4,0,2,2117,34
    2020/11/21,214,755,1701,3337,668,793,757,1227,1320,1116,1881,3046,1307,1492,1502,1039,752,584,1267,1537,1661,1006,1446,604,184,267,292,117,357,90,362,393,164,214,176,111,127,65,53,113,48,80,105,37,157,104,31,54,354,19,11,2,3,6,0,0,0,3,1,4,0,2,2150,34
    2020/11/22,215,764,1715,3360,669,799,765,1230,1328,1126,1913,3057,1321,1501,1520,1053,759,591,1291,1560,1691,1019,1461,607,187,267,296,120,360,90,370,397,164,216,182,118,131,65,54,117,49,81,106,38,157,106,31,55,358,19,11,2,3,6,0,0,0,3,1,4,0,2,2193,34
    2020/11/23,218,783,1723,3377,671,808,779,1249,1334,1142,1928,3065,1331,1521,1536,1057,761,591,1300,1567,1710,1029,1466,613,187,267,297,121,361,91,375,400,164,217,184,118,131,66,56,117,49,82,106,38,159,106,32,57,359,19,11,2,3,6,0,0,0,3,1,4,0,2,2227,34
    2020/11/24,218,787,1724,3377,670,808,779,1255,1335,1147,1939,3079,1334,1522,1548,1058,763,594,1313,1574,1724,1043,1469,626,188,270,297,121,361,94,375,403,164,218,190,117,131,68,61,117,51,82,106,39,162,108,33,56,359,19,11,2,3,6,0,0,0,3,1,4,0,2,2250,34
    2020/11/25,218,795,1750,3395,678,822,789,1276,1344,1157,1951,3109,1340,1539,1562,1063,776,600,1332,1587,1746,1053,1494,630,192,271,302,124,362,97,375,408,165,221,195,119,132,69,62,120,52,82,106,40,165,109,33,60,361,19,14,2,3,6,0,0,0,3,1,4,0,2,2282,34
    2020/11/26,220,811,1775,3414,686,830,806,1283,1368,1178,1971,3146,1356,1542,1576,1077,788,610,1354,1604,1767,1075,1506,634,193,272,308,130,364,97,381,419,166,225,199,121,132,69,65,124,52,84,106,42,165,111,33,62,365,19,14,2,3,6,0,0,0,3,1,5,0,2,2322,34
    2020/11/27,222,821,1795,3453,691,836,824,1306,1386,1192,2005,3191,1371,1563,1586,1094,798,619,1367,1630,1814,1086,1537,641,197,276,311,135,367,99,387,426,168,227,209,123,134,70,67,129,52,84,108,44,167,112,35,64,368,20,21,2,3,6,0,0,0,3,1,5,0,2,2365,34
    2020/11/28,225,832,1819,3478,694,848,836,1329,1398,1204,2024,3226,1386,1585,1615,1106,810,622,1397,1655,1832,1100,1554,666,199,282,319,140,372,100,399,433,169,228,211,130,138,71,69,135,53,89,108,45,175,115,38,65,374,21,22,2,3,6,0,0,0,3,1,6,0,2,2412,34
    2020/11/29,226,846,1827,3495,695,852,842,1345,1420,1210,2045,3233,1395,1599,1642,1124,818,637,1411,1675,1856,1112,1579,675,199,287,323,141,377,103,405,441,169,232,216,131,141,71,72,137,54,89,111,46,175,115,40,65,380,23,23,2,3,6,0,0,0,3,1,6,0,2,2446,34
    2020/11/30,227,852,1831,3512,695,863,850,1357,1422,1221,2051,3245,1397,1611,1662,1128,820,639,1423,1691,1874,1131,1582,695,201,291,336,148,379,105,407,446,171,233,219,131,142,71,74,137,55,90,112,46,176,116,41,64,382,23,24,2,3,6,0,0,0,3,1,6,0,2,2476,34
    2020/12/01,230,858,1841,3529,701,866,856,1373,1434,1232,2068,3292,1408,1619,1674,1143,826,644,1434,1711,1892,1139,1593,699,202,297,340,150,381,109,409,449,174,237,226,134,146,71,75,137,56,93,113,47,177,118,41,67,387,23,24,2,3,6,0,0,0,3,1,6,0,2,2509,34
    2020/12/02,230,860,1859,3547,711,871,867,1405,1452,1246,2094,3311,1422,1630,1690,1161,842,654,1459,1734,1922,1158,1608,717,204,298,348,152,385,111,414,466,176,243,233,139,147,71,75,137,56,95,113,47,181,118,43,66,389,23,24,2,3,6,0,0,0,3,1,6,0,2,2541,34
    2020/12/03,232,880,1882,3597,720,889,880,1429,1463,1261,2119,3347,1430,1639,1704,1172,852,664,1478,1755,1951,1176,1630,734,209,309,351,154,388,114,416,482,177,247,241,141,148,71,77,137,58,95,114,48,181,118,44,68,392,23,30,2,3,6,0,0,0,3,1,6,0,2,2570,34
    2020/12/04,236,885,1902,3626,724,890,890,1444,1475,1271,2150,3372,1440,1663,1725,1192,858,667,1491,1771,1978,1187,1641,738,209,314,358,156,394,115,418,485,180,250,246,144,152,71,78,138,58,99,118,50,182,118,45,67,398,23,30,2,3,6,0,0,0,3,1,6,0,2,2614,34
    2020/12/05,238,902,1924,3662,734,896,900,1466,1499,1290,2167,3407,1460,1683,1747,1206,872,674,1509,1796,2006,1199,1657,763,216,319,365,156,395,116,427,497,184,253,248,153,157,71,78,140,59,101,119,50,184,119,45,71,405,23,30,2,3,6,0,0,0,3,1,6,0,2,2682,34
    2020/12/06,238,903,1933,3679,736,901,902,1477,1512,1295,2184,3411,1477,1695,1755,1215,881,676,1529,1817,2011,1215,1661,771,216,321,371,163,400,117,433,503,187,261,250,158,159,72,78,143,59,103,119,50,184,120,45,71,412,23,31,2,3,6,0,0,0,3,1,6,0,2,2724,34
    2020/12/07,240,916,1936,3692,737,903,913,1493,1517,1306,2189,3418,1483,1710,1768,1221,885,678,1541,1826,2040,1224,1671,778,218,326,374,164,401,117,436,504,187,261,250,160,160,72,80,143,60,103,123,50,184,122,47,72,415,23,31,2,3,6,0,0,0,3,1,6,0,2,2767,34
    2020/12/08,241,922,1943,3707,749,904,919,1509,1524,1316,2210,3468,1489,1725,1774,1234,891,685,1548,1836,2067,1235,1681,789,222,328,377,165,403,120,441,509,190,261,254,163,162,72,80,145,60,103,123,50,185,125,47,76,419,24,31,2,3,6,0,0,0,3,1,6,0,2,2797,34
    2020/12/09,246,930,1958,3727,753,922,934,1524,1547,1332,2229,3489,1504,1739,1804,1241,905,694,1567,1857,2098,1248,1685,817,229,331,386,166,411,122,451,523,197,262,264,167,163,74,82,146,63,105,125,51,186,128,49,78,438,27,32,2,3,6,0,0,0,3,1,6,0,2,2852,34
    2020/12/10,249,937,1986,3772,760,930,946,1545,1575,1346,2246,3523,1528,1752,1818,1257,922,702,1581,1893,2137,1261,1704,842,231,337,396,166,417,123,455,540,203,264,271,169,166,77,83,147,64,106,126,52,186,128,51,81,445,29,32,2,3,6,0,0,0,3,1,6,0,2,2913,36
    2020/12/11,254,946,2004,3792,766,932,960,1566,1598,1362,2265,3557,1550,1786,1836,1275,936,707,1601,1932,2177,1279,1720,854,233,339,404,169,426,125,460,548,206,271,279,170,169,78,85,151,64,108,129,53,186,136,52,81,454,29,32,2,3,6,0,0,0,3,1,6,0,2,2959,36
    2020/12/12,256,957,2030,3824,777,938,980,1588,1613,1383,2289,3612,1568,1807,1858,1288,951,715,1623,1954,2202,1297,1738,884,235,343,408,169,433,126,469,555,212,274,284,173,172,79,88,151,69,112,132,57,187,139,53,85,458,29,32,2,3,6,0,0,0,3,1,6,0,2,3017,49
    2020/12/13,257,966,2051,3856,782,943,987,1600,1627,1399,2307,3627,1587,1820,1880,1296,967,725,1646,1972,2248,1324,1746,892,239,343,410,170,437,127,473,567,218,275,284,177,177,79,92,154,71,113,133,57,188,140,53,86,458,29,32,3,3,6,0,0,0,3,1,6,0,2,3064,50
    2020/12/14,260,977,2059,3876,782,947,995,1606,1637,1410,2317,3634,1595,1840,1895,1305,971,731,1648,1987,2261,1326,1765,908,241,342,411,174,439,128,475,574,219,276,284,178,178,79,92,155,72,113,133,57,189,141,54,85,459,30,32,3,3,6,0,0,0,3,1,6,0,2,3099,50
    2020/12/15,261,990,2073,3906,789,952,1004,1623,1649,1426,2335,3681,1607,1859,1906,1317,978,735,1667,2000,2297,1334,1786,916,247,350,413,174,445,130,481,586,223,280,286,180,179,82,92,156,73,113,134,61,189,146,54,90,466,31,32,3,3,6,0,0,0,3,1,6,0,3,3131,50
    2020/12/16,267,1010,2093,3927,793,958,1023,1646,1672,1441,2368,3738,1627,1879,1949,1337,987,743,1697,2020,2337,1353,1798,931,250,355,417,176,449,134,487,599,226,284,289,182,180,83,96,157,76,116,136,63,192,146,57,91,474,32,32,3,3,6,0,0,0,3,1,6,0,3,3202,50
    2020/12/17,272,1030,2113,3969,811,969,1038,1678,1696,1462,2403,3813,1668,1899,1973,1348,999,751,1731,2071,2378,1372,1822,968,258,363,419,180,457,137,496,611,229,285,292,187,186,84,102,159,77,119,140,65,196,150,61,103,478,32,32,3,3,6,0,0,0,3,1,6,0,3,3283,50
    2020/12/18,275,1046,2135,4003,825,976,1049,1699,1709,1479,2436,3871,1696,1930,1988,1360,1011,761,1753,2099,2416,1391,1835,991,266,373,426,182,464,143,504,622,231,286,295,191,187,90,104,164,79,125,144,67,198,151,63,106,483,32,32,3,3,6,0,0,0,3,1,6,0,3,3337,50
    2020/12/19,280,1063,2164,4047,845,982,1058,1724,1731,1509,2468,3929,1715,1949,2032,1379,1023,775,1778,2133,2454,1406,1862,998,267,378,433,182,472,144,524,634,234,288,299,196,188,91,105,167,79,127,154,67,200,152,66,109,493,33,33,3,3,6,0,0,0,3,1,6,0,3,3396,50
    2020/12/20,281,1072,2178,4081,853,989,1067,1755,1755,1523,2493,3944,1739,1960,2035,1389,1036,788,1806,2160,2490,1407,1874,1011,277,383,443,186,479,152,532,645,240,288,303,196,191,92,111,168,79,131,161,70,200,153,67,116,498,34,34,3,3,6,0,0,0,3,1,6,0,3,3456,50
    2020/12/21,287,1074,2186,4101,855,991,1074,1770,1772,1532,2508,3984,1746,1974,2061,1400,1038,793,1814,2170,2509,1418,1880,1018,279,384,443,188,481,160,536,656,240,288,303,195,196,94,112,169,80,132,161,71,200,153,69,116,498,34,34,3,3,6,0,0,0,3,1,6,0,3,3517,50
    2020/12/22,290,1082,2212,4128,868,995,1089,1789,1787,1548,2534,4070,1762,1991,2080,1416,1045,798,1829,2191,2550,1435,1888,1033,283,386,449,188,488,164,538,661,241,289,307,195,196,99,114,169,81,133,163,76,202,156,72,119,503,34,34,3,3,6,0,0,0,3,1,6,0,3,3557,50
    2020/12/23,293,1103,2249,4148,881,1007,1107,1815,1815,1585,2584,4124,1785,2010,2114,1425,1064,813,1852,2216,2569,1443,1915,1041,285,393,455,191,495,167,551,671,246,291,311,198,197,100,114,169,81,136,166,80,206,158,72,122,516,36,34,3,3,6,0,0,0,3,1,6,0,3,3656,50
    2020/12/24,297,1127,2291,4199,892,1027,1121,1840,1857,1613,2626,4168,1820,2032,2144,1436,1089,820,1886,2261,2607,1482,1939,1051,295,399,463,194,504,184,559,682,248,298,316,199,202,105,116,174,84,138,166,87,209,162,75,131,529,40,36,3,3,6,0,0,0,3,1,6,0,3,3723,50
    2020/12/25,300,1146,2336,4230,911,1033,1149,1862,1883,1642,2664,4247,1851,2065,2181,1450,1118,834,1915,2295,2662,1516,1965,1081,303,411,471,195,510,185,571,697,253,301,322,204,204,106,116,177,85,138,167,87,212,163,76,136,537,40,36,4,3,6,0,0,0,3,1,6,0,3,3787,50
    2020/12/26,308,1174,2390,4285,938,1044,1163,1897,1913,1675,2724,4329,1875,2104,2211,1465,1146,847,1945,2335,2690,1542,1984,1107,304,424,481,197,517,191,575,711,259,306,325,206,208,110,123,179,87,143,168,88,213,164,79,140,544,40,36,4,3,6,0,0,0,3,1,6,0,3,3866,50
    2020/12/27,313,1184,2411,4326,948,1054,1174,1926,1938,1686,2765,4360,1912,2117,2238,1491,1161,858,1969,2378,2722,1548,2011,1115,314,437,486,201,527,196,583,716,260,308,330,207,210,110,128,182,89,144,168,90,218,166,80,151,552,41,36,4,3,6,0,0,0,3,1,6,0,3,3948,50
    2020/12/28,316,1196,2425,4362,953,1058,1189,1932,1965,1705,2787,4379,1924,2136,2261,1500,1170,862,1993,2402,2745,1564,2018,1124,322,439,489,201,529,196,589,719,261,310,332,207,210,110,129,183,91,145,169,90,222,167,83,154,559,41,36,4,3,6,0,0,0,3,1,6,0,3,4015,50
    2020/12/29,326,1221,2458,4394,976,1067,1207,1948,1987,1726,2831,4433,1942,2167,2301,1521,1190,876,2014,2446,2801,1587,2056,1135,327,446,504,201,536,199,605,728,267,316,338,211,215,112,130,186,95,145,174,92,226,169,87,155,562,42,36,4,4,6,0,0,0,3,1,6,0,3,4105,50
    2020/12/30,327,1232,2502,4444,988,1081,1236,1964,2041,1770,2873,4466,1986,2183,2337,1536,1194,897,2043,2507,2852,1620,2099,1140,330,455,517,203,541,202,616,755,271,319,344,214,218,113,132,191,100,147,178,99,228,169,89,162,582,43,36,4,4,6,0,0,0,3,1,6,0,3,4189,52
    2020/12/31,331,1258,2547,4510,1018,1105,1244,2017,2071,1822,2958,4544,2024,2230,2389,1582,1245,916,2092,2559,2926,1657,2151,1187,346,463,523,207,549,209,631,771,278,325,354,216,224,116,135,195,108,148,188,104,236,173,90,169,590,48,36,4,5,6,0,0,0,3,1,6,0,3,4281,53
    2021/01/01,338,1270,2578,4537,1026,1109,1266,2041,2087,1849,2988,4604,2034,2238,2420,1598,1259,930,2125,2606,2964,1668,2174,1203,359,477,536,212,558,224,641,778,290,340,360,224,235,121,140,199,116,150,190,110,239,174,92,172,601,48,36,4,5,6,0,0,0,3,1,6,0,3,4345,53
    2021/01/02,346,1289,2600,4569,1045,1120,1277,2081,2126,1876,3019,4681,2074,2270,2438,1613,1283,932,2154,2639,3002,1688,2197,1218,366,479,541,215,560,235,653,796,302,343,364,227,238,123,140,201,121,152,191,112,242,177,92,174,608,49,36,4,5,6,0,0,0,3,1,6,0,3,4417,55
    2021/01/03,356,1305,2611,4640,1065,1141,1294,2106,2161,1898,3061,4712,2105,2290,2455,1637,1306,935,2177,2675,3061,1696,2230,1229,370,483,550,219,570,252,660,813,311,346,368,230,248,125,141,204,122,159,195,112,247,181,94,177,623,50,36,4,5,6,0,0,0,3,1,6,0,3,4475,55
    2021/01/04,371,1311,2641,4668,1081,1149,1315,2122,2186,1944,3119,4761,2128,2317,2511,1656,1318,944,2217,2699,3103,1710,2270,1262,379,491,559,222,573,267,675,823,317,352,369,235,255,125,142,206,127,164,196,116,252,183,96,179,631,52,36,4,6,6,0,0,0,3,1,6,0,3,4565,55
    2021/01/05,381,1334,2683,4731,1096,1192,1342,2159,2223,1982,3212,4892,2167,2340,2580,1689,1340,967,2261,2770,3143,1753,2319,1266,388,499,570,224,581,278,689,837,320,356,386,241,257,126,146,209,133,171,202,118,260,189,96,186,639,55,39,4,6,6,0,0,0,3,1,6,0,3,4650,56
    2021/04/22,711,2133,4337,7462,2026,2477,2646,4287,3831,3577,6809,9522,3843,4336,5318,3482,2946,2140,4986,5855,6311,4668,5600,3165,1036,1038,1287,713,1451,669,1593,2132,745,886,955,655,615,345,408,480,385,298,531,317,702,437,299,442,1278,151,74,6,20,23,0,0,1,4,1,7,0,3,10375,73
    2021/04/23,714,2144,4351,7487,2033,2487,2659,4315,3846,3593,6845,9596,3868,4361,5351,3506,2961,2151,5013,5904,6343,4699,5634,3183,1042,1045,1293,715,1460,674,1599,2145,750,894,956,657,618,345,409,485,388,299,538,317,709,437,299,443,1292,152,76,6,20,23,0,0,1,4,1,7,0,4,10441,74
    2021/04/24,716,2161,4382,7513,2044,2508,2689,4356,3877,3623,6867,9664,3885,4398,5386,3520,2991,2159,5041,5949,6380,4721,5678,3198,1046,1047,1300,716,1468,680,1607,2155,754,904,958,657,623,345,412,489,389,302,541,319,712,441,299,445,1306,153,76,6,20,23,0,0,1,4,1,7,0,4,10548,74
    2021/04/25,719,2179,4391,7528,2053,2516,2704,4376,3884,3644,6891,9712,3904,4414,5404,3529,3005,2175,5078,5976,6416,4742,5702,3213,1049,1052,1304,719,1473,682,1617,2166,758,908,961,661,629,346,412,492,389,303,541,320,714,444,303,445,1317,156,77,6,20,23,0,0,1,4,1,7,0,4,10639,75
    2021/04/26,722,2190,4408,7553,2059,2517,2717,4386,3889,3658,6903,9758,3920,4431,5414,3543,3009,2179,5091,6004,6430,4760,5707,3221,1053,1058,1307,723,1478,682,1620,2172,761,911,965,661,630,346,413,494,389,305,541,320,716,445,303,446,1320,156,77,6,20,23,0,0,1,4,1,7,0,4,10696,75
    2021/04/27,730,2205,4440,7590,2071,2529,2734,4415,3913,3683,6935,9792,3941,4449,5436,3562,3030,2188,5102,6037,6466,4852,5741,3240,1060,1066,1312,726,1485,687,1627,2184,768,916,968,663,634,348,415,494,391,308,548,320,728,449,304,450,1332,158,78,6,21,23,0,0,1,4,2,7,0,4,10784,74
    2021/04/28,735,2218,4471,7630,2087,2553,2766,4448,3951,3714,6968,9871,3953,4464,5466,3576,3048,2197,5141,6098,6498,4896,5793,3257,1073,1072,1315,732,1490,694,1633,2213,773,922,977,669,635,349,416,496,391,308,548,320,730,449,304,451,1343,162,78,6,21,23,0,0,1,4,2,7,0,4,10867,74
    2021/04/29,738,2239,4511,7697,2101,2573,2786,4482,3984,3734,7011,9936,3984,4503,5497,3617,3071,2209,5176,6138,6541,4915,5843,3290,1081,1075,1321,735,1509,703,1647,2242,777,926,986,678,649,357,418,502,392,313,553,320,733,454,304,451,1351,164,80,6,22,23,0,0,1,4,2,7,0,4,10938,74
    2021/04/30,743,2251,4541,7723,2109,2586,2808,4500,3997,3756,7031,10014,3997,4520,5521,3640,3079,2220,5202,6178,6570,4930,5866,3322,1087,1080,1326,735,1516,704,1650,2253,782,930,990,681,655,357,421,508,394,313,556,322,753,454,305,453,1359,165,80,6,23,23,0,0,1,4,2,7,0,4,10999,74
    2021/05/01,748,2269,4585,7802,2125,2604,2829,4543,4031,3785,7075,10090,4034,4559,5558,3672,3090,2237,5228,6220,6610,4955,5909,3357,1096,1091,1329,737,1528,708,1662,2266,792,936,1001,682,657,359,422,512,397,315,559,323,760,456,306,457,1366,166,80,6,23,23,0,0,1,4,2,7,0,4,11104,74
    2021/05/01,748,2269,4585,7802,2125,2604,2829,4543,4031,3785,7075,10090,4034,4559,5558,3672,3090,2237,5228,6220,6610,4955,5909,3357,1096,1091,1329,737,1528,708,1662,2266,792,936,1001,682,657,359,422,512,397,315,559,323,760,456,306,457,1366,166,80,6,23,23,0,0,1,4,2,7,0,4,11104,74
    2021/05/02,751,2281,4613,7822,2138,2614,2845,4580,4053,3804,7106,10171,4053,4586,5588,3688,3108,2253,5257,6272,6648,4980,5975,3381,1108,1102,1335,741,1539,710,1679,2285,798,941,1014,687,665,359,423,519,397,318,562,326,765,456,309,459,1374,168,80,6,23,23,0,0,1,4,2,7,0,4,11175,74
    2021/05/02,751,2281,4613,7822,2138,2614,2845,4580,4053,3804,7106,10171,4053,4586,5588,3688,3108,2253,5257,6272,6648,4980,5975,3381,1108,1102,1335,741,1539,710,1679,2285,798,941,1014,687,665,359,423,519,397,318,562,326,765,456,309,459,1374,168,80,6,23,23,0,0,1,4,2,7,0,4,11175,74
    2021/05/03,752,2294,4631,7853,2157,2619,2865,4599,4069,3827,7118,10219,4070,4625,5608,3712,3119,2259,5283,6301,6670,5001,6004,3425,1115,1108,1338,741,1544,713,1684,2293,801,946,1021,690,671,362,425,520,401,319,572,327,771,459,311,465,1380,168,80,6,24,23,0,0,1,4,2,7,0,4,11263,74
    2021/05/03,752,2294,4631,7853,2157,2619,2865,4599,4069,3827,7118,10219,4070,4625,5608,3712,3119,2259,5283,6301,6670,5001,6004,3425,1115,1108,1338,741,1544,713,1684,2293,801,946,1021,690,671,362,425,520,401,319,572,327,771,459,311,465,1380,168,80,6,24,23,0,0,1,4,2,7,0,4,11263,74
    2021/05/04,753,2305,4648,7869,2159,2623,2877,4637,4087,3842,7147,10262,4092,4640,5633,3732,3136,2263,5297,6338,6701,5020,6024,3437,1119,1115,1346,742,1547,713,1695,2304,807,955,1028,694,678,364,426,524,401,323,576,329,776,460,312,468,1387,168,80,6,24,23,0,0,1,4,2,7,0,4,11318,74
    2021/05/04,753,2305,4648,7869,2159,2623,2877,4637,4087,3842,7147,10262,4092,4640,5633,3732,3136,2263,5297,6338,6701,5020,6024,3437,1119,1115,1346,742,1547,713,1695,2304,807,955,1028,694,678,364,426,524,401,323,576,329,776,460,312,468,1387,168,80,6,24,23,0,0,1,4,2,7,0,4,11318,74
    2021/05/05,758,2317,4679,7882,2159,2628,2892,4652,4108,3855,7172,10332,4105,4659,5658,3753,3149,2268,5320,6363,6720,5038,6045,3453,1127,1121,1353,742,1548,718,1703,2329,808,956,1029,695,681,364,427,528,405,327,576,329,778,463,312,468,1391,168,84,6,24,23,0,1,1,4,2,7,0,4,11402,74
    2021/05/05,758,2317,4679,7882,2159,2628,2892,4652,4108,3855,7172,10332,4105,4659,5658,3753,3149,2268,5320,6363,6720,5038,6045,3453,1127,1121,1353,742,1548,718,1703,2329,808,956,1029,695,681,364,427,528,405,327,576,329,778,463,312,468,1391,168,84,6,24,23,0,1,1,4,2,7,0,4,11402,74
    2021/05/06,762,2332,4698,7908,2160,2648,2897,4682,4117,3879,7192,10381,4122,4678,5677,3764,3159,2276,5347,6404,6741,5045,6067,3468,1135,1124,1358,743,1550,720,1706,2330,814,960,1033,703,683,366,427,529,407,328,581,330,779,465,312,471,1397,169,85,6,24,23,0,1,1,4,2,7,0,4,11478,75
    2021/05/06,762,2332,4698,7908,2160,2648,2897,4682,4117,3879,7192,10381,4122,4678,5677,3764,3159,2276,5347,6404,6741,5045,6067,3468,1135,1124,1358,743,1550,720,1706,2330,814,960,1033,703,683,366,427,529,407,328,581,330,779,465,312,471,1397,169,85,6,24,23,0,1,1,4,2,7,0,4,11478,75
    2021/05/07,765,2348,4728,7974,2178,2664,2917,4722,4146,3905,7220,10441,4145,4694,5704,3779,3180,2288,5381,6432,6783,5067,6096,3491,1150,1131,1367,746,1563,723,1728,2338,819,968,1040,709,692,369,428,533,410,334,583,330,784,470,315,473,1408,169,87,6,24,23,0,1,1,4,2,7,0,4,11578,76
    2021/05/07,765,2348,4728,7974,2178,2664,2917,4722,4146,3905,7220,10441,4145,4694,5704,3779,3180,2288,5381,6432,6783,5067,6096,3491,1150,1131,1367,746,1563,723,1728,2338,819,968,1040,709,692,369,428,533,410,334,583,330,784,470,315,473,1408,169,87,6,24,23,0,1,1,4,2,7,0,4,11578,76
    2021/05/08,772,2367,4763,8017,2192,2690,2940,4773,4187,3935,7259,10517,4159,4721,5740,3802,3203,2315,5420,6484,6846,5093,6141,3523,1162,1143,1382,746,1582,726,1743,2362,829,977,1053,717,702,374,429,538,412,340,586,332,793,471,316,476,1417,169,87,6,24,23,0,1,1,4,2,7,0,4,11690,77
    2021/05/08,772,2367,4763,8017,2192,2690,2940,4773,4187,3935,7259,10517,4159,4721,5740,3802,3203,2315,5420,6484,6846,5093,6141,3523,1162,1143,1382,746,1582,726,1743,2362,829,977,1053,717,702,374,429,538,412,340,586,332,793,471,316,476,1417,169,87,6,24,23,0,1,1,4,2,7,0,4,11690,77
    2021/05/09,776,2385,4789,8082,2210,2707,2957,4806,4204,3964,7290,10587,4186,4749,5786,3835,3223,2330,5456,6536,6900,5121,6200,3553,1174,1152,1384,747,1600,731,1755,2388,833,986,1058,728,707,379,430,543,414,343,593,333,803,473,319,477,1425,169,87,9,24,23,0,1,1,4,2,7,0,4,11778,78
    2021/05/09,776,2385,4789,8082,2210,2707,2957,4806,4204,3964,7290,10587,4186,4749,5786,3835,3223,2330,5456,6536,6900,5121,6200,3553,1174,1152,1384,747,1600,731,1755,2388,833,986,1058,728,707,379,430,543,414,343,593,333,803,473,319,477,1425,169,87,9,24,23,0,1,1,4,2,7,0,4,11778,78
    2021/05/10,779,2389,4800,8112,2216,2713,2973,4820,4224,3982,7310,10618,4196,4775,5798,3853,3229,2334,5479,6556,6918,5140,6222,3576,1185,1164,1388,748,1604,733,1760,2392,836,991,1059,731,711,388,431,543,414,347,598,334,816,478,323,476,1429,169,88,9,24,23,0,1,1,4,2,7,0,4,11867,77
    2021/05/10,779,2389,4800,8112,2216,2713,2973,4820,4224,3982,7310,10618,4196,4775,5798,3853,3229,2334,5479,6556,6918,5140,6222,3576,1185,1164,1388,748,1604,733,1760,2392,836,991,1059,731,711,388,431,543,414,347,598,334,816,478,323,476,1429,169,88,9,24,23,0,1,1,4,2,7,0,4,11867,77
    2021/05/11,782,2406,4827,8160,2236,2719,2984,4850,4250,4002,7356,10659,4234,4807,5841,3874,3252,2345,5508,6591,6961,5163,6267,3612,1188,1171,1395,750,1619,740,1769,2404,843,997,1068,735,717,390,433,544,416,350,601,335,825,486,324,477,1434,169,90,9,24,23,0,1,1,4,2,7,0,4,11986,75
    2021/05/11,782,2406,4827,8160,2236,2719,2984,4850,4250,4002,7356,10659,4234,4807,5841,3874,3252,2345,5508,6591,6961,5163,6267,3612,1188,1171,1395,750,1619,740,1769,2404,843,997,1068,735,717,390,433,544,416,350,601,335,825,486,324,477,1434,169,90,9,24,23,0,1,1,4,2,7,0,4,11986,75
    2021/05/12,782,2415,4851,8183,2251,2744,3003,4877,4284,4037,7383,10738,4277,4836,5887,3896,3281,2364,5536,6631,7002,5188,6307,3633,1201,1182,1403,750,1632,749,1779,2426,846,1009,1076,742,720,393,433,547,418,355,605,336,829,496,325,478,1439,169,90,9,24,23,0,1,1,4,2,7,0,4,12097,75
    2021/05/12,782,2415,4851,8183,2251,2744,3003,4877,4284,4037,7383,10738,4277,4836,5887,3896,3281,2364,5536,6631,7002,5188,6307,3633,1201,1182,1403,750,1632,749,1779,2426,846,1009,1076,742,720,393,433,547,418,355,605,336,829,496,325,478,1439,169,90,9,24,23,0,1,1,4,2,7,0,4,12097,75
    2021/05/13,791,2431,4897,8252,2264,2756,3026,4913,4308,4062,7431,10806,4299,4869,5938,3917,3303,2372,5564,6673,7036,5214,6346,3668,1207,1190,1410,751,1656,751,1791,2440,848,1017,1080,746,732,395,434,552,421,359,607,337,834,499,327,480,1450,169,90,9,24,23,0,1,1,4,2,7,0,4,12212,75
    2021/05/13,791,2431,4897,8252,2264,2756,3026,4913,4308,4062,7431,10806,4299,4869,5938,3917,3303,2372,5564,6673,7036,5214,6346,3668,1207,1190,1410,751,1656,751,1791,2440,848,1017,1080,746,732,395,434,552,421,359,607,337,834,499,327,480,1450,169,90,9,24,23,0,1,1,4,2,7,0,4,12212,75
    2021/05/14,798,2444,4925,8280,2270,2769,3036,4936,4328,4091,7461,10872,4333,4898,5978,3948,3331,2394,5596,6708,7073,5243,6370,3686,1214,1194,1421,752,1675,754,1798,2454,856,1026,1085,755,733,397,435,553,423,360,613,341,838,501,327,482,1456,169,90,9,24,23,0,1,1,4,2,7,0,4,12304,76
    2021/05/14,798,2444,4925,8280,2270,2769,3036,4936,4328,4091,7461,10872,4333,4898,5978,3948,3331,2394,5596,6708,7073,5243,6370,3686,1214,1194,1421,752,1675,754,1798,2454,856,1026,1085,755,733,397,435,553,423,360,613,341,838,501,327,482,1456,169,90,9,24,23,0,1,1,4,2,7,0,4,12304,76
    2021/05/15,799,2457,4956,8315,2279,2787,3052,4967,4361,4109,7491,10952,4354,4910,6004,3966,3355,2405,5616,6734,7101,5256,6407,3721,1223,1203,1424,754,1688,756,1802,2462,862,1034,1091,757,734,400,435,556,423,362,616,344,844,507,329,483,1461,169,91,9,24,23,0,1,1,4,2,7,0,4,12382,76
    2021/05/15,799,2457,4956,8315,2279,2787,3052,4967,4361,4109,7491,10952,4354,4910,6004,3966,3355,2405,5616,6734,7101,5256,6407,3721,1223,1203,1424,754,1688,756,1802,2462,862,1034,1091,757,734,400,435,556,423,362,616,344,844,507,329,483,1461,169,91,9,24,23,0,1,1,4,2,7,0,4,12382,76
    2021/05/16,802,2467,4969,8332,2285,2791,3062,4992,4377,4123,7504,10995,4373,4924,6021,3975,3373,2414,5636,6750,7125,5277,6445,3740,1229,1207,1425,762,1697,760,1807,2474,869,1040,1093,757,736,402,438,556,423,364,620,346,847,510,331,485,1464,170,91,9,24,23,0,1,1,4,2,7,0,4,12433,76
    2021/05/16,802,2467,4969,8332,2285,2791,3062,4992,4377,4123,7504,10995,4373,4924,6021,3975,3373,2414,5636,6750,7125,5277,6445,3740,1229,1207,1425,762,1697,760,1807,2474,869,1040,1093,757,736,402,438,556,423,364,620,346,847,510,331,485,1464,170,91,9,24,23,0,1,1,4,2,7,0,4,12433,76
    2021/05/17,804,2471,4977,8351,2286,2802,3067,4997,4382,4135,7525,11021,4382,4938,6039,3992,3383,2419,5662,6769,7139,5286,6457,3752,1233,1213,1430,767,1701,760,1814,2478,871,1041,1096,760,742,402,440,560,425,365,620,346,850,515,333,486,1466,170,93,9,24,23,0,1,1,4,2,7,0,4,12494,76
    2021/05/17,804,2471,4977,8351,2286,2802,3067,4997,4382,4135,7525,11021,4382,4938,6039,3992,3383,2419,5662,6769,7139,5286,6457,3752,1233,1213,1430,767,1701,760,1814,2478,871,1041,1096,760,742,402,440,560,425,365,620,346,850,515,333,486,1466,170,93,9,24,23,0,1,1,4,2,7,0,4,12494,76
    2021/05/18,807,2487,5005,8394,2293,2808,3071,5021,4429,4160,7545,11055,4403,4967,6080,4018,3409,2427,5697,6792,7167,5301,6486,3764,1241,1219,1432,769,1715,760,1818,2487,872,1042,1098,762,748,402,441,562,430,367,621,349,854,515,334,489,1470,170,94,9,24,23,0,1,1,4,2,7,0,4,12592,76
    2021/05/18,807,2487,5005,8394,2293,2808,3071,5021,4429,4160,7545,11055,4403,4967,6080,4018,3409,2427,5697,6792,7167,5301,6486,3764,1241,1219,1432,769,1715,760,1818,2487,872,1042,1098,762,748,402,441,562,430,367,621,349,854,515,334,489,1470,170,94,9,24,23,0,1,1,4,2,7,0,4,12592,76
    2021/05/19,811,2491,5032,8411,2308,2831,3093,5052,4446,4191,7571,11120,4423,4988,6115,4046,3431,2439,5726,6832,7184,5322,6520,3778,1249,1226,1437,772,1733,762,1824,2497,882,1048,1102,763,756,403,443,565,435,370,624,351,856,519,334,494,1477,170,94,9,25,23,0,1,1,4,2,7,0,4,12657,76
    2021/05/19,811,2491,5032,8411,2308,2831,3093,5052,4446,4191,7571,11120,4423,4988,6115,4046,3431,2439,5726,6832,7184,5322,6520,3778,1249,1226,1437,772,1733,762,1824,2497,882,1048,1102,763,756,403,443,565,435,370,624,351,856,519,334,494,1477,170,94,9,25,23,0,1,1,4,2,7,0,4,12657,76
    2021/05/20,812,2512,5061,8471,2316,2842,3103,5072,4477,4208,7606,11181,4455,5005,6149,4070,3463,2453,5769,6858,7218,5329,6558,3790,1252,1232,1448,777,1753,763,1836,2506,889,1058,1108,765,763,407,448,566,439,371,627,354,865,521,334,495,1487,170,96,9,25,23,0,1,1,4,2,7,0,4,12739,76
    2021/05/20,812,2512,5061,8471,2316,2842,3103,5072,4477,4208,7606,11181,4455,5005,6149,4070,3463,2453,5769,6858,7218,5329,6558,3790,1252,1232,1448,777,1753,763,1836,2506,889,1058,1108,765,763,407,448,566,439,371,627,354,865,521,334,495,1487,170,96,9,25,23,0,1,1,4,2,7,0,4,12739,76
    2021/05/21,816,2519,5093,8501,2329,2855,3116,5090,4511,4222,7628,11231,4470,5022,6169,4082,3494,2465,5789,6895,7230,5342,6590,3805,1258,1236,1452,780,1763,766,1843,2513,893,1058,1111,768,767,410,450,570,441,372,628,359,865,521,334,497,1492,170,96,10,25,23,0,1,1,4,2,7,0,4,12818,76
    2021/05/21,816,2519,5093,8501,2329,2855,3116,5090,4511,4222,7628,11231,4470,5022,6169,4082,3494,2465,5789,6895,7230,5342,6590,3805,1258,1236,1452,780,1763,766,1843,2513,893,1058,1111,768,767,410,450,570,441,372,628,359,865,521,334,497,1492,170,96,10,25,23,0,1,1,4,2,7,0,4,12818,76
    2021/05/22,818,2526,5126,8525,2335,2872,3126,5111,4532,4240,7639,11279,4484,5045,6202,4102,3511,2481,5812,6915,7245,5357,6622,3822,1266,1237,1460,784,1769,769,1847,2520,895,1061,1118,770,769,411,450,570,441,375,632,360,868,526,336,502,1495,170,96,10,25,23,0,1,1,4,2,7,0,4,12873,76
    2021/05/22,818,2526,5126,8525,2335,2872,3126,5111,4532,4240,7639,11279,4484,5045,6202,4102,3511,2481,5812,6915,7245,5357,6622,3822,1266,1237,1460,784,1769,769,1847,2520,895,1061,1118,770,769,411,450,570,441,375,632,360,868,526,336,502,1495,170,96,10,25,23,0,1,1,4,2,7,0,4,12873,76
    2021/05/23,819,2531,5140,8543,2337,2882,3139,5126,4541,4255,7658,11328,4507,5054,6221,4111,3525,2488,5840,6937,7264,5374,6646,3844,1271,1244,1461,785,1782,769,1854,2532,900,1069,1122,774,773,411,453,572,441,379,632,360,869,526,336,503,1502,170,99,10,25,23,0,1,1,4,2,7,0,4,12933,76
    2021/05/23,819,2531,5140,8543,2337,2882,3139,5126,4541,4255,7658,11328,4507,5054,6221,4111,3525,2488,5840,6937,7264,5374,6646,3844,1271,1244,1461,785,1782,769,1854,2532,900,1069,1122,774,773,411,453,572,441,379,632,360,869,526,336,503,1502,170,99,10,25,23,0,1,1,4,2,7,0,4,12933,76
    2021/05/24,821,2534,5147,8555,2339,2885,3147,5134,4548,4262,7674,11340,4517,5066,6247,4118,3531,2495,5850,6949,7271,5380,6655,3856,1275,1246,1464,785,1788,770,1855,2548,901,1075,1124,776,775,413,453,575,442,380,632,360,870,527,336,506,1506,172,99,10,25,23,0,1,1,4,2,7,0,4,12998,76
    2021/05/24,821,2534,5147,8555,2339,2885,3147,5134,4548,4262,7674,11340,4517,5066,6247,4118,3531,2495,5850,6949,7271,5380,6655,3856,1275,1246,1464,785,1788,770,1855,2548,901,1075,1124,776,775,413,453,575,442,380,632,360,870,527,336,506,1506,172,99,10,25,23,0,1,1,4,2,7,0,4,12998,76
    2021/05/25,824,2547,5175,8576,2348,2897,3158,5152,4568,4275,7695,11370,4534,5086,6270,4128,3540,2500,5868,6975,7298,5390,6680,3863,1281,1248,1467,785,1801,772,1860,2557,904,1079,1131,778,778,413,458,577,443,383,634,361,871,527,336,512,1506,173,102,10,25,23,0,1,1,4,2,7,0,4,13060,76
    2021/05/25,824,2547,5175,8576,2348,2897,3158,5152,4568,4275,7695,11370,4534,5086,6270,4128,3540,2500,5868,6975,7298,5390,6680,3863,1281,1248,1467,785,1801,772,1860,2557,904,1079,1131,778,778,413,458,577,443,383,634,361,871,527,336,512,1506,173,102,10,25,23,0,1,1,4,2,7,0,4,13060,76
    2021/05/26,831,2562,5194,8613,2352,2910,3168,5175,4604,4298,7723,11432,4555,5103,6298,4146,3560,2519,5899,7010,7327,5406,6711,3876,1296,1255,1478,786,1808,773,1870,2575,908,1084,1133,783,782,414,461,579,445,385,636,362,875,529,339,523,1510,174,103,10,25,23,0,1,1,4,2,7,0,4,13119,76
    2021/05/26,831,2562,5194,8613,2352,2910,3168,5175,4604,4298,7723,11432,4555,5103,6298,4146,3560,2519,5899,7010,7327,5406,6711,3876,1296,1255,1478,786,1808,773,1870,2575,908,1084,1133,783,782,414,461,579,445,385,636,362,875,529,339,523,1510,174,103,10,25,23,0,1,1,4,2,7,0,4,13119,76
    2021/05/27,833,2577,5221,8650,2358,2915,3184,5199,4630,4317,7740,11487,4577,5124,6320,4165,3578,2529,5918,7036,7345,5422,6750,3896,1300,1257,1485,788,1819,777,1873,2588,911,1090,1140,789,789,416,464,581,447,385,639,364,879,532,339,528,1515,176,104,10,25,23,0,1,1,4,2,7,0,4,13195,76
    2021/05/27,833,2577,5221,8650,2358,2915,3184,5199,4630,4317,7740,11487,4577,5124,6320,4165,3578,2529,5918,7036,7345,5422,6750,3896,1300,1257,1485,788,1819,777,1873,2588,911,1090,1140,789,789,416,464,581,447,385,639,364,879,532,339,528,1515,176,104,10,25,23,0,1,1,4,2,7,0,4,13195,76
    2021/05/28,838,2584,5234,8669,2363,2925,3196,5216,4650,4334,7760,11533,4603,5147,6350,4186,3595,2554,5944,7061,7362,5440,6769,3910,1305,1264,1491,789,1824,779,1881,2598,916,1091,1143,796,794,416,464,585,448,386,642,364,885,533,342,537,1519,176,104,10,25,23,0,1,1,4,2,7,0,4,13260,76
    2021/05/28,838,2584,5234,8669,2363,2925,3196,5216,4650,4334,7760,11533,4603,5147,6350,4186,3595,2554,5944,7061,7362,5440,6769,3910,1305,1264,1491,789,1824,779,1881,2598,916,1091,1143,796,794,416,464,585,448,386,642,364,885,533,342,537,1519,176,104,10,25,23,0,1,1,4,2,7,0,4,13260,76
    2021/05/29,840,2593,5252,8700,2370,2936,3203,5232,4666,4345,7783,11570,4617,5168,6373,4202,3611,2574,5965,7078,7375,5453,6793,3918,1308,1267,1499,789,1829,780,1886,2605,921,1099,1145,801,799,420,468,586,448,386,643,364,886,535,342,539,1526,176,104,10,25,23,0,1,1,4,2,7,0,4,13326,76
    2021/05/29,840,2593,5252,8700,2370,2936,3203,5232,4666,4345,7783,11570,4617,5168,6373,4202,3611,2574,5965,7078,7375,5453,6793,3918,1308,1267,1499,789,1829,780,1886,2605,921,1099,1145,801,799,420,468,586,448,386,643,364,886,535,342,539,1526,176,104,10,25,23,0,1,1,4,2,7,0,4,13326,76
    2021/05/30,841,2596,5260,8724,2370,2946,3210,5245,4676,4359,7804,11608,4623,5180,6391,4217,3628,2579,6006,7095,7385,5471,6804,3926,1312,1268,1501,789,1835,781,1888,2614,924,1105,1145,804,800,423,469,589,451,387,646,366,887,535,343,542,1531,176,104,10,25,23,0,1,1,4,2,9,0,4,13381,76
    2021/05/30,841,2596,5260,8724,2370,2946,3210,5245,4676,4359,7804,11608,4623,5180,6391,4217,3628,2579,6006,7095,7385,5471,6804,3926,1312,1268,1501,789,1835,781,1888,2614,924,1105,1145,804,800,423,469,589,451,387,646,366,887,535,343,542,1531,176,104,10,25,23,0,1,1,4,2,9,0,4,13381,76
    2021/05/31,844,2598,5268,8733,2377,2950,3211,5255,4684,4367,7814,11629,4630,5200,6401,4232,3632,2583,6022,7103,7397,5477,6819,3931,1315,1271,1503,790,1835,782,1889,2615,924,1106,1146,804,801,424,469,589,451,389,648,367,888,535,343,542,1533,177,105,10,25,23,0,1,1,4,2,9,0,4,13402,76
    2021/05/31,844,2598,5268,8733,2377,2950,3211,5255,4684,4367,7814,11629,4630,5200,6401,4232,3632,2583,6022,7103,7397,5477,6819,3931,1315,1271,1503,790,1835,782,1889,2615,924,1106,1146,804,801,424,469,589,451,389,648,367,888,535,343,542,1533,177,105,10,25,23,0,1,1,4,2,9,0,4,13402,76
    2021/06/01,846,2605,5286,8761,2385,2957,3215,5282,4697,4386,7834,11658,4647,5211,6423,4247,3638,2589,6048,7136,7407,5486,6830,3939,1321,1274,1506,790,1839,786,1893,2617,927,1112,1147,812,802,425,473,591,453,390,649,367,889,536,343,542,1539,177,105,10,25,23,0,1,1,4,2,9,0,4,13453,76
    2021/06/01,846,2605,5286,8761,2385,2957,3215,5282,4697,4386,7834,11658,4647,5211,6423,4247,3638,2589,6048,7136,7407,5486,6830,3939,1321,1274,1506,790,1839,786,1893,2617,927,1112,1147,812,802,425,473,591,453,390,649,367,889,536,343,542,1539,177,105,10,25,23,0,1,1,4,2,9,0,4,13453,76
    2021/06/02,851,2612,5301,8783,2401,2969,3224,5307,4710,4404,7854,11712,4666,5221,6430,4260,3648,2604,6064,7148,7425,5496,6861,3943,1326,1278,1512,791,1844,789,1899,2622,930,1113,1148,815,807,426,475,592,454,390,651,367,889,537,343,545,1541,179,105,10,25,23,0,1,1,4,2,9,0,4,13496,76
    2021/06/02,851,2612,5301,8783,2401,2969,3224,5307,4710,4404,7854,11712,4666,5221,6430,4260,3648,2604,6064,7148,7425,5496,6861,3943,1326,1278,1512,791,1844,789,1899,2622,930,1113,1148,815,807,426,475,592,454,390,651,367,889,537,343,545,1541,179,105,10,25,23,0,1,1,4,2,9,0,4,13496,76
    2021/06/03,851,2616,5318,8817,2405,2977,3230,5326,4723,4421,7885,11752,4681,5237,6451,4277,3656,2615,6079,7170,7445,5514,6888,3952,1329,1280,1517,791,1852,789,1906,2626,933,1117,1152,817,809,431,475,592,454,391,653,368,892,539,344,547,1549,179,105,10,25,23,0,1,1,4,2,9,0,4,13543,76
    2021/06/03,851,2616,5318,8817,2405,2977,3230,5326,4723,4421,7885,11752,4681,5237,6451,4277,3656,2615,6079,7170,7445,5514,6888,3952,1329,1280,1517,791,1852,789,1906,2626,933,1117,1152,817,809,431,475,592,454,391,653,368,892,539,344,547,1549,179,105,10,25,23,0,1,1,4,2,9,0,4,13543,76
    2021/06/04,852,2626,5355,8842,2410,2982,3241,5338,4738,4443,7901,11791,4692,5256,6474,4290,3663,2622,6090,7185,7460,5527,6906,3963,1335,1286,1520,792,1859,792,1911,2634,937,1118,1153,817,812,431,476,592,456,391,654,369,895,540,346,549,1550,179,105,10,25,23,0,1,1,4,2,9,0,4,13592,76
    2021/06/04,852,2626,5355,8842,2410,2982,3241,5338,4738,4443,7901,11791,4692,5256,6474,4290,3663,2622,6090,7185,7460,5527,6906,3963,1335,1286,1520,792,1859,792,1911,2634,937,1118,1153,817,812,431,476,592,456,391,654,369,895,540,346,549,1550,179,105,10,25,23,0,1,1,4,2,9,0,4,13592,76
    2021/06/05,854,2629,5376,8860,2420,2990,3243,5353,4752,4455,7927,11830,4698,5271,6488,4311,3676,2629,6097,7201,7474,5532,6929,3971,1338,1288,1524,793,1862,793,1914,2645,939,1122,1155,818,816,432,479,592,456,392,654,369,897,544,347,549,1553,179,106,10,25,23,0,1,1,4,2,9,0,4,13652,76
    2021/06/05,854,2629,5376,8860,2420,2990,3243,5353,4752,4455,7927,11830,4698,5271,6488,4311,3676,2629,6097,7201,7474,5532,6929,3971,1338,1288,1524,793,1862,793,1914,2645,939,1122,1155,818,816,432,479,592,456,392,654,369,897,544,347,549,1553,179,106,10,25,23,0,1,1,4,2,9,0,4,13652,76
    2021/06/06,854,2637,5394,8878,2428,2993,3252,5362,4769,4461,7936,11868,4708,5287,6493,4321,3682,2637,6099,7207,7489,5539,6941,3983,1340,1291,1527,794,1869,794,1923,2655,941,1126,1155,823,817,434,479,592,456,392,655,369,897,544,347,551,1563,180,106,10,25,23,0,1,1,4,2,9,0,4,13687,76
    2021/06/06,854,2637,5394,8878,2428,2993,3252,5362,4769,4461,7936,11868,4708,5287,6493,4321,3682,2637,6099,7207,7489,5539,6941,3983,1340,1291,1527,794,1869,794,1923,2655,941,1126,1155,823,817,434,479,592,456,392,655,369,897,544,347,551,1563,180,106,10,25,23,0,1,1,4,2,9,0,4,13687,76
    2021/06/07,859,2640,5404,8895,2431,2995,3256,5367,4783,4468,7941,11884,4714,5292,6503,4332,3686,2639,6103,7214,7498,5546,6948,3989,1342,1291,1529,794,1871,794,1927,2656,943,1128,1155,825,819,434,479,592,456,393,657,369,898,544,347,552,1566,180,106,10,25,23,0,1,1,4,2,9,0,4,13726,76
    2021/06/07,859,2640,5404,8895,2431,2995,3256,5367,4783,4468,7941,11884,4714,5292,6503,4332,3686,2639,6103,7214,7498,5546,6948,3989,1342,1291,1529,794,1871,794,1927,2656,943,1128,1155,825,819,434,479,592,456,393,657,369,898,544,347,552,1566,180,106,10,25,23,0,1,1,4,2,9,0,4,13726,76
    2021/06/08,860,2645,5428,8921,2439,3001,3271,5386,4796,4479,7964,11906,4727,5301,6508,4340,3696,2647,6112,7225,7514,5548,6958,3993,1344,1292,1532,795,1873,798,1931,2660,945,1130,1156,830,820,436,480,592,459,393,660,370,903,545,347,554,1572,180,106,10,25,23,0,1,1,4,2,9,0,4,13761,76
    2021/06/08,860,2645,5428,8921,2439,3001,3271,5386,4796,4479,7964,11906,4727,5301,6508,4340,3696,2647,6112,7225,7514,5548,6958,3993,1344,1292,1532,795,1873,798,1931,2660,945,1130,1156,830,820,436,480,592,459,393,660,370,903,545,347,554,1572,180,106,10,25,23,0,1,1,4,2,9,0,4,13761,76
    2021/06/09,864,2653,5444,8946,2448,3008,3283,5401,4806,4495,7987,11948,4743,5318,6531,4346,3703,2652,6127,7243,7523,5555,6980,4005,1349,1296,1535,796,1878,801,1939,2667,948,1131,1158,832,821,437,480,597,459,393,661,370,907,547,348,554,1578,180,106,10,25,23,0,1,1,4,2,9,0,4,13792,76
    2021/06/09,864,2653,5444,8946,2448,3008,3283,5401,4806,4495,7987,11948,4743,5318,6531,4346,3703,2652,6127,7243,7523,5555,6980,4005,1349,1296,1535,796,1878,801,1939,2667,948,1131,1158,832,821,437,480,597,459,393,661,370,907,547,348,554,1578,180,106,10,25,23,0,1,1,4,2,9,0,4,13792,76
    2021/06/10,864,2664,5465,8977,2456,3017,3290,5430,4820,4507,7999,11971,4751,5328,6543,4356,3709,2657,6141,7255,7542,5570,6994,4022,1354,1299,1540,797,1893,801,1943,2671,949,1134,1160,837,824,444,482,598,459,394,662,370,910,547,348,554,1587,180,106,10,25,23,0,1,1,4,2,9,0,4,13837,76
    2021/06/10,864,2664,5465,8977,2456,3017,3290,5430,4820,4507,7999,11971,4751,5328,6543,4356,3709,2657,6141,7255,7542,5570,6994,4022,1354,1299,1540,797,1893,801,1943,2671,949,1134,1160,837,824,444,482,598,459,394,662,370,910,547,348,554,1587,180,106,10,25,23,0,1,1,4,2,9,0,4,13837,76
    2021/06/11,866,2676,5475,8996,2459,3019,3306,5450,4836,4523,8018,12005,4762,5350,6563,4370,3720,2664,6160,7270,7562,5581,7012,4026,1359,1306,1543,797,1901,802,1949,2675,955,1136,1164,839,825,446,482,601,459,394,662,372,916,548,348,555,1596,180,106,10,25,23,0,1,1,4,2,9,0,4,13858,76
    2021/06/11,866,2676,5475,8996,2459,3019,3306,5450,4836,4523,8018,12005,4762,5350,6563,4370,3720,2664,6160,7270,7562,5581,7012,4026,1359,1306,1543,797,1901,802,1949,2675,955,1136,1164,839,825,446,482,601,459,394,662,372,916,548,348,555,1596,180,106,10,25,23,0,1,1,4,2,9,0,4,13858,76
    2021/06/12,867,2688,5500,9021,2468,3024,3316,5473,4852,4533,8054,12033,4772,5369,6580,4383,3731,2672,6173,7279,7578,5591,7028,4035,1362,1306,1549,799,1907,802,1955,2687,956,1137,1167,841,829,446,485,603,461,394,662,372,921,549,348,557,1606,180,106,10,25,23,0,1,1,4,2,9,0,4,13903,76
    2021/06/12,867,2688,5500,9021,2468,3024,3316,5473,4852,4533,8054,12033,4772,5369,6580,4383,3731,2672,6173,7279,7578,5591,7028,4035,1362,1306,1549,799,1907,802,1955,2687,956,1137,1167,841,829,446,485,603,461,394,662,372,921,549,348,557,1606,180,106,10,25,23,0,1,1,4,2,9,0,4,13903,76
    2021/06/13,870,2694,5504,9032,2474,3027,3327,5489,4864,4541,8071,12064,4779,5375,6593,4395,3739,2680,6181,7288,7604,5597,7040,4039,1365,1308,1553,799,1913,802,1956,2691,958,1140,1167,842,829,446,485,603,464,394,662,372,923,550,348,557,1612,180,106,10,25,23,0,1,1,4,2,9,0,4,13922,76
    2021/06/13,870,2694,5504,9032,2474,3027,3327,5489,4864,4541,8071,12064,4779,5375,6593,4395,3739,2680,6181,7288,7604,5597,7040,4039,1365,1308,1553,799,1913,802,1956,2691,958,1140,1167,842,829,446,485,603,464,394,662,372,923,550,348,557,1612,180,106,10,25,23,0,1,1,4,2,9,0,4,13922,76
    2021/06/14,870,2698,5510,9042,2477,3030,3335,5495,4872,4549,8080,12081,4789,5382,6599,4401,3746,2682,6189,7305,7607,5602,7044,4043,1366,1309,1555,799,1914,804,1958,2693,959,1141,1168,842,830,446,485,603,464,395,662,372,924,550,348,558,1618,180,106,10,25,23,0,1,1,4,2,9,0,4,13946,76
    2021/06/14,870,2698,5510,9042,2477,3030,3335,5495,4872,4549,8080,12081,4789,5382,6599,4401,3746,2682,6189,7305,7607,5602,7044,4043,1366,1309,1555,799,1914,804,1958,2693,959,1141,1168,842,830,446,485,603,464,395,662,372,924,550,348,558,1618,180,106,10,25,23,0,1,1,4,2,9,0,4,13946,76

