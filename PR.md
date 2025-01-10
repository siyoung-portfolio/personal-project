# PR(가제)
24.10.30 ~ 진행중

## 프로젝트 설명

현직 개발자 혹은 개발을 공부하는 많은 사람들의 사이드 프로젝트를 매칭 시켜주는 서비스입니다.
프로젝트 단위에서 팀을 구하는 것 뿐만 아니라 개인이 자기PR을 통해 많은 사람들에게 알리며 많은 사이드 프로젝트 기회를 얻을 수 있게 하는 것이 핵심 서비스입니다.

## 사용된 기술
- Framework : Spring Boot 3.x.x, Spring Security
- Language : Java 17
- DB : PostgreSQL
- ETC : Websocket(Stomp) 2.3.4

## 인원
Backend 2명
Frontend 1명

## 담당업무
- Team Leader 및 기획
    - 사이드 프로젝트 구인하는 서비스는 많지만, 개인이 자기PR을 하면서 많은 사람들에게 노출할 수 있는 서비스가 없어서 자신을 알릴 수 있으면
      사이드 프로젝트를 하고 싶어하는 사람들에게 기회가 많이 가지 않을까 생각하여 기획
- 게시글/북마크 CRUD
    - 프로젝트(팀원 구인)게시글 CRUD
    - 자기PR(개인 어필)게시글 CRUD
    - 북마크 기능
    - 회원 기술스택 CRUD
- 회원가입 및 인증/인가
    - **Security**를 이용해서 **회원관련 API 개발**
- 1:1 채팅 서비스
  - Stomp기반으로 1:1 채팅 서비스 구현
  - SessionId 유무로 자동 읽음 처리 구현
  
## 트러블 슈팅

### Postman 통신 에러

- 문제 상황
  - 앤드포인트를 제대로 설정했지만 connect프레임을 보냈을 때 서버로부터 응답이 오지 않음
    ```Java
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/sub");
        config.setApplicationDestinationPrefixes("/pub");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-stomp")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
    ```
    - Postman 테스트 시 웹소켓 연결을 위한 URL
      - ws://localhost:8080/ws-stomp/websocket
      - 
    - 서버에 보낸 connect 프레임
    ```
    CONNECT
    accept-version:1.1,1.0
    heart-beat:10000,10000
    
    ```
    
    <img width="807" alt="image" src="https://github.com/user-attachments/assets/3d64d3f1-d05c-4378-b126-0e5b025bbb3c" />

  - 원인
    - connect 프레임을 보낼 때 한줄을 띄워야 함
    - 위에 처럼 한줄을 띄웠지만 Postman 측에서 인식을 하지 못함
  
  - 해결
    - 사진과 같이 Connect 프레임 수정
      
      <img width="477" alt="image" src="https://github.com/user-attachments/assets/794161ea-1f41-41ba-917e-6694912470b1" />
      
    - 해결에 도움을 받은 부분
      - https://min9805.github.io/stomp/
      - 위 사이트에서 개발자도구를 열고 한줄 띄워진 부분을 복사해서 사용
    
    - 통신이 되는 모습
      
      <img width="756" alt="image" src="https://github.com/user-attachments/assets/ed826eb1-55e2-4747-bb7e-0f3a5eb7372d" />

    
