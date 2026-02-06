# ESP32-S3 부팅 메시지 완벽 해석

## 목차
1. [ROM 부트로더 단계](#1-rom-부트로더-단계)
2. [2nd Stage 부트로더](#2-2nd-stage-부트로더)
3. [파티션 테이블](#3-파티션-테이블)
4. [이미지 세그먼트 로딩](#4-이미지-세그먼트-로딩)
5. [CPU 및 시스템 초기화](#5-cpu-및-시스템-초기화)
6. [메모리 힙 초기화](#6-메모리-힙-초기화)
7. [플래시 메모리 감지](#7-플래시-메모리-감지)
8. [애플리케이션 시작](#8-애플리케이션-시작)
9. [부팅 타임라인](#9-부팅-타임라인)
10. [약어 및 용어 설명](#10-약어-및-용어-설명)

---

## 1. ROM 부트로더 단계

### 초기 ROM 부트 정보
```
ESP-ROM:esp32s3-20210327
Build:Mar 27 2021
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **ROM 버전** | esp32s3-20210327 | ESP32-S3 ROM 부트로더 빌드 날짜 |
| **빌드일** | 2021년 3월 27일 | ROM 코드 컴파일 시점 (칩 제조 시 고정) |

### 리셋 원인 및 부트 모드
```
rst:0x15 (USB_UART_CHIP_RESET)
boot:0x28 (SPI_FAST_FLASH_BOOT)
Saved PC:0x40377786
```

| 필드 | 값 | 설명 |
|------|-----|------|
| **rst** | 0x15 | 리셋 원인 코드 |
| **리셋 타입** | USB_UART_CHIP_RESET | USB-UART를 통한 칩 리셋 (개발 중 일반적) |
| **boot** | 0x28 | 부트 모드 레지스터 값 |
| **부트 모드** | SPI_FAST_FLASH_BOOT | SPI Flash에서 고속 부팅 |
| **Saved PC** | 0x40377786 | 리셋 전 프로그램 카운터 값 (디버깅용) |

### SPI Flash 설정
```
SPIWP:0xee
mode:DIO, clock div:1
```

| 설정 | 값 | 설명 |
|------|-----|------|
| **SPIWP** | 0xee | SPI Write Protect 핀 상태 |
| **모드** | DIO | Dual I/O 모드 (2선 데이터 통신) |
| **클럭 분주** | 1 | 최고속 (분주 없음, 80MHz) |

### 부트로더 로딩
```
load:0x3fce2820,len:0x158c   (5,516 bytes)
load:0x403c8700,len:0xd24    (3,364 bytes)
load:0x403cb700,len:0x2f34   (12,084 bytes)
entry 0x403c8924
```

| 세그먼트 | 주소 | 크기 | 용도 |
|----------|------|------|------|
| **Segment 1** | 0x3fce2820 | 5,516 bytes | DRAM 데이터 세그먼트 |
| **Segment 2** | 0x403c8700 | 3,364 bytes | IRAM 코드 세그먼트 |
| **Segment 3** | 0x403cb700 | 12,084 bytes | 추가 코드/데이터 |
| **Entry Point** | 0x403c8924 | - | 2nd stage 부트로더 시작 주소 |

---

## 2. 2nd Stage 부트로더

### 부트로더 정보
```
I (18) boot: ESP-IDF v5.5.2 2nd stage bootloader
I (18) boot: compile time Feb  5 2026 14:16:34
I (18) boot: Multicore bootloader
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **ESP-IDF 버전** | v5.5.2 | 사용된 개발 프레임워크 버전 |
| **컴파일 시각** | 2026-02-05 14:16:34 | 부트로더 빌드 시간 |
| **코어 지원** | Multicore | 듀얼코어 지원 부트로더 |

### 칩 정보
```
I (18) boot: chip revision: v0.2
I (21) boot: efuse block revision: v1.3
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **칩 리비전** | v0.2 | ESP32-S3 하드웨어 버전 |
| **eFuse 블록** | v1.3 | eFuse 메모리 레이아웃 버전 |

### SPI 설정
```
I (25) boot.esp32s3: Boot SPI Speed : 80MHz
I (28) boot.esp32s3: SPI Mode       : DIO
I (32) boot.esp32s3: SPI Flash Size : 2MB
```

| 설정 | 값 | 설명 |
|------|-----|------|
| **SPI 속도** | 80MHz | Flash 통신 클럭 속도 |
| **SPI 모드** | DIO | Dual I/O (2-bit 데이터 전송) |
| **Flash 크기** | 2MB | 외장 Flash 메모리 용량 |

### RNG (난수 생성기)
```
I (36) boot: Enabling RNG early entropy source...
```
부트 단계에서 하드웨어 난수 생성기를 활성화하여 보안 초기화에 사용

---

## 3. 파티션 테이블

### 파티션 구조
```
I (40) boot: Partition Table:
I (43) boot: ## Label            Usage          Type ST Offset   Length
I (49) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (56) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (62) boot:  2 factory          factory app      00 00 00010000 00100000
I (69) boot: End of partition table
```

### 파티션 상세 분석

| # | Label | Usage | Type | SubType | Offset | Length | 크기(KB) | 용도 |
|---|-------|-------|------|---------|--------|--------|----------|------|
| **0** | nvs | WiFi data | 01 (data) | 02 (nvs) | 0x9000 | 0x6000 | 24KB | Wi-Fi 설정, NVS 저장소 |
| **1** | phy_init | RF data | 01 (data) | 01 (phy) | 0xf000 | 0x1000 | 4KB | RF 캘리브레이션 데이터 |
| **2** | factory | factory app | 00 (app) | 00 (factory) | 0x10000 | 0x100000 | 1024KB | 메인 펌웨어 애플리케이션 |

### 파티션 메모리 맵
```
0x0000 ─────────────────
       │ Bootloader    │ (부트로더 영역)
0x9000 ─────────────────
       │ NVS (24KB)    │ (비휘발성 저장소)
0xF000 ─────────────────
       │ PHY Init (4KB)│ (RF 초기화 데이터)
0x10000 ────────────────
       │               │
       │  Factory App  │ (1MB 펌웨어)
       │  (1024KB)     │
       │               │
0x110000 ───────────────
```

---

## 4. 이미지 세그먼트 로딩

### Segment 0: Flash XIP 매핑 (코드)
```
I (72) esp_image: segment 0: paddr=00010020 vaddr=3c020020 size=08670h ( 34416) map
```
- **물리 주소**: 0x00010020 (Flash)
- **가상 주소**: 0x3c020020 (Flash Cache 영역)
- **크기**: 34,416 bytes (33.6KB)
- **타입**: map (Flash에서 직접 실행, XIP)

### Segment 1: DRAM 로드
```
I (86) esp_image: segment 1: paddr=00018698 vaddr=3fc92000 size=02920h ( 10528) load
```
- **물리 주소**: 0x00018698
- **가상 주소**: 0x3fc92000 (DRAM)
- **크기**: 10,528 bytes (10.3KB)
- **타입**: load (RAM에 복사 후 사용)
- **용도**: 초기화된 데이터 변수

### Segment 2: IRAM 로드
```
I (89) esp_image: segment 2: paddr=0001afc0 vaddr=40374000 size=05058h ( 20568) load
```
- **물리 주소**: 0x0001afc0
- **가상 주소**: 0x40374000 (IRAM)
- **크기**: 20,568 bytes (20.1KB)
- **타입**: load (고속 실행을 위해 RAM 복사)
- **용도**: 시간 임계적 코드 (ISR 등)

### Segment 3: Flash XIP 매핑 (데이터)
```
I (99) esp_image: segment 3: paddr=00020020 vaddr=42000020 size=157ech ( 88044) map
```
- **물리 주소**: 0x00020020
- **가상 주소**: 0x42000020 (Flash Cache)
- **크기**: 88,044 bytes (86.0KB)
- **타입**: map (Flash에서 직접 읽기)
- **용도**: 읽기 전용 데이터

### Segment 4: 추가 코드 로드
```
I (117) esp_image: segment 4: paddr=00035814 vaddr=40379058 size=08f00h ( 36608) load
```
- **물리 주소**: 0x00035814
- **가상 주소**: 0x40379058 (IRAM)
- **크기**: 36,608 bytes (35.8KB)
- **타입**: load
- **용도**: 추가 실행 코드

### Segment 5: RTC Slow Memory
```
I (126) esp_image: segment 5: paddr=0003e71c vaddr=50000000 size=00020h (    32) load
```
- **물리 주소**: 0x0003e71c
- **가상 주소**: 0x50000000 (RTC Slow Memory)
- **크기**: 32 bytes
- **타입**: load
- **용도**: 딥슬립 중 유지되는 데이터

### 세그먼트 요약

| Segment | 타입 | 주소 영역 | 크기 | 용도 |
|---------|------|-----------|------|------|
| **0** | map | 0x3c020020 | 33.6KB | Flash 코드 (XIP) |
| **1** | load | 0x3fc92000 | 10.3KB | DRAM 데이터 |
| **2** | load | 0x40374000 | 20.1KB | IRAM 코드 |
| **3** | map | 0x42000020 | 86.0KB | Flash 데이터 (읽기 전용) |
| **4** | load | 0x40379058 | 35.8KB | IRAM 추가 코드 |
| **5** | load | 0x50000000 | 32B | RTC 메모리 |
| **총합** | - | - | **185.8KB** | - |

---

## 5. CPU 및 시스템 초기화

### 애플리케이션 로드
```
I (132) boot: Loaded app from partition at offset 0x10000
I (132) boot: Disabling RNG early entropy source...
```
- Factory 파티션(0x10000)에서 앱 로드 완료
- 초기 RNG 비활성화 (앱 단계 RNG으로 전환)

### CPU 시작
```
I (143) cpu_start: Multicore app
I (152) cpu_start: GPIO 44 and 43 are used as console UART I/O pins
I (153) cpu_start: Pro cpu start user code
I (153) cpu_start: cpu freq: 160000000 Hz
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **CPU 모드** | Multicore app | 듀얼코어 애플리케이션 |
| **UART TX** | GPIO 43 | 콘솔 출력 핀 |
| **UART RX** | GPIO 44 | 콘솔 입력 핀 |
| **시작 CPU** | Pro CPU (CPU0) | 프로토콜 CPU가 먼저 시작 |
| **CPU 클럭** | 160MHz | 동작 주파수 |

### 애플리케이션 정보
```
I (154) app_init: Application information:
I (158) app_init: Project name:     ESP-Web-Monitor
I (163) app_init: App version:      1
I (166) app_init: Compile time:     Feb  5 2026 15:28:25
I (171) app_init: ELF file SHA256:  e0e0ba4fa...
I (175) app_init: ESP-IDF:          v5.5.2
```

| 항목 | 값 |
|------|-----|
| **프로젝트명** | ESP-Web-Monitor |
| **버전** | 1 |
| **컴파일 시각** | 2026-02-05 15:28:25 |
| **ELF SHA256** | e0e0ba4fa... |
| **ESP-IDF** | v5.5.2 |

### eFuse 칩 정보
```
I (179) efuse_init: Min chip rev:     v0.0
I (183) efuse_init: Max chip rev:     v0.99
I (187) efuse_init: Chip rev:         v0.2
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **최소 리비전** | v0.0 | 펌웨어가 지원하는 최소 칩 버전 |
| **최대 리비전** | v0.99 | 펌웨어가 지원하는 최대 칩 버전 |
| **현재 리비전** | v0.2 | 현재 칩의 하드웨어 버전 |

---

## 6. 메모리 힙 초기화

### 힙 메모리 영역
```
I (191) heap_init: Initializing. RAM available for dynamic allocation:
I (197) heap_init: At 3FC95190 len 00054580 (337 KiB): RAM
I (202) heap_init: At 3FCE9710 len 00005724 (21 KiB): RAM
I (207) heap_init: At 3FCF0000 len 00008000 (32 KiB): DRAM
I (213) heap_init: At 600FE000 len 00001FE8 (7 KiB): RTCRAM
```

### 힙 메모리 맵

| 영역 | 시작 주소 | 길이 | 크기(KB) | 타입 | 용도 |
|------|-----------|------|----------|------|------|
| **영역 1** | 0x3FC95190 | 0x54580 | **337KB** | RAM | 주요 동적 메모리 |
| **영역 2** | 0x3FCE9710 | 0x5724 | **21KB** | RAM | 추가 DRAM |
| **영역 3** | 0x3FCF0000 | 0x8000 | **32KB** | DRAM | 데이터 RAM |
| **영역 4** | 0x600FE000 | 0x1FE8 | **7KB** | RTCRAM | RTC 슬로우 메모리 |
| **총 힙 크기** | - | - | **397KB** | - | malloc/new 사용 가능 |

### 힙 메모리 시각화
```
DRAM 영역 1 (337KB) ████████████████████████████████████ 84.9%
DRAM 영역 2 (21KB)  ███ 5.3%
DRAM 영역 3 (32KB)  █████ 8.1%
RTCRAM (7KB)        █ 1.8%
```

---

## 7. 플래시 메모리 감지

### Flash 칩 감지
```
I (219) spi_flash: detected chip: boya
I (221) spi_flash: flash io: dio
W (224) spi_flash: Detected size(16384k) larger than the size in the binary image header(2048k). 
                   Using the size in the binary image header.
```

| 항목 | 값 | 설명 |
|------|-----|------|
| **제조사** | Boya | Flash 칩 제조업체 |
| **I/O 모드** | DIO | Dual I/O 모드 |
| **물리 크기** | 16MB (16384KB) | 실제 Flash 칩 용량 |
| **사용 크기** | 2MB (2048KB) | 파티션 테이블에 정의된 크기 |
| **경고** | ⚠️ 크기 불일치 | 실제 칩이 더 크지만 2MB만 사용 |

**참고**: 16MB Flash가 장착되어 있지만, 파티션 설정에서 2MB만 정의되어 나머지 14MB는 미사용 상태입니다.

---

## 8. 애플리케이션 시작

### GPIO 슬립 설정
```
I (237) sleep_gpio: Configure to isolate all GPIO pins in sleep state
I (243) sleep_gpio: Enable automatic switching of GPIO sleep configuration
```
- 딥슬립 진입 시 모든 GPIO 핀 격리
- 자동 슬립 설정 전환 활성화

### Main Task 시작
```
I (250) main_task: Started on CPU0
I (260) main_task: Calling app_main()
```
- **CPU**: CPU0 (Pro CPU)에서 메인 태스크 시작
- **진입점**: `app_main()` 함수 호출

### 애플리케이션 출력
```
ESP-Web-Monitor by Jinho Jung!
This is esp32s3 chip with 2 CPU core(s), WiFi/BLE, silicon revision v0.2, 2MB external flash
Minimum free heap size: 392632 bytes
```

| 정보 | 값 |
|------|-----|
| **프로젝트** | ESP-Web-Monitor |
| **개발자** | Jinho Jung |
| **칩 모델** | ESP32-S3 |
| **CPU 코어** | 2개 (듀얼코어) |
| **무선 기능** | Wi-Fi + BLE (Bluetooth Low Energy) |
| **칩 리비전** | v0.2 |
| **외장 Flash** | 2MB |
| **최소 여유 힙** | 392,632 bytes (383.4KB) |

---

## 9. 부팅 타임라인

### 시간별 이벤트 분석

| 시간(ms) | 이벤트 | 모듈 | 단계 |
|----------|--------|------|------|
| **0** | ROM 부트로더 시작 | ROM | 1단계 |
| **~15** | 2nd stage 부트로더 로드 | ROM | - |
| **18** | 2nd stage 부트로더 시작 | boot | 2단계 |
| **40** | 파티션 테이블 읽기 | boot | 3단계 |
| **69** | 파티션 테이블 완료 | boot | - |
| **72** | 이미지 세그먼트 로딩 시작 | esp_image | 4단계 |
| **132** | 앱 로드 완료 | boot | - |
| **143** | CPU 시작 | cpu_start | 5단계 |
| **154** | 애플리케이션 정보 출력 | app_init | - |
| **191** | 힙 초기화 | heap_init | 6단계 |
| **219** | Flash 감지 | spi_flash | 7단계 |
| **250** | 메인 태스크 시작 | main_task | 8단계 |
| **260** | app_main() 호출 | main_task | 9단계 |

### 부팅 플로우차트

```
[0ms] ROM 부트로더 실행
   ↓
[~15ms] 2nd Stage 부트로더 로드
   ↓
[18ms] ESP-IDF v5.5.2 부트로더 시작
   ↓ (칩 정보 확인)
   ↓ (SPI Flash 설정)
   ↓
[40ms] 파티션 테이블 읽기
   ↓ (NVS, PHY, Factory 확인)
   ↓
[72ms] 이미지 세그먼트 로딩
   ↓ (6개 세그먼트 RAM/Flash 매핑)
   ↓
[132ms] 앱 로드 완료
   ↓
[143ms] CPU 초기화 (160MHz)
   ↓
[154ms] 애플리케이션 정보 출력
   ↓
[191ms] 힙 메모리 초기화 (397KB)
   ↓
[219ms] Flash 칩 감지 (Boya, 16MB)
   ↓
[237ms] GPIO 슬립 설정
   ↓
[250ms] Main Task 시작 (CPU0)
   ↓
[260ms] app_main() 실행
   ↓
[사용자 코드 실행]
```

### 부팅 성능 분석

| 단계 | 소요 시간 | 비율 |
|------|-----------|------|
| ROM → 2nd Boot | ~18ms | 6.9% |
| 파티션 로딩 | 22ms (18-40ms) | 8.5% |
| 세그먼트 로딩 | 60ms (72-132ms) | 23.1% |
| CPU/힙 초기화 | 59ms (132-191ms) | 22.7% |
| Flash/GPIO 설정 | 46ms (191-237ms) | 17.7% |
| Main Task 준비 | 23ms (237-260ms) | 8.8% |
| **총 부팅 시간** | **~260ms** | **100%** |

---

## 10. 약어 및 용어 설명

### 칩 및 하드웨어

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **ESP32-S3** | ESP32 Series 3 | Espressif의 3세대 Wi-Fi/BLE SoC |
| **ROM** | Read-Only Memory | 제조 시 고정된 부트로더 코드 |
| **SoC** | System on Chip | 단일 칩에 통합된 완전한 시스템 |
| **eFuse** | Electronic Fuse | 한 번 쓰기 가능한 비휘발성 메모리 |
| **PC** | Program Counter | 다음 실행할 명령어 주소 레지스터 |

### 메모리 관련

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **RAM** | Random Access Memory | 임의 접근 메모리 (휘발성) |
| **DRAM** | Data RAM | 데이터 전용 RAM, 변수 및 힙 저장 |
| **IRAM** | Instruction RAM | 명령어 전용 RAM, 고속 코드 실행 |
| **RTC RAM** | Real-Time Clock RAM | 딥슬립 중에도 유지되는 저속 메모리 |
| **RTCRAM** | RTC Slow Memory | RTC 슬로우 메모리 (7KB) |
| **Flash** | Flash Memory | 비휘발성 저장장치 (프로그램 저장) |
| **NVS** | Non-Volatile Storage | 비휘발성 저장소 (설정값 유지) |
| **XIP** | eXecute In Place | Flash에서 직접 코드 실행 (RAM 복사 없이) |
| **Heap** | Heap Memory | 동적 메모리 할당 영역 (malloc/new) |

### 주소 관련

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **paddr** | Physical Address | 물리 주소 (Flash 메모리 상의 실제 위치) |
| **vaddr** | Virtual Address | 가상 주소 (CPU가 접근하는 논리적 주소) |
| **offset** | Offset | 파티션 시작 위치 (바이트 단위) |
| **len** | Length | 길이, 크기 |

### 메모리 주소 맵

| 주소 범위 | 메모리 타입 | 용도 |
|-----------|-------------|------|
| **0x3c000000 - 0x3c800000** | Flash Cache (XIP) | 코드 실행 영역 (읽기 전용) |
| **0x3fc00000 - 0x3fd00000** | DRAM | 데이터 RAM (읽기/쓰기) |
| **0x40370000 - 0x403e0000** | IRAM | 명령어 RAM (고속 실행) |
| **0x42000000 - 0x42800000** | Flash Cache | 데이터 매핑 영역 |
| **0x50000000 - 0x50002000** | RTC Slow Memory | 딥슬립 유지 메모리 (8KB) |
| **0x600fe000 - 0x60100000** | RTC Fast Memory | RTC 고속 메모리 |

### 부팅 및 시스템

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **rst** | Reset | 리셋 원인 코드 |
| **boot** | Boot Mode | 부팅 모드 레지스터 |
| **IDF** | IoT Development Framework | ESP32 공식 개발 프레임워크 |
| **ELF** | Executable and Linkable Format | 실행 파일 포맷 |
| **SHA256** | Secure Hash Algorithm 256-bit | 256비트 보안 해시 (무결성 검증) |

### 파티션 타입

| 코드 | 타입 | 설명 |
|------|------|------|
| **00** | app | 애플리케이션 파티션 (실행 코드) |
| **01** | data | 데이터 파티션 (설정, 파일 등) |
| **ST** | SubType | 파티션 하위 유형 코드 |
| **00 (app)** | factory | 기본 펌웨어 앱 |
| **01 (data)** | phy | RF 캘리브레이션 데이터 |
| **02 (data)** | nvs | 비휘발성 저장소 |

### 통신 관련

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **SPI** | Serial Peripheral Interface | 직렬 주변장치 인터페이스 |
| **DIO** | Dual I/O | 2선 데이터 전송 모드 (MOSI+MISO) |
| **QIO** | Quad I/O | 4선 데이터 전송 모드 (더 빠름) |
| **UART** | Universal Asynchronous Receiver/Transmitter | 범용 비동기 송수신기 (시리얼) |
| **GPIO** | General Purpose Input/Output | 범용 입출력 핀 |
| **TX** | Transmit | 송신 (데이터 전송) |
| **RX** | Receive | 수신 (데이터 받기) |
| **I/O** | Input/Output | 입출력 |

### 무선 통신

| 약어 | 전체 명칭 | 설명 |
|------|-----------|------|
| **Wi-Fi** | Wireless Fidelity | 무선 랜 (IEEE 802.11) |
| **BLE** | Bluetooth Low Energy | 저전력 블루투스 |
| **RF** | Radio Frequency | 무선 주파수 |
| **PHY** | Physical Layer | 물리 계층 (무선 하드웨어) |

### CPU 관련

| 용어 | 설명 |
|------|------|
| **Pro CPU** | Protocol CPU (CPU0) - 프로토콜 처리 담당 |
| **App CPU** | Application CPU (CPU1) - 애플리케이션 처리 담당 |
| **CPU0** | 첫 번째 CPU 코어 (부팅 시 먼저 시작) |
| **CPU1** | 두 번째 CPU 코어 (필요 시 활성화) |
| **Multicore** | 멀티코어 (듀얼코어 지원) |

### 로그 레벨

| 기호 | 레벨 | 설명 |
|------|------|------|
| **I** | Info | 정보성 메시지 (정상 동작) |
| **W** | Warning | 경고 메시지 (주의 필요) |
| **E** | Error | 오류 메시지 (문제 발생) |
| **D** | Debug | 디버그 메시지 (개발용) |
| **V** | Verbose | 상세 메시지 (매우 자세함) |

### 크기 단위

| 단위 | 의미 | 바이트 환산 |
|------|------|-------------|
| **B** | Byte | 1 |
| **KB** | Kilobyte | 1,024 bytes |
| **MB** | Megabyte | 1,048,576 bytes (1,024 KB) |
| **KiB** | Kibibyte | 1,024 bytes (표준 표기) |
| **MiB** | Mebibyte | 1,048,576 bytes (표준 표기) |

### 주파수 단위

| 단위 | 의미 | 값 |
|------|------|-----|
| **Hz** | Hertz | 초당 1 사이클 |
| **MHz** | Mega Hertz | 1,000,000 Hz |
| **GHz** | Giga Hertz | 1,000,000,000 Hz |

### 숫자 표기법

| 표기 | 의미 | 예시 |
|------|------|------|
| **0x** | 16진수 (Hexadecimal) | 0x9000 = 36,864 (10진수) |
| **h** | 16진수 크기 표기 | 08670h = 34,416 bytes |
| **v** | 버전 (Version) | v5.5.2, v0.2 |

### 리셋 원인 코드

| 코드 | 의미 | 설명 |
|------|------|------|
| **0x01** | POWERON_RESET | 전원 인가 리셋 |
| **0x03** | SW_RESET | 소프트웨어 리셋 |
| **0x0C** | RTCWDT_RESET | RTC 워치독 리셋 |
| **0x15** | USB_UART_CHIP_RESET | USB-UART 칩 리셋 (개발용) |

---

## 11. 메모리 최적화 분석

### 메모리 사용 현황

| 영역 | 사용량 | 여유 | 총량 | 사용률 |
|------|--------|------|------|--------|
| **DRAM 힙** | ~4KB | 393KB | 397KB | 1% |
| **IRAM 코드** | 56KB | - | - | - |
| **Flash 코드** | 120KB | 904KB | 1024KB | 11.7% |

### 최적화 제안

1. **Flash 용량 확대**: 16MB Flash가 장착되어 있으나 2MB만 사용 중
   - 파티션 테이블 수정으로 OTA, 데이터 파티션 추가 가능
   
2. **힙 메모리**: 393KB 여유로 충분한 여유 공간 확보

3. **코드 최적화**: Factory 파티션 11.7% 사용 (여유 공간 충분)

---

## 12. 참고 자료

### 공식 문서
- [ESP-IDF 프로그래밍 가이드](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/index.html)
- [ESP32-S3 기술 레퍼런스 매뉴얼](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf)
- [ESP32-S3 데이터시트](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf)

### ESP-IDF 가이드
- [파티션 테이블](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/partition-tables.html)
- [메모리 레이아웃](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/memory-types.html)
- [부팅 프로세스](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/startup.html)
- [빌드 시스템](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/build-system.html)

### 하드웨어 리소스
- [ESP32-S3 핀 레이아웃](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/hw-reference/esp32s3/user-guide-devkitc-1.html)
- [SPI Flash 가이드](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/spi_flash/index.html)

---

## 원본 부팅 로그

```
ESP-ROM:esp32s3-20210327
Build:Mar 27 2021
rst:0x15 (USB_UART_CHIP_RESET),boot:0x28 (SPI_FAST_FLASH_BOOT)
Saved PC:0x40377786
SPIWP:0xee
mode:DIO, clock div:1
load:0x3fce2820,len:0x158c
load:0x403c8700,len:0xd24
load:0x403cb700,len:0x2f34
entry 0x403c8924
I (18) boot: ESP-IDF v5.5.2 2nd stage bootloader
I (18) boot: compile time Feb  5 2026 14:16:34
I (18) boot: Multicore bootloader
I (18) boot: chip revision: v0.2
I (21) boot: efuse block revision: v1.3
I (25) boot.esp32s3: Boot SPI Speed : 80MHz
I (28) boot.esp32s3: SPI Mode       : DIO
I (32) boot.esp32s3: SPI Flash Size : 2MB
I (36) boot: Enabling RNG early entropy source...
I (40) boot: Partition Table:
I (43) boot: ## Label            Usage          Type ST Offset   Length
I (49) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (56) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (62) boot:  2 factory          factory app      00 00 00010000 00100000
I (69) boot: End of partition table
I (72) esp_image: segment 0: paddr=00010020 vaddr=3c020020 size=08670h ( 34416) map
I (86) esp_image: segment 1: paddr=00018698 vaddr=3fc92000 size=02920h ( 10528) load
I (89) esp_image: segment 2: paddr=0001afc0 vaddr=40374000 size=05058h ( 20568) load
I (99) esp_image: segment 3: paddr=00020020 vaddr=42000020 size=157ech ( 88044) map
I (117) esp_image: segment 4: paddr=00035814 vaddr=40379058 size=08f00h ( 36608) load
I (126) esp_image: segment 5: paddr=0003e71c vaddr=50000000 size=00020h (    32) load
I (132) boot: Loaded app from partition at offset 0x10000
I (132) boot: Disabling RNG early entropy source...
I (143) cpu_start: Multicore app
I (152) cpu_start: GPIO 44 and 43 are used as console UART I/O pins
I (153) cpu_start: Pro cpu start user code
I (153) cpu_start: cpu freq: 160000000 Hz
I (154) app_init: Application information:
I (158) app_init: Project name:     ESP-Web-Monitor
I (163) app_init: App version:      1
I (166) app_init: Compile time:     Feb  5 2026 15:28:25
I (171) app_init: ELF file SHA256:  e0e0ba4fa...
I (175) app_init: ESP-IDF:          v5.5.2
I (179) efuse_init: Min chip rev:     v0.0
I (183) efuse_init: Max chip rev:     v0.99
I (187) efuse_init: Chip rev:         v0.2
I (191) heap_init: Initializing. RAM available for dynamic allocation:
I (197) heap_init: At 3FC95190 len 00054580 (337 KiB): RAM
I (202) heap_init: At 3FCE9710 len 00005724 (21 KiB): RAM
I (207) heap_init: At 3FCF0000 len 00008000 (32 KiB): DRAM
I (213) heap_init: At 600FE000 len 00001FE8 (7 KiB): RTCRAM
I (219) spi_flash: detected chip: boya
I (221) spi_flash: flash io: dio
W (224) spi_flash: Detected size(16384k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (237) sleep_gpio: Configure to isolate all GPIO pins in sleep state
I (243) sleep_gpio: Enable automatic switching of GPIO sleep configuration
I (250) main_task: Started on CPU0
I (260) main_task: Calling app_main()
ESP-Web-Monitor by Jinho Jung!
This is esp32s3 chip with 2 CPU core(s), WiFi/BLE, silicon revision v0.2, 2MB external flash
Minimum free heap size: 392632 bytes
```

---

**문서 생성일**: 2026년 2월 5일  
**칩 모델**: ESP32-S3  
**ESP-IDF 버전**: v5.5.2  
**프로젝트**: ESP-Web-Monitor  
**작성자**: AI Assistant for jvisualschool (정진호)  
**문서 버전**: 2.0 (완전판)
