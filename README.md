# ttaeyong.github.io

---
layout: default
title: 김태용 | System Programmer Portfolio
---

# 👨‍💻 김태용 (Tae Yong Kim)
**System Software & Embedded Engineer**

[![Email](https://img.shields.io/badge/Email-Contact_Me-blue?style=flat-square&logo=gmail)](mailto:ktyong1225@inha.edu)
[![GitHub](https://img.shields.io/badge/GitHub-Profile-black?style=flat-square&logo=github)](https://github.com/ttaeyong)

> 하드웨어 아키텍처에 대한 깊은 이해를 바탕으로, 제한된 자원 환경에서 최적의 성능을 끌어내는 시스템 프로그래머입니다. 메모리 계층 구조를 고려한 연산 최적화부터 RTOS 기반의 태스크 스케줄링 및 IPC 동기화 설계까지, 하드웨어와 소프트웨어의 경계에서 발생하는 병목을 규명하고 해결하는 데 강점이 있습니다. **Baekjoon Gold 1** 티어의 알고리즘 역량을 바탕으로 탄탄하고 논리적인 코드를 작성합니다.

---

## 🚀 Main Project: 하드웨어 아키텍처 기반 GEMM 최적화 (성능 15배 향상)
제한된 컴퓨팅 자원에서 하드웨어의 한계(Memory/Compute Bound)를 파악하고, 이를 극복하는 시스템 소프트웨어 최적화 역량을 기르기 위해 진행한 프로젝트입니다.

### Phase 1. VTune을 활용한 하드웨어 병목 원인 규명
단순한 실행 시간 측정이 아닌, **Intel VTune Profiler**를 활용해 CPU 파이프라인 레벨의 병목 원인을 딥다이브(Deep-dive) 분석했습니다.
* **멀티스레딩 메모리 경합:** 16 스레드 연산 시 메모리 대역폭 점유율이 99.1%에 달하는 현상을 확인하고, 메모리 접근 최적화의 필요성 입증.
* **레지스터 스필링(Register Spilling) 발견:** Loop Unrolling 적용 시 특정 명령어 라인에서 L1 Bound가 비정상적으로 급증하는 현상을 포착, x86-64 아키텍처의 레지스터 개수 한계로 인한 Spilling 임을 추론하여 Unroll 범위를 최적화.
📄 **[Phase 1. VTune 병목 분석 보고서 보기 (PDF)](/assets/pdf/Profiling_VTune_Examples.pdf)**

### Phase 2. SIMD 및 복합 최적화 적용을 통한 한계 돌파
Phase 1의 분석을 바탕으로, L1/L2 캐시 라인 크기(48KB/512KB)에 맞춘 메모리 배치와 하드웨어 가속 명령어를 적용했습니다.
* **AVX-512 Intrinsic 구현:** 512비트 벡터 레지스터를 활용하여 한 번에 16개의 정수 연산을 병렬 처리(`_mm512_add_epi32` 등 활용).
* **복합 최적화(Combined Optimization):** Cache 지역성을 극대화하는 Blocking(Size: 64)과 Branch Prediction 부하를 줄이는 Loop Unrolling, 그리고 SIMD를 결합.
* **최종 결과:** Naive 구현 대비 **실행 사이클 15배 단축** 달성 및 파라미터별 성능 개선율 시각화 도출.
📄 **[Phase 2. SIMD 구현 및 성능 시각화 보고서 보기 (PDF)](/assets/pdf/Profiling_Matrix_Multiplication.pdf)**

---

## 🛠 Low-level System & Embedded Projects

### 1. NAND Flash Translation Layer (FTL) 에뮬레이터 설계
**[Tech]** C (User-level)  |  **[Keywords]** Page-level Mapping, Out-of-place Update, Garbage Collection
* NAND 플래시의 물리적 한계인 Erase-before-write 특성을 극복하기 위한 FTL 소프트웨어 모델링.
* 가용 공간 고갈 시 발생하는 쓰기 지연 문제를 해결하기 위해, Valid Page 수가 가장 적은 희생 블록(Victim Block)을 탐색하고 데이터를 마이그레이션하는 **Greedy 알고리즘 기반의 Garbage Collection** 직접 구현.

<details>
<summary><b>💡 핵심 로직 보기 (Garbage Collection 구현체)</b></summary>
<div markdown="1">

```c
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
