[네이버 FE News](https://d2.naver.com/news/8825314) 글을 살펴보다가 React에서 세밀한 반응성을 구현하는 방법을 설명하는 [튜토리얼](https://romgrk.com/posts/reactivity-is-easy) 글을 발견하여 해당 글을 통해 배운 내용 및 실습 과정을 정리한 글을 작성해보려고 한다.

## React에서 리렌더링 최적화가 중요한 이유

React에서 특히 대규모 리스트나 데이터 테이블, 또는 복잡한 UI 상태를 다루는 경우에는 **불필요한 리렌더링을 줄이는 것**이 성능 유지에 핵심이 된다.

보통 이런 상황에서는 `React.memo`, `useMemo`, `useCallback` 같은 도구를 활용하지만, 복잡한 상태 트리와 컴포넌트 구조에서는 한계가 있다. 특히, `useContext` 기반 상태 전달은 단 하나의 값만 바뀌어도 모든 하위 컴포넌트가 리렌더링되는 문제를 가지고 있다.

## React의 selector-based reactivity란?

### 🔍 문제 상황

```
const Context = createContext();

function Grid() {
	const [focus, setFocus] = useState(0);
    const context = useMemo(() => ({ focus, setFocus }), [focus]);

    return (
    	<Context.Provider value={context}>
        	{Array.from({ length: 50 }).map((_, i) => (
            	<Cell index={i} />
            ))}
        </Context.Provider>
    );
}

function Cell({ index }) {
	const { focus, setFocus } = useContext(Context);
    const isFocused = focus === index;

    return (
    	<button onClick={() => setFocus(index)} className={clsx({ focus: isFocused })}>
        	{index}
        </button>
    );
}
```

이 방식은 간단하지만, `focus` 값이 변경될 때마다 **모든 Cell 컴포넌트가 리렌더링**된다. `useContext`를 통해 값을 받아온 모든 컴포넌트는 context 값이 바뀌면 무조건 다시 렌더링되기 때문이다.

## 35줄짜리 커스텀 store와 useSelector

이 문제를 해결하기 위해, MUI팀은 아래와 같은 **초경량 커스텀 Store**를 만들어 해결했다.

### ✅ Store 구현

```
type Listener<S> = (s: S) => void;

class Store<State> {
	public state: State;
    private listeners: Set<Listener<State>> = new Set();

    constructor(initial: State) {
    	this.state = initial;
    }

    subscribe = (fn: Listener<State>) => {
    	this.listeners.add(fn);
        return () => this.listeners.delete(fn);
    };

    update = (newState: State) => {
    	this.state = newState;
        this.listeners.forEach(listener => listener(newState));
    };
}
```

이 Store는 단순한 Pub-Sub 구조이다.

- `state`라는 값을 직접 들고 있고
- `subscribe()`를 통해 리스터너를 등록하며
- `update()`가 호출되면 모든 리스터에게 새로운 상태를 알려준다.

이 구조는 React 컴포넌트 내부의 `setState`와 연결되면, **필요한 컴포넌트만 다시 렌더링할 수 있는 미세 조정이 가능**해진다.

### ✅ useSelector 훅 구현

```
function useSelector<State, Result>(
	store: Store<State>,
    selector: (state: State, ...args: any[]) => Result,
    ...args: any[]
) {
	const [value, setValue] = useState(() => selector(store.state, ...args));

    useEffect(() => {
    	return store.subscribe(state => {
        	setValue(selector(state, ...args));
        });
    }, []);

    return value;
}
```

`useSelector()`는 `store.subscribe()`를 통해 외부 상태 변화에 반응하고, 내부적으로 `useState()`로 관리되는 값을 갱신해 렌더링을 유도한다.

selector 함수를 통해 현재 컴포넌트가 관심 있는 일부 값만 구독하게 되므로, **관심 없는 상태 변경에는 전혀 반응하지 않는다.**

### ✅ Grid에 적용해보기

```
const Context = createContext<Store<{ focus: number }>>(null!);

function Grid() {
  const [store] = useState(() => new Store({ focus: 0 }));

  return (
    <Context.Provider value={store}>
      {Array.from({ length: 50 }).map((_, i) => (
        <Cell index={i} key={i} />
      ))}
    </Context.Provider>
  );
}

const isFocus = (state: { focus: number }, index: number) => state.focus === index;

function Cell({ index }: { index: number }) {
  const store = useContext(Context);
  const focus = useSelector(store, isFocus, index);

  return (
    <button
      onClick={() => store.update({ ...store.state, focus: index })}
      className={clsx({ focus })}
    >
      {index}
    </button>
  );
}
```

이제 버튼을 클릭하면 focus 상태가 바뀌고, 바뀐 index의 Cell만 리렌더링된다.
이전에 focus였던 Cell도 리렌더링된다. (class 제거)
**나머지 Cell은 절대 리렌더링되지 않는다.**

예를 들어 5번 Cell을 클릭하면 다음과 같은 일이 일어난다:

- 기존 focus였던 예: 3 -> 5
- 3번 Cell: `isFocus === true` -> `false`로 변경 -> 리렌더링됨
- 5번 Cell: `isFocus === false` -> `true`로 변경 -> 리렌더링됨
- 그 외의 Cell: selector 결과 변화 없음 -> **리렌더링 안 됨**

## 추가 개선 사항 실습

기본적인 커스텀 store + useSelector 구조만으로도 충분히 fine-grained reactivity를 구현할 수 있지만, 사용성을 높일 수 있는 개선점들도 존재한다.

### 💡 `store.update({...})`가 번거롭다면? -> `.set()` 메서드 추가

기존 방식에서는 상태를 업데이트 할 때 다음과 같이 작성해야 한다:

```
store.update({ ...store.state, focus: 5 });
```

이 방식은 상태가 깊어질수록 점점 번거롭고 불편해질 수 있다. 따라서 아래와 같이 `.set()` 메서드를 추가해 개선할 수 있다:

```
class Store<State> {
  public state: State;
  private listeners: Set<Listener<State>> = new Set();

  constructor(initial: State) {
    this.state = initial;
  }

  subscribe = (fn: Listener<State>) => {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  };

  update = (newState: State) => {
    this.state = newState;
    this.listeners.forEach(listener => listener(newState));
  };

  set = <K extends keyof State>(key: K, value: State[K]) => {
    this.update({ ...this.state, [key]: value });
  };
}
```

이제 다음과 같이 간결한 작성이 가능해진다:

```
store.set('focus', index);
```

### 💡 React 18+의 동시성 문제 해결 -> `useSyncExternalStore`

React 18부터는 동시성 렌더링(Suspense, concurrent rendering 등)이 도입되면서 **외부 스토어에서 상태 tearing**이 발생할 수 있다. 이를 방지하기 위해 React는 공식적으로 `useSyncExternalStore` 훅을 제공한다.

따라서 기존의 `useSelector`를 다음처럼 리팩토링할 수 있다:

```
import { useSyncExternalStoreWithSelector } from 'use-sync-external-store/with-selector';

function useSelector<State, Result>(
  store: Store<State>,
  selector: (state: State, ...args: any[]) => Result,
  ...args: any[]
) {
  return useSyncExternalStoreWithSelector(
    store.subscribe,
    () => store.state,
    () => store.state,
    (state) => selector(state, ...args),
  );
}
```

### 💡 파생된 값을 매번 계산하기 부담스럽다면 -> `reselect`로 memoized selector 구현

복잡한 상태에서 계산량이 많은 파생값이 있다면, 이를 매번 다시 계산하는 것은 비효율적이다. 이런 경우 `reselect`의 `createSelector`를 활용하는 것이 유용하다.

예시: rows와 정렬 기준을 기반으로 정렬된 rows를 반환하는 selector

```
import { createSelector } from 'reselect';

const rowsSelector = (state: { rows: any[] }) => state.rows;
const sortBySelector = (state: { sortBy: string }) => state.sortBy;

const sortedRowsSelector = createSelector(
  [rowsSelector, sortBySelector],
  (rows, sortBy) => rows.toSorted((a, b) => a[sortBy].localeCompare(b[sortBy]))
);
```

그리고 다음과 같이 사용할 수 있다:

```
const sortedRows = useSelector(store, sortedRowsSelector);
```

## 느낀 점 & 활용 방안

이번 실습을 통해 React의 리렌더링 매커니즘을 보다 잘 이해하고, **컴포넌트 단위가 아닌 상태 단위로 리렌더링을 제어하는 방식**을 학습할 수 있었다.

커스텀 Store와 selector 조합으로 `React.memo` 없이 정확히 필요한 컴포넌트만 리렌더링시키는 구조가 가능하다는 것이 흥미로웠고,
Redux, Zustand, Recoil 등 무거운 상태관리 라이브러리 없이도 가볍게 상태 분리/구독을 하고 싶을 때, 혹은 복잡한 비즈니스 로직 없이 store를 직접 통제하고 싶을 때 사용하기 유용한 방식인 것 같다.

해당 방식은 [store-x-selector](https://www.npmjs.com/package/store-x-selector)라는 패키지로도 공개되어 있어, 직접 구현하지 않아도 쉽게 사용할 수 있다.
