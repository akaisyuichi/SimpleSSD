# 최근 갱신 이력을 이용한 멀티채널 FAST의 RW 로그 victim 선택

> Nexus5 원본 트레이스 비교 보고서. 소은수님 보고서의 형식과 실험 조건을 참고했으며, 로컬 FIFO FAST 기준선에서 독립 구현한 정책이다. 네 FIFO 결과를 제공된 표와 대조했다. 처리 시간·full merge·복사량·WAF·p99는 표시 자릿수까지 일치했다. copyHtoD 평균 지연은 이번 계산에서 26.67 ms, 제공 표는 26.66 ms로 차이가 있어 아래에 원자료 계산값을 유지했다.

## 아이디어 3줄 요약

**문제:** FIFO FAST는 가장 오래된 RW 로그를 회수하므로 곧 덮어써질 데이터까지 full merge로 복사할 수 있다. victim에 유효 sector가 남아 있는 LBN마다 full merge가 필요하다.

**아이디어:** full merge 대상 LBN 수가 비슷한 로그 중 최근 갱신이 덜 활발한 로그부터 회수한다. 다른 로그를 회수하며 쓰기를 계속하는 동안, 미룬 로그의 특정 LBN이 마지막 유효 sector까지 덮어써져 merge 대상에서 빠지는 효과를 기대한다.

**결과:** download와 Installing의 처리 시간이 각각 6.9%·12.8%, full merge가 8.9%·20.3% 줄었다. cameraVideo는 full merge가 1.2% 늘었고, copyHtoD는 처리 시간이 1.5% 늘어 악화됐다. 통제 실험에서는 미룬 로그의 LBN 소멸로 full merge가 2회에서 1회로 줄어드는 경로를 확인했다.

## 지표

FAST 통계 이름 앞에는 실제 로그에서 ftl. 접두사가 붙는다.

| 구분 | 지표 | 통계 이름 / 계산 방법 | 뜻 | 측정 여부 |
| --- | --- | --- | --- | --- |
| 기존 | GC 및 merge 횟수 | fast.rw_victims, fast.merge.switch/partial/full | 회수한 RW 로그 수와 종류별 merge 횟수. full은 LBN 단위 | 측정함 |
| 기존 | GC 이동 sector | fast.merge.sector_copies | merge에서 복사한 I/O unit 수. 이 Nexus5 구성은 sector당 32 KiB이며 512 B LBA와 다름 | 측정함 |
| 기존 | WAF | fast.nand_write_pages / fast.host.write_pages | 호스트 쓰기 대비 NAND 기록 sector 수 | 측정함 |
| 기존 | 처리 시간 | Trace Replayer 마지막 Tick / 10^12 | 전체 I/O의 시뮬레이션 시간(s). PC 실행 시간과 다름 | 측정함 |
| 기존 | 쓰기 평균 지연 | latency.txt의 쓰기 요청 지연 평균 | 요청 ID를 입력 trace와 대응시켜 읽기를 제외 | 측정함 |
| 기존 | 쓰기 p99 지연 | 정렬 후 ceil(0.99 × 쓰기 요청 수)번째 값 | 쓰기 요청의 꼬리 지연 | 측정함 |
| 추가 | FIFO 외 victim 선택 | fast.policy.bypass | FIFO 맨 앞 대신 다른 로그를 선택한 횟수 | 측정함 |
| 추가 | 강제 선택 | fast.policy.forced | 반복 유예 임계값에 도달한 로그를 선택한 횟수 | 측정함 |
| 추가 | 덮어쓰기에 의한 LBN 소멸 | fast.deferred.lbn_vanished.overwrite | 유예 로그에서 한 LBN의 마지막 sector가 호스트 덮어쓰기로 사라진 사건 수 | 측정함 |
| 추가 | 다른 merge / trim에 의한 소멸 | fast.deferred.lbn_vanished.merge/trim | 덮어쓰기 효과와 구분하는 소멸 원인 | 측정함 |

## 결과 (nexus5.cfg, 채움 0.9, 버퍼 끔, RW 로그 블록 8개)

각 조합 1회씩 총 8회 실행했다. 소은수님 표의 FIFO를 재현한 뒤 동일 바이너리에서 victim 정책 설정 한 줄만 바꿨다. greedy와 소은수님 정책은 이번 실행의 비교 대상에 포함하지 않았다.

