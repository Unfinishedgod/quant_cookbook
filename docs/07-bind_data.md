# 데이터 정리하기



앞 CHAPTER에서는 API와 크롤링을 통해 주가, 재무제표, 가치지표를 수집하는 방법을 배웠습니다. 이번 CHAPTER에서는 각각 csv 파일로 저장된 데이터들을 하나로 합친 후 저장하는 과정을 살펴보겠습니다.

## 주가 정리하기

주가는 data/KOR_price 폴더 내에 티커_price.csv 파일로 저장되어 있습니다. 해당 파일들을 불러온 후 데이터를 묶는 작업을 통해 하나의 파일로 합치는 방법을 알아보겠습니다.


```r
library(stringr)
library(xts)
library(magrittr)

KOR_ticker = read.csv('data/KOR_ticker.csv', row.names = 1)
KOR_ticker$'종목코드' =
  str_pad(KOR_ticker$'종목코드', 6, side = c('left'), pad = '0')

price_list = list()

for (i in 1 : nrow(KOR_ticker)) {
  
  name = KOR_ticker[i, '종목코드']
  price_list[[i]] =
    read.csv(paste0('data/KOR_price/', name,
                    '_price.csv'),row.names = 1) %>%
    as.xts()
  
}

price_list = do.call(cbind, price_list) %>% na.locf()
colnames(price_list) = KOR_ticker$'종목코드'
```




```r
head(price_list[, 1:5])
```

```
##            X X005930 X000660 X207940 X035420
## 1 2018-03-05   45200   78300  465500  156021
## 2 2018-03-06   47020   82400  449000  159426
## 3 2018-03-07   48620   82700  448000  159226
## 4 2018-03-08   49200   83500  451000  160027
## 5 2018-03-09   49740   83300  455000  160628
## 6 2018-03-12   49740   84900  458500  161429
```

```r
tail(price_list[, 1:5])
```

```
##              X X005930 X000660 X207940 X035420
## 494 2020-03-06   56500   92600  491000  179500
## 495 2020-03-09   54200   86900  494000  168000
## 496 2020-03-10   54600   89100  496000  172000
## 497 2020-03-11   52100   85500  484000  170000
## 498 2020-03-12   50800   82800  483000  166500
## 499 2020-03-13   49950   82500  456500  166000
```

1. 티커가 저장된 csv 파일을 불러온 후 티커를 6자리로 맞춰줍니다.
2. 빈 리스트인 price_list를 생성합니다.
3. for loop 구문을 이용해 종목별 가격 데이터를 불러온 후 `as.xts()`를 통해 시계열 형태로 데이터를 변경하고 리스트에 저장합니다.
4. `do.call()` 함수를 통해 리스트를 열 형태로 묶습니다.
5. 간혹 결측치가 발생할 수 있으므로, `na.locf()` 함수를 통해 결측치에는 전일 데이터를 사용합니다.
6. 행 이름을 각 종목의 티커로 변경합니다.

해당 작업을 통해 개별 csv 파일로 흩어져 있던 가격 데이터가 하나의 데이터로 묶이게 됩니다.


```r
write.csv(data.frame(price_list), 'data/KOR_price.csv')
```

마지막으로 해당 데이터를 data 폴더에 KOR_price.csv 파일로 저장합니다. 시계열 형태 그대로 저장하면 인덱스가 삭제되므로 데이터 프레임 형태로 변경한 후 저장해야 합니다.

## 재무제표 정리하기

재무제표는 data/KOR_fs 폴더 내 티커_fs.csv 파일로 저장되어 있습니다. 주가는 하나의 열로 이루어져 있어 데이터를 정리하는 것이 간단했지만, 재무제표는 각 종목별 재무 항목이 모두 달라 정리하기 번거롭습니다.


```r
library(stringr)
library(magrittr)
library(dplyr)

KOR_ticker = read.csv('data/KOR_ticker.csv', row.names = 1)
KOR_ticker$'종목코드' =
  str_pad(KOR_ticker$'종목코드', 6, side = c('left'), pad = '0')

data_fs = list()

for (i in 1 : nrow(KOR_ticker)){
  
  name = KOR_ticker[i, '종목코드']
  data_fs[[i]] = read.csv(paste0('data/KOR_fs/', name,
                                 '_fs.csv'), row.names = 1)
}
```





위와 동일하게 티커 데이터를 읽어옵니다. 이를 바탕으로 종목별 재무제표 데이터를 읽어온 후 리스트에 저장합니다.


```r
fs_item = data_fs[[1]] %>% rownames()
length(fs_item)
```

```
## [1] 237
```

```r
print(head(fs_item))
```

