---
title: "NodeJS로 WebRTC 시그널링(Signaling) 서버를 만드는 방법."
author: "Kwon Hyungjoo"
date: 2020-09-05 12:00:00 +0800
categories: javascript
tags: [javascript, nodejs, webrtc, socketio]
toc: true
toc_label: " 목차"
toc_sticky: true
---
## WebRTC 시그널링(Signaling) 서버 만들기

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

이러한 과정을 거치면 두 사용자간의 local/remote sdp 정보가 일치하게된다. 비로소 WebRTC 연결을 시작할 준비가 끝난 것이다. 물론 SDP 정보와 별도로 ICE 후보를 통해 네트워크 정보를 교환해야 한다. ICE 교환은 더 간단하게 이루어진다.

{% include figure class="center" image_path="/assets/images/posts/2020-09-05/webrtc-signaling-ice-chart.svg" alt="ICE를 교환하는 순서" caption="ICE를 교환하는 순서" %}

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

인증서의 정보를 넣으라는 내용이 나오면 모두 엔터로 넘겨버려도 문제가 발생하지 않는다. 이 인증서를 통해 node에 내장된 https 모듈을 통해 https 서버를 열 수 있다.

```javascript
const fs = require('fs');
const https = require('https');

const credentials = {
    key: fs.readFileSync('./cert.key'),
    cert: fs.readFileSync('./cert.crt'),
};

const httpsServer = https.createServer(credentials);

httpsServer.listen(9400, () => {
    console.log('HTTPS Server is running at 9400!');
});
```

다만 이렇게 서버를 열어도 접속하면 로딩만 할 뿐 아무런 페이지도 나오지 않는다. 따라서 express를 추가하여 static 파일을 제공할 수 있도록 변경해준다.

```javascript
const express = require('express');
const app = express();

app.use(express.static('public'));

const httpsServer = https.createServer(credentials, app);
```

express까지 설정하면 public 폴더 안에 들어 있는 html, js 등의 static 파일을 제공할 수 있는 상태가 된다.

### Socket&#46;io 서버 구현

https 서버가 생겼으니 socket 서버를 구현해보도록 하자.

```javascript
const SocketIO = require('socket.io');
const io = new SocketIO();

io.attach(httpsServer);

io.sockets.on('connection', socket => {
    console.log('Client connected!');
});
```

우선 위 코드를 사용하여 socket.io를 https 서버와 같이 열 수 있다. 이제 여기에 시그널링 기능을 구현할 것이다. 필요한 기능은 다음과 같다.

1. 클라이언트 등록 단계.
2. 연결 요청 / 취소 단계.
3. SDP, ICE 교환 단계.
4. 클라이언트 삭제 단계.

여기서는 1:1 연결을 위한 시그널링 서버를 구현하므로, 1단계는 식별을 위해 클라이언트(브라우저)의 이름을 등록하는 단계이다.

```javascript
const clients = new Map();

io.sockets.on('connection', socket => {
    console.log('Client connected');

    let name = null;

    // 브라우저가 자신의 이름을 보내는 경우.
    socket.on('init', initName => {
        console.log(`Client name registered. ${socket.id} = ${initName}`);

        clients.set(initName, socket.id);
        name = initName;
    });
});
```

`clients`를 [Map](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Map)으로 선언하여 클라이언트 이름과 고유 아이디를 매칭시킬 수 있도록 저장한다. 이후 브라우저가 자신의 이름을 `init` 메시지를 통해 보내면 `clients`에 저장해주면 된다.

이제 2단계를 구현해보자.

