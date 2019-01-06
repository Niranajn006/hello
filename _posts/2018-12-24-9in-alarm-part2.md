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

```pip``` 명령어를 통해서 이번 프로젝트에 필요한 라이브러리(```requests```, ```bs4```, ```pandas```, ```lxml```)들을 파이썬 환경에 설치해주자. 이제 아래 코드를 실행해보면 정상적으로 아라 게시판의 Html이 텍스트로 출력되는 것을 볼 수 있다. ```request``` 모듈의 ```get``` 메소드를 통해 구인 게시판의 url로 접속한 후 ```text``` 어트리뷰트를 출력하는 코드다.  
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

서비스 상에서 1분에 한번씩 저렇게 게시글 리스트를 크롤링 할건데 크롤링한 결과의 ```id``` 집합에서 이전 결과의 ```id``` 집합을 빼면(차집합을 구하면) 새로 등록된 글들의 알아낼 수 있다. 최신 Dataframe을 df_new, 그 전에 마지막으로 크롤링된 Dataframe을 df_prev라 하면 신규 게시글들의 ```id```는 다음과 같이 계산할 수 있다.  

{% gist bb614ccca6be7475c4556e245b320702 new_ids.py %}  

실제 서비스를 구현할 때는 df_prev를 이전 호출에서 미리 ```S3``` 버킷에 저장해놓은 뒤 다시 로딩해올 필요가 있으나 이것은 차후에 다루도록 하자.   


#### 게시글 미리보기  

이제 신규 글들의 ```id```를 알아냈다는 가정하에 각 글들의 미리보기 내용을 만들어보자. ```id=568394```인 게시글의 url은 https://ara.kaist.ac.kr/board/Wanted/568394/?page_no=1 이므로 ```id``` 값이 저 중간에 들어감을 알 수 있다. 이제 로그인을 하지 않은 상태에서 위 주소로 바로 접속을 시도하면..  

<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_fail.PNG" width="800"> <br>   
<font size="2.5">보안이 허술하지 않다..!?</font>  
</p>  

결국 어떻게든 자동으로 로그인을 해야 되는 상황에 처했다. 기존에 내가 쓰던 방식은 ```Selenium```을 사용해서 가상 웹드라이버를 만드는 것이었지만 느리고, 너무 많은 리소스를 잡아먹으며, Lambda 패키지로 만들기도 힘들기에 배제하고 ```requests``` 모듈의 ```Session``` 기능을 이용해보기로 했다. 이를 이용하면 한번 로그인에 성공하면 로그인 성공 정보가 세션에 남기 때문에 자유롭게 게시글들을 열람할 수 있다. ```Session```으로 로그인을 자동화하는 법을 찾다보니 [이 글](https://pybit.es/requests-session.html)과 [요 글](https://stackoverflow.com/questions/38021429/python-requests-login-with-redirection) 정도가 괜찮아보였다. 

아라 사이트가 어떤 식으로 로그인을 처리하는지 알아보기 위해서 크롬 개발자도구로 소스코드를 뒤적여보자.  

<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_all.png" height="500"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_code.PNG" height="500"><br>   
<font size="2.5">로그인 UI(위). Html 소스코드 상에서의 로그인 UI 코드(아래)</font>  
</p>  

위 사진 속 코드를 보니 form-action의 값이 ```/account/login``` 으로 되어있기 때문에 로그인 버튼을 누르면 ```https://ara.kaist.ac.kr/account/login/```로 리디렉션 될 것임을 예상해볼 수 있다. 그럼 어떤 파라미터들을 로그인 API의 인풋으로 받는지 알기 위해서 메인 페이지에서(꼭 메인 페이지일 필요는 없음) 개발자 도구의 네트워크 탭을 키고 아이디와 비번을 친 다음 로그인 버튼을 눌러본다.  

<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_before.PNG"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_success.PNG"><br>   
<font size="2.5">네트워크 탭을 띄워둔 로그인 하기 전 메인 페이지 화면(위). 성공적으로 로그인 한 후 나타난 네트워크 탭 화면(아래)</font>   
</p>  

사진처럼 네트워크 탭에서 거쳐간 페이지와 리소스 리스트를 보면 맨 처음에 ```/login```이라고 표시된 링크를 거쳐 가는 것을 확인할 수 있다. 실제로 마우스를 올려보면 ```https://ara.kaist.ac.kr/account/login/```을 향하고 있음을 확인할 수 있다. 다음으로 빨간색 동그라미 친 항목을 클릭해서 자세한 사항을 열람해보자.  

<p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/login_form.PNG"><br>   
<font size="2.5">로그인 API의 Form Data 항목들</font>   
</p>  

```username``` 이라는 항목에는 내가 방금 친 아이디가, ```password``` 항목에는 비밀번호가 적혀 있다. 고로 아라 사이트는 로그인 할 때 간단하게 저 두 값만 인풋으로 주면 된다는 것을 알 수 있다. 이제 파이썬 상에서 아이디와 비번을 인풋으로 서버에 넘겨주면서 로그인된 세션을 열어보자.  

이번에도 ```requests``` 모듈을 활용해서 로그인 세션을 열어보고 싶어서 구글에 ```python requests login session``` 이런식으로 검색했더니 [이 글](https://stackoverflow.com/questions/12737740/python-requests-and-persistent-sessions)이 유용했다. 아래 코드에서 내 아이디와 비번을 딕셔너리로 만든 후 로그인 링크로 접속하면서 페이로드로 넘겨주었다. 이때 ```headers``` 는 내 요청이 자동화된 스크립트에 의한 요청이 아니라 실제 유저의 브라우저에서 발생한 요청처럼 보이게 해주는 역할을 한다. 로그인된 세션을 만든 후에는 다시 원하는 게시글에 접속한 후 html에서 ```article``` 클래스로 파싱하여 게시글의 본문에 해당하는 텍스트만 추렸다.   

 {% gist bb614ccca6be7475c4556e245b320702 get_content.py %}  
 
 아니 그런데 이게 웬걸 파싱된 텍스트가 원문 내용과 다른 것이다. 
 
 <p align="center" style="color:#808080"> 
<img src="https://heartcored98.github.io/post_src/9in-alarm/wrong1.PNG"> <br> <img src="https://heartcored98.github.io/post_src/9in-alarm/wrong2.PNG"> <br>   
<font size="2.5">테스트해본 원문 글(위) 주피터 노트북에서 파싱 테스트한 결과(아래) 파싱의 상태가...?!?</font>  
</p>
 
 