| trace | 정책 | 처리 시간(s) | full merge | 복사 sector | WAF | 평균 지연(ms) | p99 지연(ms) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| download | FIFO | 126.9 | 1,219 | 648,648 | 21.79 | 10.98 | 61.4 |
| download | 최근 갱신 | 118.1 | 1,111 | 592,337 | 19.99 | 10.22 | 60.1 |
| cameraVideo | FIFO | 138.9 | 1,114 | 617,762 | 10.49 | 50.13 | 371.2 |
| cameraVideo | 최근 갱신 | 138.4 | 1,127 | 624,407 | 10.59 | 49.94 | 355.3 |
| Installing | FIFO | 199.4 | 1,830 | 976,236 | 16.29 | 11.29 | 230.8 |
| Installing | 최근 갱신 | 173.9 | 1,459 | 786,444 | 13.32 | 9.85 | 229.5 |
| copyHtoD | FIFO | 163.7 | 1,220 | 648,165 | 5.82 | 26.67 | 282.4 |
| copyHtoD | 최근 갱신 | 166.1 | 1,262 | 669,434 | 5.98 | 27.06 | 294.7 |

## 새로운 정책의 개선율 (FIFO 대비, 음수가 개선)

계산식은 (제안 정책 / FIFO - 1) × 100이다. 다른 정책과 비교한 수치를 섞지 않았다.

| trace | 처리 시간 | full merge | 복사 sector | WAF | 평균 지연 | p99 지연 |
| --- | --- | --- | --- | --- | --- | --- |
| download | -6.9% | -8.9% | -8.7% | -8.3% | -6.9% | -2.2% |
| cameraVideo | -0.4% | +1.2% | +1.1% | +1.0% | -0.4% | -4.3% |
| Installing | -12.8% | -20.3% | -19.4% | -18.2% | -12.8% | -0.5% |
| copyHtoD | +1.5% | +3.4% | +3.3% | +2.7% | +1.5% | +4.4% |

**download:** full merge -8.9%, 복사량 -8.7%, 처리 시간 -6.9%, 평균 지연 -6.9%, p99 -2.2%.

**cameraVideo:** full merge +1.2%, 복사량 +1.1%, 처리 시간 -0.4%, 평균 지연 -0.4%, p99 -4.3%.

**Installing:** full merge -20.3%, 복사량 -19.4%, 처리 시간 -12.8%, 평균 지연 -12.8%, p99 -0.5%.

**copyHtoD:** full merge +3.4%, 복사량 +3.3%, 처리 시간 +1.5%, 평균 지연 +1.5%, p99 +4.4%.

평균 지연만으로 결론짓지 않고 p99와 복사량도 함께 제시했다. 특히 cameraVideo에서는 full merge와 복사량이 늘어, 이 정책의 개선 효과가 모든 trace에 공통적이지 않다.

## 핵심 가설 검증

### 유예된 로그에서 LBN의 마지막 유효 sector가 실제로 사라지는가?

아래는 제안 정책에서 관측한 사건 수다. 한 LBN의 마지막 sector가 사라질 때 한 번 세며 lane별로 중복 집계하지 않는다.

| trace | 호스트 덮어쓰기에 의한 LBN 소멸 | 다른 merge에 의한 LBN 소멸 |
| --- | --- | --- |
| download | 87 | 830 |
| cameraVideo | 61 | 665 |
| Installing | 757 | 1498 |
| copyHtoD | 246 | 742 |

모든 trace에서 유예된 로그의 LBN이 호스트 덮어쓰기로 사라지는 현상을 관측했다. 다른 merge로 sector가 옮겨져 사라진 사건은 별도로 표시했다. 같은 로그 안에 다시 쓰여 LBN이 계속 남는 경우와 선택 victim 자신의 merge는 기대 효과로 세지 않는다.

이 수치는 소멸 사건 수이며 절약한 full merge 횟수가 아니다. 같은 LBN이 재유입됐다가 다시 사라지면 별도 사건이 된다. 전체 성능 변화에는 비용 기준과 victim 순서 변경도 함께 작용하므로 최근 갱신 이력의 기여만 분리한 결과는 아니다.

### 통제 실험: full merge 감소 경로

