# UI·상호작용·기술 구조 분석

이 문서는 실제 배포 화면에서 확인되는 UI와 동작을 기준으로 **재구현 가능한 구조**를 정리한다. 내부 소스는 확인하지 못했으므로 실제 라이브러리명·함수명·폴더 구조는 추정하지 않는다.

---

## 1. 화면 구조

전체는 4단계 wizard에 가깝다.

```text
[1 장조? 단조?]
      ↓
[2 어느 조로]
      ↓
[3 가락 짓기]
      ↓
[4 완성곡]
```

단계 번호를 항상 노출해 현재 위치를 알 수 있게 한다.

### 설계 효과

- 처음부터 모든 기능을 보여주지 않음
- 사용자의 결정 수를 줄임
- 개념 학습 순서와 UI 진행 순서가 일치
- 학생이 “지금 무엇을 하면 되는지” 쉽게 이해

---

## 2. 핵심 UI 컴포넌트

### A. ModeCard

장조/단조를 선택하는 카드.

필드 예시:

```ts
type ModeCard = {
  mode: "major" | "minor"
  title: string
  description: string
  moodWords: string[]
  audioExample: AudioExample
}
```

행동:

- 느낌 듣기
- 해당 mode로 다음 단계 이동

---

### B. KeyCard

조를 선택하는 카드.

```ts
type KeyCard = {
  id: string
  tonicMidi: number
  tonicName: string
  mode: "major" | "minor"
  keySignature: Accidental[]
  scalePitches: number[]
  solfege: string[]
  relativeKeyId?: string
  description?: string
}
```

표시 정보:

- 조 이름
- 조표
- 음계
- 으뜸음
- 나란한조
- 음계 듣기

---

### C. MeasureGuide

각 마디의 작곡 역할을 보여주는 카드/라벨.

```ts
type MeasureRole =
  | "seed"
  | "repeat"
  | "contrast"
  | "cadence"
```

예:

```ts
const roles = [
  { bar: 1, role: "seed", label: "씨앗 심기" },
  { bar: 2, role: "repeat", label: "거의 그대로" },
  { bar: 3, role: "contrast", label: "달라지기" },
  { bar: 4, role: "cadence", label: "문 닫기" }
]
```

---

### D. ChordSelector

마디마다 1·4·5도 기능화음을 선택.

```ts
type BarHarmony = {
  barIndex: number
  degree: 1 | 4 | 5
  chordPitches: number[]
}
```

학생 화면에는 구성음을 함께 보여준다.

---

### E. PitchTimeGrid

핵심 입력 UI.

```text
        1마디       2마디       3마디       4마디
높은음  □ □ □ □    □ □ □ □    □ □ □ □    □ □ □ □
       ...
낮은음  □ □ □ □    □ □ □ □    □ □ □ □    □ □ □ □
```

관찰된 현재 구조는 HTML button과 table 계열로 구현된 형태다.

### 권장 구현

시각적으로 table처럼 보여도 실제 상태는 2차원 배열이 아니라 **time event 배열**로 관리하는 편이 낫다.

```ts
type NoteEvent = {
  id: string
  bar: number
  beat: number
  pitch: number
  durationBeats: number
}
```

현재처럼 한 박에 음 하나만 허용한다면:

```ts
type SimpleCell = {
  bar: 0 | 1 | 2 | 3
  beat: 0 | 1 | 2 | 3
  pitch: number | null
}
```

으로도 충분하다.

---

## 3. 상태 모델

권장 상위 상태:

```ts
type ComposerState = {
  step: 1 | 2 | 3 | 4
  mode: "major" | "minor" | null
  keyId: string | null
  tempo: number
  accompanimentEnabled: boolean
  slowMode: boolean
  bars: BarState[]
  selectedSongSlot: string | null
  history: ComposerSnapshot[]
}
```

마디 상태:

```ts
type BarState = {
  chordDegree: 1 | 4 | 5
  notes: NoteEvent[]
}
```

---

## 4. Undo 구조

`↩ 되돌리기`가 있으므로 상태 변경 전에 snapshot을 쌓는 구조가 적합하다.

```ts
function commit(nextState) {
  history.push(currentState)
  currentState = nextState
}
```

초등용 도구에서는 정교한 command pattern보다 snapshot 방식이 구현과 안정성 측면에서 유리하다.

### history에 포함할 변경

