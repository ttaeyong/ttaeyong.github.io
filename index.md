---
layout: default
title: 김태용 | System Programmer Portfolio
---

김태용 (Tae Yong Kim)

System Software & Embedded Engineer

Email: 본인이메일@gmail.com | GitHub: github.com/본인아이디

하드웨어 아키텍처에 대한 깊은 이해를 바탕으로, 제한된 자원 환경에서 최적의 성능을 끌어내는 시스템 프로그래머입니다. 메모리 계층 구조를 고려한 연산 최적화부터 RTOS 기반의 태스크 스케줄링 및 IPC 동기화 설계까지, 하드웨어와 소프트웨어의 경계에서 발생하는 병목을 규명하고 해결하는 데 강점이 있습니다. Baekjoon Gold 1 수준의 알고리즘 역량을 바탕으로 탄탄하고 논리적인 코드를 작성합니다.

Main Project: 하드웨어 아키텍처 기반 GEMM 최적화 (성능 15배 향상)

제한된 컴퓨팅 자원에서 하드웨어의 한계(Memory/Compute Bound)를 파악하고, 이를 극복하는 시스템 소프트웨어 최적화 역량을 기르기 위해 진행한 프로젝트입니다.

Phase 1. VTune을 활용한 하드웨어 병목 원인 규명

단순한 실행 시간 측정이 아닌, Intel VTune Profiler를 활용해 CPU 파이프라인 레벨의 병목 원인을 심층 분석했습니다.

멀티스레딩 메모리 경합: 16 스레드 연산 시 메모리 대역폭 점유율이 99.1%에 달하는 현상을 확인하고, 메모리 접근 최적화의 필요성 입증.

레지스터 스필링(Register Spilling) 발견: Loop Unrolling 적용 시 특정 명령어 라인에서 L1 Bound가 비정상적으로 급증하는 현상을 포착, x86-64 아키텍처의 레지스터 개수 한계로 인한 Spilling 임을 추론하여 Unroll 범위를 최적화.

Phase 1. VTune 병목 분석 보고서 확인 (PDF)

Phase 2. SIMD 및 복합 최적화 적용을 통한 한계 돌파

Phase 1의 분석을 바탕으로, L1/L2 캐시 라인 크기(48KB/512KB)에 맞춘 메모리 배치와 하드웨어 가속 명령어를 적용했습니다.

AVX-512 Intrinsic 구현: 512비트 벡터 레지스터를 활용하여 한 번에 16개의 정수 연산을 병렬 처리(_mm512_add_epi32 등 활용).

복합 최적화(Combined Optimization): Cache 지역성을 극대화하는 Blocking(Size: 64)과 Branch Prediction 부하를 줄이는 Loop Unrolling, 그리고 SIMD를 결합.

최종 결과: Naive 구현 대비 실행 사이클 15배 단축 달성 및 파라미터별 성능 개선율 시각화 도출.

Phase 2. SIMD 구현 및 성능 시각화 보고서 확인 (PDF)

Low-level System & Embedded Projects

1. NAND Flash Translation Layer (FTL) 에뮬레이터 설계

Tech: C (User-level)

Keywords: Page-level Mapping, Out-of-place Update, Garbage Collection

Summary: NAND 플래시의 물리적 한계인 Erase-before-write 특성을 극복하기 위한 FTL 소프트웨어 모델링. 가용 공간 고갈 시 발생하는 쓰기 지연 문제를 해결하기 위해, Valid Page 수가 가장 적은 희생 블록(Victim Block)을 탐색하고 데이터를 마이그레이션하는 Greedy 알고리즘 기반의 Garbage Collection 구현.

<details>
<summary><b>주요 구현 코드 확인 (Garbage Collection 구현체)</b></summary>
<div markdown="1">

// Valid Page 수가 가장 적은 Block을 희생 블록으로 선정하여 마이그레이션 수행
void do_garbage_collection() {
    int victim_blk = -1;
    int min_valid = PAGES_PER_BLOCK + 1;
    int active_blk = g_current_pba_index / PAGES_PER_BLOCK;

    // 1. Greedy 정책: 유효 페이지가 가장 적은 희생 블록 탐색
    for (int i = 0; i < BLOCK_COUNT; i++) {
        if (i == active_blk) continue; // 현재 쓰기 중인 블록 제외
        if (blockMetaTable[i].valid_count < min_valid) {
            min_valid = blockMetaTable[i].valid_count;
            victim_blk = i;
        }
    }
    if (victim_blk == -1) return;

    // 2. 유효 페이지(Valid Page) 마이그레이션 (새로운 PBA로 복사 및 맵핑 업데이트)
    int startPba = victim_blk * PAGES_PER_BLOCK;
    for (int i = 0; i < PAGES_PER_BLOCK; i++) {
        int old_pba = startPba + i;
        int lba = PtoL[old_pba];

        if (lba != -1 && LtoP[lba] == old_pba) { 
            uint32_t data = nand_raw_read(old_pba);
            ftl_internal_write(lba, data); 
        }
    }
    
    // 3. 희생 블록 초기화 (Erase)
    ssdEraseBlock(victim_blk);
}


