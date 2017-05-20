# R을 이용한 시각화 2 - ggmap을 이용한 도시별 현재 기온 지도 그리기
ISSAC LEE  
5/17/2017  



이 포스팅은 U of Iowa의 Luke Tierney 교수님의 Visualization [강의 노트](http://homepage.divms.uiowa.edu/~luke/classes/STAT4580/weather.html#temperatures-and-locations-for-some-iowa-cities)와 데이터 과학 [지리정보 시각화](http://statkclee.github.io/data-science/geo-info.html) 사이트에서 영감을 받아 작성하였습니다.

  - 이번 포스팅의 목표: 대한민국의 각 도시별 현재 기온 지도 그리기
  
### 사용된 R 팩키지들


```r
library(tidyverse)
library(magrittr)
library(xml2)
library(jsonlite)
```




### ggmap을 이용한 지도 불러오기

`ggmap` 팩키지는 Rstudio의 Hadley Wickham과 David Kahle이 만든 지도 정보를 손쉽게 불러올 수 있도록 하는 팩키지이다. 이 팩키지는 여러 지도 소스에서 제공하는 API를 하나의 함수, `get_map()`,으로 접근할 수 있게 해주는게 특징이다. (하지만 필자가 테스트 해 본 결과 아직까지는 불안정한 모습을 보이지만 구글맵은 비교적 안정적으로 돌아가는 편이다.) 지원되는 지도 사이트에는 Google Map, Openstreet Map, Stamen Map, 그리고 뜻 밖의(?) Naver Map이 들어가 있다.

[이전 포스팅](https://github.com/issactoast/RforDataSciencePractice/blob/master/md/map1.md)에서 정의했던 대한민국의 지리적 최극단 좌표를 가지고 구글맵을 불러와보자. 아래에서 사용한 `get_map`함수에 대한 내용은 [설명서](https://cran.r-project.org/web/packages/ggmap/ggmap.pdf)를 참조하자.


```r
library(ggmap)

southKrLocation <- c(125.04, 33.06, 131.52, 38.27)
krMap <- get_map(location=southKrLocation,
                 source = "google",
                 maptype = "watercolor",
                 crop = FALSE) #roadmap, terrain, toner, watercolor
mymap <- ggmap(krMap, extent = 'device') # + theme_map() 
mymap
```

![](map2_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

`get_map`함수의 maptype 옵션을 "watercolor"로 설정하면 위와 같이 아주 예쁜 지도가 나타난다. 이렇게 준비한 지도 위에 우리가 이전 시간에 공부했던 웹상의 정보를 끌어와 지도위에 나타내어보자.

사용된 `findTempbyBox()`는 [이전 포스팅](https://github.com/issactoast/RforDataSciencePractice/blob/master/md/map1.md)에서 다루었으므로 자세한 설명은 생략하겠다.


```r
myCities <- findTempbyBox(southKrLocation, 7)
layerCities <- findTempbyBox(southKrLocation, 15)
dim(myCities); dim(layerCities);
```

```
## [1] 7 4
```

```
## [1] 172   4
```

`findTempbyBox()` 함수의 두 번째 모수는 Zoom을 나타내는데 숫자가 높을수록 자세히 표현된다. 따라서 Zoom 7로 긁어온 `myCities`에는 상대적으로 적은 숫자의 도시들이 들어있고, Zoom 15에는 우리가 요구한 범위에 속한 OpenWeatherMap에서 제공하는 도시가 전부 들어있다. 이렇게 Zoom 15를 사용하여 만든 `layerCities`의 기온 정보는 기온 등고선을 그리는데에 사용할 것이다.

다음 단계는 `myCities` 안에 들어있는 도시들 중 다른 나라에 속한 도시들을 걸러내는 방법이다.


```r
country <- myCities %$% city %>% map_chr(findCountrybyCity)
myCities <- mutate(myCities, country = country) %>% select(country, everything())
myCities <- myCities %>% filter(country == "KR")
head(myCities)
```

```
## # A tibble: 6 × 5
##   country       city      lon      lat  temp
##     <chr>      <chr>    <dbl>    <dbl> <dbl>
## 1      KR       Jeju 126.5219 33.50972 19.33
## 2      KR    Gwangju 126.9156 35.15472 26.33
## 3      KR    Daejeon 127.4197 36.32139 26.00
## 4      KR      Busan 129.0403 35.10278 26.00
## 5      KR      Seoul 126.9778 37.56826 26.81
## 6      KR Kang-neung 128.8961 37.75556 27.00
```

## 보간법(Interpolation)과 크리깅(Kriging)을 사용한 등고선 그리기

앞서 확인한 바와 같이 `layerCities`안에는 172개의 도시의 위치와 기온 정보가 들어있다. 이 정보들을 이용하여 등고선을 그릴 수 있는데, 이번 포스팅에서는 보간법과 크리깅 두 가지 방법을 사용하여 그리는 방법을 알아보자.

### 보간법(Interpolation)을 이용한 등고선 그리기

![](https://upload.wikimedia.org/wikipedia/commons/4/41/Interpolation_example_polynomial.svg)

[보간법](https://en.wikipedia.org/wiki/Interpolation)의 특징은 위의 그림에서와 같이 우리가 가지고 있는 점들(주어진 정보)을 지나는 어떤 함수를 찾음으로서 우리가 추정하는 위치의 값(알고 싶은 정보)을 예측하는 방법이라고 생각하면 된다. 즉, 주어진 도시들의 기온 정보를 이용하여 나머지 주변 지역의 기온을 어떤 매끄러운 함수로 추정하는 것이라 생각하면 좋을 것이다. `R`에서 보간법을 이용하기 위해서는 `akima` 팩키지를 이용하면 된다. 


```r
library(akima)

# Interpolation
surface <- with(layerCities, interp(lon, lat, temp, linear = FALSE))
srfc <- expand.grid(lon = surface$x, lat = surface$y)
srfc$temp <- as.vector(surface$z)
tconts <- geom_contour(aes(x = lon, y = lat, z = temp),
                       data = srfc, color = "black", na.rm = TRUE)

mymap_interp <- mymap + tconts
mymap_interp
```

![](map2_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

### 크리깅(Kriging)을 이용한 등고선 그리기

크리깅([Kriging](https://en.wikipedia.org/wiki/Kriging))의 경우 지리통계학에서 많이 사용되는 개념인데, 이 방법 역시도 큰 개념하에서는 보간법에 속하지만 추정하고자 하는 지점의 예측값이 우리가 알고 있는 정보의 linear combination으로 생각하여 결정하는 방법이라고 생각하면 된다. 즉, `layerCities` 리스트 안에 들어있지 않는 위치(도시)의 기온을 추정하고자 할 때, 리스트에 있는 가장 가까운 도시의 기온에 가장 큰 가중치를 주고 멀리 떨어진 도시의 기온은 가중치를 적게 부여하는 방식으로 예측하는 것이다.

`R`에서 크리깅을 이용하기 위해서는 `kriging` 팩키지를 이용하자.


```r
library(kriging)

surface <- with(layerCities, kriging(lon, lat, temp))$map %>% rename(temp = pred)
tconts <- geom_contour(aes(x, y, z = temp),
                       data = surface, color = "black", na.rm = TRUE, alpha = 0.4)

mymap_kriging <- mymap + tconts
mymap_kriging
```

![](map2_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

두 가지 방법 모두 거의 동일한 등고선을 결과값으로 반환하는 것을 볼 수 있다. 마지막으로 `myCities` 안의 도시 이름과 기온 정보를 지도위에 표시하는 것으로 이번 포스팅을 마치도록 하겠다. 


```r
tpoints <- geom_text(aes(x = lon, y = lat, label = temp),
                     color = "red", data = myCities)
tcities <- geom_text(aes(x = lon, y = (lat + 0.3), label = city),
                     color = "black", data = myCities)
mymap_kriging + tpoints + tcities
```

![](map2_files/figure-html/unnamed-chunk-8-1.png)<!-- -->