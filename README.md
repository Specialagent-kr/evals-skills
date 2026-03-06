# AI 코딩 에이전트를 위한 평가 스킬 (Eval Skills)

AI 코딩 에이전트가 LLM 평가(Evaluations)를 구축할 수 있도록 돕는 스킬 모음입니다.

이 프로젝트는 PM(Product Manager)이 AI 제품의 품질을 체계적으로 관리하고 평가할 수 있도록 돕는 한국어 스킬들을 제공합니다. 50개 이상의 기업을 돕고 [AI Evals 코스](https://maven.com/parlance-labs/evals?promoCode=evals-info-url)에서 학생들을 가르치며 발견한 일반적인 실수들을 방지하도록 설계되었습니다. 평가가 처음이라면 기초 자료로 [참고자료.md](참고자료.md)를 참고하세요.

> [!NOTE]
> 이 프로젝트는 PHamel Husain의 [evals-skills](https://github.com/hamelsmu/evals-skills)를 포크하여 한국어로 번역한 저장소입니다.

## 평가가 처음이신가요? 여기서 시작하세요

평가가 처음이라면 `eval-audit` 스킬부터 시작하는 것이 좋습니다. 코딩 에이전트에게 다음 지침을 전달하세요:

> <https://github.com/Specialagent-kr/evals-skills> 에서 평가 스킬 플러그인을 설치한 후, 내 평가 파이프라인에 대해 `/evals-skills:eval-audit`을 실행해줘. 각 진단 영역을 별도의 서브 에이전트로 병렬 조사하고, 결과를 하나의 보고서로 통합해줘. 감사 결과에서 권장하는 플러그인의 다른 스킬들도 함께 사용해줘.

이 감사는 완전한 해결책은 아니지만, 평가에서 흔히 발생하는 문제들을 포착해 줍니다. 또한 문제를 해결하기 위해 사용해야 할 다른 스킬들을 추천해 줍니다.

## 설치 방법 (Claude Code)

Claude Code에서 다음 두 명령어를 실행하세요:

```bash
# 1단계: 플러그인 저장소 등록
/plugin marketplace add Specialagent-kr/evals-skills

# 2단계: 플러그인 설치
/plugin install evals-skills@Specialagent-kr-evals-skills
```

업데이트 방법:

```bash
/plugin update evals-skills@Specialagent-kr-evals-skills
```

설치 후 Claude Code를 재시작하세요. 스킬은 `/evals-skills:<스킬-이름>` 형태로 나타납니다.

## 설치 방법 (npx skills)

오픈 소스인 Skills CLI를 사용하는 경우, 다음 명령어로 설치하세요:

```bash
npx skills add https://github.com/Specialagent-kr/evals-skills
```

특정 스킬 하나만 설치하려면:

```bash
npx skills add https://github.com/Specialagent-kr/evals-skills --skill eval-audit
```

업데이트 확인:

```bash
npx skills check
npx skills update
```

## 사용 가능한 스킬

| 스킬 | 기능 설명 |
|-------|-----------|
| eval-audit | 평가 파이프라인을 감사하고 심각도별로 문제를 분류하여 표시 |
| error-analysis | 트레이스 읽기 및 실패 케이스 분류 과정을 사용자에게 안내 |
| generate-synthetic-data | 차원 기반 Tuple 생성을 사용하여 다양한 합성 테스트 입력 데이터 생성 |
| write-judge-prompt | 주관적인 품질 기준을 위한 LLM-평가(LLM-as-Judge) 평가기 설계 |
| validate-evaluator | 데이터 분할, TPR/TNR, 편향 보정을 사용하여 LLM 평가 모델을 사람의 라벨과 대조하여 보정 |
| evaluate-rag | RAG 파이프라인의 검색 및 생성 품질 평가 |
| build-review-interface | 사람이 직접 트레이스를 리뷰할 수 있는 맞춤형 어노테이션 인터페이스 구축 |

스킬을 호출하려면 `/evals-skills:스킬-이름`을 입력하세요 (예: `/evals-skills:error-analysis`).

## 맞춤형 스킬 작성하기

제공되는 스킬들은 여러 프로젝트에 공통적으로 적용되는 일반적인 실수들을 다루는 시작점일 뿐입니다. 여러분의 기술 스택, 도메인, 데이터에 기반한 맞춤형 스킬이 훨씬 더 좋은 성능을 발휘할 것입니다. 여기서 시작하여 여러분만의 스킬을 작성해 보세요.

[meta-skill](meta-skill.md) 문서는 맞춤형 스킬을 구축하는 데 도움을 줄 수 있습니다.

## 스킬 그 이상의 내용

이 스킬들은 여러 프로젝트에서 공통적으로 발생하는 평가 작업들을 처리합니다. 프로덕션 모니터링, CI/CD 통합, 데이터 분석 등 프로젝트마다 달라지는 부분들은 [코스](https://maven.com/parlance-labs/evals?promoCode=evals-info-url)에서 자세히 다루고 있습니다.