</div>
</details>

2. RTOS 기반 스마트 가변 온도 경보 시스템 (ATmega128)

Tech: C, ATmega128, uC/OS-II

Keywords: RTOS, Task Scheduling, IPC (Message Queue)

Summary: uC/OS-II를 포팅하여 5개의 멀티 태스크가 동작하는 다중 센서 융합(Sensor Fusion) 제어 시스템 개발. I2C 센서 데이터 수집 태스크와 FND 디스플레이 출력 태스크 간의 자원 경합 현상을 인지하고, 태스크 간 직접 참조를 배제한 Mailbox 및 Message Queue 기반의 비동기 IPC 통신 아키텍처를 설계하여 실시간성 보장.

<details>
<summary><b>주요 구현 코드 확인 (Message Queue 기반 비동기 디스플레이 제어)</b></summary>
<div markdown="1">

// 센서 데이터 갱신 대기 중에도 디스플레이 잔상을 유지하기 위해 Accept/Pend 혼합 사용
void FndDisplayTask (void *data) { 
    while (1) {
        for (int i = 0; i < 4; i++) {
            // Non-blocking으로 큐에 새로운 온도 데이터가 있는지 확인
            display = OSQAccept(MsgQueue); 
            if(display != (void*)0){
                cur_number[0] = *(UCHAR*)display;
                cur_number[1] = *(UCHAR*)OSQPend(MsgQueue, 0, &err); // 이후 데이터는 Blocking 대기
                cur_number[2] = *(UCHAR*)OSQPend(MsgQueue, 0, &err);
                cur_number[3] = *(UCHAR*)OSQPend(MsgQueue, 0, &err);
            }
            
            // 데이터 수신 여부와 관계없이 FND 멀티플렉싱(잔상 효과) 지속 유지
            PORTC = FND_DATA[cur_number[i]];
            PORTG = fnd_sel[i];
            if (i == 1) PORTC |= 0x80;
            OSTimeDlyHMSM(0, 0, 0, 1);
        }
    }
}


</div>
</details>

3. TC275 기반 차량 주행 속도 제어 및 모니터링 시스템

Tech: C, AURIX TC275

Keywords: Bare-metal, Non-preemptive Scheduler, STM Interrupt

Summary: OS가 없는 환경(Bare-metal)에서 AURIX 보드의 System Timer(STM) 인터럽트를 활용한 Time-triggered 비선점형 스케줄러 설계. 인터럽트 루틴(ISR) 내의 연산 부하를 최소화하고, 1ms/10ms/100ms 주기의 플래그 기반 태스크 분산 처리 구현.

<details>
<summary><b>주요 구현 코드 확인 (하드웨어 타이머 기반 스케줄러)</b></summary>
<div markdown="1">

// 인터럽트 오버헤드를 줄이기 위해 주기별 실행 플래그만 설정
IFX_INTERRUPT(STM_Int0Handler, 0, 100);
void STM_Int0Handler(void) {
    IfxCpu_enableInterrupts();
    IfxStm_clearCompareFlag(g_Stm.stmSfr, g_Stm.stmConfig.comparator);
    IfxStm_increaseCompare(g_Stm.stmSfr, g_Stm.stmConfig.comparator, STM0_TICK);

    u32nuCounter1ms++;
    if ((u32nuCounter1ms % 1) == 0u)   stSchedulingInfo.u8nuScheduling1msFlag = 1u;
    if ((u32nuCounter1ms % 10) == 0u)  stSchedulingInfo.u8nuScheduling10msFlag = 1u;
    if ((u32nuCounter1ms % 100) == 0u) stSchedulingInfo.u8nuScheduling100msFlag = 1u;
}

// 메인 루프에서 플래그를 확인하여 주기에 맞는 태스크 분산 처리
void AppScheduling(void) {
    if(stSchedulingInfo.u8nuScheduling1msFlag == 1u) {
        stSchedulingInfo.u8nuScheduling1msFlag = 0u;
        AppTask1ms(); // 센서 데이터 기반 실시간 디스플레이 제어
    }
    if(stSchedulingInfo.u8nuScheduling100msFlag == 1u) {
        stSchedulingInfo.u8nuScheduling100msFlag = 0u;
        AppTask100ms(); // 속도 및 조도 체크, 시스템 상태 모니터링
    }
}


</div>
</details>

Education & Activities

현대오토에버 모빌리티 SW 스쿨 (임베디드 트랙) 수료

모빌리티 특화 임베디드 SW 아키텍처 및 시스템 제어 실무 역량 강화

Memory Systems Lab (학부 연구생)

x86-64 아키텍처 기반의 Cache Hierarchy 분석 및 GEMM 최적화 연구 진행

프로그래밍 TA 및 튜터

C++, Python, Scratch 기초 문법 및 알고리즘 튜터링 진행

대한민국 육군 (KATUSA) 병장 만기 전역

미군 부대 파견 근무를 통한 다문화 환경 협업 및 커뮤니케이션 역량 확보
