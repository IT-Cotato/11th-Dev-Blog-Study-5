# WebSocket + STOMP + SockJS로 실시간 채팅 구현하기

최근 대규모 채팅 서비스를 구축하는 프로젝트에 참여하게 되었고, 실시간 통신을 위해 **웹소켓(WebSocket)** 을 사용하게 되었다.

나는 인프라와 프론트엔드를 담당했는데, 웹소켓은 이번이 처음이어서 직접 공부하고 정리해보는 것이 좋겠다고 생각했다.

이번 글에서는 내가 왜 Websocket + SockJS + STOMP를 사용했는지, 어떻게 사용했는지를 중점으로 작성했다.

실시간 통신 방식 전반에 대해서는 아래 글에서 이미 다루었기 때문에, 이 글에서는 생략하도록 하겠다.

[https://yongaricode.tistory.com/7](https://yongaricode.tistory.com/7)

---

## 🔌 웹소켓이란?

웹소켓(WebSocket)은 **클라이언트와 서버 간의 지속적인 연결을 유지할 수 있게 해주는 통신 프로토콜**이다.

일반적인 HTTP 통신에서는 클라이언트가 요청을 보낼 때만 서버가 응답할 수 있다. 하지만 채팅처럼 서버가 먼저 클라이언트에 메시지를 전달해야 하는 상황에서는 이러한 방식으로는 한계가 있다.

웹소켓을 사용하면 클라이언트와 서버가 **한 번 연결된 후, 양방향으로 자유롭게 데이터를 주고받을 수** 있다. 따라서 채팅 서비스처럼 실시간성이 중요한 기능에서 매우 유용하게 활용된다.

## Socket.io를 선택하지 않은 이유

### Socket.io

![](https://blog.kakaocdn.net/dn/xbFsc/btsOk9VOggY/3EKXxAddiSg8ahLkQLGdK0/img.webp)

웹소켓을 처음 공부하면서 찾아본 바로는, 실제로는 ws보다는 **Socket.IO**를 더 많이 사용하는 경우가 많았다.

Socket.IO는 웹소켓을 **더 안전하고 편리하게 사용할 수 있도록 도와주는 라이브러리**다.

사용하기 쉬운 API는 물론, 자동 재연결, 백오프(Backoff), 이벤트 기반 인터페이스 등 웹소켓 위에서 동작하는 다양한 부가 기능들을 제공한다.

덕분에 프론트엔드와 백엔드 모두에서 실시간 통신 기능을 훨씬 수월하게 구현할 수 있다.

---

### Websocket ≠ Socket.io

하지만 웹소켓은 Socket.io와 같지 않다!

- **웹소켓(WebSocket)** 은 하나의 **통신 프로토콜**이고,
- **Socket.IO**는 그 위에서 동작하는 **라이브러리이자 프레임워크**다.

즉, Socket.IO는 내부적으로 웹소켓을 사용할 수 있지만, 웹소켓만 사용하는 것은 아니다.

상황에 따라 HTTP Long Polling 등의 다른 통신 방식으로도 동작할 수 있다. (Long Polling이 궁금하다면 위에 첨부해놓은 글을 참고)

따라서 "웹소켓을 사용했다"와 "Socket.IO를 사용했다"는 표현은 정확히는 다른 의미다.

많은 블로그나 자료들에서 이 둘을 동일하게 다루는 경우가 많은데, 이는 웹소켓을 처음 접하는 사람들이 대부분 Socket.IO를 통해 시작하기 때문에 생기는 일종의 오해로 보인다.

또한 Socket.IO는 각 메시지에 추가적인 메타데이터를 붙이기 때문에, 순수 WebSocket 클라이언트와는 호환되지 않는다.

즉:

- Socket.IO 서버는 일반 WebSocket 클라이언트로는 접근할 수 없고,
- 반대로 일반 WebSocket 서버도 Socket.IO 클라이언트와 호환되지 않는다.

---

### Socket.io의 다양한 기능들

그 외에도 Socket.io는 Websocket에는 없는 다양한 부가 기능을 제공한다.

- HTTP Long-Polling Fallback
- 패킷 버퍼링(Packet Buffering)
- 네임스페이스와 룸 기반 브로드캐스팅 지원 등

이런 다양한 기능들 덕분에 **Socket.IO가 웹소켓보다 더 편리하게 느껴지는 경우가 많고**, 실제로 많은 서비스에서 사용되고 있다.

우리 팀이 만든 서비스는 **단체 채팅만 존재**했기 때문에, **Socket.IO가 기본적으로 제공하는 브로드캐스트 기능이 특히 매력적으로 보였다.**

그래서 나 역시 처음엔 Socket.IO를 사용하고 싶었다.

---

### Socket.IO의 브로드캐스트는 "단일 서버 환경"에서만 기본 제공된다!!

하지만 Socket.IO의 브로드캐스트는 "단일 서버 환경"에서만 기본 제공된다.

**우리는 Auto Scaling을 두어, 멀티 서버 환경**(예: Auto Scaling, 여러 EC2 인스턴스)인데, 이런 환경에서는 서버 간 연결 상태를 공유해야 하기 때문에, **Redis 등 pub/sub 백엔드가 반드시 필요하다.**

왜냐하면…

- Socket.IO 자체만으로는 **다른 서버의 연결 정보를 알 수 없기 때문**이다.
- 서버들끼리 pub/sub 시스템을 통해 연결 상태를 **공유해야** 한다.

이 정도는 괜찮았다. 어차피 우리는 Redis pub/sub을 사용할 예정이었기 때문이다.

하지만 문제는 여기서 끝나지 않았다.

**Redis** 는 단순히 저장소 역할만 하므로, Socket.IO와 함께 제대로 동작시키려면 **socket.io-redis 어댑터**(또는 호환 어댑터)를 따로 연결해야 한다.

### 그런데…

우리 백엔드는 Spring Boot(Java) 기반이었고, Spring Boot에서 Socket.IO를 사용할 경우에도 Redis 어댑터가 필요하다.

그런데 Java용 Socket.IO에는 공식 Redis 어댑터가 존재하지 않는다. 즉, 직접 Redis Pub/Sub 연동을 구현해야 했다.

그래서 결국 팀(백엔드)에서 Socket.IO 대신 순수 WebSocket + STOMP + SockJS를 사용하기로 결정했다.

Socket.IO는 프론트와 백엔드가 같은 방식으로 구현되어야 정상적으로 작동하기 때문에, 백엔드가 Java(Spring) 기반인 우리 팀 환경에서는 사용이 어려웠다.

그래서 프론트에서도 자연스럽게 WebSocket + SockJS + STOMP 조합을 사용하게 되었다.

---

## 🪢 WebSocket + STOMP + SockJS

각 요소가 어떤 역할을 하는지 간단히 정리해보면 아래와 같다:

### WebSocket

![](https://blog.kakaocdn.net/dn/b39Jtp/btsOkQa5LzD/poONvqnVTHQ1yqQkocvkQ1/img.png)

- **기본적인 실시간 통신을 가능하게 해주는 프로토콜**
- 클라이언트와 서버 간 **양방향 통신**을 지원
- HTTP보다 가볍고, 연결을 유지한 채 실시간으로 데이터를 주고받을 수 있음

> 하지만 WebSocket만으로는 메시지를 어떤 주제로 보낼지, 구독/발행을 어떻게 처리할지 등의 고수준 기능이 없다.

---

### STOMP (Simple Text Oriented Messaging Protocol)

![](https://blog.kakaocdn.net/dn/ETJwo/btsOmHQ3tV4/XBFAzuTkErKKoQeBknpqN1/img.png)

- WebSocket 위에서 동작하는 **메시징 프로토콜**
- 마치 **HTTP의 REST처럼**, 메시지를 목적지(/topic/, /queue/ 등) 단위로 주고받을 수 있게 해줌
- 클라이언트는 특정 채널을 **구독(subscribe)** 하고, 서버는 그 채널에 **메시지를 발행(publish)** 함

> 즉, STOMP 덕분에 메시지 브로커 스타일의 통신 구조를 만들 수 있다.

---

### SockJS

![](https://blog.kakaocdn.net/dn/bPSehv/btsOmu5pogB/6PzAHfKTQyfjKxyBzkvMJ1/img.png)

- **WebSocket이 지원되지 않는 환경**을 위한 **폴백(fallback) 라이브러리**
- WebSocket을 기본으로 사용하지만, 브라우저나 네트워크 환경이 지원하지 않으면:
  - XHR streaming
  - XHR polling 등으로 자동 전환됨
- 실질적으로는 **WebSocket을 감싸는 안정성 레이어**

> 회사 네트워크, 방화벽 등으로 인해 WebSocket이 막힌 환경에서도 통신이 가능하게 해준다.

---

## 💻 구현

### 웹소켓 연결

당연한 이야기지만, 실시간으로 메시지를 주고받기 위해서는 먼저 WebSocket 연결을 맺어야 한다.

우리는 백엔드에서 만들어둔 WebSocket 엔드포인트에 클라이언트가 연결을 시도하고, 이때 헤더에 인증 토큰을 포함해서 함께 보낸다.

또한 이 연결 과정에서는 재연결 딜레이, 하트비트 설정(연결이 정상적으로 살아 있는지 주기적으로 확인하는 기능) 등 다양한 옵션들을 커스터마이징할 수 있다.

---

### WebSocket 연결 시 인증은 "한 번만"

HTTP REST와 WebSocket의 가장 큰 차이 중 하나는 **인증 방식**이다.

- **HTTP는 Stateless 프로토콜**이기 때문에,→ 요청할 때마다 매번 사용자 정보를 (예: 토큰) 함께 보내야 한다.
- 반면, **WebSocket은 Stateful 프로토콜**이기 때문에,→ **처음 연결할 때만 인증 정보를 보내면 된다.**

연결이 성립된 후에는 서버가 사용자 정보를 **세션 속성에 저장하고**, **이후의 메시지 처리에서 계속 재사용**할 수 있기 때문이다.

---

### 연결 코드 예시 (STOMP + SockJS)

```
const stompClient = new Client({
    webSocketFactory: () =>
      new SockJS(`${import.meta.env.VITE_BASE_URL}/ws-chat`),
    connectHeaders: {
      Authorization: `Bearer ${token}`,
    },
    reconnectDelay: 5000,
    heartbeatIncoming: 10000,
    heartbeatOutgoing: 10000,
    debug: (str) => console.log("[STOMP]", str),
  });
```

이렇게 설정하면,

- 처음 연결 시 백엔드에서 토큰을 검증하고,
- 성공하면 해당 사용자의 세션을 유지하면서 이후의 통신을 이어갈 수 있다.
- 네트워크가 끊기더라도 reconnectDelay 설정 덕분에 자동으로 재연결된다.

---

### 웹소켓 구독

WebSocket 연결이 성공한 후, 다음 단계는 **서버에서 보내는 메시지를 "구독"하는 것**이다.

앞서 말했듯이, STOMP는 WebSocket 위에서 동작하는 메시징 프로토콜로, /topic/, /queue/ 등 "목적지(destination)" 단위로 메시지를 주고받을 수 있게 해준다.

- 클라이언트는 특정 채널(=목적지) 을 **구독(subscribe)** 하고,
- 서버는 해당 채널에 메시지를 **발행(publish)** 한다.

---

우리는 채팅 서비스를 만들었고, 사용자 유형은 다음처럼 나뉜다:

- 팬(Fan): 아티스트와 자신의 메시지만 볼 수 있음
- 아티스트(Artist): 모든 메시지를 볼 수 있음

따라서 구독 대상도 역할에 따라 다르게 설계했다.

| 아티스트 | 아티스트 메시지, 팬 메시지 전체   |
| -------- | --------------------------------- |
| 팬       | 아티스트 메시지, 자기 자신 메시지 |

---

### 구독의 이점

한 번 구독을 설정해두면, **서버에서 해당 목적지로 보내는 모든 메시지를 자동으로 받을 수 있다.**

이는 HTTP polling처럼 계속 요청을 보내지 않아도 된다는 점에서 성능상 매우 유리하다.

---

### 구독 코드 예시 (STOMP 구독)

```
 stompClient.onConnect = () => {
    const destinations: string[] = [];

    if (isOwner) {
      destinations.push(
        `/topic/chatrooms/${chatRoomId}/messages/artist`,
        `/topic/chatrooms/${chatRoomId}/messages/fans`
      );
    } else {
      destinations.push(
        `/topic/chatrooms/${chatRoomId}/messages/artist`,
        `/topic/chatrooms/${chatRoomId}/messages/user/${userId}`
      );
    }

    destinations.forEach((destination) => {
      stompClient.subscribe(destination, (message: IMessage): void => {
        try {
          const payload: ChatMessage = JSON.parse(message.body);
          onMessage(payload);
        } catch (err) {
          console.error("[STOMP] 메시지 파싱 에러:", err);
        }
      });
    });
  };
```

###

- 구독 대상은 사용자 역할에 따라 동적으로 다르게 구성
- stompClient.subscribe() 를 사용해 STOMP 채널 구독
- 받은 메시지는 콜백 함수(onMessage)로 처리
- 메시지는 JSON 형태로 오기 때문에 반드시 JSON.parse() 필요

---

### WebSocket 연결을 훅으로 사용

앞서 살펴본 WebSocket 연결, STOMP 설정, 메시지 구독 등을 하나의 React 훅으로 묶어서 useStompClient라는 커스텀 훅으로 정리했다.

이 훅은 채팅방 컴포넌트에서 간단히 호출만 하면,

- WebSocket 연결을 맺고
- 메시지 구독을 설정하고
- 메시지를 받아 처리하며
- 컴포넌트가 언마운트되면 자동으로 연결을 해제한다.

---

### 실제 사용 예시 (채팅방 컴포넌트)

```
const stomp = createStompClient({
      token,
      chatRoomId: roomId,
      isOwner,
      userId: userId ?? undefined,
      onMessage: (msg: ChatMessage) => {
        addMessage({
          content: msg.content,
          isOwner: msg.isOwner,
          createdAt: msg.createdAt,
          nickname: msg.nickname,
        });
      },
    });

    stomp.activate();
    stompRef.current = stomp;

    return () => {
      stomp.deactivate();
    };
```

---

### 웹소켓 발행

**메시지 전송(=발행, publish)** 은 ChatInput 컴포넌트에서 처리했다.

채팅방 컴포넌트에서 stompClient 인스턴스를 props로 넘겨주고, ChatInput에서는 이를 활용해 STOMP로 메시지를 서버에 발행한다.

---

### 컴포넌트 간 구조

- ChatRoom (상위 컴포넌트)
  - WebSocket 연결 및 stompClient 관리
  - ChatInput에게 stompClient={stompRef.current}로 전달
- ChatInput (입력 컴포넌트)
  - 메시지를 입력하고 발행 처리

---

### 발행 코드

```
if (stompClient && stompClient.connected) {
      stompClient.publish({
        destination: `/app/chatrooms/${roomId}/messages`,
        body: JSON.stringify(messagePayload),
        headers: {
          "content-type": "application/json",
        },
      });
    } else {
      console.warn("[STOMP] 연결되지 않았습니다.");
    }
```

### 발행 흐름 요약

- 먼저 stompClient가 **정상적으로 연결되어 있는지 확인** (connected 체크)
- 메시지를 보낼 목적지(destination)를 지정 → 보통 /app/으로 시작하는 경로
  - destination은 구독과 다르다.
  - **발행은 /app/...**, **구독은 /topic/... 으로 나뉜다.**
  - 이때 목적지(/app/...)는 서버의 @MessageMapping 핸들러와 매핑된다.
- 메시지 본문은 JSON.stringify(...)로 직렬화하여 body에 전달

---

## 📺 예시 화면

![](https://blog.kakaocdn.net/dn/CqFQz/btsOlVPYNUE/ESivuoaTxbU2zhrwpJ3bs1/img.jpg)

위 구독에서 설명한 것 처럼 아티스트는 팬/아티스트를 구독해 모든 메세지를 볼 수 있고, 팬은 본인/아티스트를 구독해, 본인 메세지와 아티스트 메세지만 보이는 화면이다.

---

## 🥲 아쉬운 점

이번 프로젝트에서 아쉬웠던 부분 중 하나는, 유저 정보를 모두 로컬 스토리지에 저장하는 구조였다.

문제는 채팅방을 나가면 이 정보가 정상적으로 삭제되어야 하는데, 실제로는 WebSocket 연결 실패 → XHR fallback이 발생하면서,

deactivate() 함수가 예상치 않게 호출되고, 그 시점에 로컬 스토리지의 유저 정보가 조기에 삭제되는 문제가 있었다.

원인은 아마 ALB에서 WebSocket 연결이 불안정했던 것으로 보이지만, 프로젝트 기한이 짧았고… 비용 문제도 있었다.

> (3팀이 함께 썼는데, 3주 만에 300달러가 넘게 나왔다고... 아무래도 우리 팀이 범인인 듯... 😅)

결국 프로젝트 발표 당일 AWS 계정이 삭제되는 바람에, 더 이상 원인을 확인하거나 개선할 수는 없었다.

지금도 딱 떠오르는 해결책은 없지만, WebSocket의 연결 상태를 더 정확히 감지하거나, 삭제 트리거를 좀 더 명확하게 분리하는 방식으로 구현했어야 하지 않았을까 싶다.

이 부분은 다음에 비슷한 실시간 시스템을 구현할 기회가 생긴다면 꼭 다시 고민해보고 싶은 과제다.

---

## ❗️ 느낀 점

WebSocket은… 정말 신세계였다. 응답 속도가 믿기지 않을 정도로 빨랐고, 실시간이라는 게 이렇게 다르구나 싶었다.

물론 실시간이니까 당연한 걸 수도 있지만… 정말 놀랐다.

간단한 부하 테스트도 해봤는데,

- 1:1 채팅에서는 평균 지연이 약 12ms,
- 한 채팅방에 300~400명이 동시에 접속해도 2초 이내로 응답되었다. (자세한 내용은 [부하 테스트 자세히 보기](https://yongaricode.tistory.com/10#%EB%B2%88%EC%99%B8-%EB%B6%80%ED%95%98%ED%85%8C%EC%8A%A4%ED%8A%B8-1-8)
   참고)

Socket.IO를 실제로 사용해보지 못한 건 조금 아쉬웠다.

하지만 그 덕분에 오히려 ws + STOMP + SockJS 구조를 더 공부할 수 있었고, WebSocket 자체에 대한 이해도도 조금 올라간 것 같다.

특히 이 프로젝트를 하면서 정보통신공학/컴퓨터네트워크 수업 시간에 배웠던 것들이 자꾸 떠올라서, 왠지 괜히 뿌듯했다. 공부가 헛되지 않았구나...😄

이번엔 시간이 촉박했고, 인프라도 함께 맡다 보니 프론트엔드 쪽에 충분히 신경을 쓰지 못한 부분이 아쉬웠다. 다음에 웹소켓을 사용한다면 한번 해봤으니까 다음엔 좀 더 잘 구현해보고 싶다~
