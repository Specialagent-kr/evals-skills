---
name: validate-evaluator
description: >
  데이터 분할, TPR/TNR, 편향 보정을 사용하여 사람의 라벨과 대조하며 LLM 평가(Judge)를 보정합니다. 평가 프롬프트를 작성한 후(`write-judge-prompt`), 그 출력 결과를 신뢰하기 전에 일치도를 확인해야 할 때 사용하세요. 코드 기반 평가기(결정론적이며 표준 유닛 테스트로 테스트 가능함)에는 사용하지 마세요.
---

# 평가기 검증 (Validate Evaluator)

사람의 판단 기준에 맞춰 LLM 평가를 보정합니다.

## 개요

1. 사람의 라벨이 있는 데이터를 훈련(10-20%), 개발(40-45%), 테스트(40-45%) 셋으로 분할
2. 개발(Dev) 셋에서 평가를 실행하고 TPR/TNR 측정
3. 개발 셋에서 TPR과 TNR이 모두 90%를 넘을 때까지 평가 프롬프트 수정 반복
4. 별도의 테스트(Test) 셋에서 단 한 번 실행하여 최종 TPR/TNR 확인
5. 프로덕션 데이터에 편향 보정 공식 적용

## 전제 조건

- 기구축된 LLM 평가 프롬프트 (`write-judge-prompt` 결과물)
- 사람의 라벨이 있는 데이터: 실패 모드당 약 100개의 이진(Pass/Fail) 라벨이 포함된 트레이스
  - 합격(Pass) 약 50개, 실패(Fail) 약 50개를 목표로 합니다 (실제 분포가 한쪽으로 치우쳐 있더라도 균형을 맞추는 것이 좋습니다).
  - 라벨은 외부 어노테이터가 아닌 도메인 전문가가 직접 작성해야 합니다.
- 라벨링된 데이터 중 퓨샷(Few-shot) 예시로 사용할 후보군

## 핵심 지침

### 1단계: 데이터 분할 생성

사람의 라벨이 있는 데이터를 겹치지 않는 세 개의 집합으로 나눕니다:

| 분할 | 비율 | 목적 | 규칙 |
|-------|------|---------|-------|
| **훈련(Training)** | 10-20% (~10-20개) | 평가 프롬프트의 퓨샷 예시 출처 | 명확한 Pass와 Fail 사례만 포함. 프롬프트에 직접 사용됨. |
| **개발(Dev)** | 40-45% (~40-45개) | 평가기 반복 개선 및 튜닝 | 프롬프트에 절대 포함하지 말 것. 반복적으로 성능 측정에 사용됨. |
| **테스트(Test)** | 40-45% (~40-45개) | 최종 편향 없는 정확도 측정 | 개발 중에는 절대 열어보지 말 것. 마지막에 단 한 번만 사용됨. |

목표: 개발 셋과 테스트 셋을 합쳐 각 클래스(Pass/Fail)당 30~50개의 사례를 확보하세요. 실제 현실에서의 보급률이 한쪽으로 쏠려 있더라도 균형 잡힌 분할을 사용해야 TNR을 신뢰성 있게 측정할 수 있습니다.

```python
from sklearn.model_selection import train_test_split

# 첫 번째 분할: 테스트 셋 분리
train_dev, test = train_test_split(
    labeled_data, test_size=0.4, stratify=labeled_data['label'], random_state=42
)
# 두 번째 분할: 개발 셋에서 훈련용 예시 분리
train, dev = train_test_split(
    train_dev, test_size=0.75, stratify=train_dev['label'], random_state=42
)
# 결과: 약 15% 훈련, 45% 개발, 40% 테스트
```

### 2단계: 개발 셋에서 평가 실행

개발 셋의 모든 사례에 대해 평가를 실행합니다. 결과를 사람의 라벨과 비교합니다.

### 3단계: TPR 및 TNR 측정

**TPR (True Positive Rate):** 사람이 Pass라고 했을 때, 평가도 Pass라고 할 확률은?

```
TPR = (평가가 Pass라고 함 AND 사람이 Pass라고 함) / (사람이 Pass라고 함)
```

**TNR (True Negative Rate):** 사람이 Fail이라고 했을 때, 평가도 Fail이라고 할 확률은?

```
TNR = (평가가 Fail이라고 함 AND 사람이 Fail이라고 함) / (사람이 Fail이라고 함)
```