```javascript
// 연결 요청이 들어온 경우.
socket.on('request_connection', (remoteName, callback) => {
    if (!clients.has(remoteName)) {
        // 상대방 클라이언트가 없는 경우.
        if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
        return;
    }

    console.log(`Client ${name} request connect to ${remoteName}`);

    io.to(clients.get(remoteName)).emit('request_connection', name);
    if (typeof callback === 'function') callback(null);
});

// 연결 요청을 거절하는 경우.
socket.on('cancel_connection', (remoteName, callback) => {
    if (!clients.has(remoteName)) {
        // 상대방 클라이언트가 없는 경우.
        if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
        return;
    }

    console.log(`Client ${name} cancel connect with ${remoteName}`);

    io.to(clients.get(remoteName)).emit('cancel_connection', name);
    if (typeof callback === 'function') callback(null);
});
```

`request_connection` 메시지를 통해 상대방에게 연결을 요청할 수 있다. 연결을 요청하게 되면 서버가 상대방의 이름에 메시지를 보내거나 상대방을 찾을 수 없다면 callback으로 알려주게 된다. 1단계에서 `clients`에 `socket.id`를 저장했으므로 `io.to(clients.get(remoteName))`을 통해 특정한 클라이언트에게 메시지를 보낼 수 있다. 메시지를 받은 상대방은 연결을 하거나, 아니면 cancel_connection 메시지를 보내 연결을 거절했다는 것을 알려줄 수 있다.

3단계와 4단계는 아주 간단하게 구현되는데, 다음과 같다.

```javascript
// RTC 접속 정보를 공유.
socket.on('connection_info', (data, callback) => {
    if (!clients.has(data.remoteName)) {
        // 상대방 클라이언트가 없는 경우.
        if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
        return;
    }

    console.log(`Sending connection information from ${name} to ${data.remoteName}, type: ${data.type}`);

    io.to(clients.get(data.remoteName)).emit('connection_info', data);
    if (typeof callback === 'function') callback(null);
});

// 클라이언트 연결이 끊긴 경우.
socket.on('disconnect', () => {
    // 연결 끊기면 삭제.
    console.log(`Client ${name} disconnected`);

    if (clients.has(name)) {
        clients.delete(name);
    }
});
```

여기서도 2단계와 마찬가지로 `connection_info`로 데이터가 들어오면 받은 데이터를 그대로 상대방에게 전달해주기만 하면 된다. 단 3단계에서는 sdp, ice 데이터를 주고받아야 하므로 `object` 형식으로 데이터를 받고 전달해준다.

마지막 4단계는 그저 `disconnect` 이벤트가 발생하면 `clients`에서 해당되는 이름을 삭제하는 것 밖에 없다.

### 마무리

이렇게 하면 시그널링 서버의 구현이 끝난 것이다.

> 벌써요?

그렇다. 사실 시그널링 서버라는 것은 말 그대로 신호(signal)를 전달(ing)하는 역할밖에 하지 않는다. 극단적으로 보면, 1, 2, 4단계 없이 3단계만 있어도 이미 훌룡한 시그널링 서버인 것이다. 물론 그렇게 되면 클라이언트에서 구현할 부분이 늘어나므로 여기서는 아주 간단한 클라이언트 식별 단계를 넣었다.

시그널링 서버의 최종 소스는 다음과 같다.

