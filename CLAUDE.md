# 주요 3화음(I·IV·V) 연습 프로그램 — 작업 주의사항

HTML 건반으로 주요 3화음을 연습하는 프로그램. 대상 환경에 iOS Safari와 모바일 터치를 포함한다.

표시 설명
- **[검증됨]**: `00 UTILS`의 노하우 파일에 있는 내용. 예전 작업에서 겪고 해결한 것.
- **[화음 보완]**: 노하우 파일은 단음 기준이라, 3화음에 맞춰 새로 추가한 내용. 실기기 검증 전.

---

## 1. 오디오 시작 (iOS Web Audio)

**[검증됨]** 출처: `iOS_WebAudio_참고.md`

- 페이지 로드 시 `new AudioContext()`를 만들지 않는다. `let actx = null`로 선언하고 사용자 제스처 안에서 생성한다. 로드 시 만들면 iOS에서 suspended로 잠긴다.
- `ensureAudio()`로 처음 쓸 때 생성하고, `play()`와 `setVolume()` 같은 오디오 함수 안에서도 방어적으로 호출한다.
- `unlockAudio()`는 건반·버튼 `pointerdown` 핸들러의 **첫 줄**에서 호출한다. `ensureAudio()` → suspended면 `resume()` → 1샘플 무음 버퍼 재생 순서다.
- `visibilitychange`에서 복귀할 때 suspended면 `resume()`한다. iOS는 백그라운드에서 오디오를 중단한다.
- async 사슬을 끊지 않는다. `play`, `onNote`, 흰 건반·검은 건반 `pointerdown` 핸들러를 모두 `async`로 두고 `await`로 호출한다. `await`가 빠지면 resume 전에 재생을 시도한다.
- 워밍업 오실레이터를 `pointerdown`과 함께 실행하지 않는다. 워밍업 중에 `pointerup`이 오면 소리가 멈추지 않아 "첫 음이 계속 이어지는" 버그가 난다.
- 첫 음이 약하거나 살짝 늦는 현상은 iOS 특성상 완전히 없앨 수 없다. 무음 버퍼로 최소화만 가능하다.

**[화음 보완]**

- Android Chrome 등은 터치의 `pointerdown`을 오디오 허용 제스처로 치지 않고 `pointerup`/`touchend`부터 인정한다. `unlockAudio()`를 `pointerup`과 `touchend`에서도 한 번 더 호출해 첫 터치에 소리가 안 나는 경우를 막는다.
- `await resume()` 동안 손가락을 떼는 경우가 있다. `await` 뒤에 아직 눌려 있는지 확인하고, 이미 뗐다면 재생하지 않는다. 누르는 동안 지속되는 음색에서 멈추지 않는 음을 막기 위해서다.

## 2. 3화음 재생

**[검증됨]** 출처: `audio_utils(첫음부터 소리 잘 나게).py` (Web Audio로 옮겨 쓸 음색 값만)

- 배음 구성은 (배음, 진폭, 감쇠 tau초) = (1, 1.00, 1.60), (2, 0.45, 1.00), (3, 0.20, 0.55), (4, 0.06, 0.30), (5, 0.02, 0.15)이다. 진폭 합으로 나눠 정규화한다.
- 어택 8ms를 넣어 시작 잡음을 막고, 끝은 페이드아웃으로 처리한다.

**[화음 보완]**

- 세 음을 `await`로 하나씩 재생하지 않는다. `resume`을 한 번 끝낸 뒤 같은 시작 시각(`actx.currentTime + 약간`)으로 세 음을 예약해 동시에 울린다.
- 세 음이 합쳐져도 클리핑되지 않도록 음별 게인을 낮춘다(단음 대비 약 1/3부터 시작해 귀로 조정). 필요하면 `DynamicsCompressor`를 마스터에 둔다.
- 음이 끝나면 오실레이터를 `stop()`하고 노드를 끊어, 반복 연습 중에 노드가 쌓이지 않게 한다.

## 3. 건반 터치와 멀티터치

**[검증됨]** 출처: `piano_scrollbar_reference.md`, `iOS_WebAudio_참고.md`

