졸업 프로젝트의 주제가 '소스코드 유사도 판별기'로 확정되었다.
학생들이 제출한 코드 파일 간의 유사도를 분석하고, 이를 **네트워크 그래프 형태로 시각화하면 좋겠다**는 아이디어가 나와,
본격적인 개발에 들어가기 전 **React 환경에서는 네트워크 그래프를 어떻게 구현할 수 있는지** 먼저 테스트해보았다.

## 1. 라이브러리 선택: vis-network

React에서 사용할 수 있는 시각화 라이브러리들을 찾아본 결과, [Recharts](https://recharts.org/en-US), [Victory](https://nearform.com/open-source/victory/), [ApexCharts](https://apexcharts.com/), [Nivo](https://nivo.rocks/), [React-Vis](https://uber.github.io/react-vis/) 등 굉장히 다양한 도구들이 많았지만, 간편하게 네트워크 토폴로지만을 구현하기 위해서는 [vis-network](https://visjs.github.io/vis-network/docs/network/)만으로 충분한 것 같아 `vis-network`를 사용하여 구현해보기로 결정하였다.

`vis-network`는 **노드-엣지 중심 시각화 구조**로 관계 기반 데이터를 표현하는데 최적인 라이브러리이고, **간편한 사용법**을 갖고 있어 빠르게 프로토타입을 만들어보기 적합하였다.
또한, `hover`, `click`, `blur` 같은 마우스 이벤트를 간단히 붙일 수 있어서 **풍부한 인터랙션도 지원**이 가능하였다.

## 2. 네트워크 그래프 초기 구현

우선, 초기 구현 목표는 다음과 같이 잡아보았다.

> 학생 -> 노드
> 두 학생이 유사할 경우 -> 엣지로 연결
> 유사도가 높을수록 -> 엣지를 두껍고 진하게 표시

### 네트워크 그래프의 옵션 정의하기

```
    const nodes = [
      { id: 1, label: "Student A", submittedAt: "2024-04-30 14:00" },
      { id: 2, label: "Student B", submittedAt: "2024-03-28 13:00" },
      { id: 3, label: "Student C", submittedAt: "2024-04-30 14:30" },
      { id: 4, label: "Student D", submittedAt: "2024-05-01 17:00" },
    ];

    const edges = [
      {
        id: "1-2",
        from: 1,
        to: 2,
        label: "85%",
        value: 0.85,
        width: 8,
        color: { color: "#222", opacity: 0.85 },
      },
      {
        id: "2-3",
        from: 2,
        to: 3,
        label: "60%",
        value: 0.6,
        width: 5,
        color: { color: "#222", opacity: 0.6 },
      },
      {
        id: "1-4",
        from: 1,
        to: 4,
        label: "30%",
        value: 0.3,
        width: 2,
        color: { color: "#222", opacity: 0.3 },
      },
    ];

    const nodeDataSet = new DataSet(nodes);
    const edgeDataSet = new DataSet(edges);

    const data = {
      nodes: nodeDataSet,
      edges: edgeDataSet,
    };

    const options = {
      nodes: {
        shape: "dot",
        size: 20,
        font: { size: 16 },
      },
      edges: {
        font: { align: "top" },
        smooth: true,
      },
      physics: {
        enabled: true,
        stabilization: false,
      },
      interaction: {
        hover: true,
      },
    };

    const network = new Network(containerRef.current, data, options);
```

**nodes 옵션**
`nodes`옵션은 각 노드의 시각적 특성을 정의하며, 주요 속성은 다음과 같다.

- `shape`: 노드의 형태를 지정 ex) `dot`, `circle`, `box` 등
- `size`: 노드의 크기를 설정. 숫자값으로 지정하며, 기본값은 25
- `font`: 노드 라벨의 폰트 스타일을 설정
- `label`: 노드에 표시될 텍스트를 지정
- `color`: 노드의 색상을 설정
- `value`: 노드의 값을 설정하여, `scaling` 옵션과 함께 사용하면 노드의 크기를 자동으로 조절할 수 있다.

**edges 옵션**
`edges` 옵션은 노드 간의 연결선(엣지)의 시각적 특성을 정의하며, 주요 속성은 다음과 같다.

- `value`: 엣지의 가중치를 설정 (두께)
- `label`: 엣지에 표시될 텍스트를 지정
- `font`: 엣지 라벨의 폰트 스타일을 설정
- `color`: 엣지의 색상을 설정

> 이때, vis-network에서 제공하는 `.get()`이나 `.update()` 같은 메서드를 사용하기 위해서는 `nodes`와 `edges` 배열들을 `DataSet`으로 감싸줘야 한다.

**physics 옵션**
`physics` 옵션은 네트워크 그래프의 레이아웃을 결정하는 물리 엔진의 동작 방식을 설정하며, 주요 속성은 다음과 같다.

