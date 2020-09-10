---
title: "NodeJS로 WebRTC 시그널링(Signaling) 서버를 만드는 방법."
author: "Kwon Hyungjoo"
date: 2020-09-05 12:00:00 +0800
categories: javascript
tags: javascript, nodejs, webrtc, socketio
toc: true
toc_label: " 목차"
toc_sticky: true
---
# WebRTC 시그널링(Signaling) 서버 만들기

WebRTC를 사용하면서 가장 난해한 부분이 있었다. 바로 시그널링(Signaling) 서버로, 딱히 정해진 내용이 없고 자유롭게 구현하면 된다고 하여 어떻게 시작하면 될지 막막했다. 이 글에서는 최대한 간결하게 요청 수락 / 거절이 있는 1:1 화상 통화를 위한 시그널링 서버를 NodeJS와 Socket.io로 구현해볼 것이다.

## WebRTC 시그널링?

구현에 앞서 WebRTC 시그널링에 대해 간단히 알아보고 가자. 시그널링은 하나 이상의 브라우저간에 P2P 연결을 시작하기 위한 사전 데이터 교환 작업이다. 아주 쉽게 생각하면 서로의 IP를 시그널링 서버를 통해 교환하고, 각 브라우저가 그 IP로 상대방에게 접속한다고 생각하면 된다. 물론 실제로는 조금 더 복잡한 내용이 오가는데, 그 내용은 아래와 같다.

- IP 주소, 사용 가능한 포트 정보 등 네트워크 정보
- 미디어 코덱, 대여폭, 데이터 종류 등의 메타 데이터
- 보안 연결을 위한 암호화 키
- 세션 시작, 종료 등을 위한 명령 메시지

이러한 정보들은 WebRTC를 연결하기 위해 필요한 정보들이며 크게 세션 기술 프로토콜(Session Description Protocol, SDP) 메시지와 대화형 연결 수립(Interactive Connectivity Establishment, ICE) 메시지 2가지로 나누어 사용한다. 먼저 SDP 메시지를 공유하여 서로의 환경을 알아낸 뒤, ICE 메시지를 공유하여 네트워크 연결 수립이 가능한 경로를 찾아서 수립하는 것이다.

{% include figure class="center" image_path="/assets/images/posts/2020-09-05/webrtc-signaling-chart.svg" alt="WebRTC 연결이 수립되는 모습" caption="WebRTC 연결이 수립되는 모습" %}

이 구조를 [자바스크립트 세션 수립 프로토콜(JavaScript Session Establishment Protocol, JSEP)](http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-03#section-1.1)이라고 한다.

## RTCPeerConnection과 시그널링 순서

이제 본격적인 구현을 위해 어떻게 시그널링이 이루어지는지 자세히 살펴보도록 하자. WebRTC에서 다른 사용자와 연결을 하는데 사용하는 API가 바로 RTCPeerConnection이다. 이 API를 통해 SDP 세션 제안(SDP Session Description)을 생성하고 상대방의 SDP를 설정할 수 있다. 물론 ICE 후보(ICE Candidate) 또한 동일한 API를 통해 생성된다. 그 순서를 간단하게 설명하면 다음과 같다.

{% include figure class="center" image_path="/assets/images/posts/2020-09-05/webrtc-signaling-sdp-chart.svg" alt="SDP를 교환하는 순서" caption="SDP를 교환하는 순서" %}

이러한 과정을 거치면 두 사용자간의 local/remote sdp 정보가 일치하게된다. 비로소 WebRTC 연결을 시작할 준비가 끝난 것이다. 물론 SDP 정보와 별도로 ICE 후보를 통해 네트워크 정보를 교환해야 한다.

1. 사용자 1이 SDP 제안을 만들 때 onicecandidate 핸들러를 설정한다.
2. 사용자 1의 네트워크가 준비가 되면 onicecandidate 이벤트가 발생하며 ICE 후보가 만들어진다.
3. 사용자 1은 시그널링 서버를 통해 ICE 후보를 사용자 2에게 보낸다.
4. 사용자 2는 받은 ICE 후보를 addIceCandidate를 통해 등록한다.

이러한 과정을 모두 거치면 성공적으로 WebRTC 연결이 수립될 것이다. 여기서 중요한 것은 시그널링 서버는 하는 것이 사용자끼리 메시지를 전달해주는 것 밖에 없다는 점이다. 사용자가 생성한 SDP, ICE를 다른 지정한 상대방에게 전달해주면 할 일이 끝난다.

## Socket.io를 사용한 시그널링 서버 구현

서버 구현에 앞서 주의해야할 사항이 있다. WebRTC는 https로 접속된 환경에서만 동작한다. 로컬 파일을 실행하면 WebRTC 연결이 되지 않으므로 Express나 다른 라이브러리를 통해 https 서버 또한 같이 실행해야 한다.

### 프로젝트 생성

새로운 폴더를 만들고 npm을 통해 package.json을 생성한 뒤 필요한 라이브러리를 설치하자. 위에서 설명한 대로 https 서버를 올리기 위해 express도 같이 설치한다.

```console
npm init -y
npm install --save express socket.io
```

### 서버 생성

https 서버를 열기 위해서는 먼저 인증서가 있어야 한다. openssl이 설치되었다면 아래 명령을 통해 생성이 가능하다.

```console
openssl req -x509 -newkey rsa:2048 -nodes -keyout cert.key -out cert.crt -days 365
```

인증서의 정보를 넣으라는 내용이 나오면 모두 엔터로 넘겨버려도 문제가 발생하지 않는다.