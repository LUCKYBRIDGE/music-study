# music-study

초등 음악 수업에 활용할 수 있는 웹·앱·도구를 **교육 설계, 음악 이론, UI/UX, 상호작용, 기술 구조, 재구현 가능성**의 관점에서 분석하고 축적하는 저장소입니다.

단순한 기능 소개가 아니라, 실제 화면에서 확인되는 동작을 분해하여 **어떤 음악 개념을 어떤 인터페이스와 로직으로 풀었는지**, 그리고 새로운 교육용 음악 도구를 만들 때 무엇을 가져오고 무엇을 수정해야 하는지를 기록합니다.

## 분석 원칙

- 관찰 가능한 화면·텍스트·상호작용과 내부 구현 추정을 구분합니다.
- 음악 이론의 정확성을 별도로 검토합니다.
- 기능 목록보다 **학습 흐름과 문제 해결 방식**을 우선 분석합니다.
- 좋은 패턴은 서비스별 문서에서 분리해 공통 패턴으로 축적합니다.
- 재구현 시 사용할 수 있도록 상태 모델, 데이터 구조, 알고리즘 후보까지 기록합니다.
- 확인하지 못한 라이브러리·함수·서버 구조는 사실처럼 단정하지 않습니다.

분석 방식은 [Craft Anything](https://github.com/storytold/craft-anything)의 증거 기반 분석 관점을 참고합니다.

## 디렉터리

```text
music-study/
├─ references/                  # 서비스·웹·앱별 심층 분석
│  └─ major-minor-composer/
├─ patterns/                    # 여러 서비스에서 재사용할 설계 패턴
├─ benchmarks/                  # 비교 대상 서비스 목록과 비교 기준
└─ decisions/                   # 향후 자체 구현을 위한 설계 결정
```

## 현재 분석

### 장조·단조 네 마디 작곡소

- 대상: 5학년 음악
- URL: https://leixlix-pixel.github.io/toolbox/major-minor-composer.html
- 핵심: 장·단조 청음 비교 → 조 선택 → 4마디 가락 만들기 → 화음 피드백 → 완성·발표
- 분석 문서: [references/major-minor-composer/README.md](references/major-minor-composer/README.md)
- 음악 이론 검토: [references/major-minor-composer/music-theory-audit.md](references/major-minor-composer/music-theory-audit.md)
- UI·기술 구조: [references/major-minor-composer/interaction-architecture.md](references/major-minor-composer/interaction-architecture.md)
- 화면 캡처 기반 UI·UX 분석: [references/major-minor-composer/visual-ui-analysis.md](references/major-minor-composer/visual-ui-analysis.md)

## 앞으로의 분석 축

1. **학습 설계** — 도입, 탐색, 창작, 피드백, 성찰, 발표
2. **음악 표현 모델** — 오선보, 피아노롤, 리듬 그리드, 코드, 음계
3. **UI/UX** — 입력 방식, 단계 공개, 오류 방지, 모바일/태블릿
4. **음악 로직** — 음계, 조성, 화음, 리듬, 비화성음, 종지
5. **오디오 엔진** — 재생, 템포, 악기, 동시 발음, 타이밍
6. **데이터 구조** — 곡 상태, undo/redo, 저장, 공유, 내보내기
7. **수업 운영** — 작품 저장, 제출, 비교, 발표, 교사 피드백
8. **접근성** — 키보드, 음성, 색상 의존성, 터치 타깃

## 분석 예정 비교군

- Chrome Music Lab Song Maker
- Ableton Learning Music
- Groove Pizza
- Flat for Education
- Noteflight Learn
- 기타 초등 음악 창작·리듬·기보 웹 도구

각 서비스 분석이 쌓이면 개별 기능을 복제하는 것이 아니라, **초등학생이 음악 개념을 직접 조작하고 듣고 수정할 수 있는 공통 설계 패턴**을 도출하는 것을 목표로 합니다.
