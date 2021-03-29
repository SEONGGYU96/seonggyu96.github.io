---
layout: post
title: "안드로이드 : 바운드 서비스 - Messenger"
subtitle: "서비스와의 통신"
author: "GuGyu"
header-style: text
tags:
  - Android
  - AndroidFramework
  - Service
  - Background
  - BoundService
  - Messenger
---

[서비스 개요, 스타티드 서비스](https://seonggyu96.github.io/2021/03/24/started_service/)  
[바운드 서비스](https://seonggyu96.github.io/2021/03/27/bound_service/)

# Massanger

messenger는 바인더 콜백을 내부적으로 래핑해서 바운드 서비스와 클라이언트간에 Handler로 메시지를 보내고 처리하는 방식을 제공한다. 이 기법을 사용하면 AIDL을 사용하지 않고도 IPC가 가능하다. 또한 Handler를 사용하기 때문에 스레드에 안전하다.  

## 1. Messenger 클래스의 기본 내용  

- Messenger는 Parcelable 인터페이스를 구현해서 프로세스 간에 전달할 수 있는 객체이다.  
- Messenger에는 2개의 생성자가 있다. `Messenger(Handler target)`은 Handler를 감싼 것으로, 클라이언트와 바운드 서비스 양쪽에 있다. `Messenger(IBinder target)`은 Binder Proxy를 생성하는 것으로, 클라이언트에 있다.  
- aidl을 내부적으로 사용하는데, IMessenger 인터페이스로 되어 있다. 그렇다고 Messenger가 IMessenger.Stub을 구현하지는 않고 Handler의 내부 클래스인 MessengerImple에서 구현한다. Stub에서는 Handler의 `sendMessage()` 메서드를 호출하기만 한다.  
- Message 클래스에는 replyTo 라는 공개 변수가 있다. Handler에 Message를 보낼 때 replyTo에 값을 되돌려 줄 Messenger를 지정할 수 있다.  
- Messenger의 `send()` 메서드는 결과적으로 바인더 통신을 통해 Stub의 메서드를 호출한다. 리모트 통신이기 때문에 `send()` 메서드는 `RemoteException()`을 일으킬 수 있다.  

## 2. 날씨 정보 업데이트 구현  

날씨 정보를 업데이트하는 기능을 Messenger로 만들어보자. 요구사항은 다음과 같다.  

- 처음 액티비티에서 `bindService()`를 실행하면 바운드 서비스에서 날씨 정보를 가져오고 내부에서 주기적으로 날씨 정보를 가져온다.  
- 클라이언트가 추가로 바인딩되면 최신 날씨 정보를 클라이언트에 전달한다.  
- 날씨 정보를 가져올 때마다 바인딩된 모든 액티비티에 정보를 전달해서 화면에 보여준다.  
- 화면에는 새로고침 버튼이 있어서 이 버튼을 클릭하면 날씨 정보를 새로 가져오고 바인딩된 모든 클라이언트에 날씨 정보를 새로 전달한다.  

서버 코드를 보자  

```java
public class MessengerService extends Service {

    //클라이언트에서 들어오는 Message의 what에 해당하는 값들
    public static final int MSG_REGISTER_CLIENT = 1;
    public static final int MSG_UNREGISTER_CLIENT = 2;
    public static final int MSG_REFRESH = 3;

    //클라이언트에 보내는 Message의 what에 해당하는 값
    public static final int MSG_WEATHER = 11;

    public static final String WEATHER_TEXT = "weatherText";
    public static final String TEMPERATURE = "temperature";

    //스레드 개수를 1개만 생성
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    private ArrayList<Messenger> clients = new ArrayList<>();

    private Bundle lastData;

    //클라이언트에서 들어오는 Message를 처리하는 용도
    private class IncomingHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_REGISTER_CLIENT:
                    //replyTo에 전달되는 Messenger를 클라이언트 목록에 추가
                    clients.add(msg.replyTo);

                    //값이 있다면 기존 값 전달
                    if (lastData != null) {
                        Message message = Message.obtain(null, MSG_WEATHER);
                        message.setData(lastData);
                        try {
                            msg.replyTo.send(message);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                    
                case MSG_UNREGISTER_CLIENT:
                    //클라이언트 목록에서 제거
                    clients.remove(msg.replyTo);
                    break;
                    
                case MSG_REFRESH:
                    //기존 ScheduledFuture를 취소하고 업데이트
                    scheduledFuture.cancel(true);
                    fetchWeather();
                    break;
                    
                default:
                    super.handleMessage(msg);
            }
        }
    }
    
    //Handler를 감싼 Messenger를 생성
    private Messenger messenger = new Messenger(new IncomingHandler());

    private ScheduledFuture scheduledFuture;

    @Override
    public void onCreate() {
        super.onCreate();
        //서비스 생성 시 날씨 정보를 가져온다
        fetchWeather();
    }
    
    private void fetchWeather() {
        scheduledFuture = scheduler.scheduleAtFixedRate((Runnable) () -> {
            //날씨 정보를 가져와서 Bundle에 담고, 클라이언트 목록에 send()메서드로 전달
            Weather weather = callWeatherAPI();
            Message message = Message.obtain(null, MSG_WEATHER);
            Bundle bundle = message.getData();
            bundle.putString(WEATHER_TEXT, weather.weatherText);
            bundle.putInt(TEMPERATURE, weather.temparature);
            
            //최신 데이터 캐싱
            lastData = new Bundle(bundle);
            for (int i = clients.size() - 1; i >= 0; i--) {
                try {
                    clients.get(i).send(message);
                } catch (RemoteException e) {
                    //클라이언트가 종료된 상태라면 클라이언트 목록에서 제거
                    clients.remove(i);
                    e.printStackTrace();
                }
            }
        }, 0, 1, TimeUnit.MINUTES);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

다음은 클라이언트 코드이다.  

```java  
public class MessengerActivity extends Activity {

    // 클라이언트에 전달되는 Message를 처리하는 핸들러
    private class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MessengerService.MSG_WEATHER:
                    Bundle bundle = msg.getData();
                    weather.setText(bundle.getString(MessengerService.WEATHER_TEXT) +
                            ", Temperature : " + bundle.getInt(MessengerService.TEMPERATURE));
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    private Messenger outMessenger;

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 서비스가 연결되면 IBinder를 감싼 Messenger를 생성
            outMessenger = new Messenger(service);
            try {
                //replyTo에는 inMessenger를 등록한 후에 서비스에 등록하라는 Message를 보낸다
                Message msg = Message.obtain(null, MessengerService.MSG_REGISTER_CLIENT);
                msg.replyTo = inMessenger;
                outMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            Toast.makeText(MessengerActivity.this, "Connected", Toast.LENGTH_SHORT).show();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            outMessenger = null;
            Toast.makeText(MessengerActivity.this, "Disconnected", Toast.LENGTH_SHORT).show();
        }
    };

    // 핸들러를 감싼 Messenger 생성
    private Messenger inMessenger = new Messenger(new IncomingHandler());

    private TextView weather;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        weather = findViewById(R.id.title);
    }

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, MessengerService.class),
                serviceConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onStop() {
        if (outMessenger != null) {
            try {
                Message msg = Message.obtain(null, MessengerService.MSG_UNREGISTER_CLIENT);
                msg.replyTo = inMessenger;
                outMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(serviceConnection);
        super.onStop();
    }
    
    public void onClickRefresh(View view) {
        Toast.makeText(MessengerActivity.this, "Refresh.", Toast.LENGTH_SHORT).show();
        if (outMessenger != null) {
            try {
                //날씨 정보를 갱신하라는 Message를 보냄
                Message msg = Message.obtain(null, MessengerService.MSG_REFRESH);
                msg.replyTo = inMessenger;
                outMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```

이 샘플을 통해 바운드 서비스를 사용할 때의 장점을 한 가지 알 수 있다. 각 클라이언트에서 네트워크를 통한 API 호출을 매번 할 필요가 없이, 서비스 한곳에서만 네트워크 통신을 하고 모든 클라이언트에 결과를 반영할 수 있다. 바인더 콜백으로도 가능하지만 샘플과 같이 Messenger를 이용하면 더 단순해진다.

<br>
<br>


--- 
해당 포스팅은 [안드로이드 프로그래밍 Next Step - 노재춘 저](http://www.yes24.com/Product/Goods/41085242) 을 바탕으로 내용을 보충하여 작성되었습니다.

참고:  
[바인드된 서비스 개요](https://developer.android.com/guide/components/bound-services)  