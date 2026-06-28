---
title: 컴포넌트 단위 테스트 작성하기
date: 2026-06-28 19:00:00 +0900
tags: [Unit Test, Frontend, Design System]
---

디자인 시스템 컴포넌트는 회사 내부 여러 제품에 공통으로 들어가기 때문에, 동작에 대한 확실한 검증이 필요하다. 검증하지 않은 컴포넌트는 아래와 같은 문제를 일으킬 수 있다:

- 개발자가 넘긴 props가 정확히 반영되지 않아 의도한 UI가 나오지 않는 경우
- 컴포넌트가 예상한 대로 동작하지 않는 경우(예: disabled 버튼인데 클릭이 가능)
- 접근성을 준수하지 않은 경우

그래서 디자인 시스템 컴포넌트는 사전에 충분한 테스트를 설계한 후 개발하는 것이 제품의 안정성과 신뢰성을 높이는 데 큰 도움이 된다고 생각한다. 이 글에서는 단위 테스트가 **무엇을 검증해야 하는지**, 그리고 좋은 테스트 코드의 원칙을 실제 코드와 함께 정리한다.

# 단위 테스트로 무엇을 테스트하나
---
단위 테스트는 특정 모듈(단위)이 의도한 대로 동작하는지 검증하는 테스트다. 컴포넌트 하나, 함수 하나가 그 단위가 될 수 있다.

여기서 가장 중요한 원칙은 무엇을 검증 대상으로 삼느냐다.
- ❌ 구현 세부사항: 내부 상태(state), 내부 함수(handleClick)가 호출됐는가
- ✅ 사용자 행위와 결과: 버튼을 눌렀을 때 화면에 기대한 결과가 나타나는가

구현 세부사항을 검증하면, 내부 로직을 변경하거나 리팩토링할 때 테스트가 깨진다. 예를 들어 상태 관리를 `useState`에서 다른 라이브러리로 교체하면, 사용자가 보는 동작은 똑같지만 테스트는 실패한다. 테스트가 내부 구현에 묶여 있기 때문이다.

반면 사용자 행위를 검증하면, 내부 로직을 리팩토링해도 테스트는 그대로 통과한다. 테스트가 외부 명세를 고정하고, 그 아래에서 구현은 자유롭게 바뀔 수 있는 구조가 된다. 이렇게 해야 테스트가 리팩토링의 안전망이 된다.

# 좋은 테스트 코드의 원칙
## 사용자 행위를 검증한다
앞서 말한 핵심을 다시 정리하면, 테스트는 밖으로 드러난 인터페이스(사용자 행위와 그에 따른 결과)를 검증해야 한다. 내부 함수가 호출되었는지와 같은 구현이 아니라, 사용자 입장에서 무엇이 달라졌는지를 본다.

## it 설명은 명세처럼 쓴다
it에 적는 설명은 이 컴포넌트가 무엇을 보장하는지를 드러내야 한다. 구현이 아니라 역할을, 추상적인 표현이 아니라 "어떤 행위 -> 어떤 결과"를 구체적으로 적어야 한다.

```javascript
// 나쁜 예: 내부 구현을 설명
it('버튼 클릭 시 useStore의 count가 set으로 갱신된다', ...);

// 나쁜 예: 무엇이 '정상'인지 알 수 없음
it('버튼이 정상적으로 동작한다', ...);

// 좋은 예: 행위와 결과가 드러남
it('좋아요 버튼 클릭 시 좋아요 수가 1 증가한다', ...);
```

위 테스트 코드의 마지막 설명만 봐도 이 컴포넌트가 어떤 동작을 통해 어떤 결과가 나오는지 알 수 있다. 즉 기대하는 역할이 드러나는 것이다.

## Given-When-Then
테스트는 주어진 환경에서 특정 행위를 했을 때 기대한 결과가 나오는지 검증하는 구조로 쓴다.