```
## [1] "매출액"           "매출원가"        
## [3] "매출총이익"       "판매비와관리비"  
## [5] "인건비"           "유무형자산상각비"
```

다음으로 재무제표 항목의 기준을 정해줄 필요가 있습니다. 재무제표 작성 항목은 각 업종별로 상이하므로, 이를 모두 고려하면 지나치게 데이터가 커지게 됩니다. 또한 퀀트 투자에는 일반적이고 공통적인 항목을 주로 사용하므로 대표적인 재무 항목을 정해 이를 기준으로 데이터를 정리해도 충분합니다.

따라서 기준점으로 첫 번째 리스트, 즉 삼성전자의 재무 항목을 선택하며, 총 237개 재무 항목이 있습니다. 해당 기준을 바탕으로 재무제표 데이터를 정리하며, 전체 항목에 대한 정리 이전에 간단한 예시로 첫 번째 항목인 매출액 기준 데이터 정리를 살펴보겠습니다.


```r
select_fs = lapply(data_fs, function(x) {
    # 해당 항목이 있을시 데이터를 선택
    if ( '매출액' %in% rownames(x) ) {
          x[which(rownames(x) == '매출액'), ]
      
    # 해당 항목이 존재하지 않을 시, NA로 된 데이터프레임 생성
      } else {
      data.frame(NA)
    }
  })

select_fs = bind_rows(select_fs)

print(head(select_fs))
```

```
##   X2016.12 X2017.12 X2018.12 X2019.12 NA.
## 1  2018667  2395754  2437714  2304009  NA
## 2   171980   301094   404451   269907  NA
## 3     2946     4646     5358     7016  NA
## 4    40226    46785    55869    65934  NA
## 5   206593   256980   281830   286250  NA
## 6     6706     9491     9821       NA  NA
```

먼저 `lapply()` 함수를 이용해 모든 재무 데이터가 들어 있는 data_fs 데이터를 대상으로 함수를 적용합니다. `%in%` 함수를 통해 만일 매출액이라는 항목이 행 이름에 있으면 해당 부분의 데이터를 select_fs 리스트에 저장하고, 해당 항목이 없는 경우 NA로 이루어진 데이터 프레임을 저장합니다.

그 후, dplyr 패키지의 `bind_rows()` 함수를 이용해 리스트 내 데이터들을 행으로 묶어줍니다. `rbind()`에서는 리스트 형태를 테이블로 묶으려면 모든 데이터의 열 개수가 동일해야 하는 반면, `bind_rows()`에서는 열 개수가 다를 경우 나머지 부분을 NA로 처리해 합쳐주는 장점이 있습니다.

합쳐진 데이터를 살펴보면, 먼저 열 이름이 . 혹은 NA.인 부분이 있습니다. 이는 매출액 항목이 없는 종목의 경우 NA 데이터 프레임을 저장해 생긴 결과입니다. 또한 연도가 순서대로 저장되지 않은 경우가 있습니다. 이 두 가지를 고려해 데이터를 클렌징합니다.


```r
select_fs = select_fs[!colnames(select_fs) %in%
                        c('.', 'NA.')]
select_fs = select_fs[, order(names(select_fs))]
rownames(select_fs) = KOR_ticker[, '종목코드']

print(head(select_fs))
```

```
##        X2016.12 X2017.12 X2018.12 X2019.12
## 5930    2018667  2395754  2437714  2304009
## 660      171980   301094   404451   269907
## 207940     2946     4646     5358     7016
## 35420     40226    46785    55869    65934
## 51910    206593   256980   281830   286250
## 68270      6706     9491     9821       NA
```

1. `!`와 `%in%` 함수를 이용해, 열 이름에 . 혹은 NA.가 들어가지 않은 열만 선택합니다.
2. `order()` 함수를 이용해 열 이름의 연도별 순서를 구한 후 이를 바탕으로 열을 다시 정리합니다.
3. 행 이름을 티커들로 변경합니다.

해당 과정을 통해 전 종목의 매출액 데이터가 연도별로 정리되었습니다. for loop 구문을 이용해 모든 재무 항목에 대한 데이터를 정리하는 방법은 다음과 같습니다.


```r
fs_list = list()

for (i in 1 : length(fs_item)) {
  select_fs = lapply(data_fs, function(x) {
    # 해당 항목이 있을시 데이터를 선택
    if ( fs_item[i] %in% rownames(x) ) {
          x[which(rownames(x) == fs_item[i]), ]
      
    # 해당 항목이 존재하지 않을 시, NA로 된 데이터프레임 생성
      } else {
      data.frame(NA)
    }
  })

  # 리스트 데이터를 행으로 묶어줌 
  select_fs = bind_rows(select_fs)

  # 열이름이 '.' 혹은 'NA.'인 지점은 삭제 (NA 데이터)
  select_fs = select_fs[!colnames(select_fs) %in%
                          c('.', 'NA.')]
  
  # 연도 순별로 정리
  select_fs = select_fs[, order(names(select_fs))]
  
  # 행이름을 티커로 변경
  rownames(select_fs) = KOR_ticker[, '종목코드']
  
  # 리스트에 최종 저장
  fs_list[[i]] = select_fs

}

# 리스트 이름을 재무 항목으로 변경
names(fs_list) = fs_item
```

