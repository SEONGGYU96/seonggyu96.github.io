---
layout: post
title: "안드로이드 앱 설정 화면 구성하기"
subtitle: "SettingView"
author: "winter"
header-style: text
tags:
  - Android
  - CustomView
  - SantaManitto
---

## 설정 화면 ?

<img width="300" src="https://user-images.githubusercontent.com/57310034/124144862-4cfe1880-dac7-11eb-883b-8b11bfd0ace0.png"/>

이번 "산타마니또" 앱의 새로운 버전에는 위와 같은 설정 화면이 추가되었다. 유저들의 피드백을 반영하려 닉네임 수정 기능을 추가하려는데 적절한 위치가 없었기 때문이다. 마침 회원가입 플로우에서만 접근할 수 있었던 "이용약관 동의"와 "개인정보 수집 및 이용 동의" 열람을 항상 가능하도록 구현하는 게 좋다고 생각하던 참이었다. 기존에는 SUID를 이용하여 회원가입을 하고 로그아웃 기능을 지원하지 않기 때문에 해당 내용은 최초 로그인 1회 이후에는 불가능했기 때문이다.

## 계획

보기에는 굉장히 단순한 뷰라서, 이를 어떻게 구현할 것이냐 물어보면 가장 먼저 떠오르는 것은 아마 RecyclerView일 것이다. 그게 아니면 항목의 개수가 고작 3개이기 때문에 그냥 이대로 UI 컴포넌트를 배치해도 크게 문제가 되지 않는다.

그러나 나는 다음과 같이 몇 가지를 고려해 구현하고자 했다.

1. 앞으로 항목이 더 추가되거나 제거될 수 있는 가능성
2. 각 항목을 선택했을 때 실행할 로직의 다형성 보장
3. 화면의 크기를 넘어갈 경우를 대비한 스크롤 기능 탑재

1번에 대응하기 위해서는 항목의 목록을 조작하기 쉬운 인터페이스가 필요하다. `addSetting()`과 같은 메서드가 있으면 좋겠다는 생각을 했다.  
2번을 보장하기 위해서는 "항목"을 일반화하는 모델이 필요하다. 각각의 모델은 스스로의 클릭 listener를 가지고 있어야겠다.
3번을 대응하기 위해서는 ScrollingView 인터페이스를 구현하는 View를 기반으로 구현해야할 것이다. 그 중에선 RecyclerView가 가장 익숙하다.

결론은, <b>RecyclerView를 확장하는 SettingViewList를 만들고, `addSetting()` 인터페이스를 제공하도록 하자!</b>

## 구현

```kotlin
class SettingListView @JvmOverloads constructor(
        context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : RecyclerView(context, attrs, defStyleAttr) {

    ...

    private val adapter = SettingListViewAdapter()

    init {
        itemAnimator = null //불필요한 애니메이션 제거
        setHasFixedSize(true) //사이즈 고정
        overScrollMode = OVER_SCROLL_NEVER //스크롤 시 위아래 번지는 효과 제거
        layoutManager = LinearLayoutManager(context)
        setAdapter(adapter)
    }

    //항목 추가 API
    fun addSetting(title: String, listener: () -> Unit): SettingListView {
        adapter.addSetting(Setting(title, listener))
        return this //메서드 체이닝을 사용하기 위함
    }

    //커밋
    fun commit() {
        adapter.notifyDataSetChanged()
    }
    ...

    //항목을 일반화하는 모델
    private data class Setting(
        val title: String,
        val listener: () -> Unit
    )
}
```

계획한대로 만들어 본 SettingListView.kt 클래스의 일부이다. 항목들은 최초 1회를 제외하고는 동적으로 추가되거나 삭제될 일이 없기 때문에 사이즈를 고정하고, 그에 따른 애니메이션 효과도 모두 제거했다. 

### addSetting()

항목을 추가할 수 있는 API인 `addSetting()`은 `title`과 `listener`를 파라미터로 가진다. 이는 구현부에서 Setting 모델을 생성하는데 사용된다. 처음부터 `fun addSetting(setting: Setting)`으로 만들 수 있으나 굳이 파라미터로 필드를 쪼갠 이유는 간단하다. SettingListView를 사용하는 클라이언트가 항목을 추가하기 위해 알아야하는 정보를 하나라도 줄이기 위해서이다. 만약 Setting을 파라미터로 받도록 구현한다면 개발자는 Setting 클래스를 생성하기 위해 어떤 파라미터가 필요한지 도큐먼트를 찾아봐야한다. 이 단계를 생략시키기 위해 필드가 2개 뿐이니 파라미터로 쪼개는 선택을 한 것이다.  

#### 메서드 체이닝

`addSetting()`은 반환값이 있다. 바로 SettingListView 바로 자신이다. 자바로 개발할 때 남은 습관인데, 이렇게 연속으로 메서드를 호출할 경우에 메서드가 자신이 포함된 클래스 인스턴스(this)를 반환하게 되면 메서드를 연결해서 쭉 호출할 수 있다. 사실 코틀린을 사용하고있다면 이런 경우에는 apply, run 등과 같은 스코프를 사용하여 같은 효과를 낼 수 있으나 습관적으로 사용했다.

### commit()

