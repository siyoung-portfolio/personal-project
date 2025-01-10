# PR(가제)
24.10.30 ~ 진행중

## 기획 의도 및 프로젝트 설명
- 사이드 프로젝트 구인하는 서비스는 많지만, 개인이 자기PR을 하면서 많은 사람들에게 노출할 수 있는 서비스가 없어서 자신을 알릴 수 있으면 사이드 프로젝트를 하고 싶어하는 사람들에게 기회가 많이 가지 않을까 생각하여 기획하였습니다.
- 부트캠프로 개발 공부를 하는 사람이 많은 시기에 부트캠프 수료생 및 더 나아가 사이드 프로젝트를 원하는 현직 개발자와 대학생까지 많은 기회를 제공할 수 있는 사이드 프로젝트 매칭 서비스입니다.

## 사용된 기술
- Framework : Spring Boot 3.x.x, Spring Security
- Language : Java 17
- DB : PostgreSQL
- ETC : Websocket(Stomp) 2.3.4

## 인원
- Backend 2명
- Frontend 1명

## 담당업무
- 기획
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

### 채팅방 입장 이전 메시지 자동 읽음처리 오류

- 문제 상황
  - 채팅방에 입장한 유저가 마지막으로 채팅을 읽은 시간을 시점부터 입장한 시점까지 읽지 않은 메시지 읽음 처리가 되지 않음
  - 채팅방 입장시 ChatRoomMember에 관한 Select문 중복 발생
  - 채팅방 입장 시, 읽지 않은 모든 메시지 읽음처리 로직
    ```Java
    public ChatMessageResponse enterRoom(Long roomId, Long userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new IllegalArgumentException("User not found"));
        ChatRoom chatRoom = chatRoomRepository.findById(roomId)
                .orElseThrow(() -> new IllegalArgumentException("Chat room not found"));

        ChatRoomMember member = findChatRoomMember(roomId, userId);
        markAsRead(roomId, userId);
        member.setLeft(false);

        markAllMessagesAsRead(roomId, userId);

        // 입장 메시지 생성
        ChatMessage enterMessage = ChatMessage.builder()
                .chatRoom(chatRoom)
                .sender(user)
                .content(user.getNickname() + "님이 입장하셨습니다.")
                .type(MessageType.ENTER)
                .build();

        ChatMessage savedMessage = chatMessageRepository.save(enterMessage);
        return ChatMessageResponse.from(savedMessage);
    }
    ```

    ```Java
    @Transactional
    public void markAllMessagesAsRead(Long roomId, Long userId) {
        ChatRoomMember member = findChatRoomMember(roomId, userId);
        chatMessageRepository.markMessagesAsRead(roomId, member.getLastReadAt());
        member.updateLastRead();
    }
    ```
    
    ```Java
    @Modifying  // 벌크 업데이트를 위한 어노테이션 추가
    @Query("UPDATE ChatMessage m SET m.read = true " +
            "WHERE m.chatRoom.id = :roomId " +
            "AND m.sentAt > :lastReadAt " +
            "AND m.read = false")
    void markMessagesAsRead(
            @Param("roomId") Long roomId,
            @Param("lastReadAt") LocalDateTime lastReadAt
    );
    ```

- 원인
  - enterRoom 메서드에서 markAsRead라는 메서드 때문에 markAllMessageAsRead가 실행되기 전에 멤버의 마지막 읽은 시간이 바뀜
  - 따로 로직 테스트 때문에 http 메서드를 만들어서 사용한 부분을 지우지 않았던 것
  - 문제의 markAsRead
  ```Java
  public void markAsRead(Long roomId, Long userId) {
        ChatRoomMember member = findChatRoomMember(roomId, userId);
        member.updateLastRead();
    }
  ```
  - markAllMessageAsRead에서 이미 찾은 멤버를 다시 찾고 있어서 Select문 중복 발생
    ```java
    @Transactional
    public void markAllMessagesAsRead(Long roomId, Long userId) {
        ChatRoomMember member = findChatRoomMember(roomId, userId);
        chatMessageRepository.markMessagesAsRead(roomId, member.getLastReadAt());
        member.updateLastRead();
    }
    ```
- 해결
  - 메서드 서순 정리 및 불필요한 메서드 제거 및 @Transactional 추가
    ```Java
    @Transactional
    public ChatMessageResponse enterRoom(Long roomId, Long userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new IllegalArgumentException("User not found"));
        ChatRoom chatRoom = chatRoomRepository.findById(roomId)
                .orElseThrow(() -> new IllegalArgumentException("Chat room not found"));

        ChatRoomMember member = findChatRoomMember(roomId, userId);
        markAllMessagesAsRead(roomId, member);
        member.setLeft(false);

        // 입장 메시지 생성
        ChatMessage enterMessage = ChatMessage.builder()
                .chatRoom(chatRoom)
                .sender(user)
                .content(user.getNickname() + "님이 입장하셨습니다.")
                .type(MessageType.ENTER)
                .build();

        ChatMessage savedMessage = chatMessageRepository.save(enterMessage);
        return ChatMessageResponse.from(savedMessage);
    }
    ```
  - markAllMessageAsRead 매개변수 ChatRoomMember 객체를 가져오도록 수정
    ```java
    private void markAllMessagesAsRead(Long roomId, ChatRoomMember member) {
        chatMessageRepository.markMessagesAsRead(roomId, member.getLastReadAt());
        member.updateLastRead();
    }
    ```
    
    