```python
from sklearn.metrics import confusion_matrix

tn, fp, fn, tp = confusion_matrix(human_labels, evaluator_labels,
                                   labels=['Fail', 'Pass']).ravel()
tpr = tp / (tp + fn)
tnr = tn / (tn + fp)
```

단순 정확도(Accuracy)나 정밀도(Precision)/재현율(Recall) 대신 TPR/TNR을 사용하세요. 이 두 지표는 편향 보정 공식에 직접적으로 사용됩니다. 코헨의 카파(Cohen's Kappa)는 두 사람 사이의 일치도를 측정할 때만 사용하고, 평가 대 정답(Ground-truth) 비교에는 사용하지 마세요.

### 4단계: 불일치 사례 조사

평가가 사람의 라벨과 일치하지 않는 모든 케이스를 검토하세요:

| 불일치 유형 | 평가 판단 | 사람 판단 | 해결 방법 |
|-------------------|-------|-------|-----|
| **가짜 합격(False Pass)** | Pass | Fail | 평가가 너무 관대함. Fail 정의를 강화하거나 예외 사례 예시를 추가함. |
| **가짜 실패(False Fail)** | Fail | Pass | 평가가 너무 엄격함. Pass 정의를 명확히 하거나 예시를 조정함. |

각 불일치 사례에 대해 다음 중 하나를 결정합니다:

- 평가 프롬프트의 문구 수정
- 훈련 셋에서 퓨샷 예시를 교체하거나 추가
- 예외 상황에 대한 명시적인 규칙 추가
- 평가 기준을 더 구체적인 하위 체크 항목으로 분리

### 5단계: 반복 개선

평가 프롬프트를 수정하고 개발 셋에 대해 다시 실행합니다. TPR과 TNR이 안정화될 때까지 이 과정을 반복하세요.

**종료 기준:**

- **목표:** TPR > 90% 그리고 TNR > 90%
- **최소 수용치:** TPR > 80% 그리고 TNR > 80%

**일치도 개선이 정체될 때:**

| 문제 상황 | 해결 방법 |
|---------|---------|
| TPR과 TNR 모두 낮음 | 평가 모델로 더 강력한 성능의 LLM 사용 |
| 한 지표는 괜찮은데 다른 하나가 낮음 | 낮은 지표에 해당하는 불일치 사례를 집중 조사 |
| 두 지표 모두 목표치 아래에서 멈춤 | 평가 기준을 더 작고 원자적인(Atomic) 체크 항목들로 분해 |
| 특정 입력 유형에 대해 계속 틀림 | 훈련 셋에서 해당 유형에 대한 타겟팅된 퓨샷 예시 추가 |
| 라벨 자체가 일관성 없어 보임 | 사람의 라벨을 재검토; 평가 루브릭 자체의 정교화가 필요함 |

### 6단계: 테스트 셋에서 최종 측정

별도로 보관해 둔 테스트 셋에 대해 평가를 **단 한 번만** 실행합니다. 최종 TPR과 TNR을 기록하세요.

테스트 셋 결과를 보고 프롬프트를 다시 수정하지 마세요. 필요하다면 새로운 개발 데이터를 확보하여 4단계로 돌아가세요.

### 7단계 (선택 사항): 실제 성공률 추정 (Rogan-Gladen 보정)

라벨링되지 않은 프로덕션 데이터에 대한 평가 점수에는 편향이 섞여 있습니다. 정확한 합계 패스율(Pass rate)이 필요하다면, 알려진 평가의 오류율을 바탕으로 보정하세요:

```
theta_hat = (p_obs + TNR - 1) / (TPR + TNR - 1)
```

여기서:

- `p_obs` = 평가가 Pass라고 판정한 프로덕션 트레이스의 비율
- `TPR`, `TNR` = 테스트 셋 측정값
- `theta_hat` = 보정된 실제 성공률 추정치

결과값은 [0, 1] 사이로 제한(Clip)하세요. `TPR + TNR - 1`이 0에 가까우면(평가 성능이 무작위 선택과 다를 바 없음) 이 공식은 유효하지 않습니다.

**예시:**

- 평가 성능: TPR = 0.92, TNR = 0.88
- 프로덕션 트레이스 500개 중 평가가 400개를 Pass로 판정 -> p_obs = 0.80
- theta_hat = (0.80 + 0.88 - 1) / (0.92 + 0.88 - 1) = 0.68 / 0.80 = **0.85**
- 실제 성공률은 원시 데이터인 80%가 아니라 약 85%입니다.

### 8단계: 신뢰 구간 (Confidence Interval)

단일 추정값만으로는 부족합니다. 붓스트랩(Bootstrap) 신뢰 구간을 계산하세요.

```python
import numpy as np

def bootstrap_ci(human_labels, eval_labels, p_obs, n_bootstrap=2000):
    """보정된 성공률에 대한 붓스트랩 95% 신뢰 구간 계산."""
    n = len(human_labels)
    estimates = []
    for _ in range(n_bootstrap):
        idx = np.random.choice(n, size=n, replace=True)
        h = np.array(human_labels)[idx]
        e = np.array(eval_labels)[idx]

        tp = ((h == 'Pass') & (e == 'Pass')).sum()
        fn = ((h == 'Pass') & (e == 'Fail')).sum()
        tn = ((h == 'Fail') & (e == 'Fail')).sum()
        fp = ((h == 'Fail') & (e == 'Pass')).sum()

        tpr_b = tp / (tp + fn) if (tp + fn) > 0 else 0
        tnr_b = tn / (tn + fp) if (tn + fp) > 0 else 0
        denom = tpr_b + tnr_b - 1

        if abs(denom) < 1e-6:
            continue
        theta = (p_obs + tnr_b - 1) / denom
        estimates.append(np.clip(theta, 0, 1))

    return np.percentile(estimates, 2.5), np.percentile(estimates, 97.5)

lower, upper = bootstrap_ci(test_human, test_eval, p_obs=0.80)
print(f"95% CI: [{lower:.2f}, {upper:.2f}]")
```

또는 `judgy` 패키지를 사용하세요 (`pip install judgy`):

```python
from judgy import estimate_success_rate

result = estimate_success_rate(
    human_labels=test_human_labels,
    evaluator_labels=test_eval_labels,
    unlabeled_labels=prod_eval_labels
)
print(f"보정된 성공률: {result.estimate:.2f}")
print(f"95% 신뢰 구간: [{result.ci_lower:.2f}, {result.ci_upper:.2f}]")
```

## 실무 가이드라인

- LLM 평가 모델의 버전을 **명확히 고정**(Pin)하세요 (예: `gpt-4o`가 아닌 `gpt-4o-2024-05-13`). 모델이 몰래 업데이트되어 성능이 변하는 것을 방지해야 합니다.
- 평가 프롬프트를 수정하거나, 모델을 교체하거나, 프로덕션의 신뢰 구간이 비정상적으로 넓어지면 **재검증**을 수행하세요.
- 약 100개의 라벨링된 예시(Pass 50, Fail 50)를 사용하세요. 60개 미만이면 신뢰 구간이 너무 넓어집니다.
- **신뢰할 수 있는 한 명의 도메인 전문가**가 라벨링을 하는 것이 가장 효율적입니다. 불가능하다면 두 명의 어노테이터가 20~50개의 트레이스를 독립적으로 라벨링한 뒤, 불일치 사항을 해결하고 진행하세요.
- **TPR을 개선하는 것이 TNR을 개선하는 것보다 신뢰 구간을 좁히는 데 더 효과적입니다.** 보정 공식은 TPR로 나누기 때문에, 낮은 TPR은 추정 오차를 크게 증폭시켜 신뢰 구간을 넓게 만듭니다.

## 안티 패턴

- **검증 없이 평가가 그냥 잘 작동할 것이라고 가정함.** 평가는 지속적으로 실패를 놓치거나 정상 케이스를 오판할 수 있습니다.
- **단순 정확도나 일치율 사용.** 반드시 TPR과 TNR을 사용하세요. 클래스 불균형이 있는 환경에서 정확도는 기만적입니다.
- **개발/테스트 셋 데이터를 퓨샷 예시로 사용.** 이는 데이터 유출(Data leakage)입니다.
- **개발 셋 성능을 최종 정확도로 보고함.** 개발 셋 수치는 낙관적으로 편향되어 있습니다. 테스트 셋만이 편향 없는 추정치를 제공합니다.
- **편향 보정 없는 원시 평가 점수 보고.** 합계 패스율을 보고할 때는 반드시 Rogan-Gladen 공식(7단계)을 적용하세요.
- **신뢰 구간 없는 단일 추정치 보고.** 보정 결과가 85%라 하더라도 테스트 데이터가 적다면 실제로는 78-92% 사이일 수 있습니다. 이해관계자가 수치를 얼마나 신뢰할 수 있는지 알 수 있도록 범위를 함께 보고하세요.
