# KAIST PintOS Project 1 - Thread

**Pintos Project1 Thread 프로젝트** 구현 문서입니다.

본 프로젝트는 PintOS 운영체제의 스레드 관리 및 스케줄링 기능을 구현한 프로젝트로, Alarm Clock, Priority Scheduling, Advanced Scheduler(MLFQS) 세 가지 주요 기능을 구현하였습니다.

공식 문서: [https://casys-kaist.github.io/pintos-kaist/project1/introduction.html](https://casys-kaist.github.io/pintos-kaist/project1/introduction.html)

## 📎 관련 프로젝트
본 프로젝트는 KAIST PintOS 시리즈 중 **Project 1 (Thread)** 에 해당하며,  
이후 단계 프로젝트는 아래에서 확인할 수 있습니다.

- **Project 2 — User Program**  
  🔗 https://github.com/kkc1383/Krafton_Jungle_10th_Pintos_User_Program_WEEK10_11_Team11  

- **Project 3 — Virtual Memory**  
  🔗 https://github.com/kkc1383/Krafton_Jungle_Pintos_Virtual_Memory_WEEK12_13_Team12  

---

## 목차

1. [Alarm Clock](#1-alarm-clock)
2. [Priority Scheduling](#2-priority-scheduling)
3. [Advanced Scheduler (MLFQS)](#3-advanced-scheduler-mlfqs)
4. [빌드 및 테스트](#빌드-및-테스트)

---

## 1. Alarm Clock

### 개요
기존 `timer_sleep()` 함수의 busy-waiting 방식을 제거하고, 스레드를 블록 상태로 전환하여 CPU 자원을 효율적으로 사용하도록 개선했습니다.

### 주요 구현 내용

#### 데이터 구조
- **`sleep_list`**: 대기 중인 스레드를 wake_tick 기준으로 정렬하여 관리하는 전역 리스트 ([thread.c:34](pintos/threads/thread.c#L34))
- **`wake_tick`**: 각 스레드가 깨어나야 할 tick을 저장 ([thread.h:99](pintos/include/threads/thread.h#L99))
- **`sleep_elem`**: sleep_list에 삽입될 list element ([thread.h:100](pintos/include/threads/thread.h#L100))

#### 알고리즘

**`timer_sleep()` 함수 ([timer.c:87-110](pintos/devices/timer.c#L87-L110))**
```c
1. 음수 또는 0 ticks인 경우 즉시 반환
2. 현재 시간 + sleep_duration을 wake_tick에 저장
3. 인터럽트를 비활성화하여 임계 영역 보호
4. wake_tick 기준으로 sleep_list에 정렬 삽입 (wake_tick_less 비교 함수 사용)
5. thread_block()을 호출하여 스레드 블록
6. 인터럽트 복원
```

**`timer_interrupt()` 함수 ([timer.c:125-161](pintos/devices/timer.c#L125-L161))**
```c
1. 매 tick마다 호출
2. sleep_list 앞에서부터 순회
3. wake_tick <= 현재 tick인 스레드를 모두 깨움
4. wake_tick > 현재 tick인 스레드를 만나면 즉시 중단 (정렬된 리스트 활용)
```

### 설계 포인트

1. **정렬된 리스트 사용**: O(1) wake-up 성능을 위해 삽입 시 정렬
2. **인터럽트 안전성**: 임계 영역에서 인터럽트 비활성화
3. **예외 처리**: ticks <= 0인 경우 즉시 반환하여 예외 상황 처리

### 관련 파일
- [pintos/devices/timer.c](pintos/devices/timer.c)
- [pintos/threads/thread.c](pintos/threads/thread.c)
- [pintos/include/threads/thread.h](pintos/include/threads/thread.h)

---

## 2. Priority Scheduling

### 개요
기존의 Round-Robin 스케줄러를 우선순위 기반 선점형 스케줄러로 변경하고, Priority Donation을 구현하여 Priority Inversion 문제를 해결했습니다.

### 주요 구현 내용

#### 데이터 구조

**스레드 구조체 추가 ([thread.h:103-106](pintos/include/threads/thread.h#L103-L106))**
```c
int original_priority;         // 기부 전 원래 우선순위
struct list acquired_locks;    // 현재 보유 중인 락 리스트
struct lock *waiting_for_lock; // 대기 중인 락
int is_donated;                // 기부 상태 카운터
```

**락 구조체 추가 ([synch.h:23](pintos/include/threads/synch.h#L23))**
```c
struct list_elem holder_elem;  // 스레드의 acquired_locks 리스트 요소
```

#### 알고리즘

**1. 기본 우선순위 스케줄링**
- Ready 리스트를 우선순위 순으로 정렬 유지 ([thread.c:277](pintos/threads/thread.c#L277), [365](pintos/threads/thread.c#L365))
- `thread_priority_less()` 비교 함수: 높은 우선순위가 앞에 오도록 정렬 ([thread.c:779-784](pintos/threads/thread.c#L779-L784))
- 선점 시점:
  - `thread_unblock()`: 언블록된 스레드의 우선순위가 더 높으면 양보
  - `thread_yield()`: 더 높은 우선순위 스레드가 있을 때만 양보
  - `thread_set_priority()`: 새 우선순위가 낮아지면 양보

**2. Priority Donation ([synch.c:183-243](pintos/threads/synch.c#L183-L243))**

DFS 방식으로 중첩 락에 대한 우선순위 기부를 처리:

```c
donate_priority_dfs(lock, donating_priority):
  current_thread = lock.holder
  depth = 0

  while current_thread exists and depth < 8:
    if current_thread.priority >= donating_priority:
      break  // 이미 높은 우선순위를 가짐

    current_thread.priority = donating_priority

    if current_thread is READY:
      ready_list에서 재정렬  // 우선순위 변경 반영

    if current_thread.waiting_for_lock is NULL:
      break  // 더 이상 대기 중인 락이 없음

    current_thread = current_thread.waiting_for_lock.holder
    depth++
```

**3. Priority Recovery ([synch.c:267-299](pintos/threads/synch.c#L267-L299))**

락 해제 시 우선순위 복구:
```c
lock_release():
  1. acquired_locks에서 해당 락 제거
  2. 원래 우선순위로 초기화
  3. 보유 중인 다른 락들의 대기자 중 최고 우선순위 찾기
  4. 찾은 우선순위가 더 높으면 해당 값으로 갱신
  5. 필요 시 스레드 양보
```

**4. 동기화 프리미티브 수정**

- **Semaphore** ([synch.c:103-119](pintos/threads/synch.c#L103-L119))
  - `sema_up()`: `list_min()`으로 가장 높은 우선순위 대기자 깨움

- **Condition Variable** ([synch.c:367-399](pintos/threads/synch.c#L367-L399))
  - `cond_signal()`: semaphore 대기자들을 순회하여 최고 우선순위 스레드 깨움

### 설계 포인트

1. **중첩 기부 처리**: 최대 8단계 깊이의 중첩 락 체인 지원
2. **효율적인 복구**: acquired_locks 리스트로 O(n) 시간에 우선순위 복구
3. **선점 최적화**: 불필요한 context switch를 줄이기 위해 실제로 필요한 경우에만 선점
4. **인터럽트 컨텍스트 처리**: `intr_yield_on_return()`으로 인터럽트 핸들러 내 양보 지연

### 관련 파일
- [pintos/threads/thread.c](pintos/threads/thread.c)
- [pintos/include/threads/thread.h](pintos/include/threads/thread.h)
- [pintos/threads/synch.c](pintos/threads/synch.c)
- [pintos/include/threads/synch.h](pintos/include/threads/synch.h)

---

## 3. Advanced Scheduler (MLFQS)

### 개요
Multi-Level Feedback Queue Scheduler를 구현하여 공정한 CPU 시간 분배와 동적 우선순위 조정을 구현했습니다. BSD Unix의 스케줄러를 모델로 한 MLFQS는 nice 값, recent_cpu, load_average를 기반으로 우선순위를 동적으로 계산합니다.

### 주요 구현 내용

#### 데이터 구조

**Fixed-Point 연산 ([fixed-point.h](pintos/include/threads/fixed-point.h))**
- **17.14 고정소수점 형식**: 17비트 정수부 + 14비트 소수부
- `F = 1 << 14 = 16384` (1.0을 표현)
- 주요 매크로:
  - `INT_TO_FP(n)`: 정수를 고정소수점으로 변환
  - `FP_TO_INT_ROUND(x)`: 고정소수점을 반올림하여 정수로 변환
  - `MULT_FP(x, y)`: 고정소수점 곱셈
  - `DIV_FP(x, y)`: 고정소수점 나눗셈
  - 사전 계산된 상수: `FP_59_60`, `FP_1_60`

**스레드별 데이터 ([thread.h:109-110](pintos/include/threads/thread.h#L109-L110))**
```c
int nice;           // CPU 양보 의지 (-20 ~ 20)
fixed_t recent_cpu; // 최근 CPU 사용량 (고정소수점)
```

**전역 데이터 ([thread.c:37-39](pintos/threads/thread.c#L37-L39), [59](pintos/threads/thread.c#L59))**
```c
struct list mlfqs_ready_queues[64];  // PRI_MIN ~ PRI_MAX의 64개 우선순위 큐
int ready_threads_count;              // 준비 상태 스레드 수
fixed_t load_avg;                     // 시스템 부하 평균
```

#### 알고리즘

**1. 우선순위 계산 ([thread.c:424-443](pintos/threads/thread.c#L424-L443))**
```c
priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)
결과를 [PRI_MIN, PRI_MAX] 범위로 제한
```
- 매 4 ticks마다 실행 중인 스레드의 우선순위 갱신 ([timer.c:147-153](pintos/devices/timer.c#L147-L153))
- 매 초마다 모든 스레드의 우선순위 갱신 ([timer.c:155-159](pintos/devices/timer.c#L155-L159))

**2. Recent CPU 계산 ([thread.c:503-506](pintos/threads/thread.c#L503-L506))**
```c
recent_cpu = (2 * load_avg) / (2 * load_avg + 1) * recent_cpu + nice
```
- 매 tick마다 실행 중인 스레드의 recent_cpu 1 증가 ([timer.c:143](pintos/devices/timer.c#L143))
- 매 초마다 모든 스레드의 recent_cpu 재계산 ([timer.c:157](pintos/devices/timer.c#L157))
- Idle 스레드는 제외 ([thread.c:498](pintos/threads/thread.c#L498))

**3. Load Average 계산 ([thread.c:483-488](pintos/threads/thread.c#L483-L488))**
```c
load_avg = (59/60) * load_avg + (1/60) * ready_threads
```
- 매 초마다 갱신 (TIMER_FREQ ticks)
- ready_threads: 현재 실행 중인 스레드 포함 (idle 제외)

**4. 스레드 스케줄링**
- **64개 우선순위 큐 사용**: PRI_MIN부터 PRI_MAX까지 각 우선순위별 큐
- **`next_thread_to_run()`** ([thread.c:586-599](pintos/threads/thread.c#L586-L599))
  - PRI_MAX부터 아래로 스캔하여 첫 번째 비어있지 않은 큐 찾기
  - `max_priority_mlfqs_queue()` 헬퍼 함수 사용 ([thread.c:786-793](pintos/threads/thread.c#L786-L793))
- **동일 우선순위 내에서는 Round-Robin (FIFO)**

#### Timer Interrupt 통합 ([timer.c:139-160](pintos/devices/timer.c#L139-L160))

```c
매 tick마다:
  if (thread_mlfqs 활성화):
    if 현재 스레드가 idle이 아님:
      recent_cpu++

    if ticks % 4 == 0:
      현재 스레드 우선순위 갱신
      필요시 양보

    if ticks % TIMER_FREQ == 0:  // 매 초
      load_avg 갱신
      모든 스레드의 recent_cpu 갱신
      모든 스레드의 우선순위 갱신
```

#### 초기화
- **메인 스레드**: nice = 0, recent_cpu = 0 ([thread.c:575-576](pintos/threads/thread.c#L575-L576))
- **자식 스레드**: 부모의 nice와 recent_cpu 상속 ([thread.c:216-221](pintos/threads/thread.c#L216-L221))
- **Load average**: 0.0으로 시작 ([thread.c:125](pintos/threads/thread.c#L125))

#### 모드 전환
- `thread_mlfqs` 전역 boolean으로 제어
- 커널 명령줄 옵션 "-o mlfqs"로 활성화
- MLFQS 활성화 시:
  - Priority Donation 비활성화
  - `thread_set_priority()` 무시 ([thread.c:374](pintos/threads/thread.c#L374))
  - 단일 우선순위 리스트 대신 64개 큐 사용

### 설계 포인트

1. **정밀도**: 17.14 고정소수점으로 충분한 정밀도 제공
2. **효율성**: 사전 계산된 상수(59/60, 1/60)로 연산 최적화
3. **공정성**: Recent CPU와 nice 값으로 공정한 CPU 시간 분배
4. **동적 조정**: 시스템 부하에 따라 우선순위 자동 조정
5. **모듈성**: Priority Scheduling과 MLFQS가 boolean 플래그로 깔끔하게 분리

### 관련 파일
- [pintos/threads/thread.c](pintos/threads/thread.c)
- [pintos/include/threads/thread.h](pintos/include/threads/thread.h)
- [pintos/include/threads/fixed-point.h](pintos/include/threads/fixed-point.h)
- [pintos/devices/timer.c](pintos/devices/timer.c)

---

## 빌드 및 테스트

### 빌드
```bash
cd pintos/threads
make
```

### 전체 테스트 실행
```bash
make check
```

### 개별 테스트 실행

**Alarm Clock 테스트:**
```bash
make tests/threads/alarm-single.result
make tests/threads/alarm-multiple.result
make tests/threads/alarm-simultaneous.result
make tests/threads/alarm-priority.result
```

**Priority Scheduling 테스트:**
```bash
make tests/threads/priority-change.result
make tests/threads/priority-preempt.result
make tests/threads/priority-donate-one.result
make tests/threads/priority-donate-multiple.result
make tests/threads/priority-donate-nest.result
make tests/threads/priority-donate-chain.result
make tests/threads/priority-sema.result
make tests/threads/priority-condvar.result
```

**MLFQS 테스트:**
```bash
make tests/threads/mlfqs-load-1.result
make tests/threads/mlfqs-load-60.result
make tests/threads/mlfqs-recent-1.result
make tests/threads/mlfqs-fair-2.result
make tests/threads/mlfqs-fair-20.result
make tests/threads/mlfqs-nice-2.result
make tests/threads/mlfqs-nice-10.result
make tests/threads/mlfqs-block.result
```

### 테스트 결과 확인
```bash
# 전체 결과 요약
make grade

# 특정 테스트 상세 결과
cat tests/threads/[test-name].output
```

---

## 프로젝트 구조

```
pintos/
├── threads/
│   ├── thread.c          # 스레드 관리 및 스케줄링 핵심 로직
│   ├── thread.h          # 스레드 구조체 및 함수 선언
│   ├── synch.c           # 동기화 프리미티브 (lock, semaphore, condition variable)
│   └── synch.h           # 동기화 관련 구조체 및 함수 선언
├── devices/
│   ├── timer.c           # Timer interrupt 및 sleep 처리
│   └── timer.h           # Timer 관련 함수 선언
└── include/
    └── threads/
        ├── fixed-point.h # 고정소수점 연산 매크로 (MLFQS용)
        ├── thread.h      # 스레드 구조체 정의
        └── synch.h       # 동기화 구조체 정의
```

---

## 구현 하이라이트

### 동기화 메커니즘
1. **인터럽트 비활성화**: 모든 임계 영역에서 `intr_disable()` / `intr_set_level()` 사용
2. **정렬된 리스트**: `list_insert_ordered()`로 우선순위 순서 유지
3. **컨텍스트 인식 양보**: `intr_context()` 체크 및 `intr_yield_on_return()` 사용
4. **TID 할당 보호**: `tid_lock`으로 스레드 ID 카운터 보호

### 핵심 설계 원칙
1. **효율성**: Alarm Clock의 O(1) wake-up, 정렬된 리스트 활용
2. **정확성**: 인터럽트 처리로 race condition 방지
3. **모듈성**: MLFQS와 Priority Scheduling이 boolean 플래그로 분리
4. **완전성**: 최대 8단계 중첩 donation 처리
5. **정밀도**: 17.14 고정소수점으로 MLFQS 계산의 충분한 정밀도 제공

---

## 팀 정보

**KAIST PintOS Project 1 - Team 05**


본 프로젝트는 기존 PintOS의 Round-Robin 스케줄러를 개선하여 효율적인 Alarm Clock, 우선순위 기반 선점형 스케줄링, 그리고 공정한 CPU 시간 분배를 위한 MLFQS를 구현하였습니다.
