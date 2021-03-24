---
layout: post
title: "안드로이드 : 스타티드 서비스"
subtitle: "백그라운드 컴포넌트"
author: "GuGyu"
header-style: text
tags:
  - Android
  - AndroidFramework
  - Service
  - Background
---

Service는 백그라운드에서 오래 실행되는 작업을 수행할 수 있는 컴포넌트이며 UI 제공하지 않는다. 이 서비스 컴포넌트를 왜 사용하는지, 어떻게 사용하는지 위주로 알아보고자한다.  

들어가기에 앞서, 이 내용은 Android 26 미만을 타겟으로 하는 어플리케이션에만 해당한다. 26 이상부터는 백그라운드 서비스에 대한 제약이 강화되었다. 따라서 여기서 설명할 서비스를 사용하는 용도와 예시는 26 이상 부터는 적합하지 않을 수 있다. (대안으로 `JobScheduler`와 `DownloadManager` 등을 사용할 수 있다.)

## 1. 서비스의 필요성

서비스를 사용하는 이유가 뭘까? 서비스를 이야기하기 전에 아래 코드를 한 번 보자.

```java
public class CalendarApplication extends Application {
    private List<Schedule> schedules;

    @Override
    public onCreate() {
        super.onCreate();

        //캘린더 일정 불러오기
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                //약 30초가 걸렸다고 가정
                schedules = CalendarDao().getSchedules();
            }
        });
        thread.start();
        ...
    }
}
```  

실제로는 이렇게 코딩하지 않지만 앱을 실행할 때 저장된 일정들을 불러오는 캘린더 앱이 있다고 하자. 약 30초가 걸리는 작업이기 때문에 UI를 블로킹하지 않기 위해 `Thread`를 사용해 백그라운드에서 일정을 불러오고 있다. 얼핏 보면 딱히 문제가 없어보이지만 아주 치명적인 문제를 야기할 수 있다.  

이 앱이 스레드 실행을 마치지 전에 백 키로 앱을 빠져나오고나 홈 키로 나가서 다른 앱을 오랫동안 사용하면, 프로세스가 종료될 수도 있다는 것이다. OS가 해당 앱의 **우선순위**가 낮다고 판단하여 프로세스를 종료하거나, 사용자가 최근 앱 목록에서 앱을 제거해버릴 수도 있다. OS나 사용자는 앱이 백그라운드에서 무엇인가 열심히 하고 있다는 것을 모르기 때문이다. 따라서 이 30초나 걸리는 작업의 안정성을 보장할 수 없다. 이처럼 Application 뿐만 아니라 액티비티에서도 마찬가지로 시간이 오래 걸리는 작업을 스레드에서 실행한다면 안정성을 100% 보장할 수 없다.  

### 1-1. 프로세스 우선순위  

