프론트엔드 개발을 하다보면 사용자 환경에서 발생하는 버그를 실시간으로 파악하기 어려운 경우가 많은 것 같습니다. 특히 프로덕션 환경에서는 디버깅이 제한적이라 에러의 원인을 파악하기 더욱 어려웠습니다.

이번에는 저희 팀이 프로젝트에 Sentry를 도입한 과정과 경험을 공유해보려 합니다.

## 😢 "이거 무슨 에러인가요?"

저희 [Syncspot](https://syncspot.kr/) 프로젝트는 여러 사람이 모여 약속 시간과 장소를 결정하는 서비스입니다. 프로젝트가 성장함에 따라 다음과 같은 구체적인 문제점들이 발생했습니다.

### 🐛 `숨겨진 버그`

개발 환경에서는 발견되지 않지만 프로덕션 환경에서만 나타나는 에러들이 있었습니다. 예를 들어, Safari 브라우저에서만 발생하는 위치 정보 권한 관련 에러가 있었는데, 개발하는 과정에서 팀원 모두 Chrome을 사용하다 보니 발견하지 못했습니다. 또한 일부 사용자들이 약속 투표 페이지에서 투표 결과가 저장되지 않는 문제가 발생했지만, 이를 확인하지 못했습니다.

### 😭`에러에 대해 구체적인 상황 파악의 어려움`

한 사용자로부터 "친구 초대할 때 화면이 하얗게 변해요"라는 피드백을 받았는데, 어떤 단계에서 어떤 액션을 취했을 때 발생한 문제인지 정확히 파악하기 어려웠습니다. 사용자는 "그냥 버튼 눌렀는데 흰 화면만 나와요"라는 애매한 설명만 제공했기에 이를 파악하는 과정이 쉽지 않았습니다.

### 😓 `디버깅의 어려움`

한 사용자가 위치 검색 시 무한 로딩이 발생한다고 보고했지만, 어떤 검색어를 입력했는지, 어떤 네트워크 환경이었는지 알 수 없어 문제 해결하는데 오래걸렸습니다.

### 🚑 `API 에러 추적 부재`

사용자가 장소 투표 시 간헐적으로 "서버 오류" 메시지가 표시되는 현상이 있었습니다. 하지만 어떤 API 호출에서 어떤 에러가 발생했는지, 사용자의 네트워크 상태는 어땠는지 등의 상세 정보가 없어 문제 해결이 어려웠습니다.
실제로 한 사용자는 "방금 투표했는데 결과가 반영이 안됐어요"라고 문의했지만, 서버 로그에는 관련 에러가 없었고 개발하는 과정에서는 해당 문제를 재현할 수 없었습니다. (나중에 알고 보니 이 사용자는 네트워크가 불안정한 상태에서 여러 번 투표 버튼을 빠르게 클릭해서 API 요청이 충돌한 경우였습니다.)

## 🔍 Sentry 도입 결정: 왜 `Sentry` 였을까?

이러한 문제들을 해결하기 위해 몇 가지 옵션을 검토했습니다. 처음에는 콘솔 로깅을 강화하는 방안을 고려했지만, 프로덕션 환경에서는 사용자의 콘솔 로그를 직접 볼 수 없다는 한계가 있었습니다. 그다음으로 직접 에러 로깅 시스템을 구축하는 방안을 검토했으나, 이는 개발 리소스가 많이 필요하고 유지보수 부담도 컸습니다.

결국 전문 에러 모니터링 도구 도입으로 방향을 잡았고, Sentry, LogRocket, TrackJS 등을 비교 검토했습니다. 그 중에서도 Sentry가 눈에 띄었는데, 그 이유는 React와의 통합이 자연스럽고 ErrorBoundary 같은 React 특화 기능을 제공한다는 점이 매력적이었습니다. 또한 에러 발생 시 브라우저, OS, 사용자 세션 정보 등 다양한 컨텍스트를 자동으로 수집해주는 점과 세션 리플레이 기능을 통해 사용자의 행동을 추적할 수 있다는 점이 우리의 프로젝트에 필요성과 잘 부합한다고 생각하였고 Sentry는 무료 플랜을 제공해 소규모 프로젝트에서도 부담 없이 시작할 수 있다는 점과 활발한 커뮤니티와 문서가 잘 구성되어 있어 도입 과정에서 참고할 자료가 많다는 점이 좋았습니다. 따라서 이러한 이유들로 저희 팀은 최종적으로 `Sentry`를 선택하게 되었습니다.

## 🛠️ Sentry 구현하기

### 1️⃣ 기본 설정 및 초기화

Sentry를 설계할 때 가장 먼저 고려한 것은 프로덕션과 개발 환경을 분리하는 것이었습니다. 이를 위해 개발 중에는 Sentry가 활성화되지 않도록 하고, 프로덕션 환경에서만 작동하도록 구성했습니다.

`sentry.ts` 파일을 생성하여 Sentry 초기화 로직을 다음과 같이 구현했습니다.

`sentry.ts`

```javascript
import * as Sentry from '@sentry/react';

export const initSentry = () => {
  Sentry.init({
    dsn: import.meta.env.VITE_SENTRY_DSN,
    enabled: import.meta.env.PROD,
    sendDefaultPii: true,
    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration(),
    ],
    tracesSampleRate: 0.5,
    tracePropagationTargets: [
      /^\//,
      // 백엔드 api 서버, (혹시 몰라 주석으로 했습니다. 실제로는 백엔드 주소 적었어요!)
    ],
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,
    environment: import.meta.env.MODE,
  });
};
```

위의 핵심 설정 포인트들은 다음과 같습니다.

`dsn`: Sentry 프로젝트의 고유 식별자로, 환경 변수로 관리합니다.

`enabled`: 프로덕션 환경에서만 활성화하여 개발 중에는 불필요한 에러 로깅을 방지합니다.

`integrations`: 브라우저 추적과 세션 리플레이 기능을 활성화합니다.

`tracesSampleRate`: 성능 모니터링 샘플링 비율을 50%로 설정했습니다.

`replaysSessionSampleRate & replaysOnErrorSampleRate`: 일반 세션은 10%만 샘플링하고, 에러 발생 시에는 100% 리플레이를 캡처합니다.

위 초기화 코드는 애플리케이션 시작점인 `main.tsx`에서 호출하여 앱이 시작될 때 Sentry가 초기화되도록 했습니다.

### 2️⃣ 에러 경계 컴포넌트 구현

React의 렌더링 과정에서 발생하는 에러를 캡처하기 위해 Sentry의 ErrorBoundary를 활용한 컴포넌트를 구현했습니다. 해당 컴포넌트를 통해 에러가 발생했을 때 대체 UI를 보여주도록 하였습니다.

`SentryErrorBoundary.ts`

```javascript
import * as Sentry from '@sentry/react';
import { PropsWithChildren } from 'react';
import UnknownErrorFallback from './UnknownErrorFallback';

export const SentryErrorBoundary = ({ children }: PropsWithChildren) => {
  return (
    <Sentry.ErrorBoundary fallback={() => <UnknownErrorFallback />}>
      {children}
    </Sentry.ErrorBoundary>
  );
};
```

위 코드에서 ErrorBoundary를 앱의 최상위 레벨에 배치하여 렌더링 중 발생하는 모든 에러를 Sentry로 자동 보고하고, 사용자에게는 에러 메시지와 함께 홈으로 돌아갈 수 있는 UI를 제공했습니다. (이는 "흰 화면"만 보이는 문제를 해결하는 데 큰 도움이 되었습니다.)

### 3️⃣ React Router와 통합

사용자의 네비게이션 경로를 추적하기 위해 React Router v6와 Sentry를 통합했습니다.

```javascript
import * as Sentry from '@sentry/react';
import { createBrowserRouter } from 'react-router-dom';

export const sentryCreateBrowserRouter = (
  routes: Parameters<typeof createBrowserRouter>[0],
  opts?: Parameters<typeof createBrowserRouter>[1]
) => {
  return Sentry.wrapCreateBrowserRouterV6(createBrowserRouter)(routes, opts);
};
```

위의 함수를 통해 라우터를 생성하여 사용자가 페이지 간 이동할 때마다 Sentry가 자동으로 이 정보를 추적할 수 있었고, 이를 통해 "어떤 페이지에서 에러가 발생했는지"를 정확히 파악할 수 있게 되었습니다.

### 4️⃣ API 에러 캡처 유틸리티

다음으로, 백엔드 API 통신 과정에서 발생하는 에러를 효과적으로 추적하기 위한 유틸리티를 구현했습니다. 이전에는 API 에러가 발생해도 구체적인 정보를 알 수 없었지만, 아래의 유틸리티를 통해 어떤 API 호출에서 어떤 에러가 발생했는지 상세히 추적할 수 있게 되었습니다.

```javascript
export const captureApiError = (
  endpoint: string,
  error: ApiErrorResponse | AxiosError
) => {
  const context: Record<string, unknown> = {
    endpoint,
  };

  if (error instanceof AxiosError) {
    context.status = error.response?.status || 'unknown';
    context.statusText = error.response?.statusText || 'unknown';
    context.response = error.response?.data || {};
    context.config = error.config || {};
  } else {
    context.status = error.status || 'unknown';
    context.statusText = error.statusText || 'unknown';
    context.response = error.response || {};
    context.config = error.config || {};
  }

  Sentry.withScope((scope) => {
    scope.setLevel('error');
    scope.setFingerprint([`api-error-${endpoint}`]);
    scope.setContext('API Error', context);
    Sentry.captureException(error);
  });
};
```

위를 통해 에러에 대한 상세 정보와 컨텍스트를 수집하여 Sentry에 전송하였고 어떤 API 엔드포인트에서 에러가 발생했는지, 상태 코드는 무엇인지, 응답 데이터는 어떤지 등의 정보를 담아 더 효과적인 디버깅이 가능하도록 했습니다.

또한 이를 React Query의 onError 콜백에 통합하여 모든 API 에러를 자동으로 캡처하도록 설정했습니다.

```javascript
const queryClient = new QueryClient({
  defaultOptions: {
    mutations: {
      onError: (error: unknown) => {
        if (isAxiosError(error)) {
          captureApiError(error.config?.url || 'unknown', error);
          // 사용자에게 오류 알림 표시
        }
        // 기타 에러 처리
      },
    },
  },
});
```

### 5️⃣ 5. 사용자 컨텍스트 관리

에러가 발생했을 때 어떤 사용자에게 발생했는지 추적하기 위한 유틸리티도 구현했습니다. 이를 통해 특정 사용자나 사용자 그룹에서만 발생하는 문제를 파악할 수 있게 되었습니다.

```javascript
export const setSentryUser = (user: SentryUserInfo | null) => {
  if (user) {
    Sentry.setUser({
      id: user.id,
      email: user.email,
      username: user.username,
    });
  } else {
    Sentry.setUser(null);
  }
};

export const setSentryContext = (
  name: string,
  context: Record<string, ContextValue>
) => {
  Sentry.setContext(name, context);
};

export const setSentryTag = (key: string, value: string) => {
  Sentry.setTag(key, value);
};
```

로그인 성공 후 사용자 정보를 설정하고, 중요한 작업에서는 컨텍스트와 태그를 추가하여 해당 상황에서 발생한 에러를 더 잘 이해할 수 있도록 했습니다.

## 🎯 Sentry 도입 후 얻은 효과

### 👉🏻 실시간 에러 감지 및 알림

이전에는 사용자로부터 "화면이 하얘졌어요"라는 피드백을 받아도 언제, 어디서, 왜 그런 현상이 발생했는지 알 수 없었습니다. Sentry 도입 후에는 에러가 발생하는 즉시 알림을 받고, 어떤 컴포넌트에서 어떤 상황에서 에러가 발생했는지 구체적으로 파악할 수 있게 되었습니다. 실제로, 이전에 사용자가 보고한 "친구 초대 시 화면이 하얘지는 문제"는 Sentry 로그를 통해 특정 사용자가 이미 초대된 친구를 다시 초대하려 할 때 백엔드에서 409 Conflict 에러가 발생하고, 이를 제대로 처리하지 못해 렌더링 에러가 발생한다는 것을 파악할 수 있었습니다.

### 👉🏻 에러 우선순위 설정 용이

Sentry의 에러 빈도 및 영향 분석을 통해 더 중요한 문제에 집중할 수 있게 되었습니다. Sentry의 "User Impact" 기능을 통해 어떤 에러가 가장 많은 사용자에게 영향을 미치는지 파악할 수 있었습니다. 예를 들어, 소수의 Safari 사용자에게만 발생하는 CSS 에러보다 모든 모바일 사용자에게 영향을 미치는 투표 기능 버그를 우선적으로 수정하여 전체적인 사용자 경험을 빠르게 개선할 수 있었습니다.

### 👉🏻 디버깅 효율성 향상

이전에는 사용자가 "투표가 안 돼요"라고 보고해도 어떤 단계에서 어떤 이유로 문제가 발생했는지 파악하기 어려웠습니다. 이제는 Sentry의 브레드크럼과 세션 리플레이를 통해 사용자의 행동 흐름을 정확히 파악할 수 있게 되었습니다.
실제로 한 사용자가 무한 로딩 현상을 보고했을 때, Sentry 세션 리플레이를 통해 사용자가 위치 검색 시 특수문자가 포함된 검색어를 입력했고, 이 검색어가 백엔드에서 제대로 처리되지 않아 타임아웃이 발생했다는 것을 확인할 수 있었습니다.

## 👀 Sentry를 통한 실제 퍼포먼스 개선

사용자 투표 과정에서 간헐적으로 발생하던 에러를 통해 사용자 경험을 개선할 수 있었습니다.

실제 여러 사용자로부터 "투표했는데 반영이 안 돼요", "투표 중에 오류가 났어요"라는 피드백을 받았지만, 이러한 문제를 하나하나 재현하기 어려웠습니다.

하지만 Sentry를 도입한 이후에는 Sentry를 통해 수집된 데이터를 분석할 수 있었고 다음과 같은 패턴을 발견할 수 있었습니다.

- 에러는 사용자가 짧은 시간 내에 여러 투표 옵션을 빠르게 클릭할 때 발생한다.
- 백엔드에서는 동시에 발생한 여러 요청을 처리하는 과정에서 경쟁 상태가 발생한다.

이를 Sentry의 Performance Monitoring 기능을 활용하여 API 호출 타이밍과 지연 시간을 분석한 결과, 투표 API 호출이 짧은 시간 내에 여러 번 발생할 때 평균 응답 시간이 약1.5초에서 4초로 급증하는 것을 확인했습니다. 이 지연으로 인해 사용자들은 버튼을 여러 번 클릭하게 되고, 이는 더 많은 API 호출을 발생시켜 서버에 추가 부하를 주는 악순환이 발생한 것이었습니다.

> 이 문제를 해결하기 위해 두 가지 전략을 생각해봤습니다.

`프론트엔드 최적화`: 투표 버튼에 디바운싱을 적용하고, 투표 진행 중에는 버튼을 비활성화하여 중복 요청을 방지한다.

`백엔드 최적화`: 동일 사용자의 동일 옵션에 대한 중복 투표 요청을 서버 측에서 감지하여 처리한다.

위의 변경 사항을 적용한 후, Sentry 대시보드에서 투표 관련 에러가 91% 감소한 것을 확인할 수 있었습니다. (이 수치는 Sentry의 "Issues per session" 메트릭을 통해 측정했으며, 배포 전후를 비교하여 도출했습니다.)

또한 사용자 피드백에서도 "투표가 안 됨" 관련 문의가 이전보다 줄었습니다.
더불어, 투표 과정의 평균 완료 시간이 이전보다 줄게되어 전반적인 애플리케이션 성능도 크게 개선되었습니다.

## 🧐 결론 및 앞으로의 계획

Sentry 도입을 통해 저희 팀은 사용자 경험을 크게 개선할 수 있었습니다. 이전에는 사용자가 직접 문제를 보고해야만 알 수 있었던 숨겨진 에러들을 실시간으로 탐지하고, 발생 원인을 파악하며, 효율적으로 해결할 수 있게 되었습니다.

이전에는 사용자 보고로부터 에러를 인지하기까지 꽤나 오래 걸렸지만, Sentry 도입 후에는 실시간으로 감지할 수 있게 되었서 좋았던 것 같습니다.
또, 에러의 정확한 발생 지점과 컨텍스트를 알 수 있게 되면서 문제 해결에 소요되는 시간이 준 것도 좋은 점이라고 생각합니다.

결과적으로 서비스 안정성이 높아지고 에러 발생 시 더 나은 피드백을 제공하면서 전반적인 사용자 만족도 또한 높일 수 있었다고 생각합니다.

> 앞으로는 아래와 같은 내용을 생각중에 있습니다.

- `성능 모니터링 확대`

에러 트래킹 외에도 성능 모니터링을 더욱 강화하여 사용자 경험을 지속적으로 개선하고자 합니다. 저희 팀이 생각하는 핵심 기능이자 사용자가 많이 머물것으로 예상되는 투표 프로세스, 장소 추천 등의 성능 측정에 초점을 맞추어 개선해나갈 예정입니다.

- `과도한 에러 보고로 인한 노이즈 개선`

현재는 모든 에러를 Sentry로 보내도록 설정했습니다. 이로 인해 중요한 에러와 사소한 에러가 섞여 정작 중요한 문제를 놓치는 경우가 있었습니다. 이 문제를 해결하기 위해 에러의 심각도에 따라 필터링하는 로직을 추가하여
특정 타입의 에러(예를 들면 CORS 관련 에러)는 그룹화하여 관리하고 자주 발생하는 비중요 에러는 샘플링 비율 낮춰 중요한 에러에 대한 처리를 우선하는 방향으로 개선하고자 합니다.
