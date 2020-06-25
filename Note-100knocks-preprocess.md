ダミー変数化(pd.get_dummies)
===
- 指定のSeriesの変数値を疎なダミー変数列にする.
```python
tmp = pd.get_dummies(df_customer[["gender_cd"]])
```

四分位とかbinsで分類する(pd.qcut,pd.cut)
===
- 値に応じて任意のコードを割り振るとかの操作は、Series.apply(lambda x:func(x))で任意の関数を適用すればよい
- 四分位とか所定のbinsに分けられれば良ければpd.qcut(),pd.cut()が使える
```python
# 四分位分割
df_tmp["qtile"],bins = pd.qcut(df_hoge["some"],4,retbins = True)
# bins分割
df_tmp["bins"],bins = pd.qcut(df_hoge["some"],bins = [0,10,20,30,40,50,60,np.inf],right=False)
```

strアクセサ,dtアクセサでSeriesを処理する
===
- Sereisの要素にapply()で関数を適用できる
- Sereisの要素にmap()を適用する場合、引数に辞書dictを指定すると要素の置換となる。
```python
filter = df_store["tel_no"].str.contains("^[0-9]{3}-[0-9]{3}-[0-9]{4}$",regex = True)
df_store[filter]

pd.to_datetime(df_receipt["sales_epoch"],unit='s',origin='unix').dt.strftime("%m")
# queryでも使える
tmp = df_receipt.query("not customer_id.str.startswith('Z')",engine="python")

# map()でdict渡すと置換えできる
tmp = df_customer["address"].str[:3].map({"埼玉県":11,"千葉県":12,"東京都":13,"神奈川":14})
```

重複するデータの処理(duplicated)
===
- duplicated()を使うと重複してるデータがTrueとなるdfを返す
- 残す行を選択: 引数keep="first" or "last"
- 重複を判定する列を指定: 引数subset="columns"
- 重複した行の数をカウント.value_counts()を使うと個数を確認できる。
```python
# 論理否定演算子「~」を使って重複行を除くことができる
tmp = df[~df.duplicated()]
# drop_duplicates()でも除くことができる
df.drop_duplicates()
df_cnt = df_receipt[~df_receipt.duplicated(subset=['customer_id', 'sales_ymd'])]
```

条件に合致するdfを取得(df.query)
====
- df_hoge[df_hoge["any"] == some]でもいい
- df.queryの方が直感的でいいかも
- engine="python" を忘れがち
```python
tmp = tmp.query("customer_id == 'CS018205000001' & amount >= 1000")
tmp = tmp.query("customer_id == 'CS018205000001' & (amount >= 1000 | quantity >= 5)")
tmp = df_receipt.query("not customer_id.str.startswith('Z')",engine="python")
```

dfの結合(concat)
====
- indexをキーとする場合はconcatでOK
- 他にmergeがある。データをキーとする場合に使う
- 内部結合、外部結合とかも指定できる
```python
tmp1 = df_receipt[["receipt_no","receipt_sub_no"]]
tmp2 = df_receipt["sales_ymd"].astype("str").astype("datetime64")
pd.concat([tmp1,tmp2],axis=1).head(10)
```

データ型の変更(astype)
====
- Seriesのデータ型変更とかはastypeが汎用的
- astypeのデータ型は "int","float","str"とか指定できる
- 厳密には"int64" , "float128" , "unicode"とか指定できる。
- 他に "object" , "datetime64"も使える
```python
tmp = df_customer["birth_day"].astype("datetime64")
tmp = tmp.dt.strftime("%Y%m%d")
```
- 年月日を表す文字列型ならto_datetime(df["col"])でDatetime型に変更できる
- Python標準ライブラリのdatetime型と同様にstrftime()で任意のフォーマットの文字列に変換することも可能。列の要素すべてに適用する場合は上記のようにdtアクセサを利用する。


dfの変形(stack,unstack,pivot)
====
- stackは列を行にする。幾つかのcolumnをまとめて一列のデータにして縦に広げる感じ
- unstackは逆。multiindexの一列分の行を幾つかのcolumnsに展開して列を増やす感じ
- pivot()は、引数index, columns, valuesに列名を指定してデータを再形成する。
- pivot_tableは下記の感じ。aggfuncを指定できる。
```python
df_tmp = df_tmp.stack()
df_tmp = df_tmp.unstack()
df_tmp = df_tmp.pivot()

df_sales_summary = pd.pivot_table(tmp,index="era",columns="gender_cd",values="amount",aggfunc="sum")
```

Seriesに関数を適用(apply)
====
- groupbyオブジェクトに対するaggメソッドみたいな感じある
```python
tmp["era"] = tmp["age"].apply(lambda x:math.floor(x/10)*10)
```

列同士の演算、新たな列の追加
====
- ちょいちょい忘れる
```python
amounts["diff"] = amounts["amount_x"] - amounts["amount_y"]
```

グループ化と集計(groupby , agg)
====
- 便利なやつ
```python
from collections import Counter
func = lambda x:Counter(x).most_common()[0][0] # 最頻値
tmp = df_receipt.groupby("store_cd")
tmp = tmp.agg({"product_cd":func}) # funcを渡せる
# 間は「:」です。「,」だとエラーになります。

tmp = tmp.agg({"amount":"sum"})
# その他に "max" , "min" , "mean" , "median" が使える

func = lambda x:np.var(x) #分散
# func = lambda x:np.std(x) #標準偏差
tmp = tmp.agg({"amount":func})
```
