프론트엔드 개발을 하다 보면 점점 복잡해지는 UI와 유지보수 어려움 속에서, 더 안정적이고 효율적인 개발 방법이 필요해집니다. 이 글에서는 그 해답 중 두 가지인 **TypeScript**와 **Storybook**에 대해 정리해 보았습니다.

### 🧩 TypeScript란?

TypeScript는 JavaScript의 상위 확장(superset) 언어로, **정적 타입** 기능을 추가해 코드의 안정성과 유지보수성을 높여주는 언어입니다. JavaScript로 작성된 코드를 실행 전에 컴파일하여 타입 오류를 미리 잡아줍니다.

> “The goal of TypeScript is to be a static typechecker for JavaScript programs.”

### ✅ 정적 타입 vs 동적 타입

**항목정적 타이핑 (TypeScript 등)동적 타이핑 (JavaScript 등)**

| 비교 항목      | 정적 타이핑 (Static Typing) | 동적 타이핑 (Dynamic Typing) |
| -------------- | --------------------------- | ---------------------------- |
| 타입 검사 시점 | 컴파일 시점                 | 런타임                       |
| 오류 발견 시점 | 코드 작성 중                | 실행 중                      |
| 안정성         | 높음                        | 낮음                         |
| 유연성         | 낮음                        | 높음                         |

JavaScript는 자유도가 높은 대신, 런타임 오류 발생 가능성이 높습니다. 반면 TypeScript는 타입을 명확하게 정의하고 컴파일 타임에 체크하기 때문에 **에러를 사전에 방지**할 수 있습니다.

### ⚙️ TypeScript의 특징과 장점

1. 컴파일 타임에 오류 감지
   타입 불일치, 잘못된 프로퍼티 접근 등 문제를 실행 전 미리 발견할 수 있습니다.
2. 개발 편의성 향상
   에디터 자동완성, 타입 추론, 명확한 API 정의 등으로 생산성 향상과 리팩토링 용이성이 크게 증가합니다.
3. 자바스크립트 100% 호환
   기존 JS 코드와 문제없이 통합 가능하며, 점진적으로 적용할 수 있습니다.
4. 객체지향 프로그래밍 패턴 지원
   클래스, 인터페이스, 제네릭 등을 통해 더 구조화된 코드를 작성할 수 있습니다.

### 🔧 TypeScript의 작동 원리

TypeScript는 다음과 같은 컴파일러 아키텍처를 가집니다:

- **Parser**: 코드를 추상 구문 트리(AST)로 분석
- **Binder**: 각 선언을 심벌로 등록하고 참조 연결
- **Type Checker**: 타입 검사 수행
- **Emitter**: .ts → .js로 코드 변환
- **Preprocessor**: 외부 파일 정리 및 모듈 정리

이를 통해 .ts 파일은 결국 브라우저에서 실행 가능한 .js 코드로 컴파일됩니다.

### 👀 Storybook이란?

**Storybook**은 UI 컴포넌트를 독립적으로 개발, 테스트, 문서화할 수 있는 도구입니다. 컴포넌트를 직접 페이지에 넣지 않아도, Storybook에서 개별 단위로 시각적 테스트가 가능합니다.

> “Storybook is an open source tool for building UI components and pages in isolation.”

### 🛠 Storybook의 장점

- **시각적 테스트**: UI의 상태를 직접 눈으로 확인
- **컴포넌트 문서 자동화**: 사용법, 속성, 예제 등을 자동 정리
- **독립적인 개발 환경**: 다른 코드에 영향 없이 컴포넌트를 테스트
- **디자인 시스템 구축에 적합**: 일관된 UI 관리 가능

### 📦 Vite + React + TypeScript 환경에서 Storybook 설정하기

1. Storybook 설치

```
npx storybook@latest init
```

2. 절대경로 설정
   tsconfig.json

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },
});
```

.storybook/main.ts

```typescript
import { StorybookConfig } from "@storybook/react-vite";
import path from "path";

const config: StorybookConfig = {
  async viteFinal(config) {
    config.resolve = {
      alias: {
        "@": path.resolve(__dirname, "../src"),
      },
    };
    return config;
  },
};

export default config;
```

3. 실행

```
npm run storybook
```

### 🧪 Story 작성 예시 (Buttons.tories.tsx)

```typescript
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "Example/Button",
  component: Button,
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    label: "Click me!",
    primary: true,
  },
};
```

### ✨ 마무리하며

TypeScript는 코드의 안정성을 높이고, Storybook은 UI 컴포넌트를 효율적으로 개발하게 도와줍니다. 이 둘을 함께 사용하면 더 안전하고 **생산적인 프론트엔드 개발 환경**을 구축할 수 있습니다.