- `enabled`: 물리 엔진의 활성화 여부를 설정. `true`로 설정하면 노드들이 서로 밀고 당기며 자연스럽게 배치된다.
- `stabilization`: 초기 렌더링 시, 안정화 과정을 설정한다. `false`면 빠르지만 노드가 흔들릴 수 있다.

**interaction 옵션**
`interaction` 옵션은 사용자와 그래프 간의 상호작용 방식을 정의하며, 주요 속성은 다음과 같다.

- `hover`: 마우스 호버 시 노드나 엣지의 반응 여부를 설정, `true`로 설정하면 마우스를 올렸을 때 강조 효과가 나타남
- `dragNodes`: 노드를 드래그하여 위치를 변경할 수 있도록 설정, `true`로 설정하면 사용자가 노드의 위치를 직접 조정할 수 있다.
- `dragView`: 그래프 전체를 드래그하여 이동할 수 있도록 설정
- `zoomView`: 사용자가 그래프를 확대/축소할 수 있도록 설정

위 예시 데이터로 구현한 기본적인 네트워크 토폴로지의 모습은 다음과 같다.

![초기 네트워크 토폴로지](https://velog.velcdn.com/images/siiion/post/44d2348c-75aa-4350-8698-ee47be401c9c/image.png)

## 3. 인터랙션 기능 추가: 마우스 hover 시 정보 표시

노드 위에 마우스 hover 시에는 **해당 노드에 할당된 학생의 제출 시간과 유사한 학생 리스트**를 전부 표시하고,
엣지 위에 마우스 hover 시에는 **해당 엣지로 연결된 두 학생에 대한 정보**를 표시하는 기능을 추가해보았다.

### 이벤트 연결

**노드 이벤트 처리**

```
    // 노드 hover 시 학생 정보 + 유사 학생 리스트 전달
    network.on("hoverNode", function (params) {
      const node = data.nodes.get(params.node);
      const connectedEdges = data.edges.get({
        filter: (e) => e.from === node.id || e.to === node.id,
      });

      const similarStudents = connectedEdges.map((e) => {
        const otherId = e.from === node.id ? e.to : e.from;
        return {
          id: otherId,
          label: data.nodes.get(otherId).label,
          similarity: Math.round((e.value || 0) * 100),
        };
      });

      onPanelUpdate({
        type: "node",
        id: node.id,
        label: node.label,
        submittedAt: node.submittedAt,
        similarStudents,
      });
    });
```

노드 위에 마우스 호버 시 해당 노드와 연결된 엣지를 탐색하여 유사 학생 리스트를 생성하도록 하였다.
`onPanelUpdate`를 통해 상위 컴포넌트(App.js)로 제출 시간 등과 함께 해당 노드에 대한 데이터를 전달한다.

App.js에서는 다음과 같이 사이드패널로 받아온 데이터를 띄워주었다.

```
      <>
        <h3>{panelData.label}</h3>
        <p>제출 시간: {panelData.submittedAt}</p>
        <p>유사 학생 목록:</p>
        <ul>
          {panelData.similarStudents.map((s) => (
            <li key={s.id}>
              {s.label} ({s.similarity}%)
            </li>
          ))}
        </ul>
      </>
```

![인터랙션 기능 추가 - 노드](https://velog.velcdn.com/images/siiion/post/47d5989b-dfa9-49d8-81cc-a645172bfd46/image.png)

**엣지 이벤트 처리**

```
    // 엣지 hover 시 두 학생과 유사도 전달
    network.on("hoverEdge", (params) => {
      const edge = data.edges.get(params.edge);
      const fromNode = data.nodes.get(edge.from);
      const toNode = data.nodes.get(edge.to);

      onPanelUpdate({
        type: "edge",
        fromLabel: fromNode.label,
        toLabel: toNode.label,
        similarity: Math.round((edge.value || 0) * 100),
      });
    });
```

엣지 위에 마우스를 올리면 연결된 두 노드에 대한 정보와 두 노드의 유사도를 상위로 전달한다.

마찬가지로 App.js에서는 다음과 같이 사이드패널에 받아온 데이터를 띄워주었다.

```
      <>
        <h3>학생 간 유사도</h3>
        <p>
          {panelData.fromLabel} ↔ {panelData.toLabel}
        </p>
        <p>유사도: {panelData.similarity}%</p>
      </>
```

![인터랙션 기능 추가 - 엣지](https://velog.velcdn.com/images/siiion/post/ec6ed666-8fd1-4fa4-82a2-becf384b3fab/image.png)

```
    // blur 시 정보 초기화
    network.on("blurNode", () => onPanelUpdate(null));
    network.on("blurEdge", () => onPanelUpdate(null));
```

추가로 UX 개선을 위해 마우스가 노드나 엣지 위에 있지 않은 경우에는 null을 넘겨주어 사이드 패널을 초기화시키고 기본 멘트를 보여주도록 하였다.

```
    <p style={{ color: "#aaa" }}>
      그래프 위 요소에 마우스를 올려보세요
    </p>
```

![인터랙션 기능 추가 - 기본 멘트](https://velog.velcdn.com/images/siiion/post/63b1dc03-6db1-4a1b-8a1f-a393cc4f5fb4/image.png)

## 4. 전체 코드

### SimilarityGraph.js

```
import React, { useEffect, useRef } from "react";
import { DataSet, Network } from "vis-network/standalone/esm/vis-network";

function SimilarityGraph({ onPanelUpdate }) {
  const containerRef = useRef(null);

  useEffect(() => {
    const nodes = [
      { id: 1, label: "Student A", submittedAt: "2024-04-30 14:00" },
      { id: 2, label: "Student B", submittedAt: "2024-03-28 13:00" },
      { id: 3, label: "Student C", submittedAt: "2024-04-30 14:30" },
      { id: 4, label: "Student D", submittedAt: "2024-05-01 17:00" },
    ];

    const edges = [
      {
        id: "1-2",
        from: 1,
        to: 2,
        label: "85%",
        value: 0.85,
        width: 8,
        color: { color: "#222", opacity: 0.85 },
      },
      {
        id: "2-3",
        from: 2,
        to: 3,
        label: "60%",
        value: 0.6,
        width: 5,
        color: { color: "#222", opacity: 0.6 },
      },
      {
        id: "1-4",
        from: 1,
        to: 4,
        label: "30%",
        value: 0.3,
        width: 2,
        color: { color: "#222", opacity: 0.3 },
      },
    ];

    const nodeDataSet = new DataSet(nodes);
    const edgeDataSet = new DataSet(edges);

    const data = {
      nodes: nodeDataSet,
      edges: edgeDataSet,
    };

    const options = {
      nodes: {
        shape: "dot",
        size: 20,
        font: { size: 16 },
      },
      edges: {
        font: { align: "top" },
        smooth: true,
      },
      physics: {
        enabled: true,
        stabilization: false,
      },
      interaction: {
        hover: true,
      },
    };

    const network = new Network(containerRef.current, data, options);

    // 노드 hover 시 학생 정보 + 유사 학생 리스트 전달
    network.on("hoverNode", function (params) {
      const node = data.nodes.get(params.node);
      const connectedEdges = data.edges.get({
        filter: (e) => e.from === node.id || e.to === node.id,
      });

      const similarStudents = connectedEdges.map((e) => {
        const otherId = e.from === node.id ? e.to : e.from;
        return {
          id: otherId,
          label: data.nodes.get(otherId).label,
          similarity: Math.round((e.value || 0) * 100),
        };
      });

      onPanelUpdate({
        type: "node",
        id: node.id,
        label: node.label,
        submittedAt: node.submittedAt,
        similarStudents,
      });
    });

    // 엣지 hover 시 두 학생과 유사도 전달
    network.on("hoverEdge", (params) => {
      const edge = data.edges.get(params.edge);
      const fromNode = data.nodes.get(edge.from);
      const toNode = data.nodes.get(edge.to);

      onPanelUpdate({
        type: "edge",
        fromLabel: fromNode.label,
        toLabel: toNode.label,
        similarity: Math.round((edge.value || 0) * 100),
      });
    });

    // blur 시 정보 초기화
    network.on("blurNode", () => onPanelUpdate(null));
    network.on("blurEdge", () => onPanelUpdate(null));
  }, [onPanelUpdate]);

  return (
    <div
      ref={containerRef}
      style={{ height: "600px", flex: 1, border: "1px solid gray" }}
    />
  );
}

export default SimilarityGraph;
```

### App.js

```
import React, { useEffect, useRef } from "react";
import { DataSet, Network } from "vis-network/standalone/esm/vis-network";

function SimilarityGraph({ onPanelUpdate }) {
  const containerRef = useRef(null);

  useEffect(() => {
    const nodes = [
      { id: 1, label: "Student A", submittedAt: "2024-04-30 14:00" },
      { id: 2, label: "Student B", submittedAt: "2024-03-28 13:00" },
      { id: 3, label: "Student C", submittedAt: "2024-04-30 14:30" },
      { id: 4, label: "Student D", submittedAt: "2024-05-01 17:00" },
    ];

    const edges = [
      {
        id: "1-2",
        from: 1,
        to: 2,
        label: "85%",
        value: 0.85,
        width: 8,
        color: { color: "#222", opacity: 0.85 },
      },
      {
        id: "2-3",
        from: 2,
        to: 3,
        label: "60%",
        value: 0.6,
        width: 5,
        color: { color: "#222", opacity: 0.6 },
      },
      {
        id: "1-4",
        from: 1,
        to: 4,
        label: "30%",
        value: 0.3,
        width: 2,
        color: { color: "#222", opacity: 0.3 },
      },
    ];

    const nodeDataSet = new DataSet(nodes);
    const edgeDataSet = new DataSet(edges);

    const data = {
      nodes: nodeDataSet,
      edges: edgeDataSet,
    };

    const options = {
      nodes: {
        shape: "dot",
        size: 20,
        font: { size: 16 },
      },
      edges: {
        font: { align: "top" },
        smooth: true,
      },
      physics: {
        enabled: true,
        stabilization: false,
      },
      interaction: {
        hover: true,
      },
    };

    const network = new Network(containerRef.current, data, options);

    // 노드 hover 시 학생 정보 + 유사 학생 리스트 전달
    network.on("hoverNode", function (params) {
      const node = data.nodes.get(params.node);
      const connectedEdges = data.edges.get({
        filter: (e) => e.from === node.id || e.to === node.id,
      });

      const similarStudents = connectedEdges.map((e) => {
        const otherId = e.from === node.id ? e.to : e.from;
        return {
          id: otherId,
          label: data.nodes.get(otherId).label,
          similarity: Math.round((e.value || 0) * 100),
        };
      });

      onPanelUpdate({
        type: "node",
        id: node.id,
        label: node.label,
        submittedAt: node.submittedAt,
        similarStudents,
      });
    });

    // 엣지 hover 시 두 학생과 유사도 전달
    network.on("hoverEdge", (params) => {
      const edge = data.edges.get(params.edge);
      const fromNode = data.nodes.get(edge.from);
      const toNode = data.nodes.get(edge.to);

      onPanelUpdate({
        type: "edge",
        fromLabel: fromNode.label,
        toLabel: toNode.label,
        similarity: Math.round((edge.value || 0) * 100),
      });
    });

    // blur 시 정보 초기화
    network.on("blurNode", () => onPanelUpdate(null));
    network.on("blurEdge", () => onPanelUpdate(null));
  }, [onPanelUpdate]);

  return (
    <div
      ref={containerRef}
      style={{ height: "600px", flex: 1, border: "1px solid gray" }}
    />
  );
}

export default SimilarityGraph;
```

## 4. 마무리 및 회고

이번 작업을 통해 처음으로 네트워크 토폴로지를 그려보았는데, vis-network가 생각보다 다루기 쉬워 단순 시각화부터 인터랙션 처리까지 단계적으로 시도해볼 수 있었다.

간단한 샘플 데이터만으로도 충분히 구조적이고 직관적인 그래프를 그릴 수 있었고, 이를 통해 앞으로 구현하게 될 소스코드 유사도 분석 결과 시각화의 큰 틀을 그려볼 수 있었다는 점에서 의미 있는 실습이었던 것 같다.

향후에는 다음과 같은 확장 아이디어도 고려해볼 수 있을 것 같다:

- **유사도 필터 슬라이더 기능 추가**
  특정 임계값 이상의 유사도를 보이는 노드만 확인할 수 있도록 유사도 임계값을 조정하는 슬라이더를 추가하면 좋을 것 같다.

- **학생 수 증가에 따른 최적화 처리 방안**
  지금은 단 4명의 노드만을 다루고 있지만, 실제 프로젝트에서는 수십 명 이상의 학생 데이터를 다룰 가능성이 높다.
  이 경우 vis-network의 기본 설정만으로는 노드가 서로 겹쳐 보이거나, 엣지가 뒤엉켜 시각적으로 복잡해지는 문제, 렌더링 성능이 저하되는 문제 등이 발생할 수 있을 것 같아 이에 따른 최적화 처리 방안이 필요할 것 같다.

> **클러스터링**: 유사한 학생 그룹들을 묶어서 표현하고, 필요할 때만 확장해 보여주는 기능
> **동적 필터링**: 특정 유사도 기준이나 특정 학생 ID 기반으로 네트워크를 실시간 필터링
> **레이블 고정 or 하이라이팅**: 노드가 많아져도 특정 노드의 정보를 강조해서 볼 수 있도록 UI 개선
> **뷰 확대/축소 제어**: 전체를 한눈에 보기보다는 범위 제한 or 뷰포트 내 유도 기능

---

## 참고 자료

[[React] 리액트 그래프/차트 라이브러리 모음](https://velog.io/@eunjin/React-%EB%A6%AC%EC%95%A1%ED%8A%B8-%EA%B7%B8%EB%9E%98%ED%94%84%EC%B0%A8%ED%8A%B8-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC-%EB%AA%A8%EC%9D%8C)
[[React] vis-network를 이용한 네트워크 토폴로지 그리기](https://limsw.tistory.com/144)
[vis-network 공식 문서](https://visjs.github.io/vis-network/docs/network/)
