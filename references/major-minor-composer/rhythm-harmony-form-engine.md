# 리듬·화음·형식 엔진 역분석

이 문서는 화면에서 관찰되는 결과를 바탕으로 **내부 음악 엔진이 어떤 논리로 구현되어야 하는지**를 재구성한다. 실제 소스가 아니며, 관찰된 사실과 권장 구현을 구분한다.

---

# 1. 리듬: 16칸보다 onset + sustain가 핵심

## Observed

완성 화면은 한 마디를 네 박으로 표시하며:

- 음 이름: 새 음 시작
- – : 앞 음이 이어짐
- · : 쉼

이라고 설명한다.

따라서 한 마디가:

~~~text
라 – – –
~~~

라면 한 번의 A onset이 네 박 동안 지속되는 것으로 해석할 수 있다.

## Inferred

그리드 클릭은 각 박의 pitch value보다 **새 note onset을 추가하는 행동**일 가능성이 높다.

## 장점

1박 해상도만으로도 4분음표, 2분음표, 점2분음표, 온음표 길이에 해당하는 duration을 표현할 수 있다.

## 한계

- 8분음표 없음
- syncopation 제한
- tie를 명시적 기보로 표현하지 않음
- note 뒤의 명시적 쉼 입력 방식 불명

---

# 2. 쉼표 상태는 별도 데이터로 두어야 함

UI에서 빈 칸은 상황에 따라 sustain, rest, 미입력 중 하나일 수 있다.

~~~ts
type BeatState =
  | { kind: "onset", pitch: number }
  | { kind: "sustain" }
  | { kind: "rest" }
  | { kind: "empty" }
~~~

다만 canonical song model에서는 BeatState 배열보다 NoteEvent + RestEvent 구조가 확장성이 높다.

---

# 3. 화음 엔진

## Observed

마디별 I/IV/V 선택.

선택 결과에 따라 chord label, 구성음, 화음 소리, 가락 검사가 바뀐다.

## 최소 판정

~~~ts
const isChordTone =
  chord.pitchClasses.includes(note.pitchClass)
~~~

그러나 이것만으로는 음악적 피드백이 너무 거칠다.

---

# 4. 권장 화음 관계 분류

~~~ts
type Stability =
  | "stable"
  | "passing"
  | "neighbor"
  | "tension"
  | "cadential"
  | "unknown"
~~~

판정 입력:

~~~ts
analyzeNote({
  pitch,
  onset,
  duration,
  metricStrength,
  chord,
  previous,
  next,
  isPhraseEnd
})
~~~

## 단순 규칙

- Stable: 현재 triad 구성음
- Passing: 앞뒤 음 사이를 순차적으로 연결하며 약박
- Neighbor: 같은 중심음에서 한 음 위/아래로 갔다 복귀
- Tension: 화음 밖이지만 다음 음으로 해결 가능
- Cadential: 마지막 구간에서 tonic으로 해결

---

# 5. 추천 pitch 알고리즘

가장 가까운 화음 음만 추천하면 지나치게 단순하다.

권장 score:

~~~text
candidateScore =
  harmonicStability
+ melodicDistance
+ contourContinuity
+ scaleMembership
+ phraseRole
+ resolutionQuality
~~~

다음 요소를 함께 본다.

- 현재 pitch와 너무 멀지 않은가
- 앞/뒤 음과 자연스럽게 연결되는가
- 현재 scale 안인가
- 마지막 마디인가
- tonic으로 해결되는가

---

# 6. V화음 위 F 문제

C major에서 V = G-B-D.

F는 triad에는 없지만 G7 = G-B-D-F의 7음이 될 수 있다.

따라서 시스템은 “화음에 없는 잘못된 음”보다:

> 지금 선택한 3화음에는 없지만 긴장감을 만들 수 있는 음

으로 처리하는 것이 음악적으로 더 타당하다.

초등 5~6학년에서 딸림7화음을 직접 가르치지 않더라도, 내부 판정이 이를 오류로 단정할 필요는 없다.

---

# 7. 종지 엔진

현재 문 닫기 안내는 사실상 다음 규칙을 사용한다.