- 음 배치
- 음 삭제
- 마디 복사
- 음형 뒤집기
- 끝음 변경
- 예시 넣기
- 전체 삭제
- 화음 변경

재생 상태는 history에 넣지 않는다.

---

## 5. 변형 명령

### 5.1 마디 복사

```ts
copyBar(sourceIndex, targetIndex)
```

주의:

- note id는 새로 생성
- bar index는 target으로 변경
- 화음도 같이 복사할지 옵션 분리 권장

---

### 5.2 끝음 변경

```ts
replaceLastNote(barIndex, newPitch)
```

교육적으로는 무작위 변경보다:

- 현재 화음 구성음
- 다음 마디로 자연스럽게 연결되는 음
- 으뜸음

중에서 추천하는 편이 낫다.

---

### 5.3 음형 뒤집기

현재 버튼 이름은 “위아래 뒤집기”.

실제 구현은 두 종류가 가능하다.

#### 단순 visual flip

현재 사용 음역의 중앙선을 기준으로 상하 반전.

#### 음악적 inversion

기준음을 정하고 각 interval 방향을 반대로 변환.

교육적으로는 두 번째가 더 의미 있지만 초등 UI에서는 결과가 음역 밖으로 나갈 수 있어 보정 로직이 필요하다.

권장:

```ts
invertMelody(notes, pivotPitch, allowedScale)
```

변환 후 가장 가까운 scale pitch로 quantize한다.

---

## 6. 음계 모델

음악 데이터는 화면에 표시되는 한글 계이름 문자열보다 MIDI pitch number 또는 pitch class를 기준으로 저장하는 편이 안전하다.

예:

```ts
type Pitch = {
  midi: number
  letter: "C" | "D" | "E" | "F" | "G" | "A" | "B"
  accidental: -1 | 0 | 1
  octave: number
}
```

이유:

- 재생
- 오선보 배치
- 조표
- 옥타브
- 전조
- 화음 판정

을 하나의 데이터로 처리할 수 있다.

---

## 7. 화음 판정 엔진

최소 구현:

```ts
isChordTone(notePitchClass, chordPitchClasses)
```

그러나 교육용으로는 이것만으로 부족하다.

### 권장 분류기

```ts
classifyMelodicNote({
  note,
  previousNote,
  nextNote,
  chord,
  metricPosition,
  isFinalNote
})
```

출력:

```ts
type HarmonyRelation =
  | "chord-tone"
  | "passing-tone"
  | "neighbor-tone"
  | "tension"
  | "cadential-tone"
```

학생 UI에서는 명칭을 단순화:

- 화음 음
- 지나가는 음
- 다시 들어볼 음
- 끝내기 좋은 음

---

## 8. 오디오 엔진

관찰만으로 실제 라이브러리는 확인할 수 없다.

재구현 시 선택지는:

### Web Audio API 직접 사용

장점:

- 의존성 없음
- 세밀한 제어

단점:

- scheduling, envelope, polyphony를 직접 구현해야 함

### Tone.js 계열

장점:

- Transport
- tempo
- synth
- polyphony
- scheduling

구현이 쉬움

초등용 작곡 도구라면 Tone.js 같은 추상화 레이어가 실용적이다.

---

## 9. 재생 스케줄링

가장 중요한 문제는 UI 타이머가 아니라 **audio clock 기준으로 재생**하는 것이다.

나쁜 방식:

```js
setTimeout(playNote, beatMs)
```

권장:

- AudioContext time
- Transport 기반 scheduling

화면 재생 cursor는 오디오 시간을 따라가야 한다.

---

## 10. 재생 모드

현재 기능을 상태로 정리하면:

```ts
type PlaybackTarget =
  | "scale"
  | "bar"
  | "melody"
  | "harmony"
  | "full"
```

별도 엔진을 만드는 대신 동일한 scheduler에 target filter를 적용하는 것이 좋다.

---

## 11. 저장 모델

현재 서비스는 기기 내 저장을 안내한다.

권장 local 구조:

```ts
type SongSlot = {
  id: string
  category: "major" | "minor"
  index: 1 | 2
  song: Song | null
  isPresentationChoice: boolean
  updatedAt: string
}
```

### 저장 계층 권장

작은 JSON이면 localStorage로 충분하다.

하지만 향후:

- 여러 곡
- 썸네일
- 오디오
- 학생 제출물
- 버전 기록

