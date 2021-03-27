---
layout: post
title: "안드로이드 : 바운드 서비스"
subtitle: "백그라운드 컴포넌트"
author: "GuGyu"
header-style: text
tags:
  - Android
  - AndroidFramework
  - Service
  - Background
  - BoundService
---

[서비스 개요, 스타티드 서비스](https://seonggyu96.github.io/2021/03/24/started_service/)  

# 바운드 서비스  

바운드 서비스는 클라이언트-서버 인터페이스 안에서 "서버" 역할을 한다. 바운드 서비스를 사용하면 액티비티 등 컴포넌트에 서비스를 바인딩할 수 있다. 컴포넌트는 서비스에게 요청을 보내고 서비스는 응답하며 타 어플리케이션의 컴포넌트도 포함하기 때문에 프로세스간 통신(IPC)를 가능토록한다. 이런 목적으로 사용되는 서비스이기 때문에 일반적으로 바운드 서비스는 바인딩한 컴포넌트와 함께 상호작용할 때만 유지되고, 스타티드 서비스 처럼 백그라운드에서 무한히 실행되지 않는다.  

클라이언트-서버 인터페이스 개념이 익숙하지 않으면 바운드 서비스에 대한 설명을 아무리 읽어도 잘 이해가 가지 않는다. 아주 쉽게 단편적으로 설명하자면 서비스에서 제공하는 메서드를 다른 컴포넌트에서 호출할 수 있는 구조이다.  

사용 절차도 간단하다. `bindService()` 메서드를 실행해서 바인딩하고서 이후에 필요한 메서드를 호출한다.  

**bindService()**  

`Context`에 있는 `bindService()` 메서드의 시그니처는 다음과 같다.  

`public abstract boolean bindService(Intent service, ServiceConnection conn, int flags)`  

- 첫 번째 파라미터인 service는 대상 서비스를 가리킨다.  
- 두 번째 파라미터인 conn은 서비스와 연결되거나 연결이 끊길 때의 콜백이다.  
- 세 번째 파라미터인 flags에는 0을 넣을 수도 있고, Context의 BIND_XXX 상수를 넣을 수도 있다. 비트 OR 연산으로 상수를 여러 개 넣어도 된다. 가장 많이 쓰이는 상수는 `Context.BIND_AUTO_CREATE`이고, ICS(아이스크림 샌드위치)부터 추가된 상수들은 주로 서비스 프로세스의 우선순위와 관련이 있다.  

**BIND_AUTO_CREATE**  

`BIND_AUTO_CREATE` 옵션의 역할에 대해서 더 살펴보자.  

`bindService()`를 실행한다고해서 서비스에 항상 바인딩되는 것은 아니다. 서비스가 생성되어야 바인딩이 가능하다. `BIND_AUTO_CREATE` 옵션은 서비스가 생성된 게 없다면 새로 생성하도록 해준다. 이 옵션이 없다면 `bindService()`를 실행해도 서비스가 자동으로 생성되지 않는다. 어디선가 `startService()`를 먼저 실행해서 서비스를 생성해둬야한다. 따라서 스타티드 겸 바운드 서비스가 아닌 이상 `BIND_AUTO_CREATE` 옵션은 필수적이다.  

서비스에 바인딩된 클라이언트(컴포넌트)가 여러 개 남아있을 때 `stopService()`를 실행하면 어떤 일이 발생할까? `BIND_AUTO_CREATE` 옵션이 있다면 서비스가 종료되지 않는다. 모든 클라이언트가 `unbindService()`를 실행해야 서비스의 `onDestroy()`가 불린다. 반면에 옵션이 없는 경우에는 `stopService()`를 실행하면 연결이 끊기고 바로 `onDestroy()`가 호출된다.  

마지막으로, 바인딩된 클라이언트가 남아있는 상태에서도 서비스 프로세스는 메모리 문제 등으로 종료될 수 있다. 이때 `BIND_AUTO_CREATE` 옵션이 있다면 프로세스가 살아나서 재연결된다.  

<b>바운드 서비스의 용도</b>  

최신 뉴스나 인기 검색어, 날씨처럼 업데이트가 필수적인 데이터가 있다. 이 데이터를 JSON으로 결과를 리턴하는 API서버가 있을 때, 이런 데이터를 앱에서 사용하기 위한 방법에는 어떤 게 있을까?  

1. 앱에서 HTTP 호출을 통해 데이터에 직접 접근한다.  
2. HTTP 호출을 하고 결과도 객체로 리턴해주는 오픈 API jar를 만들어서 앱에서는 jar를 이용해서 데이터에 접근한다.  
3. 서비스 앱에서 데이터를 제공해주는 바운드 서비스를 만들고, 이 안에서 HTTP 호출을 하고 객체를 리턴한다. 다른 앱에서는 `bindService()`를 실행하여 서비스의 데이터에 접근한다. 이를테면 검색 앱에서는 인기 검색어 목록을 바운드 서비스로 외부에 제공할 수 있다. [Google Play In-app Billing](https://developer.android.com/google/play/billing)도 이 방식을 사용하고 있다.  
4. 외부에 공개하는 jar 내부에서 `bindService()`를 실행하고 결과를 받는다. [Google Play Service](https://developers.google.com/android/guides/overview)가 이런 형태이다. 여기에는 `connect()`와 `disconnect()` 메서드가 있는데, 내부적으로 각각 `bindService()`, `unbindService()`를 호출한다.  

3번과 4번에서 바운드 서비스를 사용한다. 아래로 갈수록 추상화 레벨이 높다. 추상화 레벨이 높을수록 좋은 게 아니라, 규모에 맞는 적절한 레벨을 선택하는 게 좋다. 오버 엔지니어링을 피하자는 얘기다.  

API로 데이터를 조회하는 예를 들었지만, 이처럼 한 곳에서 특화된 기능을 내/외부 프로세스의 여러 클라이언트에게 제공할 때도 바운드 서비스를 사용할 수 있다. 단순히 데이터를 제공하는 것은 콘텐트 프로바이더도 가능하지만, 바운드 서비스에서는 콜백을 이용한 상호 작용이 가능하다.  

<br>  

## 1. 리모트 바인딩  

리모트 바인딩 서비스는 다른 프로세스에서 접근하는 것을 전제로 설계된다. 따라서 로컬에서만 사용하는 서비스라면 리모트 바인딩 서비스를 굳이 만들 필요가 없다. 리모트 바인딩 서비스를 만드는 간단한 예제와 절차를 살펴보자.  

### 1-1. aidl 인터페이스와 생성 클래스  

[AIDL(Android Interface Definition Language)](https://developer.android.com/guide/components/aidl?hl=ko) 인터페이스는 클라이언트에게 제공할 메서드를 정의한다. 다음과 같이 src/main/aidl 디렉터리에 `IRemoteService.aidl`을 생성해보자.

```java
interface IRemoteService {
    boolean validCalendar(long calendarId, String calendarType);
}
```  

그 다음, 빌드 시키면 build/generated/aidl_source_output_dir 에 `IRemoteService.java`가 생성된다.  

```java
public interface IRemoteService extends android.os.IInterface {
    
    public static class Default implements IRemoteService { 
        @Override
        public boolean validCalendar(long calendarId, java.lang.String calendarType) throws android.os.RemoteException {
            return false;
        }

        @Override
        public android.os.IBinder asBinder() {
            return null;
        }
    }

    public static abstract class Stub extends android.os.Binder implements IRemoteService { //(1)
        private static final java.lang.String DESCRIPTOR = "IRemoteService";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR); //(2)
        }

        //(3)
        public static IRemoteService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR); // (4)
            if (((iin != null) && (iin instanceof IRemoteService))) {
                return ((IRemoteService) iin);
            }
            return new IRemoteService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        //(5)
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_validCalendar: {
                    data.enforceInterface(descriptor); //(6)
                    long _arg0;
                    _arg0 = data.readLong();
                    java.lang.String _arg1;
                    _arg1 = data.readString();
                    boolean _result = this.validCalendar(_arg0, _arg1);
                    reply.writeNoException();
                    reply.writeInt(((_result) ? (1) : (0)));
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements IRemoteService { //(7)
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public boolean validCalendar(long calendarId, java.lang.String calendarType) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR); //(8)
                    _data.writeLong(calendarId);
                    _data.writeString(calendarType);
                    //(9)
                    boolean _status = mRemote.transact(Stub.TRANSACTION_validCalendar, _data, _reply, 0);

                    if (!_status && getDefaultImpl() != null) {
                        return getDefaultImpl().validCalendar(calendarId, calendarType);
                    }
                    _reply.readException();
                    _result = (0 != _reply.readInt());
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            public static IRemoteService sDefaultImpl;
        }

        //(10)
        static final int TRANSACTION_validCalendar = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);

        public static boolean setDefaultImpl(IRemoteService impl) {
            if (Stub.Proxy.sDefaultImpl != null) {
                throw new IllegalStateException("setDefaultImpl() called twice");
            }
            if (impl != null) {
                Stub.Proxy.sDefaultImpl = impl;
                return true;
            }
            return false;
        }

        public static IRemoteService getDefaultImpl() {
            return Stub.Proxy.sDefaultImpl;
        }
    }

    public boolean validCalendar(long calendarId, java.lang.String calendarType) throws android.os.RemoteException;
}

```

(7)의 `Proxy`와 (1)의 `Stub`으로 구분되는 클라이언트와 서버는 `Parcel`을 가지고 데이터를 주고받는다. `Stub`은 추상 클래스이고 `validCalendar()` 메서드가 구현되어야 한다. 반면에 `Proxy`는 구체 클래스이다.  

(9)에서 mRemote의 `transact()` 메서드를 호출하면 `Stub`에서는 (5)의 `onTransact()` 메서드가 호출된다.  

(8)에서 Parcel에 쓰기 시작하고 (6)에서 읽기 시작한다.  

(3)에서 `asInterface()` 메서드의 내용을 보자. (2)에서 `attachInterface()`에 등록한 `Stub`을 (4)에서 `queryLocalInterface()` 메서드로 조회한다. 동일한 프로세스에 있다면 `Stub` 인스턴스가 조회되고 이를 사용한다. 내부족으로 로컬일 수도 있고 리모트일 수도 있는 바인딩이 `asInterface()` 메서드에서 결정된다.  

(10) aidl 에 선언한 메서드명을 이용해 `TRANSACTION_validCalendar`와 같이 상수로 만든 것을 볼 수 있다. 이 상수는 `transact()`와 `onTransact()` 메서드 사이에 전달되는 구분자로 사용된다. 구분자에 메서드명을 사용하는 규칙 때문에 메서드명이 동일한 게 있다면 구분되지 않는다. 그래서 aidl 인터페이스에는 메서드 오버로딩이 허용되지 않는다.  

### 1-2. Service에 Stub 구현  

서비스에서는 추상 클래스인 `Stub` 구현체를 만든다.  

```java
public class RemoteSerivce extends Service {
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    //Stub 구현
    private final IRemoteService.Stub binder = new IRemoteService.Stub() {
        @Override
        public boolean validCalendar(long calendarId, String calendarType) {
            CalendarType type = CalendarType.valueOf(calendarType);
            ...
        }
   };
}
```  

### 1-3. 클라이언트에서 서비스 바인딩  

다른 컴포넌트에서 서비스를 바인딩해서 사용하는 것도 복잡하지는 않다. `bindService()`는 바인딩 결과를 비동기로 받기 때문에 콜백으로 사용할 `ServiceConnection` 인스턴스를 `bindService()`메서드에 파라미터로 전달한다.  

```java
private IRemoteSerivce mIRemoteService;

// 커넥션 콜백 생성  
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName className, IBinder service) {
        //asInterface 메서드를 통해 로컬인 경우는 Stub 인스턴스,
        //리모트인 경우는 Proxy 인스턴스가 mIRemoteServicedp 대입된다.
        mIRemoteService = IRemoteService.Stub.asInterface(service);
    }

    // 크래시가 발생하거나 서비스가 강제 종료되었을 때 호출
    // unbindService() 때 호출되는 것이 아니다.
    @Override
    public void onServiceDisconnected(ComponentName className) {
        mIRemoteService = null;
    }
};

@Override
public void onStart() {
    super.start();
    ComponentName cName = new ComponentName("com.gugyu.servicetest", "com.gugyu.servicetest.RemoteService");
    bindService(new Intent.setComponent(cName), mConnection, Context.BIND_AUTO_CREATE);
}

public void checkValid() {
    //onServiceConnected()가 호출되기 전일 수도 있고, 서비스가 바인딩되지 않았을 수도 있음
    //null 체크 필수  
    if (mIRemoteService != null) {
        boolean valid = mIRemoteService.isValidCalendar(10L, CalendarType.NORMAL.name());
        ...
    }
}
``` 

서비스의 메서드를 로컬에서 호출할 때는 호출하는 스레드에서 서비스의 메서드가 실행된다. 반면 리모트에서 호출하는 경우에는 서비스가 속한 프로세스의 바인더 스레드 풀에서 실행되기 때문에 리모트 바인딩 서비스는 thread safety하게 만들어져야한다.

### 1-4. aidl에서 지원하는 데이터 타입  

리모트 바인딩만으로도 로컬 바인딩을 커버할 수 있지만 굳이 따로 나눈 이유가 있다. 프로세스 간에 데이터를 주고 받을 때는 [마샬링(marshaling)/언마샬링(unmashaling)](https://ko.wikipedia.org/wiki/%EB%A7%88%EC%83%AC%EB%A7%81_(%EC%BB%B4%ED%93%A8%ED%84%B0_%EA%B3%BC%ED%95%99))이 필요하기 때문에 aidl 인터페이스에 쓸 수 있는 데이터 타입이 제한된다.  

- primitive type
- String
- List: 구현체인 ArrayList 같은 타입은 쓸 수 없다. 제네릭 타입은 일부만 쓸 수 있다. `List<String>`, `List<List>` 같은 타입은 쑬 수 있지만 `List<?>`, `List<List<String>>` 같은 타입은 쓸 수 없다.  
- Map: 역시 구현체인 HashMap 같은 타입은 쓸 수 없다. 제네릭은 지원하지 않는다.  

<br>

## 2. 로컬 바인딩  

로컬 바인딩 서비스는 로컬 프로세스에서만 접근 가능한 서비스이다. 리모트 바인딩 서비스보다 훨씬 간단하고 사용할 수 있는 파라미터 타입도 늘어난다.  

### 2-1. 로컬 바인딩 서비스  

Stub을 구현할 필요가 없다.  

```java
public class LocalService extend Service {
    public final IBinder mBinder = new LocalBinder();

    public class LocalBinder extends Binder {
        public localService getService() {
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    public boolean vallidCalendar(long calendarId, CalendarType calendarType) {
        ...
    }
}
```  

### 2-2. 클라이언트에서 로컬 바인딩 접근  

```java
public class BindingActivity extends Activity {
    private LocalService mService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(Bundle savedInstanceState);
    }

    @Override
    protected void onStart() {
        super.onStart();
        Intent intent = new Intent(this, LocalService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        if (mService != null) {
            unbindService(mConnection);
        }
        super.onStop();
    }

    private ServiceConnection mConnection = new ServiceConnection() {
        
        @Override
        public void onServiceConnected(ComponentName className, IBinder service) {
            // LocalService 인스턴스를 직접 얻어낸다.
            LocalBinder binder = (LocalBinder) service;
            mService = binder.getService();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mService = null;
        }
    };

    public void checkValid() {
        if (mService != null) {
            boolean valid = mIRemoteService.isValidCalendar(10L, CalendarType.NORMAL.name());
            ...
        }
    }
}
```  

원래 안드로이드에서는 컴포넌트 간에 직접적으로 인스턴스 접근이 불가능하다. 그러나 `Binder` 객체를 통해서 직접 접근할 수 있게 되었고, `ServiceConnection` 구현체에서 이를 파라미터로 서비스 인스턴스를 얻었으니, 그 이후에는 서비스의 메서드를 직접 호출하는 방식으로 사용한다.  

### 2-3. 인터페이스를 사용한 로컬 바인딩  

위 처럼 `Binder`를 서비스 인스턴스로 직접 캐스팅하여 사용하는 것은 아무래도 억지로 끼워 맞춘 느낌이 있다. 리모트 바인딩 서비스의 기본 형태와 비슷한 형태로 변경해보자.  

`IRemoteService.aidl` 파일은 그대로 `IRemoteService` 인터페이스를 만들고, `Stub` 구현과 유사하게 `LocalBinder`에서 `IRemoteService`를 구현한다.  

```java
public class LocalService extends Service {
    private final IBinder mBinder = new LocalBinder();

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    private class LocalBinder extends Binder implements IRemoteService {
        
        @Override
        public boolean validCalendar(long calendarId, CalendarType calendarType) {
            ...
        }
    }
}
```  

```java
public class BindingActivity extends Activity {
    private IRemoteService mService;

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName className, IBinder service) {
            //인터페이스로 캐스팅하여 사용 -> 조금 더 유연함
            mService = (IRemoteService) service;
        }

        @Override
        public void onServiceDisconnect(ComponentName name) {
            mService = null;
        }
    }
}
```

<br>

## 3. 바인딩의 특성

`bindService()`를 호출하면 서비스와 엮이는 클라이언트가 하나씩 늘어난다. 이렇게 엮인 클라이언트가 남아있다면 어느 클라이언트에서 `stopService()`를 실행해도 서비스는 종료되지 않는다. 모든 클라이언트가 `unbindService()` 메서드를 호출해서 서비스와의 관계가 전부 정리되어야만 한다.  

<img src="https://developer.android.com/images/fundamentals/service_binding_tree_lifecycle.png?hl=ko"/>


### 3-1. bindService/unbindService 호출 시기  

클라이언트가 모두 `unbindService()`를 호출한다면 서비스에서는 `onUnbind()`부터 `onDestroy()`까지 호출된다. 액티비티에서 바운드 서비스를 사용할 때는 `onStart()`와 `onStop()` 에서 각각 `bindService()`와 `unbindService()`를 호출하는 것을 권장한다. `unbindService()`가 실행될 때 연결되어 있는 클라이언트 개수가 0이 되면 서비스는 종료될 수 있다.  

`onResume()`/`onPause()` 는 보다 빈번하게 트렌지션이 발생한다. 투명/다이얼로그 테마 액티비티가 뜨기만 해도 호출된다. 서비스를 생성하려면 비용이 드는데 빈번한 트랜지션에서 생성과 종료를 반복하는 것은 적합하지 않다.  

### 3-2. 대안: 콘텐트 프로바이더

서비스와 바인딩되었으면 클라이언트-서버 관계로 맺어진다. 클라이언트에서 메서드를 호출하면 서버가 결과를 호출하는 방식이다. 그런데 "조회해서 결과를 리턴한다" 라는 시나리오는 콘텐트 프로바이더와 비슷하다. 로컬 바인딩은 타입을 자유롭게 쓸 수 있으므로 크게 고민하지 않아도 되겠지만, 리모트 바인딩의 경우 단순 데이터 조회라면 콘텐트 프로바이더를 대신 쓸 수도 있다. 다만 콘텐트 프로바이더에서는 `Cursor` 타입으로 리턴해야 한다.  

```java
@Override
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    ...
    switch (URI_MATCHER.match(uri)) {
        ...
        case VALID_CALDNER:
            return validCalendar(Long.parseLong(selectionArgs[0]), CalendarType.valueOf(selectionArgs[1]));
    }
    throw new IllegalArgumentException("Unknown URI " + uri.toString());
}

private Cursor validCalendar(long calendarId, CalendarType calendarType) {
    Calendar calendar = null;
    if (calendarType == CalendarType.NORMAL) {
        ...
    } else {
        ...
    }
    MatrixCursor validCalendar(long calendarId, CalendarType calendarType) {
        cursor.newRow().add("valid");
    }
    return cursor;
}
```  

클라이언트에서는 다음과 같이 사용한다.  

```java
public boolean isValidCalendar(long calendarId, CalendarType type) {
    Cursor cursor = contentResolver.query(
        Uri.withAppendedPath(CONTENT_URI, "valid_calendar"),
        null, null,
        new String[] {String.valueOf(calendarId), type.name()}, null
    );
    return (cursor.getCount() > 0);
}
```  

### 3-3. 바운드 서비스의 작업 결과 리턴   

바운드 서비스에서도 작업이 오래 걸린다면 백그라운드 스레드에서 작업을 실행하는 것을 고려할 수 있다. 이때도 결과를 전달받으려면 4가지 방법을 고려할 수 있다.  

- `sendBroadcast()`를 통해 데이터를 전달하거나, 다시 클라이언트가 폴링해서 데이터를 가져오는 방법  
- 바인더 콜백을 메서드에 파리미터로 전달해서 결과를 받는 방법. 안드로이드 프레임워크에서 많이 쓰이며 `Toast.show()` 에도 사용된다. 안드로이드 컴포넌트의 생명주기도 system_server 프로세스의 `ActivityManagerService`에 전달된 바인더 콜백을 통해서 호출된다.  
- `bindService()` 메서드의 `Intent` 파라미터에 `ResultReceiver`를 전달하면, 서비스의 `onBind()`에서 `ResultReceiver`를 가져올 수 있다. 작업을 실행하고 `ResultReceiver`의 `send()` 메서드로 클라이언트에 결과를 다시 전달한다.  
- `Messanger`를 사용해서 양방향 통신을 할 수 있다. `Messanger`도 내부적으로 파인더 콜백을 사용한다. `Messanger`에 대해서는 별도로 포스팅할 계획이다.


ㅡ






<br>
<br>


--- 
해당 포스팅은 [안드로이드 프로그래밍 Next Step - 노재춘 저](http://www.yes24.com/Product/Goods/41085242) 을 바탕으로 내용을 보충하여 작성되었습니다.

참고:  
[바인드된 서비스 개요](https://developer.android.com/guide/components/bound-services)  
[안드로이드 인터페이스 정의 언어(AIDL)](https://developer.android.com/guide/components/aidl)