---
title: 가끔 복기할 만한 TCP Socket Programming 기초
date: 2018-03-15 10:51:24
tags: [java,socket]
categories:
  - etc.
---

# 개요

Spring에서 지원하는 Streaming이나, WebSocket에 관련해서 공부하다가, 기본 지식을 다시 복습해야할 필요성을 느꼈다.
대학 강의 내용을 토대로 가끔씩 다시 떠올릴만한 내용들을 정리해두고자 한다.

# Socket

Kernel 상에서 File Descriptor로 취급된다.
Local과 Remote의 Ip Address, Port 정보를 가지고 있다.
Data를 이 Socket을 대상으로 Read, Writer한다.

# Linux Socket Programming

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main(int argc, char *argv[]) {
  int server_socket;
  int client_socket;
  int port = 8080;
  struct sockaddr_in server_address;
  struct sockaddr_in client_address;
  int client_address_len = sizeof(client_address);
  
  char[] greeting = "Hello World!";
  
  server_socket = socket(PF_INET, SOCK_STREAM, 0); // 1

  memset(&server_address, 0, sizeof(server_address))
  serv_addr.sin_family = AF_INET;
  serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_addr.sin_port = htons(5000); 
  
  bind(server_socket, (struct sockaddr*) &server_address, sizeof(server_address)); // 2

  listen(server_socket, 1); // 3
  
  client_socket = accept(server_socket, (struct sockaddr*)&client_address, (socklen_t*) &client_address_len); // 4
  
  write(client_socket, greeting, sizeof(greeting)); // 5
  close(client_socket);
  close(server_socket);
  return 0;
}
```

Java는 위 작업들을 `new ServerSocket()`만으로도 자동으로 해주기 때문에.
Server가 Socket 통신할 때, 저수준에서 어떠한 작업들을 하는지 파악하기 위해 C로 작성해보았다.

1. socket(file descriptor)를 생성한다
  - 프로토콜 체계, TCP인지, UDP인지 결정한다
2. socket에 address 정보를 바인딩한다
3. 해당 socket이 요청을 받을 수 있도록한다
4. server socket으로 연결(connection) 요청이 들어오면, accept()는 client의 socket을 반환한다. 연결이 있을 때까지 blocking된다
5. client socket에 메시지를 쓴다

## 좀 더 살펴보기

### listen(int socket, int backlog)

`socket`에 connection queue를 붙이고 connection 대기 상태로 바꾼다.
`backlog`는 queue의 크기를 나타낸다

### client_socket = accept(server_socket, ...)

connection queue의 요청으로 connection을 맺은 후 새로운 socket을 생성해서 반환한다.
`server_socket`은 계속해서 connection 요청을 처리해야하기 때문에, connection 요청이 처리된 경우에는 새로운 socket을 생성하게 된다.
새로 생성된 socket으로 client와 통신할 수 있다.

# Socket Stream

socket을 생성하고 connection이 수립되면, 해당 소켓으로 읽기와 쓰기가 가능하다.
읽기와 쓰기는 각각의 Stream 개념으로 유지되며, 버퍼를 가진다.
`write()`를 호출하면 입력 버퍼로 데이터가 쌓이며, 원격에서 `read()`를 호출하면 데이터를 읽을 수 있다.

오해하기 쉬운 것은 데이터를 보내는 시점이다.
`write()`를 호출한다고 전송을 하는 것이 아니며, `read()`를 호출한다고 원격의 버퍼를 읽어오는 것이 아니다.

`write()`를 호출하면 socket의 쓰기 버퍼에 정보가 쌓인 상태로 있고, 원격에서 읽기 버퍼에 데이터를 쌓을 수 있는 상황인지 파악 후 통신을 시작한다.
[TCP Sliding Window](http://www.omnisecu.com/tcpip/tcp-sliding-window.php)가 이를 가능하게 해준다.

특정 socket에 read stream만 닫는다거나, write stream만 닫는 등의 방법으로 다양한 동작 방식으로 socket programming을 할 수 있다.(read thread와 write thread의 분할 등)

# Hand Shake, Send/Receive, Close

## Hand Shake : 3-Way HandShaking

TCP socket connection을 맺기 위한 일련의 과정이다.

1. client : 들려?
2. server : 어, 들려!
3. client : Ok, 알았어!

{% plantuml %}
Client -> Server : [SYN] SEQ: 100, ACK: -
Server -> Client : [SYN+ACK] SEQ: 200, ACK: 101
Client -> Server : [ACK] SEQ: 101, ACK: 201
{% endplantuml %}

**SEQ, ACK의 의미**

각자 자기 입장에서

- SEQ : N까지 보냈다
- ACK : XX부터 보내면 된다(자신이 받은 SEQ + 1)

## Send/Receive

HandShake가 완료된 이후,
client에서 300byte를 세 번 전송하는 다이어그램.

{% plantuml %}
Server -> Client  : SEQ: 200, 100-byte 
Client -> Server : ACK: 301
Server -> Client : SEQ: 301, 100-byte
Client -> Server : ACK: 402
== timeout start ==
Server ->X Client : SEQ: 402, 100-byte
== timeout end ==
Server -> Client : SEQ: 402, 100-byte
Client -> Server : ACK: 503
{% endplantuml %}

SEQ, ACK를 주고 받으면서 송수신한다.
timeout 내에 ACK가 오지 않으면 재전송한다.

## Close : 4-Way Handshaking

1. client : 이제 우리 그만 만나...
2. server : 그래, 맘 정리할 시간을 줘
3. server : 정리다했어, 그동안 미안했어
4. client : 그래 안녕

{% plantuml %}
Client -> Server : [FIN] SEQ: 1000, ACK: -
note left: ACK 올때까지 FIN_WAIT_1
Server -> Client : [ACK] SEQ: 2000, ACK: 1001
note left: FIN 올때까지 FIN_WAIT_2
note right: FIN 보낼때까지 CLOSE_WAIT
Server -> Client : [FIN] SEQ: 2001, ACK: 1001
note right: ACK 올때까지 LAST_ACK
Client -> Server : [ACK] SEQ: 1001, ACK: 2002
note left : 일정시간 TIME_WAIT\n그 후 CLOSED
{% endplantuml %}

### 우아한 종료란?

TCP가 비정상적으로 종료될 가능성이 몇가지 있다.

#### case 1: 개발자 실수(close 호출)

socket을 통해서 read, write stream이 생성된다.
`close(socket)`을 호출하면, 모든 stream을 닫아서 읽기가 불가능해진다.
즉, 한 쪽에서 `close(socket)`을 호출하면 아래와 같은 상황이 된다.

여기서 client란 먼저 FIN을 요청한 쪽으로 정의하겠다.

{% plantuml %}
== client에서 close(socket) 호출 ==
Client -> Server : [FIN] SEQ: 1000, ACK: -
Server -> Client : [ACK] SEQ: 2000, ACK: 1001
Server -> Client : [FIN] SEQ: 2001, ACK: 1001
== 감감 무소식 ==
{% endplantuml %}

##### Half-Close

이러한 경우는 client에서 write stream만 먼저 종료시키는 Half-close 개념을 사용해서 해결할 수 있다.
write stream이 종료되어도 `close()`가 호출될 때, server로 ACK가 전송된다.

#### case 2: Client가 FIN을 보낸 후, Server로부터 ACK없이 FIN을 받는 경우

- 가능성 1 : Server측 ACK가 네트워크상 유실됨
- 가능성 2 : Network Congestion 등의 문제로 FIN이 ACK보다 먼저 도착

**도식으로 나타내면 다음과 같다**

{% plantuml %}
Client -> Server : [FIN] SEQ: 1000, ACK: -
Server ->X Client : [ACK] SEQ: 2000, ACK: 1001
Server -> Client : [FIN] SEQ: 2001, ACK: 1001
Client -> Server : [ACK] SEQ: 1001, ACK: 2002
note left : 일정시간 TIME_WAIT\n그 후 CLOSED
{% endplantuml %}

##### TIME_WAIT

4-Way Handshake로 종료하도록 TCP는 정의되어 있다.
`FIN_WAIT_1` 상태에서 FIN을 먼저 받을 경우, 패킷 흐름의 무결성을 지키기 위해 `TIME_WAIT`동안 패킷를 기다린다.
패킷이 도착하거나, `TIME_WAIT` 시간이 경과하면 `CLOSED`로 바뀐다.
`TIME_WAIT`는 먼저 `FIN`을 호출한 쪽에서 생겨야한다!!!

**TCP State Diagram**

![Tcp state Diagram](https://upload.wikimedia.org/wikipedia/en/5/57/Tcp_state_diagram.png)

**Sokcet Reuse**

기본적으로 `SO_RESUSEADDR`은 `false`로 설정되어 있으며,
`true`로 설정하면 `TIME_WAIT` 상태의 포트를 사용할 수 있게 된다.

