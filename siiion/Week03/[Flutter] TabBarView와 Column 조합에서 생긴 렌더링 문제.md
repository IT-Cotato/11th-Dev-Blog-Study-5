## 문제 발생 배경

Flutter로 진행 중인 프로젝트에서, 커뮤니티 페이지를 구현하게 되었다.
커뮤니티 탭은 두 가지 종류로 나뉘어져 있었고, 각 탭에 대한 게시글 목록을 화면에 띄워야 하는 구조였다.

Flutter에서 제공하는 `TabBar`와 `TabBarView`, `DefaultTabController`를 사용해 탭 기능을 구현하고, 각 탭에 `ListView.builder()`를 넣어 다음과 같이 게시글 목록에 대한 기본 구조를 완성했다.

```
Column(
  children: [
    const HomeAppBar(),
    const TabBar(...),
    TabBarView(
      children: [
        ListView(...), // 게시글 리스트
        ListView(...),
      ],
    ),
  ],
);
```

그런데 이처럼 구현하자 각 탭의 콘텐츠가 보이지 않는 문제가 발생하였고, 다음과 같은 에러 메시지를 마주하게 되었다.

![에러 메시지](https://velog.velcdn.com/images/siiion/post/38691ff3-7816-4e94-ab57-a047d9da6b54/image.png)

로그를 찍어 확인해 본 결과 데이터는 잘 들어오고 있었고, `ListView.builder()`도 다른 페이지에서는 정상적으로 동작하였기 때문에 `TabBarView`에 문제가 있는 것이라는 생각이 들었는데, 찾아보니 `TabBarView`는 **자신의 부모 위젯으로부터 명확한 높이 정보가 전달되지 않으면 렌더링할 수 없다**는 특징이 있었다.

### 문제 정리

> `Column` 내부에 `TabBarView`를 직접 넣으면, `Column`은 `TabBarView`에게 알아서 크기를 정하라고 하기 때문에, `TabBarView`는 **무한한 높이**를 가지려 하고, 이로 인해 자식 `ListView`의 렌더링에 실패하게 된 것이다.

이 문제를 해결하기 위해, 그리고 왜 이런 현상이 발생하는지를 이해하기 위해 Flutter의 레이아웃 시스템과 위젯 간 제약 관계를 더 깊이 들여다보게 되었다.

## 해결 과정

### 1차 시도: `ListView`에 `shrinkWrap: true` 적용

처음에는 `ListView`가 내부에서 무한 높이를 요구해서 그런 건 아닐까 싶어 `shrinkWrap: true`를 적용해보았다.

```
TabBarView(
  children: [
    ListView.builder(
      shrinkWrap: true,
      itemCount: 20,
      itemBuilder: (context, index) => ListTile(title: Text('글 $index')),
    ),
    ...
  ],
)
```

이 방법은 일반적으로 `ListView`가 `Column` 안에 있을 때 사용하는 방식이지만, `TabBarView`의 렌더링 문제와는 관련이 없었다.

### 2차 시도: `SizedBox`로 높이를 강제로 지정

`TabBarView`에 직접 높이를 주는 방식도 시도해 보았다.
그러나 이 방법은 화면 전체를 유동적으로 사용할 수 없었고, 기기 화면 크기에 따라 레이아웃이 깨지거나 여백이 생기는 문제가 발생하였다.

### 최종 해결: `Expanded`로 `TabBarView` 감싸기

`Expanded` 위젯은 **남은 모든 공간을 자식에게 할당해주는 역할**을 한다.
이를 활용하여 `Column`이 자식 위젯에게 정확한 높이 정보를 주지 않는 문제를 해결할 수 있었다.

최종적으로 변경된 구조는 대략적으로 다음과 같다.

```
body: Column(
  children: [
    const SizedBox(height: 16),
    const Text('오늘의 경제톡톡', style: TextStyle(fontSize: 18)),
    const SizedBox(height: 16),
    Expanded( // 핵심!
      child: TabBarView(
        children: [
          ListView.builder(
            itemCount: 20,
            itemBuilder: (context, index) =>
                ListTile(title: Text('일반 글 $index')),
          ),
          ListView.builder(
            itemCount: 20,
            itemBuilder: (context, index) =>
                ListTile(title: Text('경제톡 $index')),
          ),
        ],
      ),
    ),
  ],
),
```

위처럼 코드를 수정하니 탭 콘텐츠가 정상적으로 렌더링되었다.

## 해결 과정에서 알게 된 위젯 관련 개념들

### 1. `Column`은 자식 위젯에게 명확한 크기를 주지 않는다

- `Column`은 세로로 위젯을 쌓는 데 사용되지만, 자식 위젯이 얼마나 커야 할지 직접 지정하지 않는다.
- 자식이 스스로 크기를 결정해야 하며, 자식이 **무한 크기(infinite height)**를 요구할 경우 `RenderBox was not laid out` 에러가 발생한다.
- 특히 `TabBarView`, `ListView`, `SingleChildScrollView` 같은 스크롤 가능한 위젯은 **부모가 크기를 정해줘야 정상 렌더링** 된다.

> `Column`안에 스크롤 가능한 위젯이 있다면 `Expanded` 또는 `SizedBox(height: ...)`를 반드시 사용하자.

### 2. `TabBarView`는 고정된 높이가 있어야 작동한다.

- `TabBarView`는 내부적으로 `PageView`를 기반으로 한다. 따라서 자신이 어느 정도 공간을 가져야 할 지 반드시 알아야 한다.
- 그렇지 않으면 자식들을 배치할 수 없어 전체 탭 콘텐츠가 렌더링되지 않는다.
- 높이를 명확히 하려면 `Expanded` 또는 `SizedBox`로 감싸야 한다.

### 3. `Expanded`는 남은 공간을 자식에게 할당하는 역할

- `Expanded`는 `Column`, `Row`, `Flex`의 자식으로 사용될 때, 해당 축의 **남은 모든 공간을 자식에게 할당**한다.
- `Expanded` 안에 `TabBarView` 또는 `ListView`를 넣으면, 이들은 정확한 크기를 가지고 안전하게 렌더링된다.

### 4. `ListView`는 무한 높이를 요구하는 위젯

- `ListView`는 기본적으로 **스크롤 가능한 전체 콘텐츠**를 렌더링하므로, 부모 위젯으로부터 "어디까지 렌더링할 수 있는지" 명확히 제약을 받아야 한다.
- `ListView`는 다음 중 하나로 감싸야 한다:
  - `Expanded`
    - `SizedBox(height: ...)`
    - `Flexible`
    - `shrinkWrap: true` (단, 성능 부담 있음)

---

Flutter는 위젯 레이아웃 구조로 손쉽게 위젯을 쌓아 UI를 구성할 수 있지만, 위젯 종류가 많은 만큼 여러 위젯들을 혼합하여 사용하다 보면 어디서 발생한지 모를 에러들을 종종 발견하게 되는 것 같다.
그런 만큼, 앞으로는 처음부터 각 위젯의 역할과 위치를 고려해 적절한 용도로 사용하는 습관을 들여야겠다는 생각을 하게 되었다.