```javascript
const fs = require('fs');
const https = require('https');
const express = require('express');
const SocketIO = require('socket.io');

const credentials = {
    key: fs.readFileSync('./cert.key'),
    cert: fs.readFileSync('./cert.crt'),
};

const app = express();

app.use(express.static('public'));

const httpsServer = https.createServer(credentials, app);

httpsServer.listen(9400, () => {
    console.log('HTTPS Server is running at 9400!');
});

const io = new SocketIO();

io.attach(httpsServer);

const clients = new Map();

io.sockets.on('connection', socket => {
    console.log('Client connected');

    let name = null;

    // 브라우저가 자신의 이름을 보내는 경우.
    socket.on('init', initName => {
        console.log(`Client name registered. ${socket.id} = ${initName}`);

        clients.set(initName, socket.id);
        name = initName;
    });

    // 연결 요청이 들어온 경우.
    socket.on('request_connection', (remoteName, callback) => {
        if (!clients.has(remoteName)) {
            // 상대방 클라이언트가 없는 경우
            if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
            return;
        }

        console.log(`Client ${name} request connect to ${remoteName}`);

        io.to(clients.get(remoteName)).emit('request_connection', name);
        if (typeof callback === 'function') callback(null);
    });

    socket.on('cancel_connection', (remoteName, callback) => {
        if (!clients.has(remoteName)) {
            // 상대방 클라이언트가 없는 경우.
            if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
            return;
        }

        console.log(`Client ${name} cancel connect with ${remoteName}`);

        io.to(clients.get(remoteName)).emit('cancel_connection', name);
        if (typeof callback === 'function') callback(null);
    });

    // RTC 접속 정보를 공유.
    socket.on('connection_info', (data, callback) => {
        if (!clients.has(data.remoteName)) {
            // 상대방 클라이언트가 없는 경우.
            if (typeof callback === 'function') callback('상대방이 접속해있지 않습니다.');
            return;
        }

        console.log(`Sending connection information from ${name} to ${data.remoteName}, type: ${data.type}`);

        io.to(clients.get(data.remoteName)).emit('connection_info', data);
        if (typeof callback === 'function') callback(null);
    });

    // 클라이언트 연결이 끊긴 경우.
    socket.on('disconnect', () => {
        // 연결 끊기면 삭제.
        console.log(`Client ${name} disconnected`);
        if (clients.has(name)) {
            clients.delete(name);
        }
    });
});
```

## Socket.io를 사용한 시그널링 클라이언트 구현

WebRTC 시그널링은 서버 구현보다 클라이언트에 구현할 내용이 더 많은 것이 사실이다. 서버는 아주 간단하게 끝났지만, 클라이언트는 구현할 내용이 훨씬 많다. 이제 클라이언트를 구현해보자.

### HTML 준비

먼저 `public` 폴더에 `index.html`을 만들고 아래 내용을 적어주자.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>WebRTC 화상 통화</title>
</head>
<body>
    <video id="video-preview" playsinline controls preload="metadata" autoplay muted></video>
    <video id="video-remote" playsinline controls preload="metadata" autoplay></video>
    <br>
    <input id="name" type="text" placeholder="내 이름" style="width:100px">
    <button id="connect" type="button">서버 접속</button>
    <input id="remote-name" type="text" placeholder="상대방 이름" style="width:100px">
    <button id="request" type="button" disabled>연결</button>

    <script src="https://cdn.jsdelivr.net/npm/jquery@3.5.1/dist/jquery.min.js"></script>
    <script src="/socket.io/socket.io.js"></script>
    <script src="./app.js"></script>
</body>
</html>
```

일단은 UI 요소를 전혀 넣지 않고 최소한의 태그들만 넣었다. Bootstrap로 조금 더 보기 좋게 만든 소스는 최종 소스 부분에 첨부하도록 하겠다.

### 웹캠 처리

이제 `app.js` 파일을 만들어 본격적으로 클라이언트 구현을 해보자. 가장 첫번째로 할 일은 `getUserMedia`를 통해 웹캠 화면을 가져오는 것이다.

```javascript
let localStream = null;

