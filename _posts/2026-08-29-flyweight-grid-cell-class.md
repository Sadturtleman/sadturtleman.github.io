---
title: "칸 하나에 문자열 하나씩 만들고 있었다 — flyweight로 판면 클래스를 18개로 줄이기"
date: 2026-08-29 17:30:00 +0900
categories: [개발, 성능]
tags: [performance, react, flyweight, memoization, 디자인패턴]
mermaid: true
---

점자 점역 데스크톱 앱의 결과 화면은 **모눈종이**다. 32칸 × 26줄짜리 판면에 점자를 한 칸씩 찍는다. 그리고 그 칸 하나하나가 DOM 요소 하나다.

여기에 드래그로 구간을 고르는 기능을 넣고 나서 뒤늦게 셈을 해봤다. 마우스를 끄는 동안 **초당 수십 번**, 한 면 920개의 칸이 전부 다시 계산된다. 칸마다 클래스 문자열을 새로 만들면서.

그래서 flyweight를 넣었다. 결과부터 말하면 **CPU는 17배 빨라졌고, 만들어지는 문자열은 920개에서 0개가 됐다.** 다만 "메모리가 줄었다"는 말은 반만 맞았다. 이 글은 그 셈과, 내가 처음에 잘못 잡았던 방향까지의 기록이다. 코드는 [Semojum/FE](https://github.com/Semojum/FE) 에 있다.

## 칸 하나가 객체 하나

판면 한 면을 그리면 이만큼이 생긴다.

| 단위 | 요소 수 |
|---|---|
| 한 줄 | 줄 div 1 + 줄번호 1 + 칸 32 = **34** |
| 한 면 | 눈금줄 34 + 26줄 × 34 = **약 920** |
| 화면에 붙는 면 | 보이는 면 + 앞뒤 2면씩 ≈ 5~6면 |
| 합계 | **약 5,000~5,500** |

총량 자체는 이미 묶여 있었다. 가상 스크롤로 화면 근처 면만 붙이고, 나머지는 [`content-visibility: auto`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility) 로 브라우저가 배치·그리기를 건너뛴다. 문서가 300면이어도 DOM은 안 늘어난다.

문제는 총량이 아니라 **다시 그리는 단위**였다. 메모가 면 단위라서, 커서가 한 칸 움직이면 그 면의 920개가 통째로 다시 조정된다. 그리고 그때마다 칸마다 이런 일을 했다.

```ts
// 칸 하나의 클래스 — 렌더마다 배열을 만들고 join한다
const cellCls = (row, selected, isCaret, highlighted, dimmed, find, inSelection) =>
  [
    'flex h-[19px] w-[19px] shrink-0 items-center justify-center …',
    'border-[#e4ebf5]',
    isCaret ? 'bg-[#5b8ce6] text-white'
      : inSelection ? 'bg-[#5b8ce6]/35'
      : find === 'active' ? 'bg-[#f9c74f] text-gray-900'
      : /* … 아홉 갈래 … */ 'bg-white',
    dimmed && !isCaret ? 'text-[#c8ccd4]' : '',
  ].join(' ');
```

## 공유할 수 있는 것과 없는 것

flyweight는 **여러 객체가 공유할 수 있는 속성(intrinsic)을 밖으로 빼서 하나만 두는** 패턴이다. 여기에 대보면 이렇게 갈린다.

- **DOM 노드는 공유할 수 없다.** 칸마다 화면에 자리가 있어야 하니 트리에 하나씩 존재해야 한다.
- **클래스 문자열은 공유할 수 있다.** 위 코드의 분기를 세어 보면 아홉 가지(커서 · 고른 구간 · 찾기 활성 · 찾기 · 커서 행 · 고정줄 · 검토필요 · 대조강조 · 기본)이고, 거기에 흐리게 on/off — **서로 다른 결과가 18개뿐**이다.

920칸을 그리는데 만들어지는 문자열의 종류가 18개라면, 매번 만들 이유가 없다.

```ts
// 아홉 가지 모양. 고르는 순서가 곧 우선순위다.
const TONE = { caret: 0, sel: 1, findActive: 2, /* … */ plain: 8 } as const;

// [모양][흐리게] 두 겹 배열. 열여덟 개를 미리 이어 붙여 둔다.
const CELL_CLASS: string[][] = TONE_CLASS.map((tone) => [
  `${CELL_BASE} ${tone}`,
  `${CELL_BASE} ${tone} ${CELL_DIM}`,
]);

export const cellClassFor = (row, selected, isCaret, highlighted, dimmed, find, inSelection) => {
  const tone = isCaret ? TONE.caret : inSelection ? TONE.sel : /* … */ TONE.plain;
  return CELL_CLASS[tone][dimmed && !isCaret ? 1 : 0];
};
```

덤이 하나 붙는다. 같은 **참조**가 나가므로 React가 `className`을 비교할 때 값 비교가 아니라 참조 비교로 끝난다.

## 재 봤다

추정만으로 넘어가기 싫어서 세 가지를 놓고 실제로 쟀다. 한 면(920칸)을 2,000프레임 그리는 셈, 즉 184만 번 호출이다.

```js
const CELLS = 920, FRAMES = 2000;
const run = (fn, label) => {
  let sink = 0;
  const t0 = performance.now();
  for (let f = 0; f < FRAMES; f++)
    for (let i = 0; i < CELLS; i++) sink += fn(i % 9, (i & 7) === 0).length;
  console.log(label, (performance.now() - t0).toFixed(1), 'ms');
};
```

| 방식 | 시간 |
|---|---|
| 1) 배열 만들고 `join` | **84~87 ms** |
| 2) flyweight + 문자열 키 | **32~33 ms** |
| 3) flyweight + 배열 색인 | **4.9~5.1 ms** |