그리고 계획에는 없던 `commit()` 메서드도 추가했다. 항목을 추가했다면 이를 어댑터에게도 알려줘야하니, `addSetting()` 구현부의 마지막에 `notifyDataSetChanged()`와 같은 notify 메서드를 호출해야할 것이다. 그러나 항목은 최소 2개 이상일 가능성이 높으므로 `addSetting()`메서드 또한 2번 이상 호출될 것인데, 매번 갱신을 요청할 필요가 있을까? 항목을 모두 추가하고 난 후에 한 번만 갱신을 요청하면 좋겠다는 생각을 했다. 항목 추가 작업을 "다 했다고 선언하다" 라는 단어가 뭐가 있을까 하니 commit이 떠올라 그렇게 이름을 붙여주었다.  

물론 `notifyXXXChanged()` API는 호출될 때 마다 곧이곧대로 반영되지 않는다. 해당 어댑터를 사용하고 있는 RecyclerView에게 데이터가 변경되었음을 알려주고, RecyclerView는 `View#requestLayout()`을 호출함으로써 메인 스레드에 레이아웃 갱신 요청을 스케줄링한다. 이는 꽤 똑똑한 알고리즘을 가지고 있어서 아직 처리되지 않은 갱신 메시지가 Queue에 남아 있다면 중복해서 메시지를 push하지 않는식으로 동작한다. 이 내용은 이전에 정리해둔 바 있으니 [여기서](https://seonggyu96.github.io/2021/01/11/ui_mechanism/#invalidate) 확인할 수 있다.

아무튼 이런 이유로 `commit()` 메서드를 통해 눈에 띄는 성능 향상을 기대할 수는 없으나 언젠가는 작은 노력들이 쌓여 좋은 결과를 만들 수 있을 수도 있을 것이라 생각하며 구현했다.  

### Adapter와 ViewHolder

```kotlin
...
    private class SettingListViewAdapter : RecyclerView.Adapter<SettingListViewHolder>() {

        private val settingList = mutableListOf<Setting>()

        fun addSetting(setting: Setting) {
            settingList.add(setting)
        }

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): SettingListViewHolder {
            return SettingListViewHolder(parent)
        }

        override fun onBindViewHolder(holder: SettingListViewHolder, position: Int) {
            //ViewHolder의 초기화 메서드 호출
            holder.onBind(settingList[position].title, settingList[position].listener)
        }

        override fun getItemCount(): Int {
            return settingList.size
        }
    }

    private class SettingListViewHolder(parent: ViewGroup) : RecyclerView.ViewHolder(LayoutInflater.from(parent.context)
            .inflate(VIEW_HOLDER_RES, parent, false)) {

        private val binding = ItemSettingBinding.bind(itemView)

        fun onBind(title: String, listener: () -> Unit) {
            binding.run {
                //클릭 리스너 설정
                root.setOnClickListener { listener.invoke() }
                textviewSettingitemTitle.text = title // 설정 타이틀 수정
            }
        }
    }
...
```

어댑터와 뷰홀더는 위와 같이 특별한 태크닉 없이 구현했다.

## 사용  

```kotlin
class SettingFragment : BaseFragment<FragmentSettingBinding>(R.layout.fragment_setting, false) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {

        binding.settinglistviewSetting
                    //설정 항목들을 연속적으로 추가
                    .addSetting("닉네임 수정") {
                        findNavController().navigate(
                                SettingFragmentDirections.actionSettingFragmentToEditNameFragment()
                        )
                    }
                    .addSetting("이용약관 동의") {
                        goToWebViewFragment(BuildConfig.TOS_URL)
                    }
                    .addSetting("개인정보 수집 및 이용 동의") {
                        goToWebViewFragment(BuildConfig.PRIVACY_POLICY_RUL)
                    }
                    //완료 후 커밋 -> NotifyDataSetChanged()
                    .commit()
        ...
    }
    ...
}
```

결과적으로 SettingListView은 위와 같이 사용되었다. 설정 화면을 구성하기 위한 모든 설정과 구현부는 커스텀뷰 안에 캡슐화되었고, 이를 사용할 클라이언트에서는 `addSetting()` 및 `commit()` API만 사용하면 된다.  

## 한계  

만약 설정 항목에 따라 다른 형태의 뷰홀더를 가져야하면 어떻게 해야할까?

<img width="300" src="https://user-images.githubusercontent.com/57310034/123569961-4a1ed180-d802-11eb-98f6-404a0fac5e54.png"/>

21년 1월 쯤 만들었던 [ECGMonitor](https://github.com/SEONGGYU96/ECGMonitorWithNRF)의 설정 화면이다. [PreferenceFragmentCompat](https://developer.android.com/guide/topics/ui/settings)을 사용하였다. 설정 항목에 각기 다른 아이콘이 들어가고 메인 타이틀과 서브타이틀이 존재하는 경우도 있다. switch와 같은 추가적인 UI 컴포넌트가 포함되기도 하며 섹션이 나누어져있다.

아직까지 이런 경우들을 대응하지는 못한다. 하지만 만약 대응한다면 다음과 같은 방법을 사용할 것 같다.

- 섹션 대응 : Section이라는 모델을 만들고 `addSection()` API를 공개
- 아이콘 대응 : `addSetting()`에 아이콘 파라미터 추가 (enum 사용)
- 뷰홀더 형태 대응 : `addSetting()`을 다양한 형태로 오버로딩하거나 사용자 지정 레이아웃을 설정할 수 있는 `addCustomSetting(view: View)` API를 공개

직접 구현해보지 않아 위 방법이 최선인지는 모르겠다. 그러나 `PreferenceFragmentCompat`에서 제공하는 다양한 API들을 보면서 참고한다면 크게 고민하지 않고 내 앱에 맞는 커스텀 설정 뷰를 만들 수 있을 거라 생각한다.  

