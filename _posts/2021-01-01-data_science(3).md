---
layout: post
title: "[Python-Quant] 금융공학 모델링 (펀드)"
category: Data_Science
tags: 
comments: true
toc: true
---
### 대한민국 대표 펀드 따라잡기
+ KODEX200(삼성자산운용): https://www.kodex.com
+ TIGER200(미래에셋자산운용): https://www.tigeretf.com
+ 두 펀드 모두 KOSPI200 지수의 변동률과 유사하게 운용하는 것을 목적으로 함
+ 펀드 자산의 95% 이상을 KOSPI200 지수 구성종목인 무식을 매수해서 수익률을 달성하는 방법 채택

+ 대표적인 지수 수익률 복제 방법
  - 지수 구성종목 주식을 그대로 보유 (KODEX200, TIGER200)
  - 지수 선물을 보유 (파생형 펀드로 자산을 굴리는 방법이 다양해진다는 장점이 있지만 자산 모집에 한계 있음 <- 퇴직연금, IRP 같은 안전자산은 파생형 펀드 투자가 불가능)

#### K10 인덱스 활용하여 인덱스 펀드 만들기
+ K10 펀드 투자설명서 (355pg)
+ 현실보다는 훨씬 단순화
+ 전에 K10 인덱스를 만들 때 썻던 코드 활용


```python
import sys
import os
import pandas as pd
import numpy as np
import re
import json
from bs4 import BeautifulSoup
from urllib.request import urlopen
import datetime as dt
from sklearn.linear_model import LinearRegression
import matplotlib as mpl
import matplotlib.pyplot as plt
import matplotlib.font_manager as fm
fname = fm.FontProperties(fname='D:/data_starfish/jupyter/malgun.ttf').get_name()
from IPython.display import Image
mpl.rc('font',family=fname)
import matplotlib.gridspec as gridspec
from jupyterthemes import jtplot
jtplot.style(theme='onedork')
%matplotlib inline
import platform
platform.system()
# 운영체제별 한글 폰트 설정
if platform.system() == 'Darwin': # Mac 환경 폰트 설정
    plt.rc('font', family='AppleGothic')
elif platform.system() == 'Windows': # Windows 환경 폰트 설정
    plt.rc('font', family='Malgun Gothic')
plt.rc('axes', unicode_minus=False) # 마이너스 폰트 설정
# 글씨 선명하게 출력하는 설정
%config InlineBackend.figure_format = 'retina'
```

```python
def stock_info(stock_cd):
    url_float = 'http://companyinfo.stock.naver.com/v1/company/c1010001.aspx?cmp_cd=' + stock_cd
    source = urlopen(url_float).read()
    soup = BeautifulSoup(source, 'lxml')
    
    temp = soup.find(id="cTB11").find_all('tr')[6].td.text
    temp = temp.replace('\r','')
    temp = temp.replace('\n','')
    temp = temp.replace('\t','')
    
    temp = re.split('/', temp)
    
    # 발행주식 수
    outstanding = temp[0].replace(',','') 
    outstanding = outstanding.replace('주','')
    outstanding = outstanding.replace(' ','')
    outstanding = int(outstanding)
    # 유동비율
    floating = temp[1].replace(' ','')
    floating = floating.replace('%','')
    floating = float(floating)
    #종목명
    name = soup.find(id="pArea").find('div').find('div').find('tr').find('td').find('span').text
    
    k10_outstanding[stock_cd] = outstanding
    k10_floating[stock_cd] = floating
    k10_name[stock_cd] = name
```

'''
한국거래소 시가총액 상위 10종목 (2020년 12월 기준)
005930 삼성전자
000660 SK하이닉스
051910 LG화학
005935 삼성전자(우)
207940 삼성바이오로직스
068270 셀트리온
035420 NAVER
005380 현대차
006400 삼성SDI
035720 카카오
'''

