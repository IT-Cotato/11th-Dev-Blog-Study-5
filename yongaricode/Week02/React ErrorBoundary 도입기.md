# React Errorboundary 도입기

최근 진행한 프로젝트에서 에러 처리를 적용해야 할 상황이 있었다.

너무 막막해서 미뤄두고 있던 에러 처리를, React의 ErrorBoundary라는 개념을 알게 되며 시도해보게 되었다.

결론부터 말하자면... **ErrorBoundary는 사용할 수 없었다.**

그래서 이번 글에서는 React ErrorBoundary가 어떤 것인지 간단히 정리하고, 내 프로젝트에서는 왜 사용할 수 없었는지 그 이유를 공유하려고 한다!

## React ErrorBoundary란?

React에서는 컴포넌트를 렌더링하는 동안 예기치 않은 오류가 발생하면 해당 컴포넌트의 트리 전체가 제거된다.

이때, 에러 처리가 없다면 사용자는 아무런 피드백 없이 깨진 화면을 마주하게 된다.

이 문제를 해결하기 위해 등장한 것이 바로 ErrorBoundary다!

ErrorBoundary는 렌더링 중 발생한 오류를 잡아주고, 그 대신 fallback UI를 보여주는 React의 특수한 컴포넌트다.

예를 들어 컴포넌트 안에서 undefined 값을 렌더링하거나 예외가 발생했을 때, 전체 앱이 죽는 것을 방지하고 대신 **예비 UI(fallback)** 를 보여줄 수 있게 해준다.

<p align="center">
  <img src="https://blog.kakaocdn.net/dn/bfEigM/btsNMd4cOPK/KgsBbYPd41giCkYwVcM4kk/img.png" width="45%" />
  <img src="https://blog.kakaocdn.net/dn/RUH7y/btsNLC4p8p0/XESrc5yx19u8tSizrltPzk/img.png" width="45%" />
</p>

오른쪽 에러 처리를 안했을 때 사용자가 보는 화면이고, 왼쪽은 에러처리를 했을 때 사용자가 볼 수 있는 화면이다.

그러니까 왼쪽 화면이 바로 Fallback UI다!

이렇게 보니까 차이가 크게 느껴지지 않나요?

## ErrorBoundary 사용

ErrorBoundary는 현재는 클래스 컴포넌트로만 구현 가능하며, 아래 두 메서드 중 하나 이상을 정의해야 한다:

- static getDerivedStateFromError(error)→ 다음 렌더링에서 fallBack UI가 보이도록 상태(state)를 업데이트한다.
- componentDidCatch(error, errorInfo)→ 에러 정보를 로깅하는 데 사용된다.

