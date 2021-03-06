データの保存と読み込み  
(to_csv,read_csv,read_table)
===
```python
# csv,UTF-8,index列含めず,header含めて保存
df_product_full.to_csv('../data/P_df_product_full_UTF-8_header.csv', encoding='UTF-8', index=False,header=True)
# 同上。Excelでの文字化け防止版
df_product_full.to_csv('data/df_product_UTF-8_header.csv',encoding='utf_8_sig',index=False)

# csv,UTF-8,headerありを読み込み
df_tmp = pd.read_csv('../data/P_df_product_full_UTF-8_header.csv')
df_tmp.head(10)
# header無い場合は header=None を指定

# tsvにもできる
df_product_full.to_csv('data/df_product_full_UTF-8_header.tsv',sep="\t",encoding='UTF-8',index=False,header=True)

# tsvの読み込み
tmp = pd.read_table('data/df_product_full_UTF-8_header.tsv',encoding='UTF-8')
tmp.head(10)
```

データの正規化、非正規化  
===
- 正規化とは、データベース内のデータを整理するプロセスのこと。 
- データを保護するために設計されたルールに従ってテーブルを作成し、それらのテーブル間のリレーションシップを確立すること、及び、冗長性と一貫性のない依存関係を排除することで、データベースの柔軟性を高めることが含まれる。
- 第１正規化（重複した列の排除）
- 第２正規化（依存関係のある列の外部化）
- 第３正規化（推移的関数従属と呼ばれる従属関係の分離・外部化)
```python
# 正規化
df_gender = df_customer[['gender_cd', 'gender']].drop_duplicates()
df_customer_s = df_customer.drop(columns='gender')

# 非正規化
df_product_full = pd.merge(df_product,
    df_category[['category_small_cd',
    'category_major_name',
    'category_medium_name',
    'category_small_name']], 
    how = 'inner', on = 'category_small_cd')
```

アンダーサンプリング
===
- 訓練データが不均衡データだと回答が多い方を予測しがちになるので、多い方のデータをアンダーサンプリング（減らす）すると汎用性が高まる
```python
from imblearn.under_sampling import RandomUnderSampler

rs = RandomUnderSampler(random_state=71)

df_sample, _ = rs.fit_sample(df_tmp, df_tmp.buy_flg)
```

時系列データの前処理  
(時系列データインデックス,resample)
===
- 時系列データは日付データindexを使うと便利
- 特にresampleによる集計が便利
```python
tmp = df_receipt.copy()
tmp["sales_date"] = pd.to_datetime(tmp["sales_ymd"].astype("str"))
tmp = tmp.set_index("sales_date")
tmp = tmp[["amount"]].resample("M").sum()

trains = [0]*3
tests = [0]*3
for i in range(3):
    trains[i] = tmp[i:12+i]
    tests[i] = tmp[12+i:18+i]

# 時系列データはランダムサンプリングすると未来のデータをカンニングしてしまうので注意が必要
def split_data(target_df, start, train_size, test_size, slide_window):
    train_start = start * slide_window
    test_start = train_start + train_size
    return df[train_start : test_start], df[test_start : test_start + test_size]

```

学習用・テスト用データの分割(train_test_split)
===
```python
from sklearn.model_selection import train_test_split
train,test = train_test_split(tmp,test_size=0.2,random_state=71)
```

ソート(sort_values)
===
- sort_valuesでキーを指定してソートできる
- 複数キーを指定してソートできる
- 第1キーは降順、第2は昇順とかって指定できる
- drop_duplicates(重複行の削除)をkeep="first" を指定して使用する場合にソート重要
```python
tmp = tmp.sort_values(["amount","customer_id"],ascending=[False,True])
df_customer_u = tmp.drop_duplicates(subset=["customer_name","postal_cd"],keep="first")
```

ジオデータの処理(緯度経度、距離の演算)
===
- 郵便番号から緯度経度を取り出したりできるぽい
- 経度（longitude）、緯度（latitude）
- ジオデータからの距離の求め方は以下の通り
```python
def calc_geo_distance(x1,y1,x2,y2):
    x1 = math.radians(x1)
    x2 = math.radians(x2)
    y1 = math.radians(y1)
    y2 = math.radians(y2)
    distance = 6371*math.acos(math.sin(y1)*math.sin(y2)+math.cos(y1)*math.cos(y2)*math.cos(x1-x2))
    return distance

tmp["distance"] = tmp[["longitude_x","latitude_x","longitude_y","latitude_y"]].apply(
    lambda x:calc_geo_distance(x[0],x[1],x[2],x[3]),axis=1
)

```

