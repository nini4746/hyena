# 📊 결과 분석 및 다음 단계 결정 가이드

각 Phase 완료 후, 결과를 보고 다음 단계를 결정해야 합니다.
아래 정보를 복사해서 보내주시면 최적의 설정을 추천해드립니다.

---

## Phase 1 완료 후

### 📋 복사할 정보

```bash
# 1. 테스트 결과 (핵심!)
cat new/models/phase1/logs/test.log

# 특히 이 부분:
# - MAE, RMSE
# - Median, P90, P95, Max
# - Within 1m/2m/3m
# - Outlier 분포 (>3m, >4m, >5m)

# 2. 학습 곡선 마지막 부분
tail -50 new/models/phase1/logs/train.log

# 특히:
# - Best epoch 몇 번?
# - Val RMSE, Val P90 추이
# - Early stopping 됐는지?
```

### 🤔 판단 기준

**결과를 보고 판단할 것**:

| 지표 | 결과 | 판단 |
|------|------|------|
| **RMSE** | < 2.3m | ✅ 좋음, Phase 2 진행 |
| | 2.3-2.6m | ⚠️ 보통, Phase 2 진행 후 재평가 |
| | > 2.6m | ❌ 나쁨, 파라미터 재조정 필요 |
| **P90** | < 1.6m | ✅ 좋음 |
| | 1.6-1.9m | ⚠️ 보통 |
| | > 1.9m | ❌ 나쁨, LR/Depth 조정 필요 |
| **Within 1m** | > 70% | ✅ 좋음 |
| | 60-70% | ⚠️ 보통 |
| | < 60% | ❌ 나쁨, 전처리 재확인 |
| **Max error** | < 12m | ✅ 정상 |
| | 12-20m | ⚠️ Outlier 많음 |
| | > 20m | ❌ 데이터 문제 가능성 |

### 📝 복사 템플릿

```
====================================================================
Phase 1 결과 리포트
====================================================================

--- 테스트 결과 ---
MAE:  [X.XXX]m
RMSE: [X.XXX]m
Median: [X.XXX]m
P90:  [X.XXX]m
P95:  [X.XXX]m
Max:  [XX.XX]m

Within 1m: [XX.X]%
Within 2m: [XX.X]%
Within 3m: [XX.X]%

--- Outlier 분포 ---
> 3m: [XXX]개 (X.X%)
> 4m: [XXX]개 (X.X%)
> 5m: [XXX]개 (X.X%)
> 6m: [XXX]개 (X.X%)

--- 학습 정보 ---
Best epoch: [XX]
Best Val P90: [X.XXX]m
Early stopping: [Yes/No]

====================================================================
```

### 💡 의사결정 예시

**예시 1: 좋은 결과**
```
RMSE: 2.2m
P90: 1.5m
Within 1m: 72%
> 5m: 156개 (1.4%)

→ ✅ Phase 2 진행
→ Outlier threshold: 5.0m (표준)
```

**예시 2: 보통 결과**
```
RMSE: 2.5m
P90: 1.8m
Within 1m: 65%
> 5m: 345개 (3.1%)

→ ⚠️ Phase 2 진행하되, Outlier 많음 주의
→ Outlier threshold: 4.0m (더 공격적으로)
→ 또는 Depth 12로 재학습 고려
```

**예시 3: 나쁜 결과**
```
RMSE: 3.1m
P90: 2.3m
Within 1m: 52%
> 5m: 678개 (6.1%)

→ ❌ Phase 2 진행 전 재조정 필요
→ 옵션 1: LR 더 낮추기 (5e-5)
→ 옵션 2: Depth 8로 줄이기 (과적합 가능성)
→ 옵션 3: Warmup 15로 늘리기
```

---

## Phase 2 완료 후

### 📋 복사할 정보

```bash
# 1. Phase 2 결과
cat new/models/phase2/logs/test.log

# 2. Phase 1과 비교
echo "=== Phase 1 ==="
grep -A 10 "기본 메트릭" new/models/phase1/logs/test.log
echo "=== Phase 2 ==="
grep -A 10 "기본 메트릭" new/models/phase2/logs/test.log

# 3. 개선폭 확인
# RMSE, P90 얼마나 줄었는지?
```

### 🤔 판단 기준

| Phase 1 → Phase 2 | 판단 | 다음 단계 |
|-------------------|------|-----------|
| RMSE 10% 이상 감소 | ✅ Gradient feature 효과 큼 | Phase 3 진행 |
| RMSE 5-10% 감소 | ⚠️ 보통 | Phase 3 진행 |
| RMSE 5% 미만 감소 | ❌ 효과 미미 | Gradient feature 재검토 |
| RMSE 증가 | ❌❌ 문제 있음 | Phase 1로 복귀 |

