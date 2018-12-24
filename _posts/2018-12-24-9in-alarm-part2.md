---
layout: post
title: 교내 커뮤니티 구인 알람 개발기 Part2 - 구인글 크롤링  
---

Part1에서 언급한대로 이번 글에서는 파이썬으로 게시판에 접속하여 글들을 모니터링하고 다뤄보자. 아래 내용을 포함한다.  
 - ```request``` 모듈을 활용해서 구인 게시판 내에 Html 소스코드를 따오고 ```BeautifulSoup```를 이용해서 ```pandas```의 Dataframe(한마디로 '표')으로 만든다.  
 - 게시글이 작성될 때 발급 받는 ```id```를 바탕으로 새로운 글들의 ```id```만 감지해서 추려낸다.   
 - 새 글들의 url로 접속하여 미리보기를 위한 글 내용을 따온다.  
 - 게시글 내용을 열람하기 위해서 자동 로그인을 구현한다.  
 
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
            <li><a href="http://otl.kaist.ac.kr" title="온라인 타임테이블 OTL">OTL</a></li>
            <li><a href="http://ftp.kaist.ac.kr" title="오픈소스 미러링 서비스 FTPKAIST 입니다.">FTPKAIST</a></li>
            <li><a href="#" id="moreLinks">more...</a></li>
        </ul>
        <div id="navigation">
            <h1><a href="/main/">아라</a></h1>
            <ul class="menu">
                <li><a href="/all/" id="menuFavorite" rel="Favorite">모아보기</a></li>
                <li><a href="#" class="category" id="menuCategory1" rel="Category1">카테고리명</a></li>
                <li><a href="#" class="category" id="menuCategory2" rel="Category2">카테고리명</a></li>
                <li><a href="#" class="category" id="menuCategory3" rel="Category3">카테고리명</a></li>
                <li><a href="#" class="category hidden" id="menuCategory4" rel="Category4">카테고리명</a></li>
                <li><a href="#" class="category" id="menuCategory5" rel="Category5">카테고리명</a></li>
                <li><a href="#" class="category" id="menuCategory6" rel="Category6">카테고리명</a></li>
            </ul>
```  

<p align="center" style="color:#808080"> 
<font size="2.5">대략 위에 처럼 나온다 (원본 html은 너무 길어서 생략된 결과다)</font>  
</p>  

## 왜 다시 만드는가?  

**돈**이 없어졌기 때문이다. ~~(자낳괴 공돌이 씁흑흑)~~  방학 때 다시 일을 하기 위해선 이런 자동화된 알람으로 무장할 필요가 있었다. 둘째로는 3개월 전에 일주일 후에 돌아오겠다는 말을 하고 사라진 것에 대한 **책임**을 지기 위해서이다.    

## 프로젝트 디자인  

#### 서비스 구성은?  

기획 중인 서비스 플로우차트는 다음과 같다.  
1. 교내 커뮤니티 중 구인란인 https://ara.kaist.ac.kr/board/Wanted 에 접속 후 게시글 목록을 따온다.  
2. 마지막으로 업데이트 된 게시글 목록과 비교하여 새로운 글을 추린다.  
3. 추려진 글들을 하나씩 접속해서 미리보기 텍스트를 추출한다.  
4. 알람으로 보낼 게시글의 제목과 미리보기 내용을 텔레그램 푸셔에 전송한다.  
5. 텔레그램 푸셔는 등록해놓은 구인 채널로 푸쉬 알림을 보내면 끝!  


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