3번은 1번의 **약 17배**다. 세 번 돌려 편차는 ±3% 안이었다.

## 조회 키를 문자열로 만들면 flyweight가 아니다

처음에 나는 2번으로 짰다. 상태를 문자열 키로 만들어 객체에서 꺼내는 방식이다.

```ts
return CELL_CLASS[`${tone}|${dimmed && !isCaret ? 1 : 0}`];   // ← 여기
```

미리 만들어 뒀으니 됐다고 생각했는데, **칸마다 `` `${tone}|${dim}` `` 라는 문자열을 새로 만들고 있었다.** 920칸이면 920개다. 만들지 않으려고 넣은 패턴인데 만드는 대상만 바뀐 셈이다.

두 겹 배열 색인으로 바꾸니 할당이 0이 되고, 같은 flyweight인데도 **6.3배**가 더 빨라졌다.

> flyweight를 넣었는지는 "미리 만들어 뒀는가"가 아니라 **"조회하는 동안 아무것도 만들지 않는가"** 로 확인해야 한다.
{: .prompt-tip }

## 그런데 줄어든 건 메모리가 아니었다

제목만 보면 메모리 이야기지만, 실제로 재보니 결이 달랐다. 만들어진 문자열을 전부 붙들고 있게 해서 힙을 쟀다.

```js
const N = 2_000_000;
const hold = new Array(N);              // 배열 자체는 측정 전에 미리 잡아 둔다
const before = process.memoryUsage().heapUsed;
for (let i = 0; i < N; i++) hold[i] = fn(i % 9, (i & 7) === 0);
const after = process.memoryUsage().heapUsed;
```

| 방식 | 200만 개를 붙들었을 때 힙 증가 | 서로 다른 값 |
|---|---|---|
| 배열 + `join` | **+86 MB** | 18개 |
| flyweight | **+0.0 MB** | 18개 |

값의 종류는 양쪽 다 18개인데 한쪽만 86MB를 쓴다. **같은 값을 담은 서로 다른 객체 200만 개**였기 때문이다.

다만 실제 앱에서 86MB가 상주하는 건 아니다. 렌더가 끝나면 그 920개는 곧바로 쓰레기가 된다. 그러니 정확히 말하면 flyweight가 줄인 것은 **상주 메모리가 아니라 GC가 치워야 할 쓰레기의 양**이고, 체감으로 돌아오는 것은 그 쓰레기를 만들고 치우는 데 쓰던 **CPU 시간**이다. 드래그처럼 초당 수십 번 도는 경로에서는 이게 프레임을 먹는다.

