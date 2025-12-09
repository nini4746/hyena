# 🚨 에러 발생 시 디버깅 가이드

스크립트 실행 중 에러가 발생하면 아래 내용을 복사해서 보내주세요.

---

## 📋 에러 발생 시 필수 정보

### 1️⃣ 기본 정보
```bash
# 어느 Phase에서 에러 발생?
Phase: [1/2/3/4]

# 어느 단계에서 멈췄나요?
Step: [전처리/학습/테스트]
```

### 2️⃣ 에러 메시지 (중요!)
```bash
# 터미널에 출력된 마지막 20-30줄 복사
# 특히 "Error", "Traceback", "Exception" 포함된 부분 전부!

예시:
Traceback (most recent call last):
  File "new/src/train_sliding.py", line 245, in <module>
    ...
RuntimeError: ...
```

### 3️⃣ 로그 파일 내용
```bash
# 에러가 학습 중 발생했다면:
tail -50 new/models/phase[X]/logs/train.log

# 에러가 테스트 중 발생했다면:
tail -50 new/models/phase[X]/logs/test.log

# 전체 로그 확인:
cat new/models/phase[X]/logs/train.log
```

---

## 🔍 에러 타입별 가이드

### A. 전처리 에러

**증상**: "preprocess" 관련 에러, 파일 없음 에러

**복사할 것**:
```bash
1. 에러 메시지 전체
2. ls data/raw | head -10  # Raw 파일 확인
3. ls new/data/  # 출력 디렉토리 확인
4. head -5 new/data/nodes_final.csv  # 노드 파일 확인
```

**예시**:
```
FileNotFoundError: [Errno 2] No such file or directory: 'data/raw/...'
```

---

### B. 학습 에러

**증상**: "train" 중 에러, CUDA/MPS 에러, 메모리 에러

**복사할 것**:
```bash
1. 에러 메시지 전체 (Traceback 포함)
2. tail -100 new/models/phase[X]/logs/train.log  # 마지막 100줄
3. 학습이 몇 epoch까지 진행됐는지?
4. 시스템 정보:
   - python --version
   - which python
   - nvidia-smi  (GPU 있으면)
```

**예시**:
```
RuntimeError: MPS backend out of memory
또는
RuntimeError: CUDA out of memory
```

---

### C. 모듈 Import 에러

**증상**: "ModuleNotFoundError", "ImportError"

**복사할 것**:
```bash
1. 에러 메시지 전체
2. pip list | grep -E "torch|numpy|pywt"  # 설치된 패키지 확인
3. which python  # Python 경로 확인
4. echo $VIRTUAL_ENV  # 가상환경 확인
```

**예시**:
```
ModuleNotFoundError: No module named 'model'
또는
ImportError: cannot import name 'HyenaModel'
```

---

### D. 데이터 로딩 에러

**증상**: "DataLoader" 에러, Shape mismatch 에러

**복사할 것**:
```bash
1. 에러 메시지 전체
2. cat new/data/sliding_mag4/meta.json  # 메타데이터 확인
3. wc -l new/data/sliding_mag4/*.jsonl  # 샘플 개수 확인
4. head -1 new/data/sliding_mag4/train.jsonl  # 샘플 형식 확인
```

**예시**:
```
RuntimeError: shape mismatch: value tensor of shape [250, 4] does not match ...
```

---

### E. 모델 체크포인트 에러

**증상**: "checkpoint" 없음, 로드 실패

**복사할 것**:
```bash
1. 에러 메시지 전체
2. ls -lh new/models/phase[X]/checkpoints/  # 체크포인트 파일 확인
3. file new/models/phase[X]/checkpoints/best.pt  # 파일 타입 확인
```

**예시**:
```
FileNotFoundError: new/models/phase2/checkpoints/best.pt
```

---

## 🎯 빠른 복사 템플릿

에러 발생 시 이 템플릿 채워서 보내주세요:

```
====================================================================
🚨 에러 리포트
====================================================================

Phase: [1/2/3/4]
Step: [전처리/학습/테스트]

--- 에러 메시지 ---
[여기에 에러 메시지 전체 복사]


--- 로그 마지막 50줄 ---
[tail -50 new/models/phase[X]/logs/train.log 결과 복사]


--- 추가 정보 ---
Python 버전: [python --version]
데이터 확인: [ls new/data/]
체크포인트: [ls new/models/phase[X]/checkpoints/]

====================================================================
```

---

## 💡 자주 발생하는 에러와 해결법

### 1. "No such file or directory: data/raw"
```bash
# 해결: Raw 데이터 확인
ls data/raw/

# 없으면:
echo "data/raw 디렉토리에 CSV 파일을 넣어주세요"
```

### 2. "MPS backend out of memory"
```bash
# 해결: Batch size 줄이기
./scripts/run_phase1.sh --batch-size 64  # 128 → 64
```

### 3. "ModuleNotFoundError: No module named 'model'"
```bash
# 해결: 현재 디렉토리 확인
pwd  # /Users/yunho/school/lstm 이어야 함

# 잘못된 위치면
cd /Users/yunho/school/lstm
```

### 4. "Phase 2 모델이 없습니다"
```bash
# 해결: Phase 순서대로 실행
./scripts/run_phase1.sh  # 먼저 Phase 1
./scripts/run_phase2.sh  # 그 다음 Phase 2
```

### 5. 학습이 너무 느림
```bash
# MPS 사용 확인
grep "MPS" new/models/phase1/logs/train.log

# "🍎 Apple Silicon GPU (MPS) 사용" 없으면 CPU로 돌고 있음
# → 정상 (MPS가 없는 환경)
```

---

## 📞 에러 리포트 예시

### 좋은 예시 ✅
```
Phase: 2
Step: 학습

--- 에러 메시지 ---
Traceback (most recent call last):
  File "new/src/train_sliding.py", line 245, in <module>
    main()
  File "new/src/train_sliding.py", line 198, in main
    model = HyenaModel(...)
RuntimeError: Expected 4D input, got 3D

--- 로그 마지막 50줄 ---
Epoch 1/400 [Train]: 100%|███████| 85/85 [02:15<00:00,  1.59s/it]
Train Loss: 0.0234
Validation...
RuntimeError: ...

--- 추가 정보 ---
Python 3.10.5
데이터: new/data/sliding_grad/ 존재
메타: {"n_features": 8, "window_size": 250}
```

### 나쁜 예시 ❌
```
에러남

또는

안됨
```

---

## 🔧 일반적인 해결 순서

1. **에러 메시지 읽기** - 무슨 에러인지 파악
2. **로그 확인** - 어디서 멈췄는지 확인
3. **파일 존재 확인** - 필요한 파일 있는지 확인
4. **위 템플릿 작성** - 정보 수집
5. **질문하기** - 템플릿 채워서 보내기

---

**💡 Tip**: 에러가 발생해도 당황하지 마세요!
에러 메시지 + 로그만 있으면 대부분 해결 가능합니다.
