동아리 공식 사이트인 COTATO 프로젝트에는 동아리 활동 기록을 확인할 수 있는 '세션 기록 페이지'가 있다.

![](https://blog.kakaocdn.net/dn/JXFvl/btsOjZeFTHz/yx6bkUS9x7m2PcRXrEP3FK/img.png)

동아리의 활동인 세션을 사진과 같이 기록할 수 있는 페이지이다. (사이트 링크 : https://www.cotato.kr/session?generationId=7)

카드를 클릭해서 상세 보기를 통해 세션의 기록과 썸네일 사진 말고도 다른 사진들도 같이 확인을 할 수 있다.

![](https://blog.kakaocdn.net/dn/JNdjb/btsOjcFupxt/4UIBGoHiNxZacdkOTzxgHK/img.png)

페이지에 어드민 권한으로 접근하면 모달 접근을 통해 세션 기록을 추가, 수정할 수 있고, 사진의 경우 하단의 카드를 드래그하여 순서를 조정할 수 있다.

![](https://blog.kakaocdn.net/dn/bTKN1X/btsOjc6Cx71/EbSgw3XMKcNq63eYMU7vuk/img.gif)

정상적으로 동작하면 위와 같이 부드럽게 DnD가 동작해야 하지만, 처음 구현을 한 당시에는 아래와 같이 **드래그를 하는 순간 드래그된 카드가 비정상적인 위치로 이동하는 문제가 발생했다.**

![](https://blog.kakaocdn.net/dn/P8R5z/btsOlwB3aLi/Xf5ukutpD2sp9n91HttFX0/img.gif)

DnD를 세션 수정 말고 다른 기능에서도 사용 중인데, 다른 기능에서는 문제가 없이 동작을 하고... 그럼 DnD 라이브러리 자체에는 문제가 없었고, 세션 수정 기능에서만 어떤 이유로 인해서 발생하는 문제였던 것이다.

다른 기능과 달리 세션 수정에서 DnD는 모달 위에서 차이점이고, 모달과 관련된 CSS 속성으로 인해서 발생하는 문제였다. 이번 글은 DnD에 드래그를 하는 로직을 분석하고, 모달의 어떤 특징 때문에 위치가 튕기는지를 분석한 글이다.

## 모달과 DnD 드래그 구현하기

해당 페이지에서는 모달과 DnD 모두 라이브러리를 사용했다. 모달은 MUI, DnD는 react-beautiful-dnd(현 @hello-pangea/dnd) 라이브러리를 사용했다.

우선 모달과 DnD가 어떻게 동작하는지 이해하기 위해서 라이브러리의 코드를 참고해서 직접 구현을 해보겠다.

### 모달 구현하기

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

const Modal = ({ children }: { children: React.ReactNode }) => {
  return createPortal(
    <div className="fixed inset-0 h-screen w-screen bg-black/10">
      <div className="absolute top-1/2 left-1/2 h-1/2 w-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-white p-8 shadow-lg">
        {children}
      </div>
    </div>,
    document.body
  );
};

export default Modal;
```

모달은 cretaePortal을 사용해서 DOM이 body 태그 하단에 위치시키고, postion: fixed 속성을 사용해서 모달 바깥쪽 배경이 불투명하게 설정했다.

그리고 모달 박스는 position: absolute와 top, left, transform 속성을 사용하여 화면 중앙에 위치시키고, children을 props로 받아 공용 Modal 컴포넌트가 되도록 구현했다.

![](https://blog.kakaocdn.net/dn/bmDpDm/btsOmsGGN8M/YEkzTb4zbc3sKkwRmcbYQ0/img.png)

### DnD 드래그 구현하기

```tsx
import React from 'react';

const DnD = () => {
// Drag 컴포넌트가 드래그 중인지 여부를 저장const [isDragging, setIsDragging] = React.useState(false);
// Drag 컴포넌트의 기존 위치를 저장const [elementPosition, setElementPosition] = React.useState({ x: 0, y: 0 });
// Drag를 시작할 때 클릭된 위치를 저장const [initialClientPosition, setInitialClientPosition] = React.useState({
    x: 0,
    y: 0,
  });
// Drag를 시작한 후 최초 클릭 위치로부터 Drag 컴포넌트의 이동 위치를 저장const [offsetPosition, setOffsetPosition] = React.useState({ x: 0, y: 0 });

/**
   * 드래그 시작 핸들러
   * 드래그 시작 시 컴포넌트의 위치와 클릭된 위치를 저장하고 드래그 상태를 true로 설정
   */const handleDragStart = (e: React.MouseEvent) => {
    e.preventDefault();

    const target = e.target as HTMLElement;
    const rect = target.getBoundingClientRect();

    setElementPosition({
      x: rect.left,
      y: rect.top,
    });
    setInitialClientPosition({
      x: e.clientX,
      y: e.clientY,
    });
    setIsDragging(true);
  };

//////
  React.useEffect(() => {
    if (!isDragging) {
      return;
    }

/**
     * 드래그 중 마우스 이동 핸들러
     * 마우스가 이동할 때마다 현재 마우스 위치와 최초 클릭 위치를 기준으로 Drag 컴포넌트의 이동 위치를 업데이트
     */const handleDragUpdate = (e: MouseEvent) => {
      e.preventDefault();

      setOffsetPosition({
        x: e.clientX - initialClientPosition.x,
        y: e.clientY - initialClientPosition.y,
      });
    };

/**
     * 드래그 종료 핸들러
     * 드래그가 끝나면 Drag 컴포넌트의 위치를 초기화하고 드래그 상태를 false로 설정
     */const handleDragEnd = (e: MouseEvent) => {
      e.preventDefault();

      setElementPosition({ x: 0, y: 0 });
      setOffsetPosition({ x: 0, y: 0 });
      setIsDragging(false);
    };

    document.addEventListener('mousemove', handleDragUpdate);
    document.addEventListener('mouseup', handleDragEnd);
    document.body.style.cursor = 'grabbing';

    return () => {
      document.removeEventListener('mousemove', handleDragUpdate);
      document.removeEventListener('mouseup', handleDragEnd);
      document.body.style.cursor = 'default';
    };
  }, [isDragging, initialClientPosition]);

  return (
    <div className="flex h-full flex-col items-center justify-between">
      <h2 className="font-meidum text-xl">DnD 드래그</h2>
      <div>
        <div
          className="h-20 w-20 cursor-grab rounded-2xl bg-pink-400"
          style={
            isDragging
              ? {
                  position: 'fixed',
                  top: elementPosition.y,
                  left: elementPosition.x,
                  transform: `translate(${offsetPosition.x}px, ${offsetPosition.y}px)`,
                  cursor: 'grabbing',
                }
              : {}
          }
          onMouseDown={handleDragStart}
        />
      </div>
    </div>
  );
};

export default DnD;
```

DnD 컴포넌트는 react-beautiful-dnd의 동작 방식에서 드래그 부분에만 필요한 로직을 간소화하여 구현했다. 굳이 저렇게 구현해야 하나? 싶기도 하지만, 실제 라이브러리에서 제공하는 기능은 복잡한 DnD 로직을 포함하고 있기에 생각보다 복잡하게 동작하고 있다.

![](https://blog.kakaocdn.net/dn/9SPxq/btsOmpb6vsE/8KfCJOqCDMQXo9jzxqKY50/img.png)

기본적이 드래그 로직은 드래그 컴포넌트에 mosuedown 이벤트를 elementPosition에는 **드래그 컴포넌트의 기존 위치**, initialClientPosition에는 사용자가 **드래그를 시작한 위치**를 저장한다. 그리고 **드래그 컴포넌트의 position을 fixed로 변경하여 레이아웃의 normal flow를 벗어나 뷰포트를 기준으로 지정된 위치에 배치되도록 한다.**

드래그 컴포넌트의 position의 absolute로 변경되었지만 기존의 위치를 유지하기 위해 **elementPostion에 저장한 값을 top, left에 적용시켜 드래그가 시작되어도 올바른 위치를 유지**하도록 한다.

드래그가 되면 사용자의 마우스 위치에 따라서 드래그 컴포넌트의 위치가 변경되어야 한다. 이는 mosuemove 이벤트를 통해서 **현재 마우스 위치와 initialClientPosition의 차이를 translate 함수를 통해 드래그 컴포넌트가 드래그된 위치로 이동**하도록 한다.

이런 플로우에서 드래그 기능을 사용하면 위치가 정상적으로 이동할까?

![](https://blog.kakaocdn.net/dn/bo3vTy/btsOlxoxqG0/olwuonezeR6163ARp08Ko1/img.gif)

세션 수정에서 드래그 시 위치가 튕기는 거처럼 직접 구현한 DnD에서 동일하게 위치가 튕기는 문제가 발생한다.

어떤 이유에서 드래그 시 컴포넌트가 비정상적으로 위치하는 것일까?

## transfrom과 containing block

왜 저런 문제가 발생하는지 결론부터 말하면 transform 속성으로 인해 **드래그 컴포넌트의 컨테이닝 블록이 뷰포트가 아닌 모달**이 되었기 때문이다.

mdn 문서의 fixed 속성에 대해서 이렇게 설명하고 있다.

> 요소를 일반적인 문서 흐름에서 제거하고, 페이지 레이아웃에 공간도 배정하지 않습니다. 대신 뷰포트의 초기 컨테이닝 블록을 기준으로 삼아 배치합니다. 단, 요소의 조상 중 하나가 transform, perspective, filter 속성 중 어느 하나라도 none이 아니라면 뷰포트 대신 그 조상을 컨테이닝 블록으로 삼습니다.

컨테이닝 식별 과정도 mdn 문서를 찾아보면 이런 설명을 볼 수 있다.

> position 속성이 absolute나 fixed 인 경우, 다음 조건 중 하나를 만족하는 가장 가까운 조상의 내부 여백 영역이 컨테이닝 블록이 될 수도 있습니다.

그리고 transform 속성에 대해서도 동일한 부분을 언급하고 있다.

> none이 아닌 값을 지정하면 새로운 쌓임 맥락을 생성합니다. 이 경우, position이 fixed 거나 absolute인 요소의 컨테이닝 블록으로서 작용합니다.

컨테이닝 블록과 뷰포트에 대한 용어를 정리하면 다음과 같다

- 컨테이닝 블록 : 요소의 위치, 크기, 비율 등을 계산할 때 기준이 되는 **부모 박스**
- 뷰포트 : 사용자가 보고 있는 **브라우저 화면 영역**

position 속성을 fixed인 경우 보통은 컨테이닝 블록이 뷰포트가 되어 가장 크기는 브라우저 크기를 기준으로, 위치는 브라우저의 좌측 상단을 기준으로 계산이 되어야 한다.

![](https://blog.kakaocdn.net/dn/bet6tc/btsOmOv14Ge/ZDDdPxWjVAQyMLc0AQUOr1/img.png)

구현된 코드를 보면 드래그가 시작되면서 position: fixed로 변경되면서 getBoundingClientRect( ) 메서드를 통해 뷰포트 기준 top, left 프로퍼티를 반환받아 elementPosition에 적용하고, 이를 다시 드래그 컴포넌트의 top, left 속성에 적용시켜 위의 그림과 같이 위치가 이동하지 않는 것을 기대하고 있었다.

**하지만 여기서 구현한 모달에는 transfrom: translate(-50%, -50%)가 적용되어 있다.**

그렇기 때문에 드래그 컴포넌트의 위치는 **fixed가 적용되어 있어도 뷰포트가 아닌 모달을 기준으로 계산**되고 있던 것이었다.

![](https://blog.kakaocdn.net/dn/8psY6/btsOmTYlqnA/1kOE59EZA8fnRe0yxknzn0/img.png)

getBoundingClientRect( ) 메서드 뷰포트 기준으로 요소의 위치를 가져오지만, 드래그 컴포넌트는 뷰포트가 아닌 모달을 기준으로 위치가 계산되고 있기 때문에 **드래그가 시작되면서 잘못된 위치에 배치**되는 결과가 발생하는 것이었다.

## 드래그 컴포넌트를 정상적으로 배치하기

모달과 DnD를 직접 구현하고, CSS 속성들을 분석한 결과 모달의 transform 속성으로 인해 드래그 컴포넌트가 잘못된 위치에 배치된다는 결론을 내릴 수 있었다.

문제를 해결하기 위해서는 DnD의 구현 방식을 수정하거나, 드래그 컴포넌트가 뷰포트를 기준으로 위치가 계산되도록 변경해야 한다. DnD의 구현 방식을 수정하는 것은 라이브러리에서 제공하기에 쉽지 않고, 간단하게 시도할 수 있는 방법은 **드래그 컴포넌트가 모달이 아닌 뷰포트를 기준으로 위치가 계산**되도록 코드를 수정하는 것이다.

### 모달의 transform 속성 제거하기

드래그 컴포넌트가 모달을 기준으로 위치가 계산되는 이유는 모달에 transform 속성이 none이 아니기 때문이고, transform 속성을 사용하지 않으면 드래그 컴포넌트가 뷰포트를 기준으로 계산되게 변경할 수 있다.

모달 컨테이너 중앙에 위치시키기 위해 top: 50%; left: 50%; transform: translate(-50%, -50%); 코드를 사용하고 있다. CSS 코드를 **flex 속성을 사용하는 방법으로 변경**하면 transform 속성을 사용하지 않더라도 모달 컨테이너를 중앙에 배치시킬 수 있다.

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

const Modal = ({ children }: { children: React.ReactNode }) => {
  return createPortal(
    <div className="fixed inset-0 h-screen w-screen bg-black/10">
      <div className="flex h-full w-full items-center justify-center">
        <div className="h-1/2 w-1/2 rounded-lg bg-white p-8">{children}</div>
      </div>
    </div>,
    document.body
  );
};

export default Modal;
```

이런 방식으로 코드를 수정하면 flex가 적용된 요소는 뷰포트 전체만큼 크기를 차지하고, justify-content: center; align-items: center;를 통해 모달 컨테이너를 중앙에 배치했다.

![](https://blog.kakaocdn.net/dn/DncVY/btsOkKvFtSe/TbbsLBnPVTM0xymcGHH8t1/img.gif)

모달에서 transform 속성을 제거하면 **드래그 컴포넌트의 컨테이닝 블록이 뷰포트가 되어 드래그 시 위치가 정상적으로 계산이 가능해진다.**

### 드래그 컴포넌트를 모달 외부에 위치

모달에서 transform 대신에 flex 속성을 사용하면 간단하게 문제를 해결할 수 있다. 하지만 모달의 코드를 변경할 수 없거나, 모달이 중앙에 위치하는 것이 아닌 특별한 위치에 배치되어야 해서 transform을 제거하지 못하는 상황이라면 어떻게 해결할 수 있을까?

모달을 구현할 때 DOM의 위치를 변경시키기 위해 createPortal을 사용했다. 그러면 드래그 컴포넌트에도 동일하게 **createPortal을 사용해서 드래그 컴포넌트의 DOM을 모달 외부에 배치**시켜 모달의 영향을 받지 않게 할 수 있다.

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

const DnD = () => {
  /*...*/ /**
   * Drag 컴포넌트를 렌더링하는 함수
   */ const renderDragComponent = () => {
    const dragComponent = (
      <div
        className="h-20 w-20 cursor-grab rounded-2xl bg-pink-400"
        style={
          isDragging
            ? {
                position: 'fixed',
                top: elementPosition.y,
                left: elementPosition.x,
                transform: `translate(${offsetPosition.x}px, ${offsetPosition.y}px)`,
                cursor: 'grabbing',
              }
            : {}
        }
        onMouseDown={handleDragStart}
      />
    );

    if (isDragging) {
      return createPortal(dragComponent, document.body);
    }

    return dragComponent;
  };

  /*...*/ return (
    <div className="flex h-full flex-col items-center justify-between">
      <h2 className="font-meidum text-xl">DnD 드래그</h2>
      <div>{renderDragComponent()}</div>
    </div>
  );
};

export default DnD;
```

드래그가 진행 중이지 않는 경우는 기존과 같은 방법으로 드래그 컴포넌트를 배치시키고, 드래그 중이면 **createPortal을 사용해 모달의 영향을 받지 않는 body 요소에 DOM을 배치시켜 컨테이닝 블록이 모달이 아닌 뷰포트가 되도록 변경할 수 있다.**

react-beautiful-dnd에서는 Draggable 컴포넌트가 제공하는 snapshot을 활용해서 컴포넌트가 드래그 중인지 확인할 수 있다.

```tsx
import { Draggable } from 'react-beautiful-dnd';

<Draggable draggableId="draggable-1" index={0}>
  {(provided, snapshot) => (
    <div ref={provided.innerRef} {...provided.draggableProps} {...provided.dragHandleProps}>
      <h4>{snapshot.isDragging ? '드래그중' : '드래그중이 아님'}</h4>
    </div>
  )}
</Draggable>;
```

### 드래그 컴포넌트의 위치를 직접적으로 수정

가장 간단하게 해결할 수 있는 방법은 모달의 transform 속성의 제거이고, 제거하기 어려우면 createPortal을 통해 DOM을 모달 외부에 배치시키는 방법이 있다. 대부분의 경우는 위 두 가지 방법으로 해결이 가능할 거 같지만, 만약 이 방법도 사용하지 못하는 상황이라면 해결할 수 있는 방법이 없을까?

시도해 볼 수 있는 다른 방법은 드래그 컴포넌트의 위치를 직접 수정하는 것이다. 위치가 뷰포트가 아닌 모달을 기준으로 계산되어 문제가 생겼다면, **style에서 위치를override 하여 해당 좌표가 모달 기준에서도 정확하게 적용**되도록 조정할 수 있다.

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

const Modal = ({ children }: { children: React.ReactNode }) => {
  return createPortal(
    <div className="fixed inset-0 h-screen w-screen bg-black/10">
      <div
        id="modal-container"
        className="absolute top-1/2 left-1/2 h-1/2 w-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-white p-8 shadow-lg"
      >
        {children}
      </div>
    </div>,
    document.body
  );
};

export default Modal;
```

우선 모달의 요소를 다른 컴포넌트에서 검색할 수 있게 id를 지정해 준다.

```tsx
import React from 'react';

const DnD = () => {
  /*...*/

  /**
   * Drag 컴포넌트를 렌더링하는 함수
   */
  const renderDragComponent = () => {
    const modalContainerElement = document.getElementById('modal-container');
    const modalContainerPosition = modalContainerElement?.getBoundingClientRect();

    return (
      <div
        className="h-20 w-20 cursor-grab rounded-2xl bg-pink-400"
        style={
          isDragging
            ? {
                position: 'fixed',
                top: elementPosition.y - modalContainerPosition.y,
                left: elementPosition.x - modalContainerPosition.x,
                transform: `translate(${offsetPosition.x}px, ${offsetPosition.y}px)`,
                cursor: 'grabbing',
              }
            : {}
        }
        onMouseDown={handleDragStart}
      />
    );
  };

  /*...*/
  return (
    <div className="flex h-full flex-col items-center justify-between">
      <h2 className="font-meidum text-xl">DnD 드래그</h2>
      <div>{renderDragComponent()}</div>
    </div>
  );
};

export default DnD;
```

드래그 컴포넌트가 뷰포트로부터 elementPosition만큼 떨어진 곳에 위치해야 하지만, 실제로는 모달로부터 elementPosition만큼 떨어진 곳에 위치해 있다. 이를 해결하기 위해 **모달 컨테이너의 위치를 가져오고, elementPositoin - modalContainerPosition만큼 떨어진 곳에 위치시킨다면 드래그 컴포넌트가 정상적인 위치에 배치될 수 있다.**

react-beautiful-dnd에서는 Draggable 컴포넌트가 제공하는 provided를 통해 라이브러리가 설정한 드래그 컴포넌트의 위치를 가져올 수 있다.

```tsx
import { Draggable, DraggableProvided, DraggableStateSnapshot DraggingStyle } from 'react-beautiful-dnd';

const renderDragComponent = (provided: DraggableProvided, snapshot: DraggableStateSnapshot) => {
  const isDragging = snapshot.isDragging;

  const modalContainerElement = document.getElementById('modal-container');
  const modalPosition = modalContainerElement?.getBoundingClientRect();

  const dragggingStyle = provided.draggableProps.style as DraggingStyle;

  const topPosition = draggingStyle.top - (isDragging ? modalPosition.top : 0);
  const leftPosition = draggingStyle.left - (isDragging ? modalPosition.left : 0);

  return (
      <div
      ref={provided.innerRef}
      {...provided.draggableProps}
      {...provided.dragHandleProps}
      style={{
        ...draggingStyle,
        top: topPosition,
        left: leftPosition,
      }}
    >
      <h4>My draggable</h4>
    </div>
  )
}

<Draggable draggableId="draggable-1" index={0}>
  {(provided, snapshot) => renderDragComponent(provided, snapshot)}
</Draggable>;
```

## 결론

이 문제는 지금으로부터 약 1년 전쯤 처음 겪었고, 당시에는 GitHub 이슈와 블로그를 참고해 createPortal을 사용하는 방식으로 해결했었다. 하지만 그때는 **라이브러리의 동작 방식을 충분히 이해하지 못한 채 단순히 동작만 되도록 처리했을 뿐이었다.** (~~해결만 해두고 방치한 사이, 해당 라이브러리는 결국 deprecated 되었다…)~~

그러던 중 ‘이 문제가 혹시 그런 구조적 원인에서 비롯된 것이 아닐까?’라는 생각이 들었고, 이번에 글을 작성하면서 라이브러리의 내부 동작 방식을 직접 분석해 보며 문제의 원인과 해결 방식을 정리해 보게 되었다.

사실 문제의 원인은 단순한 CSS 속성이었지만, 당시에는 라이브러리의 사용법에만 집중하다 보니 근본적인 원인을 파악할 수 없었다. 이번 경험을 통해 어떤 문제가 발생했을 때 그 원인이 단순하더라도 내부 동작 원리까지 이해해야 정확한 원인을 짚어낼 수 있고, 그래야 다양한 해결 방안을 스스로 제시할 수 있다는 것을 깨달았다.