### 📝 복사 템플릿

```
====================================================================
Phase 2 결과 리포트 (Phase 1 대비)
====================================================================

--- Phase 1 결과 ---
RMSE: [X.XXX]m
P90:  [X.XXX]m
Within 1m: [XX.X]%
> 5m: [XXX]개

--- Phase 2 결과 ---
RMSE: [X.XXX]m  (Δ [±X.X]%)
P90:  [X.XXX]m  (Δ [±X.X]%)
Within 1m: [XX.X]%  (Δ [±X.X]%)
> 5m: [XXX]개  (Δ [±XX]개)

--- 개선 여부 ---
RMSE 감소: [Yes/No] ([X.X]%)
P90 감소: [Yes/No] ([X.X]%)
Outlier 감소: [Yes/No] ([XXX]개)

====================================================================
```

### 💡 의사결정 예시

**예시 1: 큰 개선**
```
Phase 1: RMSE 2.4m, P90 1.7m
Phase 2: RMSE 2.1m, P90 1.4m
개선: RMSE -12.5%, P90 -17.6%

→ ✅ Gradient feature 효과 확실!
→ Phase 3 진행 (Outlier 제거)
→ Threshold: 5.0m
```

**예시 2: 작은 개선**
```
Phase 1: RMSE 2.4m, P90 1.7m
Phase 2: RMSE 2.3m, P90 1.6m
개선: RMSE -4.2%, P90 -5.9%

→ ⚠️ 효과 있지만 미미
→ Phase 3 진행하되, Phase 4(Multi-scale)가 더 중요할 듯
→ Threshold: 5.0m (표준)
```

**예시 3: 개선 없음**
```
Phase 1: RMSE 2.4m, P90 1.7m
Phase 2: RMSE 2.5m, P90 1.8m
개선: RMSE +4.2%, P90 +5.9%

→ ❌ Gradient feature가 오히려 해로움
→ Phase 1으로 복귀
→ 또는 Gradient 정규화 재검토
```

---

## Phase 3 완료 전 (Outlier 분석)

### 📋 복사할 정보

```bash
# Outlier 분석 결과
cat analysis/outputs/outlier_analysis.json

# 또는 분석 로그
cat analysis/outputs/outlier_analysis_phase2.log

# 특히:
# - Train/Val/Test별 outlier 분포
# - Threshold별 제거될 샘플 수
```

### 🤔 판단 기준: Threshold 결정

| > 5m 비율 | 추천 Threshold | 이유 |
|-----------|----------------|------|
| < 2% | 5.0m (보수적) | Outlier 적음, 데이터 손실 최소화 |
| 2-4% | 5.0m (표준) | 적당한 수준 |
| 4-6% | 4.5m (약간 공격적) | Outlier 많음, 더 제거 |
| > 6% | 4.0m (공격적) | Outlier 매우 많음, 데이터 품질 의심 |

### 📝 복사 템플릿

```
====================================================================
Outlier 분석 결과 (Phase 2 모델 기준)
====================================================================

--- Train Set ---
총 샘플: [XXXX]개
> 3m: [XXX]개 (X.X%)
> 4m: [XXX]개 (X.X%)
> 5m: [XXX]개 (X.X%)  ← 핵심!
> 6m: [XXX]개 (X.X%)

--- Val Set ---
총 샘플: [XXXX]개
> 5m: [XXX]개 (X.X%)

--- Test Set ---
총 샘플: [XXXX]개
> 5m: [XXX]개 (X.X%)

--- 통계 ---
Mean error: [X.XX]m
Median error: [X.XX]m
P90 error: [X.XX]m
Max error: [XX.XX]m

====================================================================
```

### 💡 의사결정 예시

**예시 1: Outlier 적음**
```
Train > 5m: 123개 (1.1%)
Val > 5m: 45개 (1.1%)
Test > 5m: 43개 (1.0%)

→ Threshold: 5.0m
→ 제거 샘플: ~210개 (전체의 ~1%)
→ Phase 3 진행
```

**예시 2: Outlier 보통**
```
Train > 5m: 345개 (3.1%)
Val > 5m: 123개 (3.0%)
Test > 5m: 119개 (2.9%)

→ Threshold: 5.0m (표준)
→ 제거 샘플: ~590개 (전체의 ~3%)
→ Phase 3 진행
```