```python
k10_component = ['005930','000660','051910','005935',
                 '207940','068270','035420','005380',
                 '006400','035720']
```

```python
k10_outstanding = dict()
k10_floating = dict()
k10_name = dict()

for stock_cd in k10_component:
    stock_info(stock_cd)
```

```python
# K10 구성종목의 종목 정보를 데이터프레임화
tmp = {'Outstanding': k10_outstanding,\
      'Floating': k10_floating,\
      'Name': k10_name}
k10_info = pd.DataFrame(tmp)
```

```python
# 일자별 주가 수집을 위해 날짜 포맷 정리 함수

def date_format(d):
    
    d = str(d)
    d = d.replace('/','-')
    d = d.replace('.','-')
    
    yyyy = int(d.split('-')[0])
    if yyyy < 50:
        yyyy = yyyy + 2000
    elif yyyy >= 50 and yyyy < 100:
        yyyy = yyyy + 1900
    mm = int(d.split('-')[1])
    dd = int(d.split('-')[2])
    
    return dt.date(yyyy, mm, dd)
```

```python
def historical_stock_naver(index_cd, start_date='', end_date='', page_n=1, last_page=0):
    
    if start_date:   
        start_date = date_format(start_date)   
    else:    
        start_date = dt.date.today() 
    if end_date:   
        end_date = date_format(end_date)   
    else:   
        end_date = dt.date.today()  
        
        
    naver_stock = \
    'https://finance.naver.com/item/sise_day.nhn?code=' \
    + stock_cd + '&page=' + str(page_n)
    
    source = urlopen(naver_stock).read()
    source = BeautifulSoup(source, 'lxml')   
    
    dates = source.find_all('span', class_='tah p10 gray03')  
    prices = source.find_all('td', class_='num')
    
    for n in range(len(dates)):
        
        if len(dates) > 0:
            
            this_date = dates[n].text
            this_date= date_format(this_date)
            
            if this_date <= end_date and this_date >= start_date:   

                this_close = prices[n*6].text
                this_close = this_close.replace(',', '')
                this_close = float(this_close)

                historical_prices[this_date] = this_close
                
            elif this_date < start_date:   

                return historical_prices              
            
    # 페이지 네비게이션
    if last_page == 0:
        last_page = source.find_all('table')[1].find('td', class_='pgRR').find('a')['href']
        last_page = last_page.split('&')[1]
        last_page = last_page.split('=')[1]
        last_page = int(last_page) 
        
    # 다음 페이지 호출
    if page_n < last_page:   
        page_n = page_n + 1   
        historical_stock_naver(index_cd, start_date, end_date, page_n, last_page)   
        
    return historical_prices
```

```python
# 2020년 1월 1일부터 2020년 12월 28일까지의 시세 수집

k10_historical_prices = dict()
for stock_cd in k10_component:
    historical_prices = dict()
    start_date = '2020-01-01'
    end_date = '2020-12-28'
    historical_prices = historical_stock_naver(stock_cd,start_date,end_date)
    
    k10_historical_prices[stock_cd] = historical_prices
```

```python
k10_historical_price = pd.DataFrame(k10_historical_prices)
k10_historical_price.sort_index(axis=1, inplace=True)
```