- `.key.white`, `.key.black`은 `touch-action: none`, 건반 영역은 `touch-action: pan-x`, `user-select: none`으로 둔다.
- `touchmove`를 무조건 `preventDefault()`하지 않는다. 스크롤바까지 막혀서 버벅인다. 스크롤바 영역(하단 24~28px)의 터치는 `clientY`로 구분해 `return`하고, 건반 영역만 막는다.
- `preventDefault()`를 쓰는 리스너는 `{ passive: false }`로 등록한다. `pointermove`는 `{ passive: true }`로 등록한다.
- 누르는 동안 지속되는 음색은 `pointerup`, `pointercancel`에서 정지하고, `window`에도 같은 안전망을 건다.

**[화음 보완]**

- 3화음은 손가락 세 개로 동시에 누른다. 눌린 음을 `Map<pointerId, 음>`으로 관리한다. 참고 코드처럼 `pointerup` 하나에 전체 정지를 걸면 한 손가락만 떼도 화음 전체가 끊긴다. 안전망도 해당 `pointerId`만 정지시킨다.
- 참고 코드의 `e.touches[0]`은 첫 손가락만 본다. 스크롤바 판별은 `e.touches` 전체를 검사하거나 `e.targetTouches`로 처리한다.
- **글리산도(건반을 누른 채 긁는 연주)는 필수 기능이다. 삭제하거나 끄지 않는다.** 09번 폴더 `Scale_Master_v2.7.html`의 `pointermove` 방식(`elementFromPoint`로 현재 건반을 찾아, 건반이 바뀌면 이전 음을 멈추고 새 음을 재생)을 그대로 쓴다. 멀티터치와 함께 쓰도록 `Map<pointerId, 음>`을 손가락별로 갱신하고, 다른 손가락이 누르고 있는 음은 멈추지 않는다.

## 4. 가로 스크롤 (건반이 화면보다 넓을 때만)

**[검증됨]** 출처: `piano_scrollbar_reference.md`

- 건반 컨테이너는 `position: relative; display: flex; min-width: max-content`로 둔다.
- 스크롤바 높이는 20~24px, thumb `min-width`는 40~60px로 손가락으로 잡기 쉽게 한다. Firefox용 `scrollbar-width`, `scrollbar-color`도 함께 넣는다.
- 건반 영역에 `padding-bottom: 8px`로 스크롤바 공간을 확보한다.

**참고 코드에서 그대로 쓰면 안 되는 부분**

- 참고 파일의 `.piano-key-area`는 `overflow: hidden`인데 주석은 "가로 스크롤 활성화"다. `hidden`이면 스크롤바가 나타나지 않고 손으로 스크롤할 수 없으므로 `overflow-x: auto`로 쓴다.
- iOS Safari(13 이후)는 `::-webkit-scrollbar` 꾸밈을 적용하지 않는 것으로 알려져 있다. iOS에서는 스크롤바를 잡고 끄는 방식에 기대지 말고, 건반을 화면 폭에 맞추거나 음역 이동 버튼을 두는 방안을 우선 검토한다.

## 5. 이번 작업에 쓰지 않는 것

- `window_utils(...).py`: tkinter 창 크기 조절 전용이다.
- `capture_utils(...).py`: tkinter/Windows GDI 화면 캡처 전용이다.
- `audio_utils(...).py`의 winmm 예열(무음 10회 동기 재생 등): Windows 데스크톱 Python 전용이다. 브라우저로 옮기지 않는다. 옮기면 1절의 워밍업 버그와 같은 문제가 생긴다.

## 6. 완료 전 확인

- iOS Safari 실기기: 첫 터치에 소리가 나는지, 백그라운드에서 돌아온 뒤에도 소리가 나는지.
- 세 손가락 동시 누름: 세 음이 함께 울리는지, 한 손가락만 떼면 그 음만 멈추는지, 다 뗐을 때 남는 음이 없는지.
- 글리산도: 마우스와 손가락으로 건반을 긁을 때 지나가는 음이 차례로 나는지, 뗀 뒤 남는 음이 없는지.
- 3화음 연속 재생: 찌그러짐(클리핑)이 없는지.
- 가로 스크롤이 있다면: 건반을 누를 때 페이지가 밀리지 않는지, 스크롤은 부드러운지.
