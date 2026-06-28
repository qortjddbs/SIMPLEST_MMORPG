# SIMPLEST_MMORPG

대규모 트래픽 처리를 위한 MMORPG & 스트레스 테스트 툴

본 프로젝트는 수천 명의 동시 접속자를 감당하는 MMORPG 서버 아키텍처를 검증하기 위해 개발된 전용 클라이언트 및 부하 테스트(Stress Test) 모듈입니다. 단순한 게임 플레이를 넘어, 네트워크 트래픽 처리의 한계를 테스트하고 클라이언트 구조를 최적화하는 데 집중했습니다.

주요 기술 및 아키텍처 (Technical Highlights)
1. O(1) 복잡도의 패킷 디스패처 설계 (Switch-Case 하드코딩 탈피)
일반적으로 수십 개의 패킷을 처리할 때 거대한 switch-case 문을 사용하면 유지보수가 매우 어려워집니다. 이를 해결하기 위해 함수 포인터와 람다(Lambda)를 활용한 콜백 라우팅 구조를 설계했습니다.

std::unordered_map<PACKET_TYPE, std::function<void(char*)>>를 활용하여 패킷 타입과 핸들러 함수를 매핑.

패킷 수신 시 단 5줄의 코드(DispatchPacket)만으로 조건 분기 없이 즉시 O(1) 속도로 알맞은 함수를 실행하도록 구조를 개선하여 유연성과 확장성을 확보했습니다.

2. Modern C++을 활용한 안전한 객체 생명주기 관리
수백 명의 유저와 몬스터가 시야에 들어오고 나가는(Add/Remove) 과정에서 발생할 수 있는 메모리 누수(Memory Leak)를 원천 차단했습니다.

GameManager 내에서 객체를 관리할 때 원시 포인터와 delete 대신 std::unique_ptr를 도입했습니다.

시야에서 벗어난 객체는 m_objects.erase(id) 호출 한 번으로 자동 메모리 해제되도록 구현하여, 클라이언트의 메모리 안정성을 극대화했습니다.

3. 서버 한계 돌파를 위한 더미 클라이언트 (Stress Test Tool)
서버의 병목 현상을 정확히 파악하기 위해, 화면 렌더링 없이 순수 네트워크 I/O만 발생시키는 전용 부하 테스트 모듈(NetworkModule.cpp)을 자체 제작했습니다.

클라이언트 측에서도 IOCP(I/O Completion Port) 모델을 적용하여 10,000개의 더미 소켓이 비동기적으로 패킷을 송수신할 수 있도록 구현했습니다.

단순 접속뿐만 아니라 1초마다 무작위 방향으로 이동 패킷(C2S_MOVE)을 서버로 전송하며 실질적인 연산 부하를 발생시킵니다.

서버의 응답 지연 시간(global_delay)을 실시간으로 계산하여, 지연율이 높아지면 접속 생성 속도를 조절하는 능동적인 테스트 환경을 구축했습니다.

4. SFML 기반의 최적화된 렌더링 및 UI 연출
리소스 캐싱: ResourceManager 싱글톤 클래스를 통해 텍스처와 폰트를 std::unordered_map에 한 번만 적재하여 불필요한 파일 I/O를 제거했습니다.

시각적 피드백: 컷신 연출을 위한 화면 흔들림(Screen Shake), 일정 시간 후 큐에서 자동 소멸되는 채팅 말풍선, 플레이어와 보스를 구분하는 레이더(Minimap) 시스템을 구현했습니다.

핵심 코드 스니펫 (Code Snippet)
[패킷 디스패처 (NetworkManager.cpp)]
수백 줄의 switch-case를 대체한 이벤트 기반 패킷 처리 로직입니다.

void NetworkManager::DispatchPacket(char* packet) {
    PACKET_TYPE type = reinterpret_cast<PACKET_HEADER*>(packet)->type;

    auto it = m_handlers.find(type);
    if (it != m_handlers.end()) {
        it->second(packet); // 맵에 등록된 콜백 함수를 즉시 실행 (O(1))
    } else {
        std::cout << "등록되지 않은 패킷 타입 수신: " << type << std::endl;
    }
}


[더미 클라이언트 지연율 측정 (NetworkModule.cpp)]
서버의 이동 패킷 반환 시간을 측정하여 프레임 지연율(Lag)을 산출합니다.

// 이동 요청 후 서버로부터 확정 좌표를 받았을 때 지연 시간(Lag) 계산
unsigned now_ms = static_cast<unsigned>(duration_cast<milliseconds>(high_resolution_clock::now().time_since_epoch()).count());
int d_ms = static_cast<int>(now_ms - move_packet->move_time);

if (global_delay < d_ms) global_delay++;
else if (global_delay > d_ms) global_delay--;


---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
2025년 1학기 - 후레쉬맨 모작 개발 -> https://youtu.be/xrWP2p0KoEA
2025년 2학기 - 바운스 어택 모작 개발 -> https://youtu.be/BizjzUaK3rM?si=aOZZC9m84rD2Bu-V
             - 3D 바운스볼 개발 -> https://youtu.be/3olYj93gBIk?si=OS8bPVdlrqHCQy_J
