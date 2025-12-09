# 🚀 빠른 시작 가이드

## 준비된 것

✅ `new/src/preprocess_from_csv.py` - 그리드 선형보간 포함한 전처리
✅ `new/src/model.py` - Hyena 모델
✅ `new/src/train_sliding.py` - 깔끔한 MSE Loss 학습
✅ `new/src/test_only.py` - 테스트
✅ `new/data/nodes_final.csv` - 수정된 노드 좌표
✅ `new/data/node_connections.csv` - 노드 연결 정보 (그래프 생성용)

## 준비해야 할 것

1. `data/raw/*.csv` - Raw 센서 데이터 (기존 것 사용)
2. Python 가상환경 활성화

---

## 단계별 실행

### 0. 환경 준비

```bash
cd /Users/yunho/school/lstm
source venv/bin/activate
```

### 1. 데이터 전처리 (20-30분)

```bash
python new/src/preprocess_from_csv.py \
  --raw-dir data/raw \
  --nodes-file new/data/nodes_final.csv \
  --output-dir new/data/preprocessed
```

**결과:**
- `new/data/preprocessed/*.csv` - 그리드 선형보간된 CSV
- `new/data/sliding_mag4/train.jsonl` - 학습용 (~11K samples, 230 files)
- `new/data/sliding_mag4/val.jsonl` - 검증용 (~4K samples, 87 files)
- `new/data/sliding_mag4/test.jsonl` - 테스트용 (~4K samples, 87 files)
- `new/data/sliding_mag4/meta.json` - 메타데이터

**설정:**
- Window size: 250 timesteps
- Stride: 50 (80% overlap, ~0.86m 이동)
- Stratified 분할: 모든 87개 경로가 Train/Val/Test에 포함

### 2. 학습 (2-3시간, MPS)

```bash
python new/src/train_sliding.py \
  --data-dir new/data/sliding_mag4 \
  --epochs 100 \
  --batch-size 128 \
  --lr 2e-4 \
  --hidden-dim 384 \
  --depth 10 \
  --patience 15 \
  --checkpoint-dir new/models/hyena_mag4/checkpoints
```

**결과:**
- `new/models/hyena_mag4/checkpoints/best.pt` - Best 모델 (P90 기준)

**학습 중 확인:**
- Train Loss 감소
- Val P90 감소
- Early stopping 작동 (15 epoch patience)

### 3. 테스트 (1-2분)

```bash
python new/src/test_only.py \
  --checkpoint new/models/hyena_mag4/checkpoints/best.pt \
  --data-dir new/data/sliding_mag4 \
  --hidden-dim 384 \
  --depth 10 \
  --batch-size 128
```

**확인 지표:**
- MAE (Mean Absolute Error)
- RMSE (Root Mean Square Error)
- Median, P90, P95
- Within 1m/2m/3m accuracy
- Noise robustness (1%, 5%, 10%, 20%)

---

## 기대 성능

### 목표 (기존 Huber Loss 버전보다 개선)

| 지표 | 기존 (Huber) | 목표 (MSE) |
|------|-------------|-----------|
| MAE | 1.140m | < 1.1m |
| RMSE | 3.284m | < 3.0m ← **중요!** |
| Median | 0.654m | ~0.65m |
| P90 | 1.738m | < 1.7m |
| ≤1m | 68.6% | > 68% |

**핵심**: RMSE가 감소해야 성공!

---

## 한 줄 명령어 (순차 실행)

```bash
# 전체 파이프라인
source venv/bin/activate && \
python new/src/preprocess_from_csv.py \
  --raw-dir data/raw \
  --nodes-file new/data/nodes_final.csv \
  --output-dir new/data/preprocessed && \
python new/src/train_sliding.py \
  --data-dir new/data/sliding_mag4 \
  --epochs 100 \
  --batch-size 128 \
  --hidden-dim 384 \
  --depth 10 && \
python new/src/test_only.py \
  --checkpoint new/models/hyena_mag4/checkpoints/best.pt \
  --data-dir new/data/sliding_mag4 \
  --hidden-dim 384 \
  --depth 10
```

---

## 문제 해결

### 1. "No such file or directory: new/data/preprocessed"

```bash
mkdir -p new/data/preprocessed
mkdir -p new/data/sliding_mag4
mkdir -p new/models/hyena_mag4/checkpoints
```

### 2. "ModuleNotFoundError: No module named 'model'"

```bash
# 실행 위치 확인
pwd  # /Users/yunho/school/lstm 여야 함

# 잘못된 위치면
cd /Users/yunho/school/lstm
```

### 3. RMSE가 여전히 높음

**체크리스트:**
- [ ] 전처리가 올바르게 완료됨? (`new/data/preprocessed/*.csv` 존재?)
- [ ] Sliding window 생성됨? (`new/data/sliding_mag4/*.jsonl` 존재?)
- [ ] 학습이 수렴함? (Val Loss 감소 추세?)
- [ ] Early stopping 작동? (Best model 저장됨?)

**시도해볼 것:**
```bash
# 1. 학습률 낮추기
--lr 1e-4

# 2. Depth 줄이기
--depth 8

# 3. Hidden dim 줄이기
--hidden-dim 256

# 4. Epoch 늘리기
--epochs 150 --patience 20
```

### 4. MPS 오류 (Apple Silicon)

```bash
# CPU로 강제 실행
python new/src/train_sliding.py --cpu ...
```

---

## 체크포인트

### 전처리 완료 후

```bash
ls new/data/preprocessed/ | wc -l
# → 404개 파일 (각 경로별 측정 파일)

ls new/data/sliding_mag4/
# → train.jsonl, val.jsonl, test.jsonl, meta.json
```

### 학습 완료 후

```bash
ls new/models/hyena_mag4/checkpoints/
# → best.pt

# 모델 크기 확인
du -sh new/models/hyena_mag4/checkpoints/best.pt
# → ~24MB
```

### 테스트 완료 후

출력 예시:
```
[Test Results]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 기본 메트릭:
  MAE:  1.XXXm
  RMSE: 2.XXXm  ← 이게 3.0 이하면 성공!

📈 분포:
  Median: 0.XXXm
  P90:    1.XXXm

📍 CDF:
  ≤ 1m:  XX.X%
  ≤ 2m:  XX.X%
```

---

## 다음 단계

RMSE가 개선되면:

1. **성능 비교**
   ```bash
   # 기존 vs 새 버전
   python src/test_only.py --checkpoint models/hyena_mag4/checkpoints/best.pt ...
   python new/src/test_only.py --checkpoint new/models/hyena_mag4/checkpoints/best.pt ...
   ```

2. **발표 자료 업데이트**
   - MAE, RMSE 업데이트
   - "깔끔한 MSE Loss로 RMSE XX% 개선" 추가

3. **커밋**
   ```bash
   git add new/
   git commit -m "feat: Clean MSE Loss 버전 (RMSE 개선)"
   ```

---

## 요약

1. ✅ **전처리**: 그리드 선형보간 포함 (현재 버전 유지)
2. ✅ **Loss**: MSE Loss (Huber + 가중치 제거)
3. ✅ **정규화**: COORD_CENTER=(-44.3, -0.3), COORD_SCALE=48.8
4. ✅ **nodes_final.csv**: 수정된 버전

**목표**: RMSE < 3.0m

**Good Luck! 🚀**
