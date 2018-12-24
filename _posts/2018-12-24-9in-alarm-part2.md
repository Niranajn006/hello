---
layout: post
title: 교내 커뮤니티 구인 알람 개발기 Part2 - 구인글 크롤링  
---

Part1에서 언급한대로 이번 글에서는 파이썬으로 게시판에 접속하여 글들을 모니터링하고 다뤄보자. 아래 내용을 포함한다.  
 - ```request``` 모듈을 활용해서 구인 게시판 내에 Html 소스코드를 따오고 ```BeautifulSoup```를 이용해서 ```pandas```의 Dataframe(한마디로 '표')으로 만든다.  
 - 게시글이 작성될 때 발급 받는 ```id```를 바탕으로 새로운 글들의 ```id```만 감지해서 추려낸다.   
 - 새 글들의 url로 접속하여 미리보기를 위한 글 내용을 따온다.  
 - 게시글 내용을 열람하기 위해서 자동 로그인을 구현한다.  

<br>  

#### 게시글 따오기  

아래 명령어를 통해서 이번 프로젝트에 필요한 라이브러리들을 파이썬 환경에 설치해주자. 
```
pip install requests, bs4, pandas, lxml
```

이제 아래 코드를 실행해보면 정상적으로 아라 게시판의 Html이 텍스트로 출력되는 것을 볼 수 있다. ```request``` 모듈의 ```get``` 메소드를 통해 구인 게시판의 url로 접속한 후 ```text``` 어트리뷰트를 출력하는 코드다.  
{% gist bb614ccca6be7475c4556e245b320702 get_html.py %}  

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">


<html xmlns="http://www.w3.org/1999/xhtml" lang="ko" xml:lang="ko">
    <head>
        <title>아라</title>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        <meta name="description" content="KAIST 학내 커뮤니티, 아라">
        
        <link rel="stylesheet" href="/media/CACHE/css/d8b7c9090976.css" type="text/css" media="screen">
<link rel="stylesheet" href="/media/CACHE/css/5057e10ccd3d.css" type="text/css">


        <script type="text/javascript" src="/media/CACHE/js/0b46084f93e7.js" charset="utf-8"></script>
    </head>

    <body onload="setWidth();" onresize="setWidth();">
        <ul id="topLinks">
            <li class="kaist"><a href="http://www.kaist.ac.kr"><strong>KAIST</strong></a></li>
            <li><a href="http://ara.kaist.ac.kr" title="웹아라">WebARA</a></li>
            <li><a href="http://lkin.kaist.ac.kr" title="수강지식인">LKIN</a></li>
