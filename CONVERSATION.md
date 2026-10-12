# Copilot과의 대화 기록

## 2026-10-12 세션

### 초기 요구사항

나는 Raspberry Pi Pico에서 MicroPython으로 GPIO 실습 중입니다.

**회로 연결:**
- 외부 LED: GP15에 연결
- 저항: 220Ω을 직렬로 연결
- 버튼: GP14에 연결

**실행할 동작:**
버튼을 누를 때마다 LED가 다음과 같이 작동해야 합니다.

### 요구사항 상세

1. **LED 깜빡임 속도 (3가지)**
   - 느림: 1초 간격
   - 보통: 0.5초 간격
   - 빠름: 0.1초 간격
   - 시작: 보통
   - 버튼 1회: 빠름
   - 버튼 2회: 느림
   - 버튼 3회: 보통 (반복)

2. **코드 요구사항**
   - 각 줄에 초보자용 한글 주석
   - Pin 번호가 GP 번호인지 설명

3. **하드웨어 체크리스트**
   - LED가 안 켜질 때 점검 항목

4. **버튼 입력**
   - Pin.irq() 인터럽트 방식 사용
   - 폴링(polling) 금지

5. **LED 점멸**
   - Timer를 사용해 백그라운드에서 처리
   - 메인 루프에서 sleep()으로 점멸하지 않음

6. **디바운싱**
   - 버튼을 한 번 누르면 한 단계만 변경
   - 버튼의 떨림 무시

7. **콘솔 출력**
   - 속도 변경할 때마다 현재 속도를 print로 출력

### GP 번호와 Pin 번호 설명

**중요:**
- **GP15** = GPIO 번호 (칩의 물리적 기능)
- **Pin 20** = 실제 핀 번호 (Pico 보드의 물리적 위치)

Pico 핀맵:
```
GP14 → Pin 19 (버튼)
GP15 → Pin 20 (LED)
```

MicroPython에서는 GP 번호를 직접 사용하므로 코드에서 `Pin(15, ...)` 형태로 작성합니다.

### LED가 안 켜질 때 하드웨어 체크리스트

```
체크 항목:
1. LED가 회로에 연결되어 있는가?
2. 220Ω 저항이 LED의 긴 다리(+)에 연결되어 있는가?
3. 저항의 다른 쪽이 GP15에 연결되어 있는가?
4. LED의 짧은 다리(-)가 GND에 연결되어 있는가?
5. 극성이 올바른가? (긴 다리 +, 짧은 다리 -)
6. 모든 점퍼선이 제대로 꽂혀 있는가?
7. Pico가 제대로 전원을 받고 있는가?
8. USB 연결이 안정적인가?
```

### 초기 코드 작성 및 개선

**첫 번째 버전:**
- 기본적인 LED 점멸 기능
- 버튼 인터럽트 처리
- Timer 기반 점멸
- 디바운싱 적용
- 속도 변경 시 ON/OFF 상태까지 매번 출력

**문제점:**
콘솔 화면에 ON/OFF가 계속 출력되어 현재 속도를 바로 알 수 없음

**개선 요청:**
ON/OFF 출력을 제거하고, 현재 속도만 출력하도록 수정

### 최종 코드

```python
# Raspberry Pi Pico MicroPython
# LED 속도 제어 예제

from machine import Pin, Timer
import time

# ========== 핀 번호 설정 ==========
LED_PIN = 15      # LED는 GP15에 연결
BUTTON_PIN = 14   # 버튼은 GP14에 연결

# ========== 속도 설정 ==========
SPEEDS = {
    'slow': 1000,      # 느림: 1초마다 깜빡임
    'normal': 500,     # 보통: 0.5초마다 깜빡임
    'fast': 100        # 빠름: 0.1초마다 깜빡임
}

SPEED_CYCLE = ['normal', 'fast', 'slow']

# ========== 전역 변수 ==========
led_state = 0
current_speed_index = 0
last_button_press = 0
debounce_time = 200

# ========== GPIO 초기화 ==========
led = Pin(LED_PIN, Pin.OUT, value=0)
button = Pin(BUTTON_PIN, Pin.IN, pull=Pin.PULL_UP)

# ========== Timer 생성 ==========
led_timer = Timer()

# ========== LED 점멸 함수 ==========
def toggle_led(timer):
    global led_state
    led_state = 1 - led_state
    led.value(led_state)
    # ON/OFF 출력 제거 - 현재 속도만 출력하도록 변경

# ========== 버튼 인터럽트 함수 ==========
def button_callback(pin):
    global last_button_press, current_speed_index
    
    current_time = time.ticks_ms()
    
    # 디바운싱
    if time.ticks_diff(current_time, last_button_press) < debounce_time:
        return
    
    last_button_press = current_time
    
    # 속도 단계 변경
    current_speed_index = (current_speed_index + 1) % len(SPEED_CYCLE)
    
    speed_name = SPEED_CYCLE[current_speed_index]
    speed_ms = SPEEDS[speed_name]
    
    # 현재 속도만 출력
    print(f"현재 속도: {speed_name} ({speed_ms}ms)")
    
    # 타이머 재설정
    led_timer.deinit()
    led_timer.init(period=speed_ms, mode=Timer.PERIODIC, callback=toggle_led)

# ========== 버튼 인터럽트 등록 ==========
button.irq(trigger=Pin.IRQ_FALLING, handler=button_callback)

# ========== 초기 설정 ==========
start_speed = SPEED_CYCLE[current_speed_index]
led_timer.init(period=SPEEDS[start_speed], mode=Timer.PERIODIC, callback=toggle_led)

print("=" * 40)
print("Raspberry Pi Pico LED 속도 제어 시작")
print("시작 속도: normal (500ms)")
print("버튼을 누르면: normal -> fast -> slow -> normal")
print("=" * 40)

# ========== 메인 루프 ==========
while True:
    time.sleep(0.1)
```

### 핵심 개념 정리

**1. Pin.irq() - 인터럽트 방식**
- 버튼을 눌렀을 때 (HIGH → LOW) 자동으로 함수 호출
- 폴링(계속 확인)하지 않아도 됨
- CPU 효율이 좋음

**2. Timer - 백그라운드 점멸**
- 타이머가 일정 시간마다 콜백 함수 호출
- 메인 루프에서 sleep()으로 블로킹하지 않음
- 다른 작업도 동시에 처리 가능

**3. 디바운싱 - 떨림 무시**
- 버튼이 물리적으로 떨려서 여러 번 입력될 수 있음
- 200ms 동안 같은 입력 무시
- 한 번 누르면 한 단계만 변경됨

**4. 콘솔 출력 최적화**
- ON/OFF를 매번 출력하면 콘솔이 복잡함
- 속도 변경할 때만 현재 속도 출력
- 콘솔을 깔끔하게 유지

### 최종 완성

✅ 모든 요구사항 충족
- 3단계 속도 제어
- 초보자용 상세 주석
- GP vs Pin 번호 설명
- 하드웨어 체크리스트
- 인터럽트 기반 버튼 입력
- Timer 기반 LED 점멸
- 디바운싱 적용
- 현재 속도만 출력

**파일:**
- main.py - Pico에 업로드하면 자동 실행
- README.md - 사용 설명서
- CONVERSATION.md - 이 대화 기록
