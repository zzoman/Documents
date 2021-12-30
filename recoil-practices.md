# Recoil Practices

우리 팀은 약 3개월 정도 Recoil을 사용해 왔다.

단순히 전역 상태관리를 대체할 용도로 Recoil을 도입하였지만, 이왕 사용하기 시작했다면 가능한만큼 최대한 Recoil으로 대체하기로 했다. 단순 상태값은 모두 atom으로 대체하였고, 비동기 데이터 요청과 useMemo같은 파생 데이터도 모두 Recoil Selector로 대체하여 대부분의 데이터 관련 로직을 컴포넌트 외부로 이관했다.

그 과정에서 기술적인 문제점도 있었겠지만, 기술적 문제점은 공식 문서나 기존의 React 개발방법을 통해 비교적 쉽게 해결할 수 있었다. 보다 어려웠던 부분은 Recoil 상태(State)의 네이밍이나 기존 코드와의 협응성 등 Recoil 코드를 운용하는 방법이었는데, 몇가지 대표적인 사례를 소개하고자 한다.

## Recoil State 네이밍 컨벤션

처음에는 [공식 가이드](https://recoiljs.org/docs/introduction/getting-started)에 맞추어 용도에 따라 State, Query 접미사를 사용하였다.

```ts
const postsState = atom();
const postsQuery = selector();
```

그러나 코드를 작성하다보면 순수한 Recoil 상태의 갯수는 많지 않고, 대부분은 다른 상태에 의존성을 가지는 파생 상태 (Derived State)가 훨씬 많아지게 된다. 목적에 따라 조금씩 다른 파생 상태의 이름을 짓다보면 그때부터 천하제일 이름짓기 대회가 시작된다.

```ts
const postsByWriterIdState = selector();
const postsOnlyNotReportedState = selector());
```

구체적인 이름 짓기는 어쩔 수 없다고 하더라도 무수히 길어지는 이름을 볼때마다 State, Query 등의 접미사가 조금씩 부담스러워진다. 단순히 접미사를 지워보는 것도 방법일 수 있으나, 그럴 경우 컴포넌트에서 사용시 모듈명과 변수명이 겹치는 문제가 발생한다.

`as` 구문을 사용하여 새로운 이름을 할당하는 방법도 있으나, 이름짓기가 어려워 선택한 길의 결과로 또 다른 이름을 짓고 모습은 달갑지 않다.

```tsx
import { posts as postsState } from "src/states/posts";
const Posts = () => {
  const posts = useRecoilValue(postsState);
};
```

### 객체로 추상화된 모듈 만들기

아마도 많은 사람들이 Recoil 상태값을 목적에 따라 하나의 ES Module로 나누어 사용하고 있을 것이다.  
우리도 예를 들면 `posts`에 관련한 Recoil State 를 하나의 `src/states/posts.ts` 파일로 모듈화하여 사용하고 있었다.

```ts
// src/states/posts.ts
export const postsState = atom();
export const postsByWriterIdState = selector();
export const postsOnlyNotReportedState = selector());
```

여기서 개별로 각각의 상태값을 추출하는 대신, 이름을 붙힌 객체로 추상화된 구조를 제공하는 것으로 문제를 해결했다.

```ts
// src/states/posts.ts
const posts = atom();
const postsByWriterId = selector();
const postsOnlyNotReported = selector());

export const postsState = {
  posts,
  postsByWriterId,
  postsOnlyNotReported,
}
```

`State` 접미사를 유지함으로 Recoil 상태라는 것을 명확히 표시할 수 있고, `postsState`라는 추가된 구조를 통해 이름만으로 용도를 이해하기가 쉬워졌으며, Recoil 상태명의 중복도 방지할 수 있게 되었다. 무엇보다 편리한 장점은, 컴포넌트에서 모듈명과 변수명의 충돌을 구조적으로 완전히 차단할 수 있다는 것이다.

```tsx
import { postsState } from "src/states/posts";
const Posts = () => {
  const posts = useRecoilValue(postsState.posts);
};
```

## Recoil과 Custom Hook의 혼용 문제점

Custom Hook과 Recoil은 호환성이 좋지 않다.

우리 팀은 하나의 데이터 모델에 필요한 유틸 함수들을 하나의 Custom Hook으로 엮어서 제공하고 있었다. 기존에 Context 로 작성된 전역 상태값들을 Recoil로 대체하면서 하나의 Custom Hook이 여러 개의 상태를 의존하게 되었다.

```tsx
const usePosts = () => {
  const userIds = useRecoilValue(usersState.userIds);
  const posts = useRecoilValue(postsState.posts);
  const postsByWriterId = useRecoilValue(postsState.postsByWriterId(userIds));
  const postsOnlyNotReported = useRecoilValue(postsState.postsOnlyNotReported);
  // ...
  return {
    getPosts,
    getReportedPosts,
    getPostsByUser,
  };
};
```

의존하게 되는 Recoil 상태가 비동기요청을 수행하는 경우, Suspense에 의해 해당 컴포넌트 트리가 데이터의 로딩 중에는 Fallback으로 렌더링된다는 것을 유념하자.

하나의 Custom Hook이 여러 개의 Recoil 상태를 의존하는 경우, 그리고 그 Hook이 여러개의 컴포넌트에서 무분별하게 사용되는 경우, 예기치 못한 경우에 웹서비스 전체가 최상위 Suspense에 의해 Fallback으로 렌더링되는 경우가 발생한다.

컴포넌트가 직접 비동기요청을 수행하는 Recoil 상태를 의존할 경우, 비교적 원인을 찾기 수월하고 해당 컴포넌트를 Suspense로 감싸주면 되지만, 해당 Recoil 상태가 Custom Hook에 의해 의존되는 경우, 이를 디버깅하거나 해결하기가 쉽지 않다.

### Custom Hook 작성 컨벤션 정의

보다 쉬운 디버깅과 명확한 이해를 위해서, 우리는 다음과 같은 컨벤션을 정했다.

- 되도록 Custom Hook은 Recoil 상태와 무관한 로직을 처리하는 용도로만 활용한다.
- Recoil 상태 구독이 필요할 경우, 단일 함수를 반환하는 형태의 Custom Hook만을 작성한다.
- 가능하면 Custom Hook이 아닌, 상태 의존을 포함하는 최소 단위의 단일 컴포넌트로 분리하여 재사용한다.

이에 따라 앞서 소개한 예시는 다음과 같이 분리되었다.

```tsx
const useGetPosts = () => {
  const posts = useRecoilValue(postsState.posts);
  // ...
  return getPosts;
};

const useGetPostsByUser = () => {
  const userIds = useRecoilValue(usersState.userIds);
  const postsByWriterId = useRecoilValue(postsState.postsByWriterId);
  // ...
  return getPostsByUser;
};
```

이로서 사용하고자 하는 Custom Hook의 Recoil 상태 의존여부를 쉽게 파악할 수 있게 되었으며, 문제가 발생할 경우 해당 Hook을 사용하는 컴포넌트를 Suspense로 감싸 쉽게 해결할 수 있게 되었다.

## Writable Selector 패턴

[Writable Selector](https://recoiljs.org/docs/api-reference/core/selector/#writeable-selectors)의 `set`은 이해하기 쉽지않다.  
단순하게 설명하자면, Selector의 상태값으로 다수의 다른 Atom을 한번에 변경할 수 있는 기능이다.

["When to use Writable Selectors in RecoilJS"](https://dev.to/yoniweisbrod/when-to-use-writeable-selectors-in-recoiljs-45b2)는 언제 Writable Selector가 유용한지, 어떤 코드 패턴으로 활용할 수 있는지를 잘 소개하고 있지만, TypeScript를 사용하는 환경에서는 설득되기 힘들다.

```tsx
const removeFoodSelector = selector({
  key: "removeFoodSelector",
  set: ({ set, get }, foodToRemove) => {
    const currentOrder = get(order);
    const foodToRemoveIndex = currentOrder.findIndex(
      (val) => val === foodToRemove
    );
    set([
      ...currentOrder.slice(0, foodToRemoveIndex),
      ...currentOrder.slice(foodToRemoveIndex + 1),
    ]);
  },
});
```

해당 문서에서 최종적으로 제안하는 위 형태의 Writable Selector는 아주 유려해 보이지만, TypeScript를 사용할 경우 - 공식 문서와 마찬가지로 - 반드시 해당 Selector에는 `get` 필드가 설정되어야 하기 때문에 setter 로서의 역할 뿐만 아니라 getter 로서의 역할도 마땅히 수행해야한다.

그리고 `set`의 두번째 인자인 `newValue`(이 예제에서는 `foodToRemove`)는 반드시 해당 Selector의 Generic Type과 동일해야 하기 때문에, 이 제약 조건을 다 지키자면 해당 Selector는 무의미한 `get` 필드를 의무적으로 설정해야만 하는, 어딘가 못나고 쓰기 싫은 모양새가 되고 만다.

```tsx
interface Food = {};
type Order = Food[];

export const foodAtom = atom<Food>();

export const removeFoodSelector = selector<Food>({
  key: "removeFoodSelector",
  get: ({ get }) => {
    return get(foodAtom);
  },
  set: ({ set, get }, foodToRemove) => {
    const currentOrder: Order = get(order);
    const foodToRemoveIndex = currentOrder.findIndex(
      (val) => val === foodToRemove
    );
    set([
      ...currentOrder.slice(0, foodToRemoveIndex),
      ...currentOrder.slice(foodToRemoveIndex + 1),
    ]);
  },
});
```

Writable Selector를 패턴으로 활용하기 위해 이런 무의미한 `get` 필드를 설정하는 것은 비생산적일 뿐만 아니라, `footAtom`을 직접 사용하고싶은 경우 둘 중 어떤 상태를 사용해야하는지 헷갈리게 만들기도 한다. 또한 `get` 필드를 가지고 있어 상태로 활용할 수 있음에도 동사로 시작하는 변수명(removeFoodSelector)은 어색하기만 하다.

그러나 여전히 하나의 상태 변경을 통해 여러개의 상태를 업데이트하고자 하는 필요성은 생기기 마련이다.

### 상태의 집합을 반환하는 Selector

이 문제점을 적절한 Writable Selector의 이름과, ES module을 활용하여 개선하고자 했다.

```tsx
const aState = atom<A>();
const bState = atom<B>();
export const cState = atom<C>();
export const dState = atom<D>();

interface Combined {
  a: A;
  b: B;
}

export const combinedState = selector<Combined>({
  key: "combinedState",
  get: ({ get }) => {
    const a = get(aState);
    const b = get(bState);
    return { a, b };
  },
  set: ({ get, set }, newValue) => {
    set(aState, newValue.a);
    set(bState, newValue.b);
    if (newValue.a !== undefined) {
      set(cState, !get(cState));
    }
    if (newValue.b !== undefined) {
      set(dState, !get(dState));
    }
  },
});
```

한번의 변경으로 업데이트하고 싶은 총 네개의 상태, aState/bState/cState/dState가 있다고 가정해 보자.

1. aState와 bState는 변경을 위한 인자로 활용되는 상태이고, 나머지는 이 변경으로 영향을 받는 상태이다.
2. aState와 bState를 인자로 받을 수 있도록 Writable Selector의 `get`이 두 상태값을 함께 반환하는 객체로 설정하고, 이를 위한 Generic Type인 `Combined`를 설정한다.
3. `set` 필드에 적절한 변경을 작성한다.
4. aState와 bState는 export하지 않고, `combinedState`를 export하여 상태를 사용한다.

이렇게 함으로 Writable Selector 스스로도 사용가능한 상태가 될 수 있으며, `set`을 활용하여 다른 상태의 변경도 일으킬 수 있다. 또한 적절히 모듈을 export하여 혼란스럽게 aState/bState/combinedState를 컴포넌트에서 사용하는 일을 방지할 수 있게 되었다.