**예시 3: Outlier 많음**
```
Train > 5m: 678개 (6.1%)
Val > 5m: 234개 (5.7%)
Test > 5m: 221개 (5.4%)

→ 선택지:
  1. Threshold 4.0m (더 공격적) → 약 10% 제거
  2. Threshold 5.0m (표준) → 약 6% 제거
  3. Phase 2 결과 재검토 (왜 이렇게 많은지?)

→ 추천: Threshold 4.5m (절충안)
→ 또는 Phase 2 재학습 (LR, Depth 조정)
```

---

## Phase 3 완료 후

### 📋 복사할 정보

```bash
# Phase 2 vs Phase 3 비교
echo "=== Phase 2 (Outlier 제거 전) ==="
grep -A 10 "기본 메트릭" new/models/phase2/logs/test.log
echo "=== Phase 3 (Outlier 제거 후) ==="
grep -A 10 "기본 메트릭" new/models/phase3/logs/test.log

# Outlier 필터링 정보
cat analysis/outputs/outlier_filter.log | grep -A 5 "총 제거"
```

### 🤔 판단 기준

| 지표 | 기대 | 판단 |
|------|------|------|
| RMSE | 5-10% 감소 | Outlier 제거 효과 |
| P90 | 5-15% 감소 | 중요! |
| Max error | < 5m | ✅ 목표 달성 |
| Within 1m | 증가 | 데이터 품질 향상 |

### 📝 복사 템플릿

```
====================================================================
Phase 3 결과 리포트 (Outlier 제거 효과)
====================================================================

--- Outlier 제거 정보 ---
Threshold: [X.X]m
제거된 샘플:
  Train: [XXX]개 (X.X%)
  Val:   [XXX]개 (X.X%)
  Test:  [XXX]개 (X.X%)

--- Phase 2 결과 ---
RMSE: [X.XXX]m
P90:  [X.XXX]m
Max:  [XX.XX]m
Within 1m: [XX.X]%

--- Phase 3 결과 ---
RMSE: [X.XXX]m  (Δ [±X.X]%)
P90:  [X.XXX]m  (Δ [±X.X]%)
Max:  [XX.XX]m  (Δ [±XX.X]%) ← 중요!
Within 1m: [XX.X]%  (Δ [±X.X]%)

--- 목표 달성 여부 ---
RMSE ≤ 2.0m: [Yes/No]
Max ≤ 5m: [Yes/No]
Within 1m ≥ 80%: [Yes/No]

====================================================================
```

### 💡 의사결정 예시

**예시 1: 목표 달성**
```
Phase 3 결과:
  RMSE: 1.9m (목표 달성!)
  P90: 1.2m
  Max: 4.8m (목표 달성!)
  Within 1m: 82% (목표 달성!)

→ ✅ Phase 3까지로 충분!
→ Phase 4는 선택사항 (더 개선 원하면)
→ 결과 만족스러우면 여기서 종료 가능
```

**예시 2: 목표 근접**
```
Phase 3 결과:
  RMSE: 2.1m (목표: 2.0m)
  P90: 1.3m
  Max: 5.2m (목표: 5.0m)
  Within 1m: 78% (목표: 80%)

→ ⚠️ 거의 다 왔음!
→ Phase 4 진행 추천 (Multi-scale + Attention)
→ 예상: RMSE 1.8-2.0m, 목표 달성 가능
```

**예시 3: 추가 개선 필요**
```
Phase 3 결과:
  RMSE: 2.3m
  P90: 1.5m
  Max: 6.1m
  Within 1m: 72%

→ ❌ 목표 미달
→ 선택지:
  1. Phase 4 진행 (Multi-scale)
  2. Depth 12로 재학습
  3. Direction features 추가 (Phase 3.5)
```

---

## 🎯 빠른 의사결정 플로우

```
Phase 1 완료
  │
  ├─ RMSE < 2.3m → Phase 2 진행
  ├─ RMSE 2.3-2.6m → Phase 2 진행 (주의)
  └─ RMSE > 2.6m → 파라미터 재조정

Phase 2 완료
  │
  ├─ 개선 > 10% → Phase 3 진행 (threshold 5.0m)
  ├─ 개선 5-10% → Phase 3 진행 (threshold 4.5-5.0m)
  └─ 개선 < 5% → Phase 1 복귀 or Phase 3 skip

Phase 3 완료
  │
  ├─ 목표 달성 → 종료 or Phase 4 (선택)
  ├─ 목표 근접 → Phase 4 진행
  └─ 목표 미달 → Depth/LR 재조정 or Phase 4
```

---

**💡 Tip**: 각 Phase 결과를 위 템플릿에 맞춰 보내주시면,
다음 단계 설정(Threshold, LR, Depth 등)을 정확히 추천해드립니다!