OS가 프로세스를 제거해야할 때 우선순위를 부여하는 방식은 [공식문서](https://developer.android.com/guide/components/activities/process-lifecycle)에 나름 명료하게 나와있다.  

1. 포그라운드 프로세스 : 안드로이드 컴포넌트가 포그라운드에서 실행되는 프로세스이다. 메모리가 부족할 때에도 가장 마지막까지 남을 수 있는 프로세스이다. 다음 조건 중 하나라도 해당하면 프로세스가 포그라운드에 있는 것으로 간주된다.
    - 사용자와 상호 작용하는 액티비티를 가지고 있다. (`onResume()`이 실행된 상태)
    - `onReceive()`가 실행 중인 브로드캐스트 리시버를 가지고 있다.  
    - `onCreate()`, `onStart()`, `onDestroy()` 중 하나를 실행 중인 서비스를 가지고 있다.  

2. 가시(visible) 프로세스 : 포그라운드 컴포넌트를 가지고 있지는 않지만 사용자가 보는 화면에 아직 영향이 있는 프로세스이다.  
    - `onPause()`가 실행되었으나 가시 상태인 액티비티를 가지고 있을 때(다른 프로세스의 다이얼로그 테마나 투명한 액티비티가 가렸을 때)
    - `Service.startForeground()`를 통해 포그라운드 서비스로 실행 중인 `Service`가 있을 때
    - 가시 액티비티에 바운드된 서비스가 실행 중일 때  

3. 서비스 프로세스 : `startService` 메서드로 시작된 Service를 유지하는 프로세스이다. 사용자가 지금 보고 있는 화면과 직접적인 연관은 없지만 일반적으로 사용자가 관심을 가진 작업(ex 백그라운드 네트워크 데이터 업로드/다운로드)을 실행한다. 따라서 메모리가 많이 부족하지 않다면 서비스 프로세스까지는 실행 상태를 유지하려고 한다. 하지만 과도하게 오래 실행된다면 (ex 30분 이상) 우선 순위가 강등될 수 있다.  

4. 캐시 프로세스 : 하나 이상의 액티비티 인스턴스가 존재하지만 사용자에게 보이지 않고(`onStop()` 메서드가 호출되고 반환을 마친 상태) 활성화된 컴포넌트가 없는 프로세스이다. 따라서 다른 곳에서 메모리가 필요할 때 언제든 종료될 수 있다. 이는 LRU 목록으로 유지되며 일반적으로 가장 오래된 프로세스부터 종료된다.  

공식 문서에 나타나있는 조건들 말고도 세부적이고 다양한 조건들이 존재하긴하지만 우선 순위에 가장 큰 영향을 끼치는 것들은 보다시피 활성화된 **컴포넌트**의 유무이다. 따라서 앞에서 제기한 작업의 안전성을 보장하기 위해서는 서비스를 활용하면 된다.  

```java
public class CalendarApplication extends Application {
    private List<Schedule> schedules;

    @Override
    public onCreate() {
        super.onCreate();

        startService(new Intent(this, LoadSchedulesService.class));
        ...
    }
}

public class LoadSchedulesService extends Service {
    private List<Schedule> schedules;

    @Override
    public onCreate() {
        super.onCreate();

        //캘린더 일정 불러오기
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                //약 30초가 걸렸다고 가정
                schedules = CalendarDao().getSchedules();
            }
        });
        thread.start();
        ...
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```  

이렇게 하면 앱을 바로 백그라운드로 보내더라도 서비스의 생명주기 메서드가 실행 중일 때는 우선순위가 가장 높은 포그라운드 프로세스에 있다가 생명주기 메서드가 리턴되고 난 이후에는 세 번째 우선순위인 서비스 프로세스에 남는다.  

<br>

## 2. 서비스에 대한 오해  

### 2-1. 메인 스레드? 백그라운드 스레드?

서비스를 언급할 때 자주 나오는 얘기가 백그라운드상에서 실행되는 컴포넌트라는 것이다. 서비스는 액티비티처럼 눈에 보이는 가시 컴포넌트가 아니라는 의미로 백그라운드를 이야기하는 것이지, 메인 스레드가 아닌 별도 스레드에서 실행하는 것으로 착각하면 안된다. 서비스의 생명주기 메서드는 UI스레드에서 실행되므로 오래 걸리는 작업을 수행하려면 별도의 스레드를 생성해야한다.  

### 2-2. 싱글턴?  

서비스는 앱에서 1개의 인스턴스밖에 생기지 않는다. 따라서 일부러 싱글턴 객체를 만들고 그 안에서 백그라운드 스레드를 실행하는 수고를 하지 않아도 된다. 서비스를 이용하면 된다. 

<br>

## 3. 서비스 시작 방법  

`Context`에는 서비스를 시작하는 방법으로 `startService()`와 `bindService()` 메서드가 있다. (안드로이드 버전 26이상 부터는 `startForegroundService()` 메서드도 제공한다.)  

<img src="https://developer.android.com/images/service_lifecycle.png."/>  

서비스는 보통 스타티드 서비스(Unbounded Service)와 바운드 서비스(Bound Service)로 존재하는데, 스타티드이면서 바운드일 수 있다. 스타티드이면서 바운드인 서비스는 코드도 복잡하고 고려할 것이 많기 대문에 피하는 게 좋지만 어쩔 수 없이 사용해야하는 경우도 있다. 예를 들어 음악 재생 화면이 있을 때 화면을 종료해도 음악을 들을 수 있으려면 스타티드 서비스를 이용해야한다. 그런데 다시 화면에 진입할 때 재생 중인 음악 정보를 화면에 보여줘야 한다면 바운드 서비스이기도 해야 한다.  

## 4. 스타티드 서비스  

스타티드 서비스는 `startService()` 메서드나 `startForegroundService()`로 시작된다. `startForegroundService()`를 사용해 서비스를 시작한 경우, 5초 이내에 서비스의 `startForeground()`를 호출해 알림을 표시해야한다. 그렇지 않으면 서비스를 중단시키고 ANR로 간주한다. 이후 내용에서는 이 두 메서드를 `startService()` 하나로 통칭하겠다.

그러나 `startService()`를 호출한다고해서 서비스가 곧바로 시작되는 것은 아니다. 메인 루퍼의 메시지큐에 메시지가 들어가서 메인 스래드를 쓸 수 있는 시점에 서비스가 시작된다. `startService()` 메서드는 곧바로 ComponentName을 리턴하고 다음 라인을 진행한다. 이는 `Intent Bundle`에 파라미터를 전달하고 서비스에 작업하도록 **요청**하는 역할을 할 뿐이다.  

### 4-1. onCreate(), onStartCommand()  

서비스가 처음 생성되는 경우에는 `onCreate()`를 거쳐서 `onStartCommand()` 메서드를 실행한다. 그 이후에 `startService()`를 호출하면 `onCreate()`는 거치지 않고 `onStartCommand()` 메서드가 실행된다. `onCreate()`는 서비스에 필요한 리소스 등을 준비하는 작업을 하고, `onStartCommand()`는 이름 그대로 명령을 매번 처리하는 역할을 한다. 그래서 액티비티와는 다르게 서비스의 `onCreate()` 메서드에는 전달된 `Intent`를 사용할 수 없다.  

표준 패턴은 `onStartCommand()`에서 백그라운드 스레드를 생성하고 스레드에서 명령에 따른 작업을 수행하는 것이다. 여기서 백그라운드 스레드를 생성하지 않고 작업을 바로 수행하면, 작업의 속도에 따라 UI 업데이트에 영향을 끼칠 수 있다.  

### 4-2. 컴포넌트 간 통신  

<b>브로드캐스트</b>  
서비스에서 작업 진행 상황에 따라 액티비티에 메시지를 보내려면 일반적으로 브로드캐스트를 사용한다. 예를 들어 서버와 동기화를 하는 `SyncService`가 있는데, 화면에서 버튼을 눌러 이를 시작한다. 동기화 도중에는 화면에 ProgressBar로 진행 중을 표시하고, 동기화가 끝나면 종료 메시지를 표시하려고 한다. 이때 액티비티에서는 브로드캐스트 리시버를 등록하고 서비스에서는 `sendBroadcase()`를 실행한다.  

<b>ResultReceiver</b>  
또 다른 방식으로는 서비스를 시작하는 `Intent`의 `ResultReceiver`를 통해 값을 되돌려줄 수도 있다. **단방향** 메시지 전달이라면 이 방식이 간편하다.  

```java
//액티비티
private View syncLayout, progressBar;
private TextView syncMassage;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);
    ...
    Intent intent = new Intent(this, SyncService.class);
    //서비스를 시작하면서 resultResceiver를 전달함
    intent.putExtra(Constant.EXTRA_RECEIVER, resultReceiver);
    startService(intent);
}

private Handler hander = new Handler(Looper.getMainLooper());

private ResultReceiver resultReceiver = new ResultReceiver(handler) {
    @Override
    protected void onReceiveResult(int resultCode, Bundle resultData) {
        if (resultCode == Constant.SYNC_COMPLETED) {
            //동기화 종료 표시
        }
    }
};
```  

`ResultReceiver`를 생성할 때 생성자에는 `Handler` 인스턴스를 넣을 수도, null을 넣을 수도 있다. 서비스의 백그라운드 스레드에서 `ResultReceiver`의 `send()` 메시지를 호출하는데, 결과를 받는 쪽에서 UI를 업데이트하기 때문에 `Handler`를 거쳐 메인 루퍼의 메시지큐에 메시지를 넣은 것이다. 호출하는 쪽과 받는 쪽이 둘 다 백그라운드 스레드에서 동작하거나, UI와 관련이 없다면 null을 사용해도 된다.  

```java
//서비스
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            //... 동기화 작업

            //intent로 전달된 ResultReceiver 꺼내기
            final ResultReceiver receiver
                = intent.getPacelableExtra(Constant.EXTRA_RECEIVER);

            //ResultReceiver의 send()메서드로 작업 완료 알림
            receiver.send(Constant.SYNC_COMPLETED, null);
            stopSelf();
        }
    }).start();
    return START_NOT_STICKY;
}
```  

여기서는 작업 종료 메시지만 보냈지만 작업 진행률이나 실패 메시지와 같은 다양한 결과를 보낼 수도 있다.  

### 4-3. 서비스 재시작 방식  

가용 메모리가 낮거나 포커스를 갖고 있는 액티비티의 시스템 리소스를 복구해야할 때, 안드로이드 시스템은 서비스를 강제 종료시킬 수 있다. 포커스된 액티비티에 바인딩되었거나 포그라운드 서비스의 경우, 종료 가능성이 희박하긴하지만 안심할 수는 없다. 또한 서비스가 시작되어 장시간 실행 중이라면 우선순위가 조금씩 낮아지고 강제 종료될 가능성도 높아진다. 스타티드 서비스는 강제 종료 후 가능한 한 빨리 시스템에서 서비스를 재시작한다.  

하지만 그렇다고 서비스가 강제 종료되면 시스템이 알아서 재시작한다고만 알고 넘어가면 안된다. 서비스가 언제 재시작하는지, 재시작을 안 하는 조건은 무엇인지 알아야 서비스를 안정적으로 다룰 수 있다.  

#### 4-3-1. onStartCommand()  

스타티드 서비스에서는 `onStartCommand()` 메서드에서 리턴하는 int 상수를 가지고 재시작 방식을 제어한다. `onStartCommand()` 메서드의 시그니처는 다음과 같다.  

`public int onStartCommand(Intent intent, int flags, int startId)`  

리턴 값으로 사용되는 int 상수를 하나씩 살펴보자.  

**START_NOT_STICKY**

강제 종료되면 재시작하지 않는다. 명시적으로 `startService()`를 실행할 때만 의미 있는 작업에 사용된다. 예를 들어 화면에 보여줄 뉴스를 API로 가져와서 저장할 수 있는데, 메모리 이슈로 서비스가 강제 종료되었다면 `startService()` 명령을 기다려서 최신 뉴스를 다시 가져오는 것이 나을 것이다.  

**START_STICKY**  

기본 리턴 값이다. 서비스가 정상적으로 종료되지 않았다면 메모리가 확보되었을 때 자동으로 재시작한다. 재시작 시에는 다시 `onStartCommand()`를 호출하는데 이때 `Intent` 파라미터가 null로 전달되기 때문에 재시작하면서 NPE를 발생시킬 가능성이 있다. 따라서 전달된 `Intent`를 사용하지 않고 내부 상태 변수만 사용하는 서비스에 적합하다. 예를 들어 SNS 앱에서 새로운 메시지가 몇 개나 왔는지 정기적으로 API를 호출해서 확인한다면 `Intent` 파라미터가 전달될 필요가 없다. 이때는 재시작할 때 `Intent` 파라미터가 null이어도 무관하다.  

**START_REDELIVER_INTENT**  

재시작하면서 `onStartCommand()`에 `Intent`를 다시 전달하여 실행한다. 어떻게든 해당 파라미터를 가지고 실행해야 하는 서비스가 이에 해당한다. 쇼핑몰 앱에서 API를 통해 특정 상점의 상품 목록을 가져온 후 DB에 저장하는 경우를 예로 들 수 있다.  

<b>백그라운드 서비스에서의 한계</b>  

`START_STICKY`나 `START_REDELIVER_INTENT`는 서비스가 시스템에 의해 종료된 경우, 재시작하는 상수라하였다. 어떤 상황에 어울리는지 예시도 함께 설명했으나 사실 백그라운드 서비스는 해당 상수의 사용이 어렵다.  

[공식문서](https://developer.android.com/about/versions/oreo/android-8.0-changes.html#back-all)에 따르면, Android 8.0 이상 부터는 캐시된 프로세스 상태이고 포그라운드 컴포넌트가 없다면 백그라운드 서비스를 실행하는데 여러가지 제약이 따른다. 시스템에 의해 `START_STICKY`, `START_REDELIVER_INTENT`를 리턴한 백그라운드 서비스가 종료되면 곧이어 서비스를 재시작하려고 시도한다. 그러나 서비스를 재시작하기 위해 실행한 프로세스는 액티비티도 함께 재시작하지는 않기 때문에 캐시된 프로세스 상태이고, 포그라운드 컴포넌트가 없는 상태이다. 따라서 백그라운드 서비스를 실행할 때 `IllegalStateException` 이 발생한다.  

> java.lang.IllegalStateException: Not allowed to start service Intent { cmp=com.gugyu.servicetest/.SleepService }: app is in background uid UidRecord{3a1d18a u0a84 SVC  idle change:idle|uncached procs:1 proclist:29836, seq(0,0,0)}

따라서 Android 8.0 이상을 대상으로 하는 앱이라면, `JobScheduler`나 `DownloadManager`, 혹은 `ForegroundService`를 고려해보자.  

<br>

#### 4-3-2. 펜딩 서비스 재시작  

서비스가 시작되기 전에 앱의 다른 컴포넌트에서 크래시가 발생하면 어떻게 될까? `Application`의 `onCreate()`에서 `startService()`를 실행했는데 메인 액티비티의 `onCreate()` 메서드 내에서 크래시가 발생해 프로세스가 종료되었다면, 서비스는 정상적으로 실행될까? 결론부터 말하자면 앱을 새로 실행하면서 서비스를 시작한다. (만약 앱이 중단되었다는 다이얼로그가 뜬 경우에는 "앱 닫기" 버튼을 누르지 말고 "앱 정보" 등 다른 버튼을 누르면 앱이 곧이어 재시작 되는 것을 확인할 수 있다.)  

처음 프로세스가 시작될 때 `Application` 의 `onCreate()` 메서드에서 `startService()`를 실행해도, 메시지 큐의 순서상 액티비티가 먼저 시작되고 서비스는 그 다음에 시작된다. 이때 시작 액티비티의 `onCreate()`에서 크래시가 발생하면 프로세스는 죽지만 `com.android.server.am.ActivityServices`에서 팬딩 서비스를 실행하기 위해서 다시 프로세스를 띄운다. 프로세스가 뜰 때 액티비티는 띄우는 대상이 아니므로 `Application`을 생성한 이후에 바로 서비스만 시작된다.  

하지만 마찬가지로 8.0 이상 부터는 백그라운드 서비스를 재시작하면서 `IllegalStateException`가 발생하니 주의하도록 하자.  

#### 4-3-3. 불필요한 재시작 방지  

서비스를 재시작할 수 있도록 했지만 작업이 다 끝났는데도 서비스를 재시작하는 경우가 발생할 수 있다. 이는 서비스를 정상적으로(명시적으로) 종료하지 않았기 때문이다. 서비스를 종료하는 방법은 두 가지가 있다.  

- `Context`의 `stopService()`  
- `Service`의 `stopSelf()`  

`stopService()` 메서드는 앱을 사용하는 내내 실행되는 서비스나 포그라운드 서비스를 사용하는 경우에 주로 사용되지만 그런 케이스가 많지 않다. 특히 백그라운드 서비스의 경우에는 이런 방법은 권장되지 않는다. 백그라운드 서비스는 필요할 때만 동작하고 작업을 마치는 즉시 바로 종료하는 것이 좋다.  

`Service`의 `stopSelf()` 메서드는 `Context`의 `stopService()` 메서드와 역할이 동일하지만 `Service` 내에서 호출한다는 것이 다르다. 서비스에서 할 일이 끝났으면 백그라운드 스레드든 아니든 직접 `stopSelf()`를 실행해서 서비스를 명시적으로 종료해야한다. 이때 서비스는 `onDestory()`까지 실행된다.  

만약 서비스를 명시적으로 종료하지 않는다면, 할 일은 다 끝났는데 서비스는 시작된(started) 상태로 남아 불필요하게 메모리를 차지하고 있다. 어느 순간에 메모리 이슈로 서비스가 강제 종료되면 리턴 상수가 `START_STICKY`나 `START_REDELIVER_INTENT`인 경우에 의도치 않게 재시작하는 일이 생긴다.  

<br>

### 4-4. 멀티 스레드 이슈  

스타티드 서비스는 멀티 스레드 이슈를 주의해야한다. 여러 곳에서 `startService()`를 동시에 호출할 수 있다. 어차피 UI 동작은 단일 스레드 모델을 따르기 때문에 `onCreate()`, `onStartCommand()` 메서드가 한 번에 하나씩만 호출되지만, 스레드는 여러 개가 동시에 실행될 수 있다.  

#### 4-4-1. 멤버 변수  

`onStartCommand()`에서 백그라운드 스레드를 시작하면 여러 스레드가 동시에 실행될 수 있고, 이때 값을 잘못 공유하면 문제가 발생할 여지가 생긴다. 예를 들어 API를 통해 특정 정보를 요청해야하는데, 전달된 `Intent`의 id 값을 서비스의 멤버 변수로 쓰면 어떻게 될까? 의도치 않게 요청한 id 값이 도중에 변경되어 데이터가 잘못 저장될 수 있다.  

#### 4-4-2. stopSelfResult()  

여러 클라이언트에서 `startService()`를 실행한다면 모든 작업이 끝나고 나서야 서비스를 종료해야한다. 그러나 모든 작업이 끝나는 시점을 어떻게 알 수 있을까? 무작정 진행 중인 작업이 끝날 때 마다 `stopSelf()`를 호출하면 다른 클라이언트가 요청한 작업에서 문제가 발생한다.  

이런 경우에 대비해서 `stopSelfResult(int startId)` 메서드가 존재한다. `startId`는 `onStartCommand()`에 전달된 값으로, `startId`가 가장 최근에 시작된 것이라면 그때에만 서비스를 종료한다. 이 메서드를 사용하면 각각의 작업이 끝날 때 마다 `stopSelfResult()`를 실행해도 모든 작업이 끝나고 나서야 서비스가 종료된다.  

<br>

### 4-5 외부 프로세스에서의 서비스 접근

intent-filter를 사용하면 암시적 인텐트로써 외부에서 서비스를 실행할 수 있다.  

```xml
<service android:name=".RemoteService" android:process":remote">
    <intent-filter>
        <action android:name="com.gugyu.servicetest.REMOTE_SERVICE" />
    </intent-filter>
</service>  
```

서비스를 시작할 때는 action name에 지정된 값을 사용한다.  

`startService(new Intent("com.gugyu.servicetest.REMOTE_SERVICE"))`;  

위 코드를 해당 앱뿐만 아니라 다른 앱에서 실행해도 `RemoteService`를 실행할 수 있다.  

만약 action name이 동일한 서비스가 여러 개 있다면 어떻게 될까? 액티비티의 경우는 action name이 동일하면 액티비티를 선택하라는 다이얼로그를 띄워주고, 브로드케스트 리시버는 action name이 동일한 모든 브로드캐스트 리시버를 실행한다. 서비스는 intent-filter의 `android:priority` 속성 값을 비교해서 높은 것을 실행하고, 같다면 시스템이 랜덤으로 선택해서 실행한다. 결국 암시적 인텐트로 서비스를 실행하면 문제가 발생할 수 있다.  

```xml
<service android:name=".RemoteService" android:process":remote">
    <intent-filter android:priority="999">
        <action android:name="com.gugyu.servicetest.REMOTE_SERVICE" />
    </intent-filter>
</service> 
```  

intent-filter를 적용하지 않고 명시적 인텐트로 외부에서 서비스를 직접 시작할 수도 있다.  

```java
ComponentName cName = new ComponentName("com.gugyu.servicetest", "com.gugyu.servicetest.RemoteService");
startService(new Intent.setComponent(cName));
```  

명시적 인텐트를 통해 외부 프로세스에서 서비스를 시작할 수 있도록 허용하려면  `AndroidManifest.xml`에 `android:exported` 속성을 true로 설정해야 한다. 그리고 암시적 인텐트와는 다르게 `ComponentName`에 정확한 패키지명과 정확한 서비스 클래스의 이름을 명시해야한다.  

그리고 마찬가지로 8.0 이상 부터는 명시적 인텐트로 서비스를 실행할 때에도 제약이 있어 예외가 발생할 수 있다.  

#### 4-5-1. 암시적 인텐트 제한  

롤리팝부터는 암시적 인텐트로 서비스를 실행할 수 없다. `startService()`, `bindService()` 둘 다 마찬가지다. 암시적 인텐트로 서비스를 시작하면 보안적으로 위험하기 때문이다. 요청한 Intent에 어느 서비스가 응답할지 모르는데다 사용자는 어느 서비스가 시작되는지 볼 수 없기 때문에 더욱 문제가 크다.  

> java.lang.IllegalArgumentException: Service intent must be explicit  

롤리팝 이전 버전을 타겟으로 한다고 해도 경고 메시지가 뜬다.  

<br>

### 4-6. IntentService 클래스  

서비스에서 멀티 스레딩이 필요한 경우는 많지 않다. 동시에 여러 요청을 처리할 필요가 없다면 `IntentService`를 활용하는 것이 좋다. 내부적으로 1개의 백그라운드 스레드를 가지고 전달된 `Intent`를 순차적으로 처리할 수 있다. 핸들러와 유사한데, 실제로 내부적으로 `HandlerThread`를 사용한다.  

```java
public class NewsReaderService extends IntentService {

    ///IntentService에는 기본 생성자가 없기 때문에 직접 생성자를 추가해줘야한다.
    //생성자에 들어가는 name 파라미터는 백그라운드 스레드의 스레드 명으로 사용된다.
    public NewsReaderService() {
        super("NewsReader");
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        ...
    }
}
```  

`IntentService`는 `Service`를 확장하여 추상 메서드들을 구현해두었다. `onStartCommand()`의 기본 리턴 값은 `START_NOT_STICKY`인데, 이 값을 변경하려면 생성자에서 `setIntentRedelivery(true)` 를 호출하면 된다.  

구조는 단순하다. `onCreate()`에서 `HandlerThread`를 생성하고 루퍼와 연결된 `Handler` 를 만든다. `onStartCommand()` 메서드는 실행될 때마다 `Handler`에 메시지를 보내고 `handleMessage()`는 우리가 구현한 `onHandleIntent()` 메서드를 실행한다. 그리고 작업이 끝나면 `stopSelf(int startId)`를 호출해서 서비스를 종료한다.  

#### 4-6-1. 주의 사항  

`IntentService`를 사용할 때 내부 구조를 이해하지 못하면 문제가 생길 수 있다.

예를 들어 백그라운드 스레드에서 Toast를 띄우려고 하면 에러가 발생한다. Toast는 내부 클래스 TN에서 Handler로 기본 생성자를 사용하는 부분이 있기 때문이다. 따라서 백그라운드 스레드에서는 Looper가 없기 때문에 별도로 `Looper.prepare()`를 먼저 실행해줘야 Toast를 띄울 수 있다.  

그러나 백그라운드 스레드에서 동작하는 메서드인 `IntentService`의 `onHandleIntent()`에서 Toast를 띄우면 에러가 발생하지 않고 성공한다. `IntentService`가 내부적으로 사용한 `HandlerThread`에서 이미 `Looper.prepare()`를 실행시켰기 때문이다. 그러나 여기에는 치명적인 문제가 있다.  

`Toast.show()`는 바인더 통신을 통해 system_server 프로세스에서 `NotificationManagerService`의 `enqueToast()` 메서드를 호출한다. 이때 파라미터로 바인터 콜백이 전달되고, 콜백에서는 2가지 작업이 수행된다. 화면에 Toast를 보여주는 작업과 일정 시간 (`Toast.LENGTH_SHORT` 등) 이후에 제거하는 작업이다.  

그런데 `IntentService`에서는 `onHandleIntent()`를 실행한 이후에 바로 `stopSelf()`를 호출하고, `onDestroy()`에서는 `HandlerThread`가 생성한 `Looper`를 종료하는 `Looper.quit()`를 호출한다. 루퍼가 종료되면 더이상 루퍼의 메시지큐로 전달되는 콜백이 실행되지 않는다.  

루퍼가 종료되면서 Toast를 보여주는 콜백과 제거하는 콜백이 모두 실행되지 않을 수도 있고, 보여주는 콜백은 실행되었지만 제거하는 콜백은 실행되지 않을 수도 있다. 특히 후자의 경우는 액티비티를 종료해도 계속 토스트가 남아있을 수 있다. 이런 경우는 시점 문제라서 재현이 잘 되지도 않고 원리를 모르면 해결할 수도 없다.  

<br>

### 4-7. 서비스 중복 실행 방지  

`onStartCommand()`에서 스레드를 생성하면, 서비스 명령이 전달될 때 마다 스레드가 계속 생성된다. 그러나 경우에 따라 스레드가 이미 시작되어 있으면 나머지는 스킵해야할 때도 있다.  

예를 들어 메모 앱에서 서버와 지속적으로 데이터를 동기화한다고 해보자. 앱을 시작할 때나 동기화 버튼을 누를 때, 그리고 메모를 추가할 때도 동기화를 실행한다. 그러나 이들 작업은 여러 개를 동시에 실행하면 문제가 발생할 여지가 있다.  

이렇게 한 가지 작업에만 충실하고 나머지는 확실히 스킵해야한다면 `onCreate()`에서 스레드를 생성하고 작업을 수행하면 된다. 처음 `startService()`를 호출할 때만 `onCreate()`가 실행될 것이고, 작업이 끝나고 `stopSelf()`를 호출한다면 문제가 없다.  

`IntentService`와 유사해 보이긴하지만 `IntentService`는 실행 요청된 명령들을 **순차적으로** 모두 다 처리한다. 작업이 이미 수행 중이라면 최초 실행된 명령 외에는 스킵하고 싶은 목적과는 어긋난다.  

물론 `onStartCommand()`에서 스레드를 생성하면서도 동일하게 다른 명령들을 스킵할 수 있다. 작업이 수행 중인지 나타내는 `isRunning` 멤버 변수를 두고 `true`일 때는 바로 메서드를 리턴해버리면 된다.  

혹은 `ThreadPoolExecutor`를 사용해서 스레드 개수를 1로 고정하는 방법도 있다.  

```java
//DiscardPolicy -> 가용 가능한 스레드가 없을 때는 요청을 무시
private ThreadPoolExecutor executor = new ThreadPoolExecutor(1, 1, 0,
    TimeUnit.SECONDS, new SynchronousQueue<Runnable>(),
    new ThreadPoolExecutor.DiscardPolicy());

@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    executor.submit(new Runnable() {
        @Override
        public void run() {
            ...
            stopSelf();
        }
    });
    return START_STICKY;
}
```  



<br>
<br>


--- 
해당 포스팅은 [안드로이드 프로그래밍 Next Step - 노재춘 저](http://www.yes24.com/Product/Goods/41085242) 을 바탕으로 내용을 보충하여 작성되었습니다.

참고:  
[인텐트 및 인텐트 필터](https://developer.android.com/guide/components/intents-filters?hl=ko)  
[백그라운드 실행 제한](https://developer.android.com/about/versions/oreo/background?hl=ko)  
[서비스 개요](https://developer.android.com/guide/components/services?hl=ko)