~~~text
bar 3 harmony = V
bar 4 harmony = I
last melody pitch = tonic
last duration = long
→ strong closure
~~~

이를 점수화하면:

~~~ts
closureScore =
  harmonyResolution(V_to_I)
+ melodyEndsOnTonic
+ finalNoteDuration
+ metricPlacement
~~~

학생에게는 점수를 보여주지 않고:

- 안정적으로 끝남
- 조금 열린 느낌
- 계속 이어질 것 같은 느낌

처럼 청감 중심으로 표현한다.

---

# 8. 마디 역할 엔진

네 마디 역할은 자동 판정 결과가 아니라 **instructional scaffold**다.

~~~ts
const barRoles = [
  "seed",
  "repeat",
  "contrast",
  "closure"
]
~~~

학생이 규칙을 어겨도 입력을 막기보다 설명을 제공해야 한다.

---

# 9. form similarity 엔진

## Observed

동일한 마디 → a  
조금 다른 마디 → a′  
빈 마디 → -

## 권장 feature vector

~~~ts
type BarFingerprint = {
  rhythmPattern: number[]
  pitchSequence: number[]
  intervalSequence: number[]
  contour: ("up"|"down"|"same")[]
  onsetCount: number
}
~~~

## 권장 similarity 구성

- rhythm similarity
- interval/contour similarity
- exact pitch match
- onset count

를 함께 본다.

단순 pitch 일치율 하나만 보면 전조된 동일 패턴을 다른 아이디어로 오판할 수 있다.

---

# 10. a′ 판정 시 transposition 문제

예:

~~~text
C D E
D E F#
~~~

는 exact pitch는 모두 다르지만 interval 구조와 contour가 같다.

따라서 interval sequence와 contour를 함께 비교해야 한다.

---

# 11. melodic contour

완성 화면의 선은 note pitch를 y 좌표로 변환하고 duration을 가로 길이로 표현하는 구조로 볼 수 있다.

~~~ts
y = mapPitchToY(pitch)
~~~

권장 개선:

- 음 시작점 marker
- sustain은 연속선
- rest는 gap
- 마디 경계 세로선
- tap 시 계이름/음이름

---

# 12. audio scheduling

16칸 grid라도 setTimeout보다 AudioContext clock을 기준으로 scheduling해야 한다.

~~~ts
eventStart =
  transportStart +
  beatsToSeconds(event.onsetBeat, bpm)
~~~

화음은 같은 onset에 3개 pitch를 동시에 scheduling한다.

느리게 모드는 note data를 수정하지 않고 tempo만 변경한다.

---

# 13. 한 마디만 / 가락만 / 화음만

동일한 Song State에서 range와 track filter만 바꾼다.

~~~ts
play({
  range: barRange | wholeSong,
  tracks: ["melody"] | ["harmony"] | ["melody","harmony"]
})
~~~

별도 재생 데이터를 만들지 않는 것이 좋다.

---

# 14. inversion 구현 권장

학생용 표현: 위아래 뒤집기

권장 알고리즘:

1. 기준음 선택
2. 각 음의 scale degree 차이 계산
3. 방향 반전
4. 현재 scale 안으로 보정
5. 허용 음역 밖이면 octave correction

~~~ts
invertedDegree =
  pivotDegree - (degree - pivotDegree)
~~~

chromatic pitch보다 scale degree 기준이 초등용 tonal composer에 적합하다.

---

# 15. 끝음 바꾸기 구현 권장

버튼을 눌렀을 때 무작위 pitch보다 bar role에 맞는 후보를 생성한다.

2마디:

- tonic 이외의 stable tone
- 3마디로 연결되는 음
- 직전 음에서 가까운 음

4마디:

- tonic 우선

같은 끝음 변경이라도 마디 역할에 따라 candidate generator를 달리한다.

---

# 16. internal model과 학생용 언어 분리

내부:

~~~text
tonic
dominant
subdominant
non-chord tone
metric strength
interval
~~~

학생 UI:

~~~text
집 같은 음
문 닫기
화음 음
지나가는 음
~~~

두 층을 분리해야 학년이나 난이도에 따라 표현만 바꿀 수 있다.