```python
k10_historical_price = k10_historical_price.fillna(method='bfill')
if k10_historical_price.isnull().values.any():
    k10_historical_price = k10_historical_price.fillna(method='ffill')
k10_historical_price.head(3)
>> \begin{table}[]
\begin{tabular}{lllllllllll}
\hline
\rowcolor[HTML]{373E4B} 
\multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{000660}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{005380}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{005930}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{005935}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{006400}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{035420}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{035720}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{051910}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{068270}}} & \multicolumn{1}{r}{\cellcolor[HTML]{373E4B}{\color[HTML]{D4D8EC} \textbf{207940}}} \\ \hline
\rowcolor[HTML]{414B5E} 
{\color[HTML]{DBDFEF} \textbf{2020-12-28}}                                   & {\color[HTML]{DBDFEF} \textbf{115500.0}}                                           & {\color[HTML]{DBDFEF} 189500.0}                                                    & {\color[HTML]{DBDFEF} 78700.0}                                                     & {\color[HTML]{DBDFEF} 72900.0}                                                     & {\color[HTML]{DBDFEF} 559000.0}                                                    & {\color[HTML]{DBDFEF} 281000.0}                                                    & {\color[HTML]{DBDFEF} 373000.0}                                                    & {\color[HTML]{DBDFEF} 814000.0}                                                    & {\color[HTML]{DBDFEF} 333500.0}                                                    & {\color[HTML]{DBDFEF} 789000.0}                                                    \\
\rowcolor[HTML]{3B4455} 
{\color[HTML]{DBDFEF} \textbf{2020-12-24}}                                   & {\color[HTML]{DBDFEF} \textbf{118000.0}}                                           & {\color[HTML]{DBDFEF} 187000.0}                                                    & {\color[HTML]{DBDFEF} 77800.0}                                                     & {\color[HTML]{DBDFEF} 72800.0}                                                     & {\color[HTML]{DBDFEF} 563000.0}                                                    & {\color[HTML]{DBDFEF} 282000.0}                                                    & {\color[HTML]{DBDFEF} 374000.0}                                                    & {\color[HTML]{DBDFEF} 818000.0}                                                    & {\color[HTML]{DBDFEF} 347500.0}                                                    & {\color[HTML]{DBDFEF} 794000.0}                                                    \\
\rowcolor[HTML]{414B5E} 
{\color[HTML]{DBDFEF} \textbf{2020-12-23}}                                   & {\color[HTML]{DBDFEF} \textbf{116000.0}}                                           & {\color[HTML]{DBDFEF} 185000.0}                                                    & {\color[HTML]{DBDFEF} 73900.0}                                                     & {\color[HTML]{DBDFEF} 69900.0}                                                     & {\color[HTML]{DBDFEF} 554000.0}                                                    & {\color[HTML]{DBDFEF} 284000.0}                                                    & {\color[HTML]{DBDFEF} 377500.0}                                                    & {\color[HTML]{DBDFEF} 806000.0}                                                    & {\color[HTML]{DBDFEF} 355000.0}                                                    & {\color[HTML]{DBDFEF} 796000.0}                                                    \\ \hline
\end{tabular}
\end{table}
```

```python
# 유동주식 비율 반영한 시가총액 구하기
k10_historical_mc = k10_historical_price * k10_info['Outstanding'] * k10_info['Floating'] * 0.01
k10_historical_mc.tail(3)
>> 
```
```python
len(dates)
>> 6
```

```python
len(prices) # 체결가, 등략률, 거래량, 거래대금 => 4X6 ==> 첫 번째 값만 필요 (0,4,8...추출)
>> 24
```

```python
for n in range(len(dates)):
    this_date = dates[n].text
    this_date = date_format(this_date)
    
    this_close = prices[n*4].text # 0,4,8...
    this_close = this_close.replace(',','')
    this_close = float(this_close)
    this_close
    
    print(this_date, this_close)
>> 2020-12-15 370.88
2020-12-14 371.56
2020-12-11 372.24
2020-12-10 369.37
2020-12-09 371.47
2020-12-08 363.45
```

```python
# 페이지 내비게이션
# '맨뒤' 검사
paging = source.find('td', class_ = 'pgRR').find('a')['href']
# 페이지 번호만 추출
paging = paging.split('=')[1]
paging
>> '617'
```

```python
# 페이지번호를 숫자 형식으로 변환
last_page = source.find('td', class_='pgRR').find('a')["href"]
last_page = last_page.split('&')[1]
last_page = last_page.split('=')[1]
last_page = int(last_page)
```

#### 데이터 추출 기능의 함수화