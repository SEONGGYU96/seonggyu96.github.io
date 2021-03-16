---
layout: post
title: "안드로이드 프로세스 재시작 후 발생하는 데이터 손실"
subtitle: "상태 보존의 중요성"
author: "GuGyu"
header-style: text
tags:
  - Android
  - NPE
  - SavedInstanceState
  - ViewModel
  - SharedPreference
---

[액티비티 생명주기](https://seonggyu96.github.io/2021/03/02/android_activity/) 에서도 다뤘지만, 안드로이드 OS는 메모리 부족 등의 이유로 백그라운드에 있는 앱 프로세스를 종료시키는 경우가 있다. 사용자가 다시 앱을 실행시키면 소멸 직전 최상단에 있던 액티비티/프래그먼트를 재생성해서 보여주는데, 이때 런타임에 생성된 데이터나 객체 인스턴스가 손실되어 다양한 문제를 일으킬 수 있다.

## UI 데이터 손실  

사용자가 마지막으로 보고 있던 화면이 회원가입 폼이었다고 하자. 이메일과 비밀번호, 자동 가입 방지 문자 등을 모두 입력한 후 가입 버튼을 누르기 전에 전화가 걸려와 앱 프로세스가 백그라운드로 이동했다. 정상적인 시스템 상태라면 전화를 받는다고해서 백그라운드 프로세스가 종료되는 일은 없지만 가능성이 0%는 아니다.  
*(컴포넌트의 실행 여부 등 프로세스의 우선 순위에 영향을 끼치는 요소가 많다.)*  

사용자가 전화를 끊고 회원가입을 마무리 하려고 앱을 다시 실행했으나 안타깝게도 앱 프로세스는 OS에 의해 종료된 상태였고, OS는 최소한의 양심으로 마지막 액티비티/프래그먼트를 `onCreate()`부터 다시 시작시켜주었다. 하지만 사용자는 기존에 입력했던 정보를 다시 입력해야했다.  

앱 프로세스가 재실행되면 발생하는 문제 중 가장 눈에 띄는 것은 위와 같이 UI에 입력한 데이터의 손실이다. 사용자가 입력한 데이터는 런타임 중에 생성되었기 때문에 프로세스가 종료되면 함께 사라지는 것은 당연하다.  

다행히 안드로이드 프레임워크에서는 이를 해결하기 위한 방법을 제공하고 있다.  

### SavedInstanceState  

Activity와 Fragment 모두 `onSaveInstanceState()` 와 `onRestoreInstanceState()` 콜백을 제공한다.

```kotlin
override fun onSaveInstanceState(outState: Bundle?) {
    // Save the user's current game state
    outState?.run {
        putInt(STATE_SCORE, currentScore)
        putInt(STATE_LEVEL, currentLevel)
    }

    // Always call the superclass so it can save the view hierarchy state
    super.onSaveInstanceState(outState)
}

companion object {
    val STATE_SCORE = "playerScore"
    val STATE_LEVEL = "playerLevel"
}
```  
위 처럼 `onSaveInstanceState()` 콜백을 오버라이딩해서 액티비티/프래그먼트가 재생성되더라도 복구하고싶은 **간단한** 데이터를 `Bundle`객체에 저장할 수 있다. `Bundle` 객체에 저장하기 때문에 직렬화/역직렬화가 수행되어야하기 때문에 간단하지 않은 데이터라면 권장하지 않는다.

예제 코드는 공식 문서에서 발췌했으며 현재 게임 스코어와 레벨을 저장하고 있다.  

```kotlin
override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    // Always call the superclass so it can restore the view hierarchy
    super.onRestoreInstanceState(savedInstanceState)

    // Restore state members from saved instance
    savedInstanceState.run {
        currentScore = getInt(STATE_SCORE)
        currentLevel = getInt(STATE_LEVEL)
    }
}
```  
복원할 상태가 있다면, 액티비티/프래그먼트가 재실행될 때 `onRestoreInstanceState()`콜백이 호출된다. 따라서 해당 콜백을 오버라이딩해서 저장한 값들을 다시 UI에 반영할 수 있다. 파라미터로 전달된 `Bundle` 객체는 `onSaveInstanceState()`에서 값을 저장한 그 `Bundle` 객체이다.  

복원할 상태가 없다면 콜백 자체가 실행되지 않는다. 따라서 자바의 경우 굳이 `Bundle`에 대한 null check를 하지 않아도 된다.
 
그런데 파라미터로 전달된 `savedInstanceState: Bundle?`이 조금 익숙하지않은가? 그렇다. `onCreate()` 콜백에서도 동일한 파라미터가 존재한다.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState) // Always call the superclass first

    // Check whether we're recreating a previously destroyed instance
    if (savedInstanceState != null) {
        with(savedInstanceState) {
            // Restore value of members from saved state
            currentScore = getInt(STATE_SCORE)
            currentLevel = getInt(STATE_LEVEL)
        }
    } else {
        // Probably initialize members with default values for a new instance
    }
    // ...
}
```  

`onCreate()`는 인스턴스 생성 시 반드시 호출되기 때문에 전달되는 `Bundle`은 nullable 하다. 그러나 복원할 값이 있다면 `onRestoreInstanceState()`와 동일한 `Bundle`이다. 차이점이 있다면 실행 시점이다. `onCreate()`는 액티비티/프래그먼트 생명주기 중 가장 처음에 실행되지만 `onRestoreInstanceState()`는 `onStart()` 이후에 실행된다. 이를 인지하여 상황에 맞게 골라 사용하면 된다.  

<img width="400" src="https://user-images.githubusercontent.com/57310034/111283715-c2fa8300-8682-11eb-9e3f-cc9739c4ee4f.png"/>

참고로 `onSaveInstanceState()`는 대칭적으로 `onStop()` 이후에 호출된다.  

<img width="400" src="https://user-images.githubusercontent.com/57310034/111283926-f3dab800-8682-11eb-97c4-84a1ff230e87.png"/>  

SavedInstanceState를 이용한 데이터 저장/복구는 구성 변경으로 인해 액티비티/프래그먼트가 재생성될 때에도 활용이 가능하다.  

### intent 복구  

참고로 액티비티를 처음 실행할 때 `intent`에 어떤 값들을 담아 전달했다면, 이 데이터들도 완전히 복구된다.  

<br>

## ViewModel과 싱글턴 인스턴스의 손실  

사실 UI 데이터에 대한 손실에 대한 걱정은 비교적 사소한 데이터에 속하기 때문에 어플리케이션 자체에 심각한 문제를 일으키는 경우는 잘 없다. 하지만 ViewModel이나 싱글턴으로 생성한 앱 전역 인스턴스들도 재성성되는 것은 큰 문제를 야기할 수 있다.  

ViewModel은 그 목적상 비교적 "복잡한 데이터"를 많이 다룬다. 여기서 "복잡한 데이터"라 함은 단순히 UI에 표시할 문자열이나 정수 값들이 아니라 객체 단위의 데이터들이다. 공식 문서에서도 구성 변경으로 인한 액티비티 재실행 시 데이터 복구를 위해 간단한 데이터를 제외하고는 ViewModel 사용을 권장하고 있다. 그러나 런타임 중에 사용자와의 상호작용으로 인해 변경된 값들을 가지고 있는 ViewModel 또한 재생성되기 때문에 사용자가 같은 데이터를 생성하기 어려운 데이터라면 다른 방법으로 데이터를 저장하고 있어야한다.  

그리고 가장 심각한 문제를 일으키는 것이 바로 싱글턴 인스턴스들이다. DB인스턴스나 SharedPreference 등 앱 전역에서 접근하고 데이터를 공유할 수 있는 객체는 싱글턴으로 만들어두는 경우가 많다. 그러나 많은 개발자들이 다른 컴포넌트/모듈에서 변경한 데이터 값이 유지되어 있을 것이라 확신하고 별다른 안전장치 없이 데이터를 사용하는 실수를 한다. DB나 SharedPreference는 최종 커밋까지 마친 데이터라면 문제가 없지만 성능 향상을 위해 임시 데이터를 갖거나 쓰기 작업을 미루는 경우도 있기 때문이다.  

내가 한 실수 중 하나는, 유저가 최초 로그인 후 발급받은 accessToken을 싱글턴 객체에 담아 사용한 일이었다. 한 번 발급받은 후에는 앱을 종료할 때 까지 쭉 사용할 데이터라고 생각했기 때문에 싱글턴 객체에 담아 아무런 안전 장치 없이 다양한 뷰모델에서 사용했고, 결국 앱을 백그라운드에 보냈다가 한참 뒤에 실행하면 비정상 종료를 일으켰다.  

이렇게 앱 전반에 걸쳐 중요한 역할을 하는 데이터는 아무리 임시 데이터라하더라도 `SharedPreference`에 저장해두는 것을 권장한다.  

<br>

## 앱 프로세스 종료 시나리오 테스트  

그렇다면 위에 언급한 위험 요소들에 대한 대응 코드를 모두 작성하고 테스트할 때는 어떻게 환경을 구성할 수 있을까? 무작정 앱을 백그라운드로 보내 kill 당하기를 기다릴 수는 없는 노릇이다. 이때는 adb 명령어를 사용해서 프로세스를 강제로 kill할 수 있다.  

1. 테스트하고자 하는 액티비티/프래그먼트를 최상단에 띄운다.
2. 앱을 백그라운드로 보낸다. (홈 버튼 등)
3. `$adb shell am kill <package>` 명령어를 실행한다. (예: `adb shell am kill com.example.test`)
4. 앱을 다시 실행한다.  

그럼 앱을 처음 실행하는 것 처럼 약간 딜레이가 발생하지만 마지막으로 머물러있던 액티비티/프래그먼트가 띄워지고 백스택, `intent`, `SavedInstanceState` 등이 복구된다. 나머지 데이터들은 자신이 의도한대로 복구가 되었는지 확인해보자.  



<br>
<br>


--- 
참고:  
[안드로이드 앱 강제 종료 재현하기](https://dhna.tistory.com/411)  
[프로세스 및 애플리케이션 수명 주기](https://developer.android.com/guide/components/activities/process-lifecycle)
