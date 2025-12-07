# Pythonコード

# 楽天証券→銘柄リスト→Excel出力

```python
#プログラム実行前に次の1と2が必要です

#1. マーケットスピードIIのログイン
#2. エクセルを起動　空白のブックを開く。RSS接続する。

#3. このプログラムを実行する
#4. 開いているエクセル画面を出し、プログラムが終了するまで待つ
#5. 終了したら、作成し保存したリストを確認する

import win32com.client #RSSへ接続するためのエクセル用
from openpyxl import Workbook #エクセル保存用
import logging
import sys
from datetime import datetime

# ログ設定
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler(f'meigara_list_{datetime.now().strftime("%Y%m%d_%H%M%S")}.log'),
        logging.StreamHandler(sys.stdout)
    ]
)

try:
    xl = win32com.client.GetObject(Class="Excel.Application")   #今、開いている空白のブック
    logging.info("Excelへの接続に成功しました")
except Exception as e:
    logging.error(f"Excelへの接続に失敗しました: {str(e)}")
    sys.exit(1)

try:
    xl.Visible = True    

    wb=Workbook()   #保存用にエクセルブックを作成
    ws=wb.active
    ws['A1']="証券コード"
    ws['B1']="銘柄"
    ws['C1']="現在値"
    ws['D1']="単位株数"
    ws['E1']="購入可能金額"
    ws['F1']="PER"
    ws['G1']="PBR"
    ws['H1']="ROE"
    ws['I1']="出来高"
    logging.info("ワークブックを作成し、ヘッダーを設定しました")
except Exception as e:
    logging.error(f"ワークブック作成エラー: {str(e)}")
    sys.exit(1)

start_no=1300 #証券コード開始番号
step_no=500 #RSSでは1度に500銘柄まで取得可能
end_no=9999 #証券コード終了番号 

n=0
total_processed=0
logging.info(f"データ取得開始: 証券コード {start_no} から {end_no} まで")

try:
    for j in range(start_no, end_no, step_no):
        batch_start=j
        batch_end=min(j+step_no-1, end_no)
        logging.info(f"バッチ処理開始: {batch_start} - {batch_end}")
        
        try:
            xl.Range("A1:I500").ClearContents()
            logging.debug("Excel範囲 A1:I500 をクリアしました")
        except Exception as e:
            logging.warning(f"Excel範囲クリアエラー: {str(e)}")
            continue
        
        for i in range(step_no):  #このループはRSS関数を入力して値を表示させる
            try:
                stock_no=j+i
                if stock_no<=end_no:        
                    xl.Cells(i+1, 1).Formula = str(stock_no)    #株式コード
                    xl.Cells(i+1, 2).Formula = f"=RssMarket({stock_no}, \"銘柄名称\")" #銘柄
                    xl.Cells(i+1, 3).Formula = f"=RssMarket({stock_no}, \"現在値\")"   #現在値
                    xl.Cells(i+1, 4).Formula = f"=RssMarket({stock_no}, \"単位株数\")" #単位株数
                    xl.Cells(i+1, 5).Formula = f"=C{i+1}*D{i+1}"   #購入可能金額
                    xl.Cells(i+1, 6).Formula = f"=RssMarket({stock_no}, \"PER\")" #PER
                    xl.Cells(i+1, 7).Formula = f"=RssMarket({stock_no}, \"PBR\")"   #PBR
                    xl.Cells(i+1, 8).Formula = f"=G{i+1}/F{i+1}*100"   #ROE (PBR/PER)            
                    xl.Cells(i+1, 9).Formula = f"=RssMarket({stock_no}, \"出来高\")"  #出来高
                    total_processed+=1
            except Exception as e:
                logging.error(f"証券コード {stock_no} のデータ設定エラー: {str(e)}")
                continue         

        for k in range(step_no):   # このループは、存在する証券コードで、PBRが0より大きく1未満のものを、市況リストとして保存用エクセルに入れていく
            try:
                stock_no=j+k
                if stock_no<=end_no:        
                    stock_name=xl.Cells(k+1, 2).Value
                    pbr=xl.Cells(k+1,7).Value
                    
                    if stock_name and pbr:
                        try:
                            pbr_float=float(pbr)
                            if stock_name!="" and 0.0 < pbr_float < 1.0:
                                n=n+1
                                ws[f'A{n+1}']=xl.Cells(k+1,1).Value
                                ws[f'B{n+1}']=xl.Cells(k+1,2).Value
                                ws[f'C{n+1}']=xl.Cells(k+1,3).Value
                                ws[f'D{n+1}']=xl.Cells(k+1,4).Value
                                ws[f'E{n+1}']=xl.Cells(k+1,5).Value
                                ws[f'F{n+1}']=xl.Cells(k+1,6).Value
                                ws[f'G{n+1}']=xl.Cells(k+1,7).Value
                                ws[f'H{n+1}']=xl.Cells(k+1,8).Value
                                ws[f'I{n+1}']=xl.Cells(k+1,9).Value
                                logging.debug(f"証券コード {stock_no} を保存しました")
                        except ValueError as ve:
                            logging.debug(f"証券コード {stock_no} のPBR変換エラー: {str(ve)}")
                            continue
            except Exception as e:
                logging.error(f"証券コード {stock_no} のデータ保存エラー: {str(e)}")
                continue
        
        logging.info(f"バッチ処理完了: {batch_start}-{batch_end}, 現在の有効データ数: {n}")

except Exception as e:
    logging.error(f"メイン処理でエラー: {str(e)}")
    logging.warning("部分的なデータを保存します")

# リストを保存
try:
    filename=f'stock_data_{datetime.now().strftime("%Y%m%d_%H%M%S")}.xlsx'
    wb.save(filename)
    logging.info(f"データ保存完了: {filename}")
    logging.info(f"処理サマリー: 処理件数 {total_processed}, 有効データ {n} 件")
    print(f"データ保存完了: {filename}")
    print(f"処理サマリー: 有効データ {n} 件を保存しました")
except Exception as e:
    logging.error(f"ファイル保存エラー: {str(e)}")
    print(f"エラー: ファイルの保存に失敗しました - {str(e)}")
```


