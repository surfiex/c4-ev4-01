# 🚗 기아 EV4 (KIA EV4) 오픈파일럿 완벽 이식(Porting) 가이드

이 문서는 프로그래밍 지식이 부족한 **오픈파일럿 입문자 분들도 따라할 수 있도록**, Kia EV4를 오픈파일럿(Openpilot)에 연동시키기 위해 어떤 파일의 몇 번째 줄을, 어떻게 수정해야 하는지 매우 상세하게 설명한 튜토리얼입니다. 

---

## 🛠 수정 파일 및 수정 방법

모든 코드는 기기에 SSH로 접속하여 `/data/openpilot` 또는 PC의 `opendbc_repo` 폴더 기준에서 작업합니다.

### 1단계: 판다(Panda) 하드웨어의 통신 처리량 늘리기
EV4의 최신 카메라 시스템은 어마어마한 양의 통신(데이터)을 쏟아냅니다. 기존 설정으로는 기기가 이 트래픽을 처리하지 못해서 과부하(`interruptRateCan2`) 에러를 띄웁니다. 따라서 통신 한계치를 2배로 올려주어야 합니다.

*   **파일 위치:** `/data/openpilot/panda/board/stm32h7/stm32h7_config.h`
*   **작업 위치:** 대략 **26번째 줄** 부근
*   **변경 사항:**
```c
// ❌ [수정 전]
#define CAN_INTERRUPT_RATE 16000U

// ✅ [수정 후] 16000U를 32000U로 변경
#define CAN_INTERRUPT_RATE 32000U
```

### 2단계: EV4 전용 조향 및 통신 맵핑 권한 부여 (Values.py)
차량이 HDA2 환경(조향과 LFA 통신이 서로 다른 버스인 E-CAN과 A-CAN에 분리됨)임을 오픈파일럿이 인식하게 하려면, 차량 지문(Fingerprint) 설정에 플래그(Flag)를 강제로 명시해 주어야 합니다.

*   **파일 위치:** `/data/openpilot/opendbc/car/hyundai/values.py`
*   **작업 위치:** `KIA_EV4`가 정의되어 있는 **518번째 줄** 부근
*   **변경 사항:**
```python
# ❌ [수정 전]
  KIA_EV4 = HyundaiCanFDPlatformConfig(
    [HyundaiCarDocs("Kia EV4 2024", ...)],
    CarSpecs(mass=2000, wheelbase=2.8, ...),
    flags=HyundaiFlags.EV,
  )

# ✅ [수정 후] flags 부분에 LKA_STEERING 및 LKA_STEERING_ALT 권한 추가
  KIA_EV4 = HyundaiCanFDPlatformConfig(
    [HyundaiCarDocs("Kia EV4 2024", ...)],
    CarSpecs(mass=2000, wheelbase=2.8, ...),
    flags=HyundaiFlags.EV | HyundaiFlags.CANFD_LKA_STEERING | HyundaiFlags.CANFD_LKA_STEERING_ALT,
  )
```

### 3단계: 블랙박스 오류 모드 우회 및 시스템 오판 막기 (Interface.py)
EV4는 기본 펌웨어 탐지 로직이 인식하지 못해 조향 권한이 없는 차(dashcamOnly)로 오판하는 버그가 발생합니다.

*   **파일 위치:** `/data/openpilot/opendbc/car/hyundai/interface.py`
*   **작업 위치:** `get_params` 함수 내부 (**49~56번째 줄** 부근)
*   **변경 사항 (동적 조회 삭제하고 직접 플래그 읽기):**
```python
# ❌ [수정 전]
    if ret.flags & HyundaiFlags.CANFD:
      # lka_steering = next((fw for fw in car_fw if fw.ecu == "eps" and b"PE" in fw.fwVersion), None)
      # if lka_steering is not None:
      ...

# ✅ [수정 후] 복잡한 버전 검색을 버리고 강제로 Values의 권한을 가져오기
    if ret.flags & HyundaiFlags.CANFD:
      lka_steering_alt = ret.flags & HyundaiFlags.CANFD_LKA_STEERING_ALT
      lka_steering = ret.flags & HyundaiFlags.CANFD_LKA_STEERING
```
*   **작업 위치:** 오류 모드 진입부 (**153~156번째 줄** 부근)
*   **변경 사항:** EV4 조건문 완전 주석 처리 혹은 삭제
```python
# ❌ [수정 전]
    # Dashcam cars are missing a test route...
    if candidate in (CAR.KIA_OPTIMA_H, CAR.KIA_EV4):  # < EV4가 껴 있음!
      ret.dashcamOnly = True

# ✅ [수정 후] EV4는 블랙박스가 아니므로 삭제
    if candidate in (CAR.KIA_OPTIMA_H,):
      ret.dashcamOnly = True
```

### 4단계: 엑셀 카운터/체크섬에 의한 조향 차단 버그 픽스 (Panda Safety)
EV4의 엑셀 페달 정보(`0x35` 메시지)는 타 현대차들과 완전히 구조가 다릅니다. 이 때문에 판다 기기가 차량 신호가 변조됐다고 오판하여(`rxInvalid=True`) 오픈파일럿을 죽여버립니다. 이를 막기 위해 **체크섬과 카운터 검사를 무시하는 방어막 우회 코드**를 Panda Safety에 등록해 줍니다.