위 과정을 거치면 fs_list에 총 237리스트가 생성됩니다. 각 리스트에는 해당 재무 항목에 대한 전 종목의 연도별 데이터가 정리되어 있습니다.


```r
saveRDS(fs_list, 'data/KOR_fs.Rds')
```

마지막으로 해당 데이터를 data 폴더 내에 저장합니다. 리스트 형태 그대로 저장하기 위해 `saveRDS()` 함수를 이용해 KOR_fs.Rds 파일로 저장합니다.

Rds 형식은 파일을 더블 클릭한 후 연결 프로그램을 R Studio로 설정해 파일을 불러올 수 있습니다. 혹은 `readRDS()` 함수를 이용해 파일을 읽어올 수도 있습니다.

## 가치지표 정리하기

가치지표는 data/KOR_value 폴더 내 티커_value.csv 파일로 저장되어 있습니다. 재무제표를 정리하는 방법과 거의 동일합니다.


```r
library(stringr)
library(magrittr)
library(dplyr)

KOR_ticker = read.csv('data/KOR_ticker.csv', row.names = 1)
KOR_ticker$'종목코드' =
  str_pad(KOR_ticker$'종목코드', 6, side = c('left'), pad = '0')

data_value = list()

for (i in 1 : nrow(KOR_ticker)){
  
  name = KOR_ticker[i, '종목코드']
  data_value[[i]] =
    read.csv(paste0('data/KOR_value/', name,
                    '_value.csv'), row.names = 1) %>%
    t() %>% data.frame()

}
```

먼저 티커에 해당하는 파일을 불러온 후 for loop 구문을 통해 가치지표 데이터를 data_value 리스트에 저장합니다. 단, csv 내에 데이터가 \@ref(tab:valuesample)와 같이 행의 형태로 저장되어 있으므로, t() 함수를 이용해 열의 형태로 바꿔주며, 데이터 프레임 형태로 저장합니다.


Table: (\#tab:valuesample)가치지표의 저장 예시

 value       x     
-------  ----------
  PER     Number 1 
  PBR     Number 2 
  PCR     Number 3 
  PSR     Number 4 


```r
data_value = bind_rows(data_value)
print(head(data_value))
```

```
##     PER   PBR      PCR     PSR X1
## 1 13.87 1.134    6.571  1.2942 NA
## 2 29.83 1.253    9.264  2.2252 NA
## 3    NA 6.936 2961.208 43.0506 NA
## 4 46.76 4.193   20.097  4.1356 NA
## 5 76.70 1.383    7.701  0.8397 NA
## 6 83.59 8.312   57.271 22.2819 NA
```

`bind_rows()` 함수를 이용하여 리스트 내 데이터들을 행으로 묶어준 후 데이터를 확인해보면 PER, PBR, PCR, PSR 열 외에 불필요한 NA로 이루어진 열이 존재합니다. 해당 열을 삭제한 후 정리 작업을 하겠습니다.


```r
data_value = data_value[colnames(data_value) %in%
                          c('PER', 'PBR', 'PCR', 'PSR')]

data_value = data_value %>%
  mutate_all(list(~na_if(., Inf)))

rownames(data_value) = KOR_ticker[, '종목코드']
print(head(data_value))
```

```
##          PER   PBR      PCR     PSR
## 005930 13.87 1.134    6.571  1.2942
## 000660 29.83 1.253    9.264  2.2252
## 207940    NA 6.936 2961.208 43.0506
## 035420 46.76 4.193   20.097  4.1356
## 051910 76.70 1.383    7.701  0.8397
## 068270 83.59 8.312   57.271 22.2819
```



```r
write.csv(data_value, 'data/KOR_value.csv')
```

1. 열 이름이 가치지표에 해당하는 부분만 선택합니다.
2. 일부 종목은 재무 데이터가 0으로 표기되어 가치지표가 Inf로 계산되는 경우가 있습니다. `mutate_all()` 내에 `na_if()` 함수를 이용해 Inf 데이터를 NA로 변경합니다.
3. 행 이름을 티커들로 변경합니다.
4. data 폴더 내에 KOR_value.csv 파일로 저장합니다.