## ゴールデンクロス検知

```python
import yfinance as yf
import pandas as pd
import matplotlib.pyplot as plt

# CSVファイルから銘柄リストを読み込み
csv_file = 'nikkei225.csv'  # CSVファイルのパスを指定
df_tickers = pd.read_csv(csv_file, encoding='utf-8')

# tickersリストを作成（tickerカラムを使用）
tickers = df_tickers['ticker'].tolist()

# 対象銘柄リスト（例: 日経２２５）
#tickers = [ 
#    ]

# パラメータ
short_window = 25   # 短期移動平均（25日）
long_window = 75   # 長期移動平均（75日）
lookback_days = 75 # 過去75日分のデータを取得（75日SMAを計算するため）
cross_window = 5    # ゴールデンクロスをチェックする直近の日数

def detect_golden_cross(ticker, name=None):
    # 株価データを取得
    stock_data = yf.download(ticker, period=f"{lookback_days}d", interval="1d")
    
    if stock_data.empty:
        return None, None
    
    # 短期および長期移動平均を計算
    stock_data['SMA_short'] = stock_data['Close'].rolling(window=short_window).mean()
    stock_data['SMA_long'] = stock_data['Close'].rolling(window=long_window).mean()
    
    # ゴールデンクロス検出
    result = {
        'ticker': ticker,
        'is_golden_cross': False,
        'last_price': stock_data['Close'].iloc[-1],
        'cross_date': None
    }
    
    # 最新5日間でゴールデンクロスが発生したかチェック
    for i in range(-cross_window, 0):
        current_short = stock_data['SMA_short'].iloc[i]
        current_long = stock_data['SMA_long'].iloc[i]
        prev_short = stock_data['SMA_short'].iloc[i-1]
        prev_long = stock_data['SMA_long'].iloc[i-1]
        
        # 短期SMAが長期SMAを上抜け（ゴールデンクロス）
        if prev_short <= prev_long and current_short > current_long:
            result['is_golden_cross'] = True
            result['cross_date'] = stock_data.index[i].date()
            break
    
    return result, stock_data

# 各銘柄をチェック
golden_cross_stocks = []
for ticker in tickers:
    result, stock_data = detect_golden_cross(ticker)
    if result and result['is_golden_cross']:
        golden_cross_stocks.append(result)

# 結果を表示
print("ゴールデンクロスが発生した銘柄:")
for stock in golden_cross_stocks:
    print(f"銘柄: {stock['ticker']}, 最新価格: {stock['last_price']:.2f}, "
          f"ゴールデンクロス発生日: {stock['cross_date']}")

# オプション: ゴールデンクロス銘柄のチャートをプロット
if golden_cross_stocks:
    last_ticker = golden_cross_stocks[-1]['ticker']
    _, stock_data = detect_golden_cross(last_ticker)
    
    plt.figure(figsize=(10, 6))
    plt.plot(stock_data.index, stock_data['Close'], label='Close Price')
    plt.plot(stock_data.index, stock_data['SMA_short'], label=f'{short_window}-day SMA')
    plt.plot(stock_data.index, stock_data['SMA_long'], label=f'{long_window}-day SMA')
    plt.title(f"{last_ticker} - Golden Cross")
    plt.xlabel("Date")
    plt.ylabel("Price")
    plt.legend()
    plt.grid()
    plt.show()
else:
    print("ゴールデンクロスは見つかりませんでした。")
```

