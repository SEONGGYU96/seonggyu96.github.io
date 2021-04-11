---
layout: post
title: "안드로이드 앱, 다국어 지원하기"
subtitle: '영어'
author: "GuGyu"
header-style: text
tags:
  - Android
  - WASK
  - Project
  - naccoro
  - English
  - Translate
---
---
<br>

WASK를 미국 및 영어권 국가에 출시하기 위해 영어 번역을 시도하였다.  
미국은 중국과는 다르게 구글 플레이스토어를 사용할 수 있기 때문에 별다른 출시 이슈는 없을 것으로 예상된다.  
그러니 마음 편하게 디바이스 설정에 따라 앱에 사용될 언어를 바꿔주도록 설정해보자.
<br>

# 미리보기
## 한국어 설정 시
<p float="left">
<img width="300" src="https://user-images.githubusercontent.com/57310034/101230123-e5c1e680-36e6-11eb-8b1e-7cb540619083.png"/>
<img width="300" src="https://user-images.githubusercontent.com/57310034/101230165-19047580-36e7-11eb-8bdd-0a37cf4a38b2.png"/>  
</p>
<br>

## 영어 설정 시
<p float="left">
<img width="300" src="https://user-images.githubusercontent.com/57310034/101230193-42bd9c80-36e7-11eb-8ecd-d56d39892b07.png"/>
<img width="300" src="https://user-images.githubusercontent.com/57310034/101230740-69c99d80-36ea-11eb-9374-82a83aa16b02.png"/>  
</p>
<br>

# 언어 현지화

## strings.xml 추가
<img width="600" float="left" src="https://user-images.githubusercontent.com/57310034/101230835-27549080-36eb-11eb-91be-72eb2f1d810f.png"/>  

위와 같이 strings 리소스 폴더에서 `Values Resource File` 을 새로 생성한다.  
<br>

<img width="600" float="left" src="https://user-images.githubusercontent.com/57310034/101231395-4d2f6480-36ee-11eb-953e-a1ba8bcb97f8.png"/>   

`File name` 은 "strings.xml"로 동일하게 해준다. 이 상태로 OK를 누르면 당연히 에러가 발생한다. 이미 동일한 이름의 리소스 파일이 존재하기 때문이다.  
하지만 `Locale` 이라는 `Qualifier` 를 추가해줌으로서 또 하나의 식별자를 부여해 생성할 수 있다.  
`Qualifier` 는 공식 문서에 따르면 `개별 구성` 이라는 뜻의 `한정자`로 번역되고 있다.  
국가나 기본 설정 언어 뿐만 아니라 사용하는 기기 등 다양한 앱 실행 환경에 따라 **개별적으로** 적용할 수 있는 리소스를 식별하는 것이다.  
우리는 디바이스 기본 언어에 따라 해당 리소스를 참조하도록 하고싶기 때문에 `Locale` 한정자를 추가한다.  
<br>

<img width="600" float="left" src="https://user-images.githubusercontent.com/57310034/101231787-e8c1d480-36f0-11eb-8883-d20a401fd263.png"/> 

그리고 매칭될 언어와 국가를 선택한 후에 OK 버튼을 눌러 리소스를 생성한다.  
<br>

<img width="300" float="left" src="https://user-images.githubusercontent.com/57310034/101231732-8bc61e80-36f0-11eb-9a03-b5272fd11dcb.png"/>  

미국 국기가 적용된 리소스가 추가되었다 !  
<br>

## 번역 및 확인

<img width="600" float="left" src="https://user-images.githubusercontent.com/57310034/101231875-62f25900-36f1-11eb-96a6-320d5dc6aac3.png"/>  

해당 파일에 기존 파일을 복붙한 후, 영어로 번역해주었다.  
(검수 받기 전 파파고 + 짧은 영어 끈으로 번역한 것이므로 어색한 표현은 무시하면 된다...)  
<br>

<img width="300" float="left" src="https://user-images.githubusercontent.com/57310034/101231945-f035ad80-36f1-11eb-8173-652ec81ea1b7.png"/>  

그럼 이제 언어 설정에 따라서 코드 내에서 `@string/~` 를 통해서 참조하는 리소스 파일이 자동으로 변경된다.  
즉, 여기서는 `setting_replacement_cycle` 이라는 이름을 가진 문자열 값을 얻기 위해서 기본적으로는 일반 `strings.xml` 파일을 열어 해당 이름을 찾아 문자열 값을 참조하지만, 영어로 설정된 상태에서는 `en-rUS` 한정자가 적용된 `strings.xml` 파일을 열어 해당 이름의 문자열을 참조하게 되는 것이다.
<br>

# 그 외 현지화

저렇게만 해서 현지화가 끝나면 정말 좋겠으나 다른 문제점도 존재한다.

## 날짜 표기법

<img width="300" float="left" src="https://user-images.githubusercontent.com/57310034/101236767-102b9800-3717-11eb-96fd-c014d4901723.png">  

WASK는 마스크 교체 기록을 확인할 수 있는 달력뷰가 있다.  
그리고 해당 달력뷰의 상단에는 지금 보여지고 있는 달력이 몇 년도 몇 월을 나타내는지 `YYYY년 MM월` 형식으로 보여주고 있다.  
하지만 미국에서는 연도와 월만 표기하는 경우에 `November 2020` 혹은 `Nov 2020` 과 같이 표기한다고 한다.  
이와 같은 부분들은 단순히 대응되는 문자열 리소스를 변경한다고 해결되는 문제가 아니기 때문에 코드 상에서 언어에 따라 구현을 달리 해야한다.  
<br>

### 설정된 언어에 따라 분기하기

```java
private void setLocaleDateType() {
    //앱이 실행되고 있는 디바이스의 local 설정 값을 가져온다. (국가)
    Locale locale = getResources().getConfiguration().locale;
    //해당 국가의 언어를 가져온다.
    String language = locale.getLanguage();
    if (language.equals("en")) { //영어로 설정되어있는 경우
        setEnglishDateType();
    } else { //그 외
        setKoreanDateType(); 
    }
}
```
따라서 기존 구현부를 커스텀뷰로 묶어둔 다음, 생성자에서 위와 같은 조건문을 삽입하여주었다.  
그럼 날짜를 보여주는 뷰가 생성될 때 설정된 언어에 맞게 어순이 정렬된 상태로 생성이 되고, 기존에 사용하던 비즈니스 로직에서는 원래 하던데로 `Date` 값을 넣어주기만 하면 된다.  
<br>

<img width="300" float="left" src="https://user-images.githubusercontent.com/57310034/101237154-d0b27b00-3719-11eb-8ae7-7788d4134ec0.png"/>  

영어 버전에서 날짜 형식 보여주기 성공 !

---
그 외에도 다이얼로그의 날짜 어순, `day | days` 문제 등 다양한 수정이 필요한데, 모두 위 분기문으로 처리하면 된다.  
나는 이를 커스텀뷰로 해결하였는데 이는 다음 링크에서 볼 수 있다.  
[안드로이드 현지화를 위한 커스텀 텍스트뷰 만들기](https://seonggyu96.github.io/2020/12/06/WASK_custom_view/)
<br>

끝