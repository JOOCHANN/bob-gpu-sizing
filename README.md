# Bob 온프렘 GPU 산정기

IBM Bob을 고객 환경에 자체 호스팅(온프렘)할 때 필요한 GPU 수와, 그 GPU로 쓸 수 있는 개발자 수를 추정하는 IBM 내부 참고용 페이지입니다.

주소: https://bob-gpu-sizing.joochan.workers.dev/

> IBM 공식 견적이나 성능 보증이 아닙니다. 고객 제안에 숫자를 쓰기 전에는 고객 GPU에서 Bob 실제 작업으로 실측해 확인해야 합니다.

## 화면 구성

| 화면 | 내용 |
|---|---|
| GPU 산정 | 모델과 개발자 수를 기준으로 GPU 종류별 필요 장 수 |
| 가진 GPU로 찾기 | 보유 GPU와 장 수를 기준으로 쓸 수 있는 모델과 인원 |
| 모델 성능 | 코딩·에이전트 지수와 주요 벤치마크, 성능 대비 필요 GPU 수 |
| 비교함 | 여러 구성을 같은 조건으로 나란히 비교 |
| 용어와 근거 | 용어 설명, 계산 방식, 모델·GPU 값별 출처 |

표의 행을 누르면 상세 화면으로 이동합니다. 상세 화면에는 계산 과정, 단계별 계산식, 근거 링크가 있습니다. 모델 정보에는 압축 방식별 Hugging Face 저장소 링크가 있습니다.

## 사용 방법

1. 개발자 수를 입력합니다. (기본 200명)
2. 모델을 고릅니다. 목록은 IBM 공식 지원, 공식 모델 계열, 고객 직접 연결(BYOM)로 나뉘고, 모델명 앞 색 태그는 모델 제조사입니다.
3. GPU별 필요 장 수를 확인합니다. 가정은 '계산 가정 바꾸기'에서 바꿀 수 있습니다.

바꾼 설정은 각자 브라우저에만 저장됩니다.

## 계산 방식

vLLM으로 서빙하는 경우를 기준으로 계산합니다.

### 필요한 GPU 수 (모델만 올리기)

- 모델 파일을 GPU 메모리에 올리고 큰 작업(128K 토큰) 1건이 들어가는 가장 작은 GPU 묶음입니다.
- vLLM 레시피의 최소 VRAM(가중치 × 1.2)보다 적게 잡지 않습니다.
- 레시피와 대조한 결과, 모델·압축 방식·GPU 683개 조합 중 645개는 레시피 기준과 같고 38개는 더 보수적이며, 더 적게 나온 경우는 없습니다.

### 쓸 수 있는 개발자 수

```
개발자 수 = 이론 동시 세션 × 실사용 보정(25%) ÷ 같은 순간 쓰는 비율(25%)
```

1. **이론 동시 세션:** 아래 두 한도 중 먼저 닿는 쪽입니다.
   - 메모리: 모델을 싣고 남은 GPU 메모리(KV 캐시)에 작업이 몇 건 들어가는지
   - 속도: 1명당 초당 20토큰 이상을 유지하면서 몇 명까지 받을 수 있는지. 속도 계산식은 vLLM 레시피 실측 10건과 맞췄고, 예측은 실측의 0.82~1.11배입니다.
2. **실사용 보정 25%:** imec가 실제 코딩 에이전트 작업으로 측정했을 때, 무리 없이 쓸 수 있던 동시 세션은 이론값의 27~44%였습니다. 여기서는 모든 사례보다 낮은 25%만 인정합니다.
3. **같은 순간 쓰는 비율 25%:** 전체 개발자 중 같은 순간 Bob의 답을 받고 있는 사람의 비율입니다. 하루 평균이 아니라 사용이 몰리는 시간을 기준으로 합니다.

기본 가정에서는 두 비율이 같아서 이론 동시 세션 1개당 개발자 1명으로 계산됩니다.

예시: Laguna S 2.1 (FP8), H200 4장
이론 동시 세션 214명 → 실사용 보정 25% → 53명 → 같은 순간 비율 25%로 나눔 → 개발자 약 212명

### 기본 가정

| 항목 | 기본값 | 바꿀 수 있는 값 |
|---|---|---|
| 작업 규모 (Bob이 한 번에 읽는 코드·대화 양) | 64K 토큰 | 32K, 128K, 최대 270K |
| 1명당 응답 속도 최저선 | 초당 20토큰 | 15, 30 |
| 같은 순간 쓰는 비율 | 25% | 10~50% |
| 실사용 보정 | 25% | 32%(실측 평균), 100%(이론값) |
| KV 캐시 정밀도 | FP8 | BF16 |
| 어텐션 데이터 병렬 | 끔 | 켬 (결과가 달라지는 모델에서만 표시) |
| GPU 메모리 | 공칭 GB(10진) 기준, 90% 사용 | |

### Bob 관련 한도