## CSV作成

```python
import pandas as pd

# 以前のリスト形式（nikkei225_codes）をここに貼り付け
nikkei225_codes = [
    ['3236', 'プロパスト'],
    ['1699', 'NF原油先物'],
    ['1605', 'INPEX'],
    ['2317', 'システナ'],
    ['8316', '三井住友フィ'],
    ['8035', '東京エレク'],
    ['2267', 'ヤクルト'],
    ['9104', '商船三井'],
    ['2760', '東京エレク'],
    ['6758', 'ソニークル'],
    ['7581', 'サイゼリヤ'],
    ['4592', 'サンパイス'],
    ['9101', '日本郵船'],
    ['9433', 'KDDI'],
    ['7003', '三井E＆S'],
    ['3133', '海帆'],
    ['2644', 'GX半導体'],
    ['1557', 'SPDRS'],
    ['9432', 'NTT'],
    ['2244', 'GXUS'],
    ['7203', 'トヨタ自動'],
    ['7777', 'スリー・デ'],
    ['6920', 'レーザーテ'],
    ['8058', '三菱商事'],
    ['3003', 'ヒューリッ'],
    ['5401', '日本製鉄'],
    ['2914', '日本たばこ'],
    ['1540', '純金上場信'],
    ['4661', 'オリエンタ'],
    ['7974', '任天堂'],
    ['1671', 'WTI原油'],
    ['9434', 'ソフトバン'],
    ['1570', 'NF日経レ'],
    ['316A', 'IFREE'],
    ['7201', '日産自動車'],
    ['5243', 'NOTE'],
    ['4755', '楽天グル'],
    ['2432', 'ディー・エ'],
    ['8897', 'MIRAR'],
    ['4540', 'ツムラ'],
    ['9348', 'ispace'],
    ['1357', 'NF米経タ'],
    ['4676', 'フジ・メデ'],
    ['7608', 'ジーエック'],
    ['3350', 'メタプラネ'],
    ['7011', '三菱重工業'],
    ['8105', '堀田丸正'],
    ['8698', 'マネックス'],
    ['8783', 'abc'],
    ['3905', 'データセグ'],
    ['2334', 'イオレ'],
    ['6203', '豊和工業'],
    ['6857', 'アドバンテ'],
]

# DataFrameに変換し、tickerに'.T'を追加
df = pd.DataFrame(nikkei225_codes, columns=['code', 'name'])
df['ticker'] = df['code'] + '.T'

# CSVに保存
df[['ticker', 'name']].to_csv('nikkei225.csv', index=False, encoding='utf-8')
print("CSVファイル 'nikkei225.csv' を生成しました。")
```