```  
<p align="center" style="color:#808080"> 
<font size="2.5">너무 길어서 생략된 html임</font>  
</p>  

구글에 Html에서 Table을 손쉽게 추출하는 법을 몇개 검색한 결과 [여기](https://srome.github.io/Parsing-HTML-Tables-in-Python-with-BeautifulSoup-and-pandas/)의 방식이 제일 깔끔하면서도 강인해보였다. 그냥 쓰니까 몇가지 오류가 나서 어딘가를 수정했따. 최종 Html 파서인 ```HTMLTableParser```는 [여기](https://gist.github.com/heartcored98/bb614ccca6be7475c4556e245b320702#file-parser-py)에 올려두었으며 어디를 어떻게 고친건지는 3개월 전 일이므로..      
<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/simple.jpg" width="300"> <br>   
<font size="2.5">...</font>  
</p>  

아래 코드를 통해 최종적으로 Dataframe을 얻을 수 있었고 N, 작성자, 말머리 등등의 불필요한 열은 삭제해주자. 게시글의 ```id```는 인덱스로 설정해주었으며 이 인덱스는 나중에 새로운 글을 찾는데 사용될 것이다.  
{% gist bb614ccca6be7475c4556e245b320702 get_df.py %}  
<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/df.PNG" width="800"> <br>   
<font size="2.5">아주 이쁘게 파싱됐다</font>  
</p>  

서비스 상에서 1분에 한번씩 저렇게 게시글 리스트를 크롤링 할건데 크롤링한 결과의 ```id``` 집합에서 이전 결과의 ```id``` 집합을 빼면(차집합)을 구하면 새로 등록된 글들의 알아낼 수 있다. 최신 Dataframe을 df_new, 그 전에 마지막으로 크롤링된 Dataframe을 df_prev라 하면 신규 게시글들의 ```id```는 다음과 같이 계산할 수 있다.  

{% gist bb614ccca6be7475c4556e245b320702 new_ids.py %}  

실제 서비스를 구현할 때는 df_prev를 이전 호출에서 미리 ```S3``` 저장해놓은 뒤 다시 로딩해올 필요가 있으나 이것은 차후에 다루도록 하자.   

#### 게시글 따오기  

#### 게시글 미리보기  

이제 신규 글들의 ```id```를 알아냈다는 가정하에 각 글들의 미리보기 내용을 만들어보자. ```id=568394```인 게시글의 url은 https://ara.kaist.ac.kr/board/Wanted/568394/?page_no=1 이므로 ```id``` 값이 저 중간에 들어감을 알 수 있다. 이제 로그인을 하지 않은 상태에서 위 주소로 바로 접속을 시도하면..  

<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_fail.PNG" width="800"> <br>   
<font size="2.5">보안이 허술하지 않다..!?</font>  
</p>  

결국 어떻게든 자동으로 로그인을 해야 되는 상황에 처했다. 기존에 내가 쓰던 방식은 ```Selenium```을 사용해서 가상 웹드라이버를 만드는 것이었지만 느리고, 너무 많은 리소스를 잡아먹으며, Lambda 패키지로 만들기도 힘들기에 배제하고 ```requests``` 모듈의 ```Session``` 기능을 이용해보기로 했다. 이를 이용하면 한번 로그인에 성공하면 로그인 성공 정보가 세션에 남기 때문에 자유롭게 게시글들을 열람할 수 있다. ```Session```으로 로그인을 자동화하는 법을 찾다보니 [이 글](https://pybit.es/requests-session.html)과 [요 글](https://stackoverflow.com/questions/38021429/python-requests-login-with-redirection) 정도가 괜찮아보였다. 

 




<br>

#### 서버는 어디에?   
 
저번 버전에서는 한달에 약 3.5달러 밖에 과금이 되지 않는 AWS의 [LightSail](https://aws.amazon.com/ko/lightsail/) 서버를 한 대 빌려서 사용했었는데, 돈 벌자고 만드는건데 월 지출이 발생하는 것도 그렇고 요즘 대세인 PasS를 시도해보고 싶어서 이번엔 AWS의 [Lambda](https://aws.amazon.com/ko/lambda/) 함수 형태로 만들어서 1분에 한번씩 호출하기로 했다.  

Lambda 서비스는 한달에 약 1백만회까지 요청이 무료라서(고마워요 AWS!) 호출 주기가 1분이면 월 약 43,000회 정도의 호출이 발생하므로 돈을 한 푼도 안 쓰고 서비스가 가능하다.       

    
<br>
#### 구현은 어떻게?  
앞서 언급한 다섯 가지의 서비스 구성 요소들을 어떻게 구현할지 하나씩 생각해보았다.  

1번의 경우 해당 링크가 퍼블릭이기 때문에 파이썬의 ```request``` 모듈 등을 활용해서 html을 긁어오고 ```beautifulsoup```을 이용해서 Table 파싱을 하면 될 것 같다.  

2번의 경우는 일반적인 서버라면 마지막 게시글 리스트를 임시 파일로 저장하거나 메모리에 계속 저장해놓고 비교할 수 있지만 Lambda의 경우 매 호출 시 초기화 되기 때문에 내부 변수를 선언하더라도 사용할 수 없게 된다. 이에 대한 대안 몇가지를 적어봤다.  

- [S3](https://aws.amazon.com/ko/s3/)에 게시글 상태를 ```.txt```나 ```.csv``` 형태로 매 호출이 끝날 때 저장하고 호출이 시작될 때 다시 로딩해온다.  
- [ElastiCache](https://aws.amazon.com/ko/elasticache/)를 활용해서 캐시 서버를 만든다. (너무 비싸고 높은 응답속도도 필요 없는 상황이므로 배제한다.)
- [DynamoDB](https://aws.amazon.com/ko/dynamodb/)나 [RDS](https://aws.amazon.com/ko/rds/?nc2=h_m1) 등의 서비스를 이용해서 게시글들에 대한 DB를 구축한다.  

이번 프로젝트는 최소 비용, 최대 성과가 목표이므로 가볍게 S3 버켓에 게시글들에 대한 ```pandas``` 데이터프레임을 ```.csv``` 형태로 저장하기로 했다.  

3번의 경우는 글의 내용을 보려면 사이트에 로그인을 해야 하는데 이를 자동화시키기 위해서는 2가지 방법이 있는데 후자를 먼저 시도해보고 정 안되면 전자의 방식을 택하기로 했다.  

- ```Selenium```을 사용해서 가상 웹 드라이버를 만들고 로그인을 한 후에 게시글을 열람한다.  
    - 장점: 가장 쉽고 확실하게 작동함
    - 단점: ChromeDriver를 사용해야 하며 Lambda에 패키지로 만들어서 올리기가 힘들며 쓸데 없이 리소스를 많이 먹음
- ```Request``` 모듈의 ```Session```을 활용해서 로그인 상태를 유지한다.  
    - 장점: 적은 리소스와 패키지만으로도 구현 가능
    - 단점: 페이지 내에서 로그인 기작이 어떤 식으로 일어나는지 네트워크를 분석해볼 필요가 있다.  
    
4,5번의 경우는 저번 버전에서도 사용했던 텔레그램 파이썬 모듈인 ```telegram```을 사용하기로 했다.

올해를 넘기면 귀차니즘이 너무 세져버리므로 3~4일 내에 조져보도록 한다.  



<p align="center" style="color:#808080"><b>·&nbsp;&nbsp;&nbsp;&nbsp;·&nbsp;&nbsp;&nbsp;&nbsp;·</b><br></p>  

<p align="right" style="color:#808080"> 
<font size="2.5">이 글은 Part2에서 이어집니다.</font>  
</p>