## flyweight만으로는 부족했다

여기서 멈추면 절반이다. 문자열을 안 만들어도 **React는 여전히 920개를 훑는다.** 할당은 없어졌지만 diff 비용은 그대로다.

그래서 [`React.memo`](https://react.dev/reference/react/memo) 를 한 겹 더 내려 **줄 단위**로 메모했다. 커서·선택·호버는 한두 줄에만 영향을 주므로, 다시 그리는 양이 920에서 34가 된다.

```mermaid
flowchart LR
    A["커서 한 칸 이동"] --> B{"메모 단위"}
    B -->|"면 단위 (전)"| C["920개 요소 diff<br/>+ 920개 문자열 생성"]
    B -->|"줄 단위 (후)"| D["34개 요소 diff"]
    D --> E["공유 테이블 18개<br/>조회만, 생성 0"]
    C --> F["프레임 지연"]
    E --> G["같은 참조 → 참조 비교"]
```

메모를 세우려면 넘기는 값이 렌더마다 같아야 한다. 여기서 세 군데를 손봤다.

- 고른 구간을 **객체 대신 숫자 두 개**(`selFrom`, `selTo`)로 넘긴다. 객체면 렌더마다 새 참조라 메모가 늘 깨진다.
- 마스크가 없는 줄에 `?? []` 를 쓰고 있었다. **렌더마다 새 빈 배열**이라 이것 하나로 메모 전체가 무력해진다. 공유 상수로 바꿨다.
- 블록 테두리에 필요한 앞뒤 줄 비교는 상위에서 하고 **불리언만** 넘긴다. 줄 조각이 전체 줄 배열을 들면 그 참조가 바뀔 때마다 전부 다시 그려진다.

세 번째가 특히 그렇다. `?? []` 같은 한 줄이 `React.memo` 를 통째로 무력화한다. 메모를 붙일 때는 **넘기는 값 중에 렌더마다 새로 만들어지는 게 없는지**부터 훑어야 한다.

## 정리 — 그리고 아직 못 잰 것

- 공유할 수 있는 것과 없는 것을 먼저 가른다. **DOM 노드는 못 나눠 쓰고, 그 부수 객체는 나눠 쓸 수 있다.**
- 경우의 수를 세어 본다. 18가지뿐이면 920번 만들 이유가 없다.
- **조회 자체가 할당이면 flyweight가 아니다.** 문자열 키 대신 색인을 쓰면 6.3배 차이가 났다.
- flyweight는 **할당을 없애고**, 메모는 **훑는 양을 줄인다.** 둘은 대체재가 아니라 짝이다.

마지막으로 솔직하게 남길 것. 위 숫자는 전부 **함수 단위 벤치마크**다. 실제 앱에서 드래그할 때 프레임이 몇 ms 줄었는지는 아직 못 쟀다. 판면의 진짜 다음 단계는 줄 하나를 텍스트 한 덩어리로 그리고 강조를 오버레이 사각형으로 얹어 **DOM을 30분의 1로** 줄이는 것인데, 그건 칸 단위 배경색을 다시 설계해야 해서 실측 결과를 보고 정할 생각이다.

숫자 없이 "빨라졌을 것"이라고 적는 것보다는, 잰 것과 못 잰 것을 갈라 적는 편이 나중의 나에게 쓸모 있다.

## 참고 자료

- [React — `memo`](https://react.dev/reference/react/memo) : props가 그대로면 다시 그리지 않게 하는 방법과, 메모가 깨지는 흔한 이유
- [React — `useMemo`](https://react.dev/reference/react/useMemo) : 렌더마다 새로 만들어지는 값을 붙잡아 두기
- [MDN — `content-visibility`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility) : 화면 밖 요소의 배치·그리기를 건너뛰기
- [MDN — `Map`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Map) : 키 순서가 보장되는 자료구조(상한을 둔 캐시에 쓸 때)
- 이번 작업 커밋: [`c6fd4cd`](https://github.com/Semojum/FE/commit/c6fd4cde0c343eda22f15a07510d8e1b39f03050) — `BrailleGrid.tsx`
