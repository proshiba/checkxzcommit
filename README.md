# XZ Utilsの件で、Jia Tan氏のcommitメッセージを分析してみました

色々話題になっているので、少し調べてみる。といってもほとんど調べてあるため、少し気になったこととしてコミットメッセージで特徴的な文章や繰り返しているスペルミスがないかを調べたので記録として残しておきます。  

## 結果

1. 誤字として以下の2つが繰り返し使われていました。
   1. guarentee -> 正=guarantee : 3回同じ誤字が使われていました。
   2. depend -> 正=depened : 9回同じ誤字が使われていました。
2. 言い回しに特徴的なものはありませんでした。

これらは全てOpenAIのAPIを使って確認したものです。  
一応備忘録敵に最後にやり方についてのメモの残しています。

全ての結果は、[コミットメッセージの誤字etc](./results/スペルミスを含む全てのcommit-message.txt)に残してあります。

## コミットの日時とタイムゾーン

[activity-file](./results/committer-or-author-activity.txt)に全て残してあります。  

- ほとんどは中国のタイムゾーンである UTC+8:00になっていました。しかしこれは偽装のための可能性があります
- UTC+2:00、UTC+3:00がいくつかのコミットで確認できました。
    - UTC+2:00: 1回のみ。(2022-11-07 (Mon) 14:24:14 +0000 (16:24 +0200)) このタイムゾーンは東欧で例えばチェコとかです。
    - UTC+3:00: 6回、タイミングもバラバラでこのタイムゾーンがありました。このタイムゾーンの有名な国はロシアです。
        - 2022-06-16 (Thu) 14:32:19 +0000 (17:32 +0300)
        - 2022-07-25 (Mon) 15:20:01 +0000 (18:20 +0300)
        - 2022-07-25 (Mon) 15:30:05 +0000 (18:30 +0300)
        - 2022-09-08 (Thu) 12:07:00 +0000 (15:07 +0300)
        - 2022-10-06 (Thu) 18:53:09 +0000 (21:53 +0300)
        - 2023-06-27 (Tue) 14:27:09 +0000 (17:27 +0300)

## やり方について

結構雑なやり方であるが、同じように分析したい人に向けて一応書いておく。

### ページ構成などを確認

断っておくこととして、始めはcurlで取得する処理をしていたが、別にこの方法でなくても大丈夫。（当たり前だが）  
実際にはpython/requestsとかで全部スクリプトにした方がいいと思う。

まず始めに、以下のgitサイトを見てページ構成などを確認した。
- https://git.tukaani.org/

結果、Jia Tanがauthorであるものを検索できたためこれを確認。
- https://git.tukaani.org/?p=xz.git;a=search;s=Jia+Tan;st=author

この際、トップのページと`pg=1から4`までの5ページを取れば全てのコミットIDが取得できると確認した。  

### commit IDの取得

以下の2コマンドでcommitIDを取得

```bash
curl 'https://git.tukaani.org/?p=xz.git;a=search;pg=1;s=Jia+Tan;st=author' | grep "commit</a>" | cut -d '|' -f 1 | cut -d '"' -f 4 > links.txt
for i in $(seq 4); do echo curl "https://git.tukaani.org/?p=xz.git;a=search;pg=$i;s=Jia+Tan;st=author" | grep "commit</a>" | cut -d '|' -f 1 | cut -d '"' -f 4 >> links.txt; done
```

### commitメッセージの取得

各コミット用のリンクを取得したら、このリンクの内容をベースに各コミットの内容をチェックする。
Note: 以下の処理は、一度限りで作ったものですし、途中でhtmlを保存したりする処理もあったので、余計な処理が入っていたりでとても読みづらいです。ご容赦ください。

```python
import json
import bs4
import requests

f=open("./links.txt")
links = f.read().strip().split("\n")
for l in links:
    tmp=l.split(";h=")
    tmp
    if len(tmp) == 1:
       continue
    key=tmp[1]
    v=requests.get("https://git.tukaani.org"+l)
    htmltext=v.text
    soup = bs4.BeautifulSoup(htmltext)
    message = soup.find("div", "page_body").text
    users = soup.find("table", "object_header")
    lines = users.find_all("tr")
    commitinfo = { "author" : {"name":lines[0].text, "datetime": lines[1].text}, "committer": { "name":lines[2].text, "datetime":lines[3].text}}
    commitinfo["message"] = message
    results[key] = commitinfo
    print("get Commit Info at -> ".format(key))
    time.sleep(1)
```
### 取得したcommit メッセージをOpenAIのAPIを使って分析。

**OpenAIのAPIは有償**です。私はjia tanがauthor+committerのメッセージを全てチェックした際に3ドルほどかかりました。  
以下はざっくり全て誤字の有無などをチェックした際のプロンプトなどです。  

```python
commit_messages = [ each["message"] for each in results.values() ]

system_prompt = """
ユーザーが入力した文章について教えてください。
1. 誤字はありますか？
   - Yesの場合は、その文字と正しいスペルを回答してください。
   - Noの場合は、Noとのみ回答してください。
2. 特徴的な言い回しはありますか？
   - Yesの場合は、どういうタイプの人に多い言い回しかを回答してください。
   - Noの場合は、Noとのみ回答してください。

例1）
1. dictionaly(dictonary)
2. No

例2）
1. No
2. シンガポールでよく使われるスペルです
"""

msg_system = {"role" : "system", "content" : system_prompt}

results = []
for each_msg in commit_messages:
   user_input=each_msg
   msg_user = {"role": "user", "content": user_input}
   messages= [msg_system, msg_user]
   completion = client.chat.completions.create(messages=messages, model="gpt-4")
   results.append({"msg" : user_input, "result": completion.choices[0].message.content})
```

