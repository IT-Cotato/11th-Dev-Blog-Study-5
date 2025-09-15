# Virtual DOM 알아보기

> "왜 React는 Virtual DOM을 사용하나요?"

주니어 개발자 면접에서 이런 질문을 받았습니다. 그때 저는 "Virtual DOM이 실제 DOM보다 가볍고 성능이 좋아서요"라고 대답했는데요.
면접관분께서는 "그럼 Virtual DOM이 없으면 성능이 무조건 안 좋은가요?"라는 follow-up 질문을 하셨고, 그때 저는 제대로 답하지 못했습니다.

이 경험을 계기로 Virtual DOM에 대해 깊이 있게 공부해보기로 했습니다. Virtual DOM이 정말 "성능"만을 위한 것인지, 그리고 React가 이를 통해 실제로 해결하고자 했던 문제는 무엇인지 함께 알아보도록 하겠습니다.

## DOM, 웹의 기초가 되는 인터페이스

DOM(Document Object Model)을 이해하기 위해, 우리가 자주 마주치는 웹 개발 상황을 생각해봅시다. 사용자가 쇼핑몰에서 상품을 장바구니에 담거나, SNS에서 좋아요를 누르면 어떤 일이 일어날까요? 브라우저는 이러한 변경사항을 어떻게 화면에 반영할까요?

이런 동작들의 핵심에는 `DOM`이 있습니다.

DOM은 HTML 문서의 프로그래밍 인터페이스로, 웹 페이지가 로드되면 브라우저는 이 문서를 파싱하여 트리 구조의 DOM을 생성합니다.

예를 들어, 다음과 같은 간단한 장바구니 HTML이 있다고 해볼까요?

```html
<div class="cart">
  <h1>장바구니</h1>
  <p class="item-count">상품 1개</p>
  <ul class="item-list">
    <li>맥북 프로 16인치</li>
  </ul>
</div>
```

이 HTML은 아래와 같은 트리 구조의 DOM으로 표현됩니다.