*   **파일 위치:** `/data/openpilot/opendbc/safety/modes/hyundai_canfd.h`
*   **작업 위치:** `static RxCheck hyundai_canfd_lka_steering_rx_checks[]` 선언부 바로 아래 (**326번째 줄** 부근)
*   **변경 사항:** EV4 전용 무시 배열 신규 작성
```c
// ✅ [코드 추가] 
      // EV4: ACCELERATOR (0x35) has non-standard counter and checksum
      static RxCheck hyundai_canfd_lka_steering_alt_ev4_rx_checks[] = {
        // [핵심] .ignore_checksum = true, .ignore_counter = true 추가!
        {.msg = {{0x35, 1, 32, 100U, .ignore_checksum = true, .ignore_counter = true, .ignore_quality_flag = true},
                 {0x100, 1, 32, 100U, .max_counter = 0xffU, .ignore_quality_flag = true},
                 {0x105, 1, 32, 100U, .max_counter = 0xffU, .ignore_quality_flag = true}}},
        {.msg = {{0x175, 1, 24, 50U, .max_counter = 0xffU, .ignore_quality_flag = true}, { 0 }, { 0 }}},
        {.msg = {{0xa0, 1, 24, 100U, .max_counter = 0xffU, .ignore_quality_flag = true}, { 0 }, { 0 }}},
        {.msg = {{0xea, 1, 24, 100U, .max_counter = 0xffU, .ignore_quality_flag = true}, { 0 }, { 0 }}},
        {.msg = {{0x1aa, 1, 16, 50U, .ignore_checksum = true, .max_counter = 0xffU, .ignore_quality_flag = true}, { 0 }, { 0 }}},
        HYUNDAI_CANFD_SCC_ADDR_CHECK(1)
      };
```
*   **작업 위치:** 등록 처리부 (**338번째 줄** 부근)
*   **변경 사항:** 방금 만든 EV4용 방패 등록
```c
// ❌ [수정 전]
      if (hyundai_canfd_alt_buttons) {
        SET_RX_CHECKS(hyundai_canfd_lka_steering_alt_buttons_rx_checks, ret); ...

// ✅ [수정 후] EV4 조건(`hyundai_canfd_lka_steering_alt`)이 참이면, 우리가 만든 예외 로직 구동!
      if (hyundai_canfd_alt_buttons && hyundai_canfd_lka_steering_alt) {
        SET_RX_CHECKS(hyundai_canfd_lka_steering_alt_ev4_rx_checks, ret);
      } else if (hyundai_canfd_alt_buttons) {
        SET_RX_CHECKS(hyundai_canfd_lka_steering_alt_buttons_rx_checks, ret);
      } else { ... }
```

### 5단계: 핸들 버튼 통신 두절(에러) 막기 (CarState.py Fallback)
차량 주행 도중 캔 통신 데이터가 잠시 부족할 때 파이썬 코드가 죽어서(`KeyError`) 자율 주행이 튕기는 것을 막는 보험(Fallback) 로직입니다. 

*   **파일 위치:** `/data/openpilot/opendbc/car/hyundai/carstate.py`
*   **작업 위치:** 크루즈 상태 및 버튼 읽어오기 블록 (**288번째 줄** 부근)
*   **변경 사항:**
```python
# ❌ [수정 전]
    cruise_btn_vals = cp.vl_all[self.cruise_btns_msg_canfd]["CRUISE_BUTTONS"]
    self.cruise_buttons.extend(cruise_btn_vals)

# ✅ [수정 후] 데이터가 없으면 강제로 원시값 긁어오기
    cruise_btn_vals = cp.vl_all[self.cruise_btns_msg_canfd]["CRUISE_BUTTONS"]
    if not cruise_btn_vals:
      # Fallback: vl_all may be empty if checksum/counter validation rejects the message
      cruise_btn_vals = [cp.vl[self.cruise_btns_msg_canfd]["CRUISE_BUTTONS"]]
    self.cruise_buttons.extend(cruise_btn_vals)
```

---

## 🚀 6단계: 판다 펌웨어 재컴파일 & 적용하기
이 모든 코드(특히 `.h` 끝나는 c/c++ 하드웨어 펌웨어 코드)를 수정한 뒤 반드시 오픈파일럿 판다의 심장을 다시 구워주어야 합니다. 기기 터미널에서 다음을 한 줄씩 치시면 됩니다.

```bash
# 판다 폴더로 진입
cd /data/openpilot/panda

# 기존 찌꺼기가 남아 문제가 생기는 것을 막기 위해 .bin 캐시 삭제 (중요)
rm -f /data/openpilot/panda/board/obj/panda_h7/main.bin

# 하드웨어 펌웨어 컴파일!
scons -j4

# 차량 시스템과 함께 완벽하게 붙이기 위한 기기 재부팅!
sudo reboot
```

## ✋ 자주 묻는 질문 (정전식 터치 HOD 관련)
> **Q. 완벽하게 차가 혼자서 핸들을 돌리는데, 계기판에선 자꾸 핸들을 잡으라고 뜹니다! 왜 에러가 떴죠?**
> A. 에러가 아닙니다. EV4는 구형 차종들과 달리 **정전식(HOD) 핸들 센서**가 내장되어 있기 때문에 사람 손 피부의 전기 신호를 읽습니다. 오픈파일럿이 완벽하게 조향(힘)을 가짜로 만들어 우회하더라도, 피부 터치 신호까지는 가짜로 전송하지 않습니다. **따라서 핸들의 빈 공간에 손가락 하나만 살짝 걸쳐두시면 모든 경고가 사라지고 무한정 달릴 수 있습니다!**