- Bob은 작업 1건에 최대 270K 토큰까지 쓰고, 그중 약 20K는 다음 답변용으로 남겨 둡니다. 기본 설정에서는 90%쯤 차면 앞의 대화를 요약해 줄입니다.
- 작업 규모를 '최대 270K'로 두면 모든 사용자가 한도까지 찬 작업을 하는 경우로 계산합니다. 모델의 서빙 한도가 더 짧으면 그 한도를 씁니다.
- 서빙 한도가 Bob 최대보다 짧은 모델(gpt-oss 2종, MiniMax M3, MiniMax M2.7, GLM-4.7)은 화면에 따로 표시됩니다.
- 에어갭 설치는 안전 검사용 가드레일 모델(gpt-oss-20b)을 함께 두도록 안내하고 있습니다. 산정 표에는 들어 있지 않으므로 GPU 1장 정도를 별도로 잡아야 합니다.

## 데이터 (2026-11-10 기준)

**모델 28종**

| 구분 | 모델 |
|---|---|
| IBM 공식 지원 (2) | Nemotron 3 Ultra, Laguna S 2.1 |
| 공식 모델 계열 (5) | Nemotron 3 Super, Nemotron 3 Nano, Nemotron 3.5 Lightning, Laguna M.1, Laguna XS 2.1 |
| 고객 직접 연결, BYOM (21) | Kimi K2.7 Code, Kimi K2.6, Kimi K3, GLM-5.3, GLM-5.2, GLM-5.3 Flash, GLM-4.7, Qwen3.5 397B, Qwen3.5 122B, Qwen3.6 35B, Qwen3.8 27B, Qwen3.8 Flash-Next, Qwen3.8 2.4T, Qwen3-Coder 480B, DeepSeek V4-Pro, DeepSeek V4-Flash, MiniMax M2.7, MiniMax M3, gpt-oss-120b, gpt-oss-20b, Mistral Small 4 |

Bob 문서의 에어갭 지원 목록에는 Nemotron 3.5 Super와 Mistral Medium 3.5(IBM Z용 Premium Package 전용)도 있지만, 이 페이지에는 아직 들어 있지 않습니다.

**GPU 10종:** A100 80GB, H100 80GB, H200 141GB, B200 180GB, B300 270GB, L40S 48GB, RTX PRO 6000 Blackwell 96GB, MI300X 192GB, MI325X 256GB, MI355X 288GB

**성능 지표**

| 지표 | 출처 |
|---|---|
| 코딩 지수, 에이전트 지수 | Artificial Analysis (OpenRouter Models API 제공) |
| τ²-Bench Airline | OpenRouter |
| Terminal-Bench 2.1 | Vals AI |
| SWE-bench Verified, SWE-bench Pro | BenchLM 모음 (제조사 발표 위주) |

'제조사'로 표시된 점수는 모델 제조사가 발표한 값이라 독립 측정과 바로 비교하기 어렵습니다.

## 한계

- 사용자 수는 작업 규모와 같은 순간 쓰는 비율에 가장 크게 좌우됩니다. 고객의 저장소 크기와 사용 시간대를 먼저 확인하는 것이 좋습니다.
- 공식 지원 모델은 아직 NVIDIA GPU 실측값이 없습니다. 실측이 나오면 보정값을 다시 맞출 예정입니다.
- AMD(ROCm) GPU는 실측 메모리 여유가 계산보다 작게 나온 사례가 있어 더 낮춰 보는 편이 안전합니다.
- 어텐션 데이터 병렬을 켠 결과는 실사용 보정(25%)을 맞출 때의 조건과 달라 실제보다 많게 나올 수 있습니다.
- 추론 가속(MTP, EAGLE 등)은 반영하지 않았습니다.
- GPU 단가는 들어 있지 않습니다. 장 수가 같아도 GPU 종류에 따라 비용 차이가 큽니다.

## 주요 출처

- vLLM recipes 저장소: https://github.com/vllm-project/recipes
- vLLM recipes 최소 VRAM 공식: https://github.com/vllm-project/recipes/blob/main/SCHEMA.md
- imec, 에이전트 코딩 동시 세션 실측: https://aistack.imec-int.com/blog/gpu-self-hosting
- IBM Bob 문서, 컨텍스트 한도: https://bob.ibm.com/docs/ide/core-concepts/context-window-management
- IBM Bob 문서, 지원 모델: https://bob.ibm.com/docs/ide/enterprise/on-premises/model-gateway/supported-models
- OpenRouter Models API: https://openrouter.ai/api/v1/models
- Vals AI Terminal-Bench 2.1: https://vals.ai/benchmarks/terminal-bench-2-1
- BenchLM SWE-bench Verified: https://benchlm.ai/benchmarks/swe-bench-verified

모델별·GPU별 값의 출처는 페이지의 '용어와 근거' 화면에 있습니다.

## 파일 구성과 업데이트

| 파일 | 내용 |
|---|---|
| `index.html` | 페이지 전체. 데이터, 계산식, 화면이 이 파일 하나에 들어 있습니다. |
| `README.md` | 이 문서 |

페이지는 Cloudflare에 올려 서비스합니다. 수정할 때는 Cloudflare 대시보드의 이 프로젝트에 새 `index.html`을 올려 배포합니다.
