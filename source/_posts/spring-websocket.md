---
title: Spring WebSocket 소개
date: 2018-03-30 00:29:13
tags: [spring, websocket]
categories: 
  - spring
---

# 개요

Web Browser에서 Request를 보내면, Server는 Response를 준다. HTTP 통신의 기본적인 동작 방식이다. 하지만 Server에서 Client로 특정 동작을 알려야하는 상황도 있다. 예를 들어 Browser로 Facebook에 접속해 있다가, 누군가 친구가 글을 등록하는 경우, 혹은 Web Browser로 메신저를 구현하는 경우다. WebSocket이 있기 전에는 이를 Polling이나 Long polling 등의 방식으로 해결했었다. 하지만 WebSocket의 등장으로 Server-Client 간의 실시간 통신이 가능하게 되면서, 앞으로 Long polling은 역사의 뒤안길으로 사라질 것 같다.

<!-- more -->

WebSocket이란 HTTP 환경에서 전이중 통신(full duplex, 2-way communication)을 지원하기 위한 프로토콜로, [RFC 6455](https://tools.ietf.org/html/rfc6455)에 정의되어 있다. HTTP 프로토콜에서 Handshaking을 완료한 후, HTTP로 동작을 하지만, HTTP와는 다른 방식으로 통신을 한다.

본 문서는 WebSocket을 소개하며, 간단하게 Web Chatting을 지원하는 Application을 작성하여 Spring에서 어떻게 WebSocket을 사용할 수 있도록 지원하는지 소개한다.

# Handshake

WebSocket은 HTTP 기반으로 Handshaking을 한다. 어떠한 방식인지 잠깐 훑어보자.

## Handshake 요청

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: v10.stomp, v11.stomp, my-team-custom
Sec-WebSocket-Version: 13
```

- `Connection: Upgrade` : HTTP 사용 방식을 변경하자.
- `Upgrade : websocket` : WebSocket을 사용하자.
- `Sec-WebSocket-Protocol: xxx, yyy, zzz` : WebSocket을 쓰면서 이 중에서 protocol을 골라서 쓰자.
- `Sec-WebSocket-Key` : 보안을 위한 요청 키.

## Handshake 응답

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `101 Switching Protocols` : Handshake 요청 내용을 기반으로 다음부터 WebSocket으로 통신할 수 있다.
- `Sec-WebSocket-Accept` : 보안을 위한 응답 키 - `base64.encode(Sec-WebSocket-Key.concat(GUID))`

# WebSocket Sevrer를 운용할 때의 유의사항

- HTTP에서 동작하나, 그 방식이 HTTP와는 많이 상이하다.
  - REST한 방식의 HTTP 통신에서는 많은 URI를 통해 application이 설계된다.
  - WebSocket은 하나의 URL을 통해 Connection이 맺어지고, 후에는 해당 Connection으로만 통신한다.
- Handshake가 완료되고 Connection을 유지한다.
  - 전통적인 HTTP 통신은 요청-응답이 완료되면 Connection을 close한다. 때문에 이론상 하나의 Server가 Port 수의 한계(`n<65535`)를 넘는 client의 요청을 처리할 수 있다.
  - WebSocket은 Connection을 유지하고 있으므로, 가용 Port 수만큼의 Client와 통신할 수 있다.

Spring의 지원을 받아 Web Application Server에서 HTTP를 지원하면서 WebSocket도 지원할 수 있다. 하지만 HTTP와 WebSocket의 개념이 많이 상이하다보니, WebSocket을 사용해야 한다면 전용 Server를 구축하는 편이 운영하기 쉬울 것으로 판단된다.

# 언제 쓰면 좋을까?

[Spring Reference](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/web.html#websocket-intro-when-to-use)을 참조하면, `자주 + 많은 양의 + 지연이 짧아야하는 통신`을 할 수록 WebSocket이 적합하다고 설명하고 있다. 주로 채팅이나 게임이 이러한 요구사항을 가질 것이다. 단순한 알림 성격의 뉴스 피드 같은 정보에는 polling이나 streaming 방식이 더욱 단순하고 효율적인 솔루션이 될 수 있다.

# 지원하는 Browser

| ![지원브라우저](support-browsers.png)|
| - |
| https://caniuse.com/#feat=websockets |

> IE는 10부터 지원한다

# SockJS

IE 8, 9는 여전히 많은 인터넷 사용자가 사용하고 있는 브라우저이나, 해당 버전에서는 WebSocket을 지원하지 않는다. Spring에서는 [sockjs-client](https://github.com/sockjs/sockjs-client/)와 합을 맞추어서, 기존 설정에 큰 변경없이, 마치 WebSocket Polyfill을 사용하는 것과 같은 효과를 낸다.

## Spring WebSocket 설정

**WebSocket 기본 설정**

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/myHandler").setAllowedOrigins("*");
    }
}
```

**WebSocket SockJS 설정**

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/myHandler").setAllowedOrigins("*").withSockJS();
    }
}
```

`.withSockJS()`만 추가되었다. Client에서는 `GET /myHandler/info`를 호출해서 Server의 정보를 취득하며, Client의 지원 여부에 따라 Long polling이나 Polling으로 통신한다. Server 측에서는 위 설정 외에는 소스 코드의 변함 없이 운영할 수 있다.

# SockJS 기반 채팅서버 예제

서버에서 미리 정의한 채팅방(`ChatRoom`)에 들어가있는 사용자(`Session`) 간에 채팅을 지원한다.

## Gradle 설정

```gradle
dependencies {
  compile('org.springframework.boot:spring-boot-starter-mustache')
  compile('org.springframework.boot:spring-boot-starter-web')
  compile('org.springframework.boot:spring-boot-starter-websocket')

  compile('org.webjars.bower:jquery:3.3.1')
  compile('org.webjars:sockjs-client:1.1.2')
  compile('org.webjars:webjars-locator:0.30')

  runtime('org.springframework.boot:spring-boot-devtools')
  compileOnly('org.projectlombok:lombok')
  testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## Spring Application 설정

**application.yml**

```yml
spring.mustache.suffix: .html
```

**WebSocketConfig**

```java
@Profile("!stomp")
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Autowired
    private ChatHandler chatHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 해당 endpoint로 handshake가 이루어진다.
        registry.addHandler(chatHandler, "/ws/chat").setAllowedOrigins("*").withSockJS();
    }
}
```

`WebSocketHandlerRegistry`에 `WebSocketHandler`의 구현체를 등록한다. 등록된 Handler는 특정 endpoint(`"/wa/chat"`)로 handshake를 완료한 후 맺어진 connection의 관리한다. `WebSocketHandler`의 구현체(`ChatHandler`)의 내용은 아래에서 살펴보겠다.

## 채팅방 입장 구현

### ChatRoom

```java
@Getter
public class ChatRoom {
    private String id;
    private String name;
    private Set<WebSocketSession> sessions = new HashSet<>();
    
    // 채팅방 생성
    public static ChatRoom create(@NonNull String name) {
        ChatRoom created = new ChatRoom();
        created.id = UUID.randomUUID().toString();
        created.name = name;
        return created;
    }
}
```

채팅방은 `id`, `name`, `sessions`로 구성된다. `WebSocketSession`은 spring에서 WebSocket connection이 맺어진 세션을 가리킨다. 편하게 고수준 `socket`이라고 생각하자. 해당 session을 통해서 메시지를 보낼(`sendMessage()`) 수 있다.

### ChatRoomRepository

```java
@Repository
public class ChatRoomRepository {
    private final Map<String, ChatRoom> chatRoomMap;

    public ChatRoomRepository() {
        chatRoomMap = Collections.unmodifiableMap(
                Stream.of(ChatRoom.create("1번방"), ChatRoom.create("2번방"), ChatRoom.create("3번방"))
                      .collect(Collectors.toMap(ChatRoom::getId, Function.identity())));

        chatRooms = Collections.unmodifiableCollection(chatRoomMap.values());
    }

    public ChatRoom getChatRoom(String id) {
        return chatRoomMap.get(id);
    }
    
    public Collection<ChatRoom> getChatRooms() {
        return chatRoomMap.values();
    }
}
```

테스트를 위한 용도로 UUID로 채팅방 id를 지정하여, 3개의 채팅방을 생성해두었다.

### ChatController

```java
@Controller
@RequestMapping("/chat")
public class ChatRoomController {

    private final ChatRoomRepository repository;
    private final AtomicInteger seq = new AtomicInteger(0);

    @Autowired
    public ChatRoomController(ChatRoomRepository repository) {
        this.repository = repository;
    }

    @GetMapping("/rooms")
    public String rooms(Model model) {
        model.addAttribute("rooms", repository.getChatRooms());
        return "/chat/room-list";
    }

    @GetMapping("/rooms/{id}")
    public String room(@PathVariable String id, Model model) {
        ChatRoom room = repository.getChatRoom(id);
        model.addAttribute("room", room);
        model.addAttribute("member", "member" + seq.incrementAndGet()); // 회원 이름 부여

        return "/chat/room";
    }
}
```

채팅방에 진입을 하기위한 Controller다. `/chat/rooms`를 통해서 채팅방의 목록을 확인할 수 있고, `/chat/rooms/{id}`를 통해서 특정 id의 채팅방에 입장할 수 있다.

### room-list.html

```html
<h1>채팅방 목록</h1>
<ul>
    {{#rooms}}
    <li><a href="/chat/rooms/{{id}}">{{name}}</a></li>
    {{/rooms}}
</ul>
```

### room.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{room.name}}</title>
    <script src="/webjars/jquery/dist/jquery.min.js"></script>
    <script src="/webjars/sockjs-client/sockjs.min.js"></script>
</head>
<body>
<h1>{{room.name}}({{room.id}})</h1>
<div class="content" data-room-id="{{room.id}}" data-member="{{member}}">
    <ul class="chat_box">
    </ul>
    <input name="message">
    <button class="send">보내기</button>
</div>
<script>
    $(function () {
        var chatBox = $('.chat_box');
        var messageInput = $('input[name="message"]');
        var sendBtn = $('.send');
        var roomId = $('.content').data('room-id');
        var member = $('.content').data('member');

        // handshake
        var sock = new SockJS("/ws/chat");

        // onopen : connection이 맺어졌을 때의 callback
        sock.onopen = function () {
            // send : connection으로 message를 전달
            // connection이 맺어진 후 가입(JOIN) 메시지를 전달
            sock.send(JSON.stringify({chatRoomId: roomId, type: 'JOIN', writer: member}));
            
            // onmessage : message를 받았을 때의 callback
            sock.onmessage = function (e) {
                var content = JSON.parse(e.data);
                chatBox.append('<li>' + content.message + '(' + content.writer + ')</li>')
            }
        }

        sendBtn.click(function () {
            var message = messageInput.val();
            sock.send(JSON.stringify({chatRoomId: roomId, type: 'CHAT', message: message, writer: member}));
            messageInput.val('');
        });
    });
</script>
</body>
</html>
```

- `new SockJS("/ws/chat")` : handshake를 한다.
- `sock.onoepn = function() { ... }` : handshake가 완료되고 connection이 맺어지면 실행된다.
- `sock.send(string)` : socket을 대상으로 문자열을 전송한다.
- `sock.onmessage = function(evt) { ... }` : socket에서 정보를 수신했을 때 실행된다. `evt.data`로 정보가 들어온다.

### ChatMessage

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ChatMessage {
    private String chatRoomId;
    private String writer;
    private String message;
    private MessageType type;
}
```

client와 주고 받을 모델이다.

### ChatHandler

```java
@Slf4j
@Profile("!stomp")
@Component
public class ChatHandler extends TextWebSocketHandler {

    private final ObjectMapper objectMapper;
    private final ChatRoomRepository repository;

    @Autowired
    public ChatHandler(ObjectMapper objectMapper, ChatRoomRepository chatRoomRepository) {
        this.objectMapper = objectMapper;
        this.repository = chatRoomRepository;
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {

        String payload = message.getPayload();
        log.info("payload : {}", payload);

        ChatMessage chatMessage = objectMapper.readValue(payload, ChatMessage.class);
        ChatRoom chatRoom = repository.getChatRoom(chatMessage.getChatRoomId());
        chatRoom.handleMessage(session, chatMessage, objectMapper);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        repository.remove(session);
    }
}
```

`ChatHandler`는 `WebSocketHandler`의 구현체이다. `WebSocketHandler`는 다음 메서드를 가지고 있다.

- `afterConnectionEstablished(WebSocketSession session)` : connection이 맺어진 후 실행된다.
- `handleMessage(WebSocketSession session, WebSocketMessage<?> message)` : message 타입에 따라 `handleTextMessage()`, `handleBinaryMessage()`를 실행한다.
  - 본문에서 상속한 `TextWebSocketHandler`는 `handleTextMessage(WebSocketSession session, TextMessage message)`를 실행한다.
- `afterConnectionClosed(WebSocketSession session, CloseStatus status)` : close 이후 실행된다.

message가 들어오는 경우, `message.getChatRoomId()`로 채팅방을 찾아 메시지를 전파한다.

### ChatRoom

```java
@Getter
public class ChatRoom {
    private String id;
    private String name;
    private Set<WebSocketSession> sessions = new HashSet<>();

    public void handleMessage(WebSocketSession session, ChatMessage chatMessage, ObjectMapper objectMapper) {

        if (chatMessage.getType() == MessageType.JOIN) {
            join(session);
            chatMessage.setMessage(chatMessage.getWriter() + "님이 입장했습니다.");
        }
        
        send(chatMessage, objectMapper);
    }

    private void join(WebSocketSession session) {
        sessions.add(session);
    }

    private <T> void send(T messageObject, ObjectMapper objectMapper) {

        TextMessage message = new TextMessage(objectMapper.writeValueAsString(messageObject));
        sessions.parallelStream().forEach(session -> session.sendMessage(message));
    }
}
```

## 실행

![스크린샷1](websocket-example1.png)