if (navigator.getUserMedia) {
    navigator.getUserMedia(
        { video: true, audio: true },
        resultStream => {
            localStream = resultStream;
            $('#video-preview')[0].srcObject = resultStream;
        },
        error => {
            console.log('웹캠 권한을 거부하였거나 오류가 발생했습니다.');
            console.error(error);
        }
    );
} else {
    console.log('브라우저가 웹캠을 지원하지 않습니다.');
}
```

이렇게 하면 웹캠으로부터 video와 audio가 합쳐진 스트림을 가져오게 된다. 만약 자신의 웹캠에 마이크가 없다면 단순히 `audio`를 `false`로 바꿔주면 된다. 저장하고 서버로 들어가보면 벌써 왼쪽 비디오 태그에 웹캠이 보여지는 모습을 볼 수 있다. 만약 웹캠이 없거나, 웹캠 권한을 거절하였다면 콘솔창에서 오류 메시지를 확인할 수 있다.

{% include figure class="center" image_path="/assets/images/posts/2020-09-05/webrtc-signaling-client-01.jpg" alt="웹캠이 보여지는 모습" caption="웹캠이 보여지는 모습" %}

개인정보 문제로 웹캠은 간단한 나무토막으로 대체되었다.

### Socket&#46;io 클라이언트 구현

이제 웹캠을 가져왔으니 서버에 연결하는 부분을 구현하자.

```javascript
let socket = null;

$('#connect').on('click', event => {
    name = $('#name').val();

    if (!name) {
        alert('이름을 입력해주세요!');
        return;
    }

    socket = io();

    socket.on('connect', () => {
        $('#connect').text('접속 완료');
        $('#connect').prop('disabled', true);
        $('#request').prop('disabled', false);

        socket.emit('init', name);
    });

    socket.on('disconnect', () => {
        $('#connect').text('접속 끊김');
    });

    // 연결 요청을 받은 경우
    socket.on('request_connection', remoteName => {
        requestConnection(remoteName);
    });

    // 상대방이 연결 요청을 거절한 경우.
    socket.on('cancel_connection', () => {
        alert('상대방이 연결을 거절하였습니다.');
    });

    // 연결 중에 통신하는 내용.
    socket.on('connection_info', data => {
        connectionInfo(data);
    });
});
```

연결 버튼을 누르면 socket&#46;.io 연결을 시작하고, `init` 메시지를 보내 내 이름을 등록하는 부분이다. 그리고 `socket.on`으로 들어오는 메시지를 처리하는 부분도 추가하였다. 이제 실제 브라우저에 접속하여 연결 버튼을 누르면 접속 완료라는 메시지가 뜨고, 서버에도 아래와 같은 로그를 보게 될 것이다.

```console
Client connected
Client name registered. hyNpIF9TZUWW_pcCAAAA = pc01
```

이러면 서버에 접속하는 부분도 구현이 끝났다. 물론 실질적으로 처리하는 함수인 `requestConnection`와 `connectionInfo`가 아직 없으므로 통화 요청 메시지를 받을 수 있겠지만 통화를 연결할 수는 없다.

### WebRTC Peer Connection 구현

마지막으로 가장 복잡한 WebRTC Peer Connection을 구현해야 한다. Peer Connection을 만들고 SDP Offer를 전송하는 부분은 다음과 같은 순서로 진행한다.

1. `new RTCPeerConnection`을 통해 새로운 연결을 만든다.
1. `RTCPeerConnection`에 `getUserMedia`로 가져온 `MediaStream`을 추가한다.
1. `negotiationneeded` 이벤트가 발생하면 `createOffer`를 통해 SDP Offer를 생성한다.
1. 생성한 SDP를 `setLocalDescription`로 저장한다.
1. 상대방에게 SDP Offer를 보낸다.

이를 코드로 구현해보자.

```javascript
let peerConnection = null;

// PeerConnection을 생성하는 부분은 2곳에서 사용되므로 함수로 작성하자.
const createPeerConnection = () => {
    if (peerConnection) return;

    // STUN 서버는 Google의 공개된 서버를 사용한다.
    peerConnection = new RTCPeerConnection({ iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] });

    peerConnection.onicecandidate = event => {
        // 이 이벤트가 발생하면 ICE 교환을 시작해야 한다.
        onIceCandidate(event);
    };

    peerConnection.ontrack = event => {
        // 원격에서 스트림(트랙)을 받으면 그 스트림을 화면에 표시해줘야 한다.
        $('#video-remote')[0].srcObject = event.streams[0];
    };
};
```