![](https://velog.velcdn.com/images/cllaude/post/a807f555-ba43-406d-ad38-6b13ace99cdd/image.png)

## 현대 웹에서 마주치는 DOM의 한계

이제 사용자가 장바구니에 새로운 상품을 추가했다고 가정해봅시다. 우리는 다음과 같은 JavaScript 코드로 DOM을 업데이트해야 합니다.

```javascript
const itemList = document.querySelector('.item-list');
const newItem = document.createElement('li');
newItem.textContent = '에어팟 프로 2세대';
itemList.appendChild(newItem);

const itemCount = document.querySelector('.item-count');
itemCount.textContent = '상품 2개';
```

이러한 DOM 직접 조작 방식에는 몇 가지 문제가 있습니다.

1. **복잡성 증가**: 상품 추가, 수량 변경, 삭제 등 다양한 상태 변화를 모두 DOM 조작으로 관리하다 보면 코드가 금방 복잡해집니다.

2. **성능 이슈**: 각각의 DOM 조작은 브라우저가 레이아웃을 다시 계산하고(reflow), 화면을 다시 그리도록(repaint) 만듭니다.

3. **일관성 관리의 어려움**: 장바구니의 상품 목록, 총액, 할인가 등 여러 요소가 서로 연관되어 있을 때, 이들의 상태를 일관성 있게 유지하기가 어렵습니다.

## Virtual DOM, DOM 조작의 새로운 패러다임

이러한 문제들을 해결하기 위해 React는 Virtual DOM이라는 접근 방식을 도입했습니다. Virtual DOM은 실제 DOM의 가벼운 복사본으로, 자바스크립트 객체 형태로 메모리에 존재합니다.

앞서 본 장바구니 예시를 Virtual DOM으로 표현하면 이렇게 됩니다.

```javascript
// Virtual DOM의 자바스크립트 객체 표현
const virtualDOM = {
  type: 'div',
  props: { className: 'cart' },
  children: [
    {
      type: 'h1',
      props: {},
      children: ['장바구니'],
    },
    {
      type: 'p',
      props: { className: 'item-count' },
      children: ['상품 1개'],
    },
    {
      type: 'ul',
      props: { className: 'item-list' },
      children: [
        {
          type: 'li',
          props: {},
          children: ['맥북 프로 16인치'],
        },
      ],
    },
  ],
};
```

이제 새로운 상품을 추가할 때는 어떻게 될까요?

React는 다음과 같은 과정을 거칩니다.

1. 상태 변경 감지: 새로운 상품이 추가되면 React는 이를 감지합니다.
2. 새로운 Virtual DOM 생성: 변경된 상태를 반영한 새로운 Virtual DOM 트리를 만듭니다.
3. Diffing: 이전 Virtual DOM과 새로운 Virtual DOM을 비교합니다.
4. 실제 DOM 업데이트: 차이가 있는 부분만 실제 DOM에 적용합니다.

![](https://velog.velcdn.com/images/cllaude/post/fd74cebf-138c-4d90-8b86-22fc5849071e/image.png)

이러한 방식의 장점은 개발자가 DOM을 직접 조작하지 않고, 상태 변경만 선언적으로 처리하면 된다는 것입니다.

이에 따라 장바구니에 상품을 추가하는 코드는 이렇게 단순화됩니다.

```javascript
function ShoppingCart({ items }) {
  return (
    <div className="cart">
      <h1>장바구니</h1>
      <p className="item-count">상품 {items.length}개</p>
      <ul className="item-list">
        {items.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

![](https://velog.velcdn.com/images/cllaude/post/6f0e2e65-9a3a-4339-b2db-226f199dd070/image.png)

## Virtual DOM으로 얻는 이점

Virtual DOM의 진정한 가치는 단순히 성능 향상이 아닙니다. 오히려 다음과 같은 이점들이 더 중요합니다.

1. **선언적 UI 프로그래밍**: DOM을 직접 조작하는 대신, UI가 어떻게 보여야 하는지만 선언하면 됩니다.

2. **상태 관리 단순화**: 복잡한 UI 상태 변화를 React가 자동으로 처리해줍니다.

3. **크로스 브라우저 호환성**: Virtual DOM이 브라우저별 DOM 처리 방식의 차이를 추상화해줍니다.

4. **서버 사이드 렌더링 용이성**: Virtual DOM은 브라우저 DOM에 의존하지 않아 서버에서도 렌더링이 가능합니다.

## React의 렌더링 프로세스 이해하기

이제 React가 실제로 어떻게 Virtual DOM을 처리하는지 자세히 살펴보겠습니다. React의 렌더링 프로세스는 크게 `Render Phase`와 `Commit Phase`로 나뉩니다.

### 🫣 Render Phase: 변경 사항 계산하기

Render Phase에서는 다음과 같은 작업들이 이루어집니다.

1. **컴포넌트 렌더링**: 컴포넌트의 render 함수를 호출하여 새로운 Virtual DOM을 생성합니다.
2. **Diffing**: 이전 Virtual DOM과 새로운 Virtual DOM을 비교합니다.

위에서 예로들은 장바구니 예시에서 장바구니에 새로운 상품을 추가할 때 React는 이 두 상태를 비교하여 실제로 변경된 부분(새로 추가된 에어팟 프로 항목)만을 찾아냅니다.

```javascript
function CartItem({ name, price }) {
  return (
    <li className="cart-item">
      <span>{name}</span>
      <span>{price}원</span>
    </li>
  );
}

// 이전 상태
const prevItems = [{ id: 1, name: '맥북 프로 16인치', price: 3600000 }];

// 새로운 상태
const newItems = [
  { id: 1, name: '맥북 프로 16인치', price: 3600000 },
  { id: 2, name: '에어팟 프로 2세대', price: 359000 },
];
```

### 🔨 Commit Phase: 변경 사항 적용하기

Commit Phase는 React가 계산된 변경 사항을 실제 DOM에 반영하는 단계입니다. 이 단계에서는 모든 변경이 일괄적으로 적용되며, 중간에 중단될 수 없습니다. 즉 위에서 장바구니에 상품을 추가할 때, 새로운 상품이 화면에 나타나는 것과 함께 상품 개수도 동시에 업데이트되어야 하므로, 하나의 작업처럼 처리되어야 합니다.

또한 `Commit Phase` 단계에서는 실제 DOM 조작과 함께 여러 중요한 작업들이 수행됩니다. `componentDidMount`나 `componentDidUpdate` 같은 생명주기 메서드가 호출되어 컴포넌트가 화면에 완전히 반영된 후의 작업을 처리할 수 있습니다. `useEffect` 훅의 실행이나 DOM 노드에 대한 ref 업데이트도 이 시점에 이루어집니다.

또, 주목할 점은 Commit Phase가 동기적으로 실행된다는 것입니다.
이는 모든 변경 사항이 화면에 반영되기 전까지 다른 작업이 중간에 끼어들 수 없다는 것을 의미합니다. 이러한 특성 덕분에 React는 일관된 UI 상태를 보장할 수 있습니다.

## 효율적인 상태 업데이트: Batch Update

React는 여러 상태 업데이트를 하나의 리렌더링으로 묶어서 처리하는 'Batch Update' 기능을 제공합니다. 이는 특히 여러 상태가 동시에 변경되는 경우에 유용합니다.

```javascript
function handleAddToCart(product) {
  // 이 세 가지 상태 업데이트는 하나의 리렌더링으로 처리됩니다
  setCartItems((items) => [...items, product]); // 상품 추가
  setTotalPrice((price) => price + product.price); // 총액 업데이트
  setItemCount((count) => count + 1); // 상품 개수 업데이트
}
```

이러한 배치 처리를 통해 여러 번의 리렌더링이 한 번으로 줄어들게 되고 모든 상태 업데이트가 동시에 반영되게 된다는 이점이 있습니다.

## 🙇🏻 마치며

처음 면접에서 받았던 "왜 React는 Virtual DOM을 사용하나요?"라는 질문에 대한 답을 찾아가는 과정에서, 그동안 정확히 알지 못했던 Virtual DOM의 역할을 좀 더 명확히 이해할 수 있었습니다. 알아보는 과정에서 Virtual DOM은 단순한 성능 최적화 기법이 아니라, 복잡한 DOM 조작을 추상화하여 개발자가 UI의 상태에만 집중할 수 있도록 도와주는 구조임을 알게 되었습니다.

UI가 특정 상태일 때 어떻게 보여야 하는지만 선언하면 되고, 실제 DOM 변경은 React가 내부적으로 효율적으로 처리하는 구조 덕분에 코드가 간결해지고, 상태 변화에 따른 UI 업데이트도 훨씬 더 일관되게 이루어질 수 있었던 것 같습니다. 이러한 학습을 통해 결국 Virtual DOM은 React가 복잡한 UI를 효과적으로 관리하기 위해 선택한 중요한 설계 방식이라는 점을 자연스럽게 이해할 수 있었습니다.