- **Given**: 테스트를 위해 세팅하는 환경 (렌더링 등)
- **When**: 검증 대상이 되는 행위 (주로 사용자 상호작용)
- **Then**: 예상 결과 검증

```javascript
it('버튼을 클릭하면 화면에 바나나가 표시된다', async () => {
  // Given: 버튼이 있는 컴포넌트를 렌더링한다
  render(<FruitButton />);

  // When: 사용자가 버튼을 클릭한다
  await user.click(screen.getByRole('button', { name: '과일 보기' }));

  // Then: 화면에 '바나나'가 나타난다
  expect(screen.getByText('바나나')).toBeInTheDocument();
});
```

## 실제 코드로 보기: Base UI Button의 `disabled` 테스트
Base UI의 [`Button.test.tsx`](https://github.com/mui/base-ui/blob/master/packages/react/src/button/Button.test.tsx)에서 `disabled` 동작을 검증하는 테스트다.

```tsx
it('native button: uses the disabled attribute and is not focusable', async () => {
  const handleClick = vi.fn();

  // Given: disabled 상태의 버튼을 렌더링한다
  const { user } = await render(<Button disabled onClick={handleClick} />);

  const button = screen.getByRole('button');

  // Then(상태): 비활성화 속성이 적용되고 포커스 대상에서 제외된다
  expect(button).toHaveAttribute('disabled');

  // When: 사용자가 Tab으로 포커스를 시도한다
  await user.keyboard('[Tab]');
  // Then: 포커스되지 않는다
  expect(button).not.toHaveFocus();

  // When: 사용자가 클릭/키보드로 활성화를 시도한다
  await user.click(button);
  await user.keyboard('[Enter]');
  // Then: 핸들러가 호출되지 않는다
  expect(handleClick.mock.calls.length).toBe(0);
});
```
> 실제 테스트는 `onMouseDown`, `onPointerDown`, `onKeyDown` 등 더 많은 핸들러를 함께 검증한다. 여기서는 흐름을 보기 위해 `onClick` 중심으로 줄였다.

이 한 케이스에 앞서 본 원칙이 모두 들어 있다.

1. **사용자 행위를 검증한다.** Base UI Button 내부의 커스텀 훅 `useButton`에서 어떤 변수가 바뀌는지 들여다보지 않는다. 사용자가 실제로 하는 행위 — `Tab`으로 포커스 시도, 클릭, 키 입력 — 를 흉내 내고 그 결과만 본다.
2. **`it` 설명이 명세다.** `'native button: uses the disabled attribute and is not focusable'`만 읽어도 이 테스트가 무엇을 보장하는지 알 수 있다. "어떤 대상(native button)이 / 어떤 동작(disabled 속성 사용, 포커스 불가)을 하는가"가 제목에 그대로 드러난다.
3. **Given-When-Then 구조다.** disabled 버튼을 렌더링하고(Given), 포커스·클릭·키 입력을 시도한 뒤(When), 포커스되지 않고 핸들러가 불리지 않음을 확인한다(Then).

내부 구현을 한 번도 들여다보지 않으면서, "비활성화된 버튼은 포커스되지 않고 상호작용을 받지 않는다"는 **명세를 그대로 검증**한다. `useButton`의 구현을 어떻게 리팩토링하든, 이 동작만 유지되면 테스트는 통과한다.

## 정리

- 단위 테스트는 **구현이 아니라 사용자 행위와 결과**를 검증한다. 그래야 리팩토링해도 깨지지 않는다.
- `it` 설명은 **명세처럼** — 구현이 아니라 역할을, 구체적인 "행위 → 결과"로 쓴다.
- **Given-When-Then** 구조로 시나리오를 표현한다.
- Base UI `Button`의 `disabled` 테스트는 이 원칙들을 충실히 따른다. 내부 훅을 들여다보지 않고, 사용자 행위와 관찰 가능한 결과만으로 명세를 검증한다.