dfの結合  
(pd.merge)
===
- pd.merge(df_left,df_right,how="inner",on="keycol")で onの列をキーにして積集合結合、和集合結合、左外部結合とかできる
```python
# onに指定するcolの名前が違ってもleft_on,right_onで指定できる
# 当然ながらキーの内容は同種のものでないと機能しない
tmp = pd.merge(df_customer_1,df_store,how="inner",left_on="application_store_cd",right_on="store_cd")

```

欠損値処理  
(isnull, dropna, fillna, isnan)
===
- 欠損値の有無の確認、削除、補完、条件処理
```python
# 欠損値の数の確認
df_product.isnull().sum()
# 欠損値の削除
df_product_1 = df_product.dropna()
# 欠損値の補完
df_product_2 = df_product.fillna({"unit_price":pricemean,"unit_cost":costmean})
# 欠損値の補充（任意の関数）
func = lambda x:x[1] if np.isnan(x[0]) else x[0]
df_product_4["unit_price"] = df_product_4[["unit_price","price_median"]].apply(func,axis=1)

```

外れ値処理
===
手法の一例
- 標準偏差による除外
- 「第一四分位数-1.5×IQR」よりも下回るもの、または「第三四分位数+1.5×IQR」を超えるものの除外

データサンプリング
===
- sampleはランダムサンプリング
- train_test_splitは層化抽出（元データのデータ割合を維持したままデータサンプリングする）
```python
df_customer.sample(frac=0.01)[:10] # 1%抽出

from sklearn.model_selection import train_test_split
# train_test_splitを使って層化抽出(10%)
train,test = train_test_split(df_customer,test_size=0.1,stratify=df_customer["gender"])
```

日時型の処理と計算  
(datetimeの計算, relativedelta)
===
- relativedeltaを使うのが無難
- シンプルな計算だと日数ベースぽい？
```python
tmp["date1"] = pd.to_datetime(pd_hoge["date"].astype("str"))
tmp["date2"] = pd.to_datetime(pd_hoge["date"])
# 日数を得る
tmp["elapsed_date"] = tmp["date1"] - tmp["date2"]
# 月数を得る
func1 = lambda x:relativedelta(x[0],x[1]).years*12+relativedelta(x[0],x[1]).months
tmp["elapsed_month"] = tmp[["date1","date2"]].apply(func1,axis=1)
# 年数を得る
func2 = lambda x:relativedelta(x[0],x[1]).years
tmp["elapsed_year"] = tmp[["date1","date2"]].apply(func2,axis=1)
# 秒数を得る
func3 = lambda x:x[0].timestamp() - x[1].timestamp()
tmp["elapsed_sec"] = tmp[["date1","date2"]].apply(func3,axis=1)
# 曜日の処理
func4 = lambda x:x - relativedelta(days=x.weekday())
tmp["monday"] = tmp["date"].apply(func4)
```

preprocessing(標準化、正規化)の適用  
(preprocessing.scale, .minmax_scale)
===
- preprocessing.scaleは平均0、標準偏差1で標準化
- preprocessing.minmax_scaleは最小値0、最大値1で正規化
```python
from sklearn import preprocessing
from sklearn.model_selection import train_test_split

# sklearnのpreprocess.scaleを利用すると各データの標本標準偏差を得られる
tmp["amount_ss"] = preprocessing.scale(tmp["amount"])
tmp["amount_mm"] = preprocessing.minmax_scale(tmp["amount"])
```

np関数の適用  
(np.floor, np.ceil, np.round,  
np.log10, np.log,   
np.nanmean, np.nanmedian)
===
- math.floorとかはNaNあるとエラーになるけどnp関数ならNaNをスルーしてくれる
- Series.mean(skipna = True)とかもできる
```python
pricemean = np.round(np.nanmean(df_product["unit_price"]))
```

ダミー変数化(pd.get_dummies)
===
- 指定のSeriesの変数値を疎なダミー変数列にする.
```python
tmp = pd.get_dummies(df_customer[["gender_cd"]])
```

四分位を取得する(np.percentile,somedf.quantile)
===
- np.percentileでダイレクトに四分位とか取得できる
- somedf.quantileではリストで取得できる
- IQRは第一四分位と第三四分位の差である
```python
# 四分位取得
q25 = np.percentile(somedf["hoge"],q=25)
q75 = np.percentile(somedf["hoge"],q=75)
iqr = q75-q25
# 四分位をリストで取得する
qts = somedf["hoge"].quantile([0,0.25, 0.5, 0.75, 1.0])
qts = list(qts["hoge"])
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

# unix秒の場合
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