까지 갈 경우 IndexedDB가 적합하다.

---

## 12. 발표곡 선택

`⭐ 이걸로 발표`는 boolean 하나보다 **presentationSongId** 한 개로 관리하는 편이 안전하다.

```ts
type SongShelf = {
  slots: SongSlot[]
  presentationSongId: string | null
}
```

이렇게 해야 두 곡이 동시에 발표곡으로 선택되는 오류를 방지할 수 있다.

---

## 13. 인쇄

완성곡 인쇄 시 반드시 별도의 print CSS를 둔다.

```css
@media print {
  .controls,
  .navigation,
  .audio-buttons {
    display: none;
  }

  .score {
    break-inside: avoid;
  }
}
```

향후 실제 오선보를 넣는다면 인쇄 출력은 SVG 기반이 가장 안정적이다.

---

## 14. QR

QR은 최소 두 종류로 분리한다.

### Entry QR

서비스 자체 URL.

### Song QR

특정 작품을 공유할 수 있는 URL.

서버를 쓰지 않는다면 곡 JSON을 URL fragment에 압축하는 방식도 가능하지만 길이가 커질 수 있다.

수업 서비스에서는 작품 ID 기반 서버 저장이 더 현실적이다.

---

## 15. 모바일/태블릿 UX

초등 수업에서는 태블릿 사용을 전제로 다음을 권장한다.

- pitch cell 최소 터치 영역 44px 이상
- 드래그로 음을 그리는 기능은 선택적으로 제공
- 한 손가락 스크롤과 음 입력 충돌 방지
- 세로 화면에서는 음역보다 가로 시간축이 잘리기 쉬우므로 가로 스크롤 또는 landscape 안내
- 재생 버튼은 입력 그리드와 가까이 배치
- undo는 항상 보이는 위치에 유지

---

## 16. 접근성

button 기반 grid는 접근성 관점에서 장점이 있다.

반드시 추가할 것:

```html
<button
  aria-label="2마디 3박, 미"
  aria-pressed="true">
</button>
```

키보드:

- 방향키: 셀 이동
- Enter/Space: 음 입력
- Delete/Backspace: 제거

색상만으로 화음 관계를 표현하지 않는다.

---

## 17. 실제 오선보 연결

향후 가장 중요한 확장점.

현재:

```text
Pitch Grid → 소리
```

권장:

```text
              ┌→ Web Audio
Pitch Grid → Song Model
              ├→ Harmony Analyzer
              └→ Staff Renderer
```

오선보는 별도 입력기가 아니라 **같은 Song Model의 두 번째 view**여야 한다.

그래야 그리드와 악보가 항상 일치한다.

---

## 18. 권장 컴포넌트 구조

```text
App
├─ ProgressNav
├─ ModeStep
│  └─ ModeCard
├─ KeyStep
│  └─ KeyCard
├─ ComposerStep
│  ├─ MeasureGuide
│  ├─ ChordSelector
│  ├─ PitchTimeGrid
│  ├─ PlaybackControls
│  ├─ TransformationTools
│  └─ HarmonyFeedback
├─ ResultStep
│  ├─ StaffView
│  ├─ MelodyContour
│  ├─ PerformanceGuide
│  └─ PrintView
└─ SongShelf
```

---

## 19. 권장 데이터 흐름

```text
학생 입력
   ↓
Action
   ↓
Song State
   ├─→ Grid View
   ├─→ Staff View
   ├─→ Audio Scheduler
   ├─→ Harmony Analyzer
   └─→ Local/Cloud Storage
```

핵심은 **오디오, 화면, 악보, 분석 결과를 각각 따로 저장하지 않는 것**이다.

항상 하나의 canonical Song State에서 파생해야 한다.

---

## 20. 구현 우선순위

### Phase 1

- major/minor 선택
- key 정의
- 4×4 pitch grid
- 재생
- 마디별 화음
- undo
- 로컬 저장

### Phase 2

- 반복/변형 도구
- 화음 관계 피드백
- 여러 곡 슬롯
- 발표곡 선택
- 인쇄

### Phase 3

- 실제 오선보
- 8분음표/쉼표 등 리듬
- 학생 제출
- 교사 모아보기

### Phase 4

- 화음 밖 음의 문맥적 분석
- 작품 비교
- 음악적 피드백
- 협업 작곡

이 순서가 기능 복잡도와 수업 효과의 균형이 좋다.