동일한 실제 호스트 쓰기 순서로 RW 로그 3개를 채운 뒤 가장 오래된 로그의 데이터를 다시 썼다. 갱신 이력은 해당 쓰기들로 생성했으며 미래 요청이나 인위적인 이력 주입을 사용하지 않았다.

1. FIFO는 오래된 로그의 LBN을 먼저 full merge한다.
2. 제안 정책은 다른 로그 하나를 즉시 회수한 뒤 호스트 쓰기를 계속한다.
3. 그 쓰기가 미룬 로그의 해당 LBN을 모든 lane에서 덮어써 로그를 비운다.
4. 다음 회수는 빈 로그를 full merge 없이 erase한다.

| lane 수 | FIFO full merge | 제안 정책 full merge | 결과 |
| --- | --- | --- | --- |
| 1 | 2 | 1 | 통과 |
| 2 | 2 | 1 | 통과 |
| 4 | 2 | 1 | 통과 |
| 8 | 2 | 1 | 통과 |

통제 테스트는 PAL/DRAM 없이 FTL 상태와 merge 횟수를 검사한다. 앞의 Nexus5 성능 표는 별도로 NAND 타이밍을 포함한 시뮬레이터에서 측정했다.

## 구현 방법

각 RW 로그의 LBN별 유효 sector 수를 유지한다. 비용은 로그 안의 유효 LBN 개수로 근사한다. 최근 4,096번의 호스트 sector 쓰기 구간에서 LBN별 덮어쓰기 횟수를 센다. 최초 쓰기도 구간을 진행시키지만 갱신 횟수에는 더하지 않는다. GC 복사와 trim은 갱신 이력을 만들지 않는다.

활동도(log) = Σ[유효 sector 수(log, LBN) × 최근 덮어쓰기 횟수(LBN)] / 로그의 전체 유효 sector 수

1. 빈 로그가 있으면 가장 오래된 빈 로그부터 회수한다.
2. 빈 로그가 없고 유예 횟수가 8회 이상인 로그가 있으면 그중 가장 오래된 로그를 강제 선택한다.
3. 그 외에는 최소 비용보다 LBN 수가 최대 1개 많은 후보까지 허용하고 활동도가 가장 낮은 로그를 선택한다.
4. 동률이면 오래된 로그를 선택하고, 남은 LBN을 기존 방식으로 merge해 victim 하나를 즉시 회수한다.

8회는 강제 선택 임계값이다. 빈 로그 우선과 여러 강제 대상의 순서가 있어 정확히 8회 이내 회수를 보장하지는 않는다. LBN 수는 실제 복사량의 근사 비용이며 정확한 NAND 비용은 아니다.

설정: FASTVictimPolicy=1, FASTHistoryWindow=4096, FASTCostSlack=1, FASTMaxDeferrals=8. 기본 FASTVictimPolicy=0은 FIFO다. FAST 활성화는 MappingMode=1, 멀티채널 구성은 SuperblockSize=C로 설정한다.

## 참고

### 재현 조건

| 항목 | 설정 |
| --- | --- |
| 기준 구현 | 로컬 멀티채널 FAST 72a923f. 소은수님 정책을 합치지 않음 |
| 설정 출처 | SimpleSSD fast-lbn-merge 930c76e의 config/nexus5.cfg 및 standalone acc3b6d의 config/sample.cfg |
| NAND 구성 | Channel=2, Package=2, Die=2, Plane=2, Block=512, Page=256, PageSize=16384. 총 물리 64 GiB |
| 슈퍼블록 | SuperblockSize=C, 멀티플레인 ON. FTL 페이지 64 KiB, lane 2개, sector당 32 KiB |
| OP / 초기 상태 | OP 10%, FillingMode=1, FillRatio=0.9, InvalidPageRatio=0.3 |
| warmup | 두 정책 모두 FIFO warmup, 측정 시작 시 갱신 이력을 비움 |
| 캐시 / 버퍼 | 읽기·쓰기 캐시 OFF. 로컬 기준선에 RW 쓰기 버퍼가 없어 버퍼 OFF와 같은 경로 |
| 로그 | 공유 SW 로그 1개, RW 로그 상한 8개 |
| 재생 | Mode=1, TimingMode=0, IOLimit=0. 전체 I/O, 원래 주소·크기 유지 |
| 원본 | log156_download, log161_cameraVideo, log181_Installing, log125_copyHtoD |
| 변환 | 저장소 trace/convert_nexus5.py 사용. 기존 세 파일과 SHA256 일치. cameraVideo는 같은 방식으로 추가 변환 |

