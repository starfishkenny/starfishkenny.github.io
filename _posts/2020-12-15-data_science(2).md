---
layout: post
title: "[Python-Quant] 퀀트 모델링"
category: Data_Science
tags: 
comments: true
toc: true
---
### 주가데이터를 수집하고, 그래프를 그려보고, 회귀분석까지 진행
  + reference: <파이썬을 활용한 금융공학 레시피>, 김용환
  + 네이버와 다음의 주가데이터 활용
  + KOSPI200 --> 네이버, S&P500 --> 다음 

#### 네이버금융에서 KOSPI200 일별시세 자료로의 접근
  + KOSPI200 화면 --> '일별시세'에서 '프레임소스보기' --> 'view-source:' 부분을 제외한 url을 주소창에 입력
  + url구조 확인 (https://finance.naver.com/sise/sise_index_day.nhn?code=종목코드&page=페이지번호)

```python
index_cd = 'KPI200'
page_n = 1
naver_index = 'https://finance.naver.com/sise/sise_index_day.nhn?code=' + index_cd + \
'&page=' + str(page_n)
```

```python
# urllib.request의 urlopen
from urllib.request import urlopen
source = urlopen(naver_index).read()
source
>> b'\n\n\n\n\n\n\n<html lang="ko">\n<head>\n<meta http-equiv="Content-Type" ...
```

```python
# BeautifulSoup
from bs4 import BeautifulSoup
source = BeautifulSoup(source, 'lxml')
print(source.prettify())
>> <html lang="ko">
 <head>
  <meta content="text/html; charset=utf-8" http-equiv="Content-Type"/>
  <title>
   네이버 금융 ...
```

```python
# 필요한 자료는 '날짜'와 '체결가(=종가지수)'
# 원하는 항목에 마우스 오른쪽 --> 검사
# 우리가 필요한 데이터는 모두 <td> 태그에 들어있음을 확인
td = source.find_all('td')
len(td)
>> 54
```

```python
# XPath (XML Path Language) 활용 --> 데이터를 대량으로 제공하는 사이트에서 원하는 데이터의 위치를 찾아낼 때 편리
# Python -> 숫자를 0부터 셈 / XPath -> 숫자를 1부터 셈 ==> 1을 빼줘야함
# /html/body/div/table[1]/tbody/tr[3]/td[1]
source.find_all('table')[0].find_all('tr')[2].find_all('td')[0]
>> <td class="date">2020.12.15</td>
```

```python
d = source.find_all('td', class_='date')[0].text
d
>> '2020.12.15'
```

```python
# datetime 라이브러리 활용
import datetime as dt
yyyy = int(d.split('.')[0])
mm = int(d.split('.')[1])
dd = int(d.split('.')[2])
this_date = dt.date(yyyy, mm, dd)
this_date
>> datetime.date(2020, 12, 15)
```

```python
# 함수화
def date_format(d):
    d = str(d).replace('-','.') # 날짜 구분자가 '-'로 되어있을 때 '.'으로 변환
    yyyy = int(d.split('.')[0])
    mm = int(d.split('.')[1])
    dd = int(d.split('.')[2])
    
    this_date = dt.date(yyyy, mm, dd)
    return this_date
```

```python
# 지수 가져오기
# XPath -> /html/body/div/table[1]/tbody/tr[3]/td[2]
this_close = source.find_all('tr')[2].find_all('td')[1].text
this_close = this_close.replace(',','') # 쉼표 제거
this_close = float(this_close)
this_close
>> 370.88
```

```python
p = source.find_all('td', class_ = 'number_1')[0].text
p
>> '370.88'
```

```python
dates = source.find_all('td', class_='date')
prices = source.find_all('td', class_='number_1')
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