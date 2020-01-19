---
layout: post
title: WebSocket 세션 연결 후 1분 뒤 자동 disconnect
tag: Websocket, Android, Java

---

WebRTC 영상통화의 연결/통화종료/SDP/Candidate 메시지 교섭을 위해 클라이언트는 시그널링 서버의 Websocket세션 생성을 요청하고 이를 통해 통신한다. 

현재 영상통화 앱에서 WebSocket을 통해 성공적으로 모든 교섭을 마치고 WebRTC 영상통화를 하는 도중 Client 측에서 다음과 같은 로그를 남기고 WebSocket세션이 끊기는 현상을 발견했다.

```java
01-16 11:20:44.579 14044 14842 D de.tavendo.autobahn.WebSocketReader: run() : ConnectionLost
01-16 11:20:44.580 14044 14842 D de.tavendo.autobahn.WebSocketReader: WebSocket reader ended.
01-16 11:20:44.584 14044 14837 D de.tavendo.autobahn.WebSocketConnection: fail connection [code = CONNECTION_LOST, reason = WebSockets connection lost
01-16 11:20:44.584 14044 14837 D de.tavendo.autobahn.WebSocketReader: quit
01-16 11:20:44.585 14044 14843 D de.tavendo.autobahn.WebSocketWriter: WebSocket writer ended.
01-16 11:20:44.586 14044 14837 D WebSocketChannelClient: WebSocket connection closed. Code: CONNECTION_LOST. Reason: WebSockets connection lost. State: REGISTERED
01-16 11:20:44.586 14044 14837 D de.tavendo.autobahn.WebSocketConnection: worker threads stopped
01-16 11:20:44.591 14044 14841 D de.tavendo.autobahn.WebSocketConnection: SocketThread exited.
```

WebSocket 세션이 WebRTC 영상통화 자체에는 영향이 없었지만 통화를 종료를 하거나 추가 참여자에 대한 통지를 송수신하기 위해서는 필수였다.

패킷 분석을 해본결과 정상적인 WebSocketConnection Close 핸드셰이크이 없이 비정상적으로 종료한 경우라 패킷을 통해 어느쪽에서 (서버, 클라) 먼저 세션을 종료한지 파악하기 어려웠다.

조금 더 관찰해보니 세션을 통해 마지막으로 메시지를 보내고 정확하게 1분 뒤 세션이 끊기는 것을 확인했다.

```java
01-16 11:19:44.788 14044 14837 D maxst:WebSocketChannelClient: Send WS Message kkj C->WSS: {"candidates":{"mainCandidate":{"candidate":"candidate:414306594 1 udp 41885439 116.122.157.158 18588 typ relay raddr 222.106.175.59 rport 56614 generation 0 ufrag PRl3 network-id 3 network-cost 10","id":"0","label":0,"type":"candidate"}},"messageType":"notification","methodName":"candidate","peerId":"ae7f42d6-686e-449d-a745-a4c13b039edc","sessionId":"1_a243d30865604c1494ad9ac8735c39dc","uuid":"e7ad778a-1c8f-466e-bead-96bd49f8ea33"}
---
01-16 11:20:44.579 14044 14842 D de.tavendo.autobahn.WebSocketReader: run() : ConnectionLost
```

이걸 봐서는 비정상적인 버그가 아닌 의도된 disconnect라 보여 클라이언트가 사용하고 있는 WebSocket library tavendo.autobahn를 분석해보았다. 해당 library는1분 동안 메시지를 read 하지 않으면 -1를 reader에 전달해 timeout 세션 종료를 하고 있는 것을 발견했다. 세션을 유지하기 위해서는 일종의 keepalive 메시지를 서버와 주기적으로 통신이 필요했고 RFC 문서를 통해 WebSocket은 [5.5.2 Ping](https://tools.ietf.org/html/rfc6455#section-5.5.1),  [5.5.3 Pong](https://tools.ietf.org/html/rfc6455#section-5.5.3)이라는 Control Frame을 사용해서 세션을 유지하여야 했다.

결론적으로 서버와 Ping/Pong 메시지를 주고 받아야 했고 이를 구현해야 되었다. 추가로 구글링을 해본 결과, 사용하고 있던 Android WebSocket library가 너무 구버전이었고 Auto-ping 지원을 하지 않아 최신 버전인 autobahn.crossbar.io로 library를 업그레이드를 했다.

 WebSocket 세션을 생성할 때 ping interval를 설정해주면 자동으로 Ping Control Frame을 전송해준다.

```java
...
Log.d(TAG, "Connecting WebSocket to: " + socketUrl);
WebSocketConnection ws = new WebSocketConnection();
wsObserver = new WebSocketObserver();
try {
  WebSocketOptions opt = new WebSocketOptions();
  opt.setAutoPingInterval(20);
  opt.setAutoPingTimeout(10);
  ws.connect(socketUrl, wsObserver, opt);
} catch (WebSocketException e) {
  reportError("WebSocket connection error: " + e.getMessage());
}
```

추가로 Signaling 서버는 org.springframework.web.socket의 AbstractWebSocketHandler에 구현된 인터페이스로 Ping/Pong 메시지를 처리하고 있다.

이렇게 클라이언트 측 소스를 간단히 수정으로 하고 나니 5분 이상 영상통화 aging test를 해도 아무 문제 없었고 Ping/Pong 패킷이 정상적으로 수신되고 있는 것을 확인할 수 있었다.

```
No.	Time	Source	Destination	Protocol	Length	Info
2128	6.680654368	116.122.157.151	218.145.218.25	WebSocket	506	WebSocket Text [FIN] 
7251	42.391746031	218.145.218.25	116.122.157.151	WebSocket	72	WebSocket Ping [FIN] [MASKED]
7256	42.392797223	116.122.157.151	218.145.218.25	WebSocket	68	WebSocket Pong [FIN] 
7331	42.635392157	218.145.218.25	116.122.157.151	WebSocket	72	WebSocket Ping [FIN] [MASKED]
7336	42.636439465	116.122.157.151	218.145.218.25	WebSocket	68	WebSocket Pong [FIN] 
```