### 해석의 한계와 검증 범위

- 각 조합 1회 실행이다. 같은 입력의 결정적 시뮬레이션 결과이며 통계적 유의성 검정은 수행하지 않았다.

- 쓰기 평균·p99는 요청 ID를 원래 trace의 연산·offset·길이와 대응시켜 읽기를 제외했다. 전체 요청 완료와 두 정책의 입력 SHA256·설정·요청 수·호스트 쓰기 sector 수 일치를 검증했다.

- 추가 정책 CPU/DRAM 연산 비용은 기존 고정 지연 모델에 반영하지 않았다. 시뮬레이션 처리 시간 감소를 실제 하드웨어 개선율로 단정할 수 없다.

- 실험 종료 시 남은 로그를 강제로 drain하지 않았다. LBN의 최근 갱신이 모든 sector의 향후 소멸을 보장하지 않아, 일부 sector만 반복 갱신되면 효과가 작을 수 있다.

- 비용 기준만 사용하는 대조군은 추가하지 않았다. 최근 갱신 이력과 비용 제한의 기여도를 따로 정량화하려면 추가 실험이 필요하다.

- 정합성 검사 177,862단계를 통과했다. 부분 lane 쓰기·순차 쓰기·trim·format·최신 위치·유효 sector 수·블록 보존·RW 메타데이터 일치·이력 크기를 검사했다.

- 비용 제한·동률·빈 로그 우선·강제 선택·이력 만료·같은 로그 재쓰기 오집계 방지·소멸 원인 분류·통계 초기화도 통제 테스트로 검증했다.

- 별도 build-recent-update에서 빌드했으며 기존 DRAMPower deprecated 경고만 -Werror 예외로 처리했다.

- 실험 중 기존 sample.cfg·Nexus5 설정·트레이스·기존 결과와 기준선 HEAD를 보존했다. 결과 검증 후 독립 정책을 fast-recent-update 브랜치에 정리했다.

### 코드 및 실험 출처

- 구현: ftl/fast.cc, ftl/fast.hh, ftl/config.cc, ftl/config.hh
- 테스트: test/ftl/test_fast.cc
- 설정 예제: config/recent-update-gc.cfg (Nexus5, 정책 활성화)
- 기준선: 72a923f849ceb6d1a62d49807f89701b7785414c
- standalone 설정 출처: acc3b6d800b5247a247455ebdbcaa1dab9e34e6b
- Nexus5 설정 출처: 930c76e0e605858d08b9dd77bb5160dd947c7f43
- FIFO 표 비교 출처: 사용자가 제공한 소은수님 노션 보고서 화면
- 실험 원자료·재현 스크립트는 standalone 작업 폴더 experiments/recent-update에 보존했다.

### 트레이스 SHA256

- log156_download.txt: 0acf11d85fb558fb63c5d9d136950fc898f7f9970f251ea0f45971a0d48dbcd4
- log161_cameraVideo.txt: 8ca92b59240c45346d676f4bff3830b4bd090b093a2b596bb7386e07132ffe94
- log181_Installing.txt: d11d0643e86b49ba8862d951b6eaeea507f5db2508b824a88d388a6470921d36
- log125_copyHtoD.txt: b91d3ff7d9a3f0202dd1bad5d23f44a49bf813e0da911fb61ec2b8b6a5616683


### 설정 및 실행

config/recent-update-gc.cfg는 실제 Nexus5 비교 실험의 SSD 설정 예제다.
FASTVictimPolicy를 0으로 바꾸면 같은 조건의 FIFO 기준선을 실행할 수 있다.
FASTRWGCPolicy=0은 참고 설정에서 남은 키이며 본 독립 정책의 선택자는
FASTVictimPolicy다. 실행 예제에서는 혼동을 피하기 위해 해당 키를 제외했다.

standalone 시뮬레이터에서 Mode=1, TimingMode=0, IOLimit=0으로 두고
trace/convert_nexus5.py로 변환한 입력을 지정한다. LatencyLogFile을 설정하고
쓰기 요청만 골라 평균·p99를 계산해야 한다.

로컬 Git 작성자 정보와 게시할 본인 저장소가 확인되기 전에는
검증된 코드와 실험 결과를 포함한다.