클래스를 직접 작성하기 싫다면 [react-error-boundary](https://github.com/bvaughn/react-error-boundary)를 사용하면 된다.

이렇게 클래스를 작성했으면 Error Boundary를 적용하고 싶은 컴포넌트를 감싸주면 된다.

```jsx
import { ErrorBoundary } from "./components/common/ErrorBoundary";
import MyPage from "./components/MyPage";
import Home from "./components/Home";

const App = () => {
  return (
    <div>
      <Home />
      <ErrorBoundary>
        <MyPage />
      </ErrorBoundary>
    </div>
  );
};
```

이렇게 감싸면, MyPage에서 발생한 오류만 잡을 수 있는 것이다. Home에서 발생한 오류는 잡지 못한다.

```jsx
// App.tsx 또는 Layout.tsx
<ErrorBoundary>
  <RouterProvider router={router} />
</ErrorBoundary>
```

이렇게 하면 Router 전체에서 렌더링 중 발생한 에러를 감지하고 fallback UI로 대체할 수 있다.

### ErrorBoundary의 한계

하지만 ErrorBoundary도 모든 에러를 잡진 못한다...

ErrorBoundary는 렌더링 도중 생명주기 메서드 및 그 아래에 있는 전체 트리에서 에러만 잡아낼 수 있다.

따라서 다음과 같은 에러는 포착하지 못한다.

### 1. 이벤트 핸들러

이벤트 핸들러는 **React의 렌더링 흐름 바깥**이기 때문에 Error Boundary가 감지할 수 없다.

```jsx
const MyComponent = () => {
  const handleClick = () => {
    throw new Error('버튼 클릭 중 오류 발생!');
  };

  return <button onClick={handleClick}>클릭</button>;
};

// 아래처럼 ErrorBoundary로 감싸도 위 에러는 못 잡음<ErrorBoundary>
  <MyComponent />
</ErrorBoundary>;
```

이렇게 감싸도, onClick(이벤트 핸들러)에서 발생한 에러는 잡지 못한다. 이걸 해결하기 위해서는 이벤트 핸들러 안에서 try/catch를 직접 사용해야 한다!

### 2. 비동기적 코드: setTimeout, fetch, async/await 등

이 역시 렌더링 흐름과 별개이기 때문에 감지되지 않는다.

```jsx
useEffect(() => {
  setTimeout(() => {
    throw new Error("비동기 타이머 오류!");
  }, 1000);
}, []);
```

에러를 잡기 위해서는 비동기 함수 안에서도 try/catch로 감싸야 한다.

### 3. 서버 사이드 렌더링

Error Boundary는 **클라이언트에서만 작동하기 때문에** 서버에서의 렌더링 도중 발생한 에러는 Error Boundary에 의해 감지되지 않는다.

즉, Next.js 같은 SSR 환경에서 발생한 에러는 ErrorBoundary로 못 잡는다!

따라서 SSR 환경에서는 서버 측에서 직접 에러 핸들링 로직을 구현해야 한다. (예: getServerSideProps에서 try/catch)

### 4. 자식에서가 아닌 에러 경계 자체에서 발생하는 에러

Error Boundary 컴포넌트 자체 내부에서 발생한 에러는 자신이 감지하지 못하기 때문에, 그 에러는 더 바깥의 다른 ErrorBoundary가 잡아야 한다.

```jsx
<ErrorBoundary2>
  <ErrorBoundary>
    {/* 내부에서 오류 발생 */}
    <Component />
  </ErrorBoundary>
</ErrorBoundary2>
```

ErrorBoundary가 실패할 수도 있으므로 최상단에 하나의 ErrorBoundary를 더 둘 수도 있다.

## ErrorBoundary는 어디서 사용해야할까?

- **앱 전체에 한 번**: 최상단(App.tsx 또는 Layout)에 ErrorBoundary를 추가해서 앱 전체의 예기치 않은 렌더링 에러를 막을 수 있다.
- **중요한 UI 단위별로 나누기**: 예를 들어 게시판, 프로필 카드, 지도 컴포넌트 등 외부 API나 복잡한 로직이 있는 UI는 별도로 ErrorBoundary로 감싸는 것이 좋다. 이렇게 하면 특정 컴포넌트에서 에러가 나더라도 전체 화면이 깨지는 것을 막을 수 있다.

## 네트워크 요청에서 ErrorBoundary를 사용하고 싶다면?

내가 ErrorBoundary를 실제 프로젝트에 도입하지 못했던 이유는, 앞서 말한 제한 사항들 때문이었다.

특히 나는 백엔드 API와 통신 중 발생한 네트워크 오류를 잡고 싶었지만, ErrorBoundary는 렌더링 중 발생하는 오류에만 대응할 수 있었기 때문에 적합하지 않았다.

그런데 에러 핸들링을 할 때 ErrorBoundary를 많이들 쓰고 있다고 해서, 제대로 알아보지 않고 도입했다가 결국 원하는 방식대로 동작하지 않아 사용을 포기하게 되었다...(반성합니다)

**그런데… 왜 다들 ErrorBoundary를 사용하고 있었을까?**

알고 보니 많은 개발자들이 **React Query**를 함께 사용하고 있었던 것이다!

React Query는 다음과 같은 특징을 갖고 있다:

- 내부적으로 네트워크 요청 오류를 잡아서 컴포넌트에서 throw한다.
- ErrorBoundary와 호환되도록 설계되어 있다.
- QueryClient에서 throwOnError 옵션을 true로 설정하면, 발생한 에러가 다음 렌더링 사이클에서 throw되어 ErrorBoundary가 감지할 수 있게 된다.

즉, 많은 블로그나 튜토리얼에서 "ErrorBoundary로 네트워크 오류를 처리할 수 있다"고 말하는 이유는,

**React Query와 함께 사용할 때만 가능한 시나리오**였던 것이다...

## 나의 대안...

그래서 나는 그냥 **axios interceptor에서 에러를 처리**하기로 했다!

이 [블로그 글](https://velog.io/@bokjunwoo/Axios-%EC%9D%B8%ED%84%B0%EC%85%89%ED%84%B0%EC%97%90%EC%84%9C-%EC%97%90%EB%9F%AC-%ED%95%B8%EB%93%A4%EB%A7%81%ED%95%98%EA%B8%B0)을 참고해서 에러 핸들링 구조를 잡았다.

처음엔 401 관련 오류만 interceptor에서 처리하고 있었는데, 생각해보니 굳이 try-catch를 매번 쓰느니 interceptor에서 다 처리해도 되겠다 싶었다. 그래서 지금은 대부분의 에러 처리를 interceptor에 몰아서 하고 있다.

그리고 나는 공통 API 요청 함수를 따로 만들어 사용하고 있다.

getResponseData라는 함수인데, 이 함수 내부에는 이미 try-catch가 있기 때문에, "에러 처리를 interceptor 대신 여기서 해도 되지 않을까?" 하는 고민이 들었다.

어차피 모든 요청이 이 함수를 거치니까, 여기서 에러를 통합적으로 처리해도 전역적인 대응이 가능하다고 생각했기 때문이다.

### 🤔 getResponseData vs. Axios Interceptor

| 구분                  | 장점                             | 장점                                                                            |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------- |
| **getResponseData**   | API 단위의 UI 분기 처리에 유리함 | 요청별로 에러 메시지를 다르게 보여주거나, 사용자 피드백을 조절해야 할 때 유용함 |
| **Axios Interceptor** | 전역 공통 처리에 유리함          | 토큰 만료 등 모든 요청에 공통 적용되는 처리를 반복 없이 설정 가능               |

나는 POST 요청일 때는 toast를 띄우고, GET 요청일 때는 fallback UI 하나만 보여주면 되었기 때문에,

개별 UI 처리 로직이 필요하지 않았고, 에러 처리는 전부 인터셉터에서 처리하기로 결정했다.

getResponseData 함수에서는 에러가 발생해도 별도의 처리는 하지 않고, 단순히 throw만 하도록 두고 있다.

하지만 이후 백엔드에서 커스텀 에러 코드를 내려주고, 요청별로 세부적인 UI 분기가 필요해진다면, 그때는 인터셉터 + getResponseData 조합으로 나눠서 처리하는 방식을 고려할 계획이다!

하지만... 검색했을 때 글이 많이 나오지 않는 걸로 봐서... 남들이 안하는데엔 이유가 있으려나? 하는 생각도 있다... 아마도 이 방식은 **네트**워크 요청 중 발생하는 에러에만 대응할 수 있어서 그런 것 같다... 나도 다음엔 react query를 사용해...  React Error boundary를 사용해보고 싶다...
