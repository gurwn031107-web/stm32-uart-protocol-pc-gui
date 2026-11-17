# STM32 UART 패킷 프로토콜 + PC 제어 프로그램

NUCLEO-F411RE와 PC가 UART로 주고받는 바이너리 패킷 프로토콜을 정의하고, 보드 펌웨어와 PySide6 PC 프로그램을 같이 만든 프로젝트입니다.
PC에서 보드의 LED 4개를 켜고 끄고, 온습도와 거리를 요청해 화면과 그래프로 보여 줍니다.

- 기간: 2026.11.04
- 보드: NUCLEO-F411RE (STM32F411RE)
- PC: Python, PySide6, pyserial, pyqtgraph

## 구성

| 폴더 | 내용 |
|---|---|
| [`firmware`](firmware) | STM32CubeIDE 프로젝트. 패킷 수신 상태 머신과 명령 처리 (`Core/Src/driver/uart_protocol.c`, `main.c`) |
| [`pc_gui`](pc_gui) | PC 제어 프로그램. 포트 자동 감지, LED 제어, 센서 표시, 실시간 그래프, 통신 로그, 보드 없이 돌려 보는 시뮬레이터 |

## 패킷 형식

```
| SOF (0xAA) | LEN | CMD | PAYLOAD (LEN-1 바이트) | CHECKSUM |
LEN      = CMD 1바이트 + PAYLOAD 길이 (1 ~ 15)
CHECKSUM = (LEN + CMD + PAYLOAD 합) & 0xFF
```

- 115200bps, 8N1, USART2(PA2 / PA3)를 ST-LINK 가상 COM 포트로 사용
- 응답 CMD는 요청 CMD + 0x80 (예: `0x14` LED 상태 요청 → `0x94` 응답)

| 요청 CMD | 내용 | 응답 |
|---|---|---|
| `0x10` ~ `0x13` | LED0 ~ 3 켜기 / 끄기 | `0x90` ~ `0x93` + OK / ERR |
| `0x14` | LED 상태 요청 | `0x94` + LED 4개 상태 |
| `0x20` | 온습도 요청 | `0xA0` + 온도 정수 / 소수, 습도 정수 / 소수 |
| `0x30` | 거리 요청 | `0xB0` + 거리 2바이트 (cm) |

## 펌웨어

- 1바이트씩 수신 인터럽트를 받아 `WAIT_START → WAIT_LEN → WAIT_DATA → WAIT_CHK` 상태 머신으로 패킷을 조립합니다.
- LEN이 0이거나 버퍼보다 크면 바로 처음 상태로 돌아가고, 체크섬이 맞을 때만 명령 처리 콜백을 부릅니다.
- 명령 처리는 콜백 함수로 분리해서, 프로토콜 코드(`uart_protocol.c`)와 보드 동작 코드(`main.c`)를 나눴습니다.
- LED는 PB15, PB14, PB13, PC4에 연결했습니다.
- 온습도와 거리 응답은 프로토콜과 PC 프로그램을 먼저 맞추기 위해 고정값으로 보내고 있습니다. 실제 DHT11 / HC-SR04 측정값을 넣는 것이 다음 단계입니다.

## PC 프로그램

```bash
cd pc_gui
pip install -r requirements.txt
python main.py          # 실제 보드 (포트 자동 감지)
python main.py --sim    # 보드 없이 가상 보드로 실행
```

- ST-LINK 포트(VID 0x0483)를 찾아 LED 상태 요청을 보내고, 올바른 응답이 오는 포트에 자동으로 연결합니다. USB를 다시 꽂으면 자동으로 다시 연결합니다.
- 요청은 하나씩 보내고, 응답이 오거나 300ms 타임아웃이 난 뒤 다음 요청을 보냅니다. 3번 연속 타임아웃이면 "보드 응답 없음"을 표시합니다.
- 시리얼 통신은 별도 스레드(`serial_worker.py`)에서 처리해서 화면이 멈추지 않게 했습니다.
- `pytest`로 패킷 생성 / 해석, 펌웨어 호환성, 통신 스레드를 테스트합니다.

자세한 화면 구성과 예시 패킷은 [`pc_gui/README.md`](pc_gui/README.md)에 있습니다.

## 빌드 (펌웨어)

1. STM32CubeIDE에서 `File > Import > Existing Projects into Workspace`로 `firmware` 폴더 열기
2. `.ioc`를 열고 Generate Code (HAL 드라이버는 저장소에서 제외)
3. Build 후 Run
