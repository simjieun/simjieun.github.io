# React Query: API 중복 호출과 Sentry 로그 과다 적재 문제 분석

> API 오류 발생 시 동일한 API가 여러 번 호출되고, Sentry에 동일 에러 로그가 수십 번 쌓이는 현상을 분석하고 해결한 과정을 기록함.

---

## 문제 1: Sentry에 동일 에러 로그가 수십 번 쌓임

### 문제

API 500 에러 발생 시 Sentry에 동일한 에러가 수십 번 중복 적재되는 현상이 발생함.

`retry: false` 설정이 되어 있어 API 재시도는 없는데도 불구하고, 에러 1건에 Sentry 이벤트가 수십 개씩 쌓이고 있었음.

### 원인

`useMemo` 내부에서 Sentry 로깅(부수효과)을 실행하고 있었기 때문.

```tsx
// ❌ 문제 코드
const errorIds = useMemo(
  () =>
    queries
      .map((query, idx) => {
        if (query.isError) {
          // useMemo 안에서 부수효과 실행 → 리렌더링마다 중복 호출
          loggingException(query.error);
          return ids[idx];
        }
        return null;
      })
      .filter(Boolean),
  [queries, ids],
);
```

`useMemo`는 순수한 값 계산 목적으로 설계됨. 그런데 React Query는 쿼리 상태가 `pending → error`로 전환되는 과정에서 **내부적으로 여러 번 상태를 업데이트**하며 리렌더링을 발생시킴.

리렌더링마다 `queries` 배열은 새로운 참조를 반환하기 때문에 `useMemo`의 의존성이 변경된 것으로 간주되어 재실행됨. 결과적으로 API는 1번만 호출되어도 `loggingException`이 N번 호출됨.

```
API 호출 1번
  → React Query 상태 업데이트 (pending → error, 내부 여러 단계)
    → 리렌더링 발생
      → queries 참조 변경
        → useMemo 재실행
          → loggingException 호출 ← 수십 번 반복
```

### 해결 방안

부수효과는 `useMemo`가 아닌 `useQueries`의 `onError` 콜백에서 처리함.

React Query **v4**에서 `onError` 콜백은 쿼리가 에러 상태로 전환될 때 **정확히 1번만** 호출됨.

> **React Query v5:** 쿼리 옵션의 `onError` 콜백이 제거됨. v5 환경에서는 아래 문제 3의 `QueryCache.onError` 방식을 바로 적용해야 함.

```tsx
// ✅ 해결 코드
const queries = useQueries({
  queries: ids.map((id) => ({
    queryKey: ['data', id],
    queryFn: () => fetchData(id),
    // onError: 에러 전환 시 1번만 실행됨 (React Query가 보장)
    onError: (error: unknown) => {
      if (axios.isAxiosError(error)) {
        loggingException(error);
      }
    },
  })),
});

// useMemo는 순수 계산만 담당
const errorIds = useMemo(
  () => queries.map((q, idx) => (q.isError ? ids[idx] : null)).filter(Boolean),
  [queries, ids],
);
```

---

## 문제 2: API가 여러 번 호출됨

### 문제

단일 리소스에 대한 API임에도 페이지 진입 시 동일한 API가 **4번** 호출되는 현상이 발생함.

특히 Chrome에서 API를 차단한 상태로 페이지에 진입하면 재현이 확실함.

### 구조 파악

문제가 된 커스텀 훅(`useDataQuery`)은 여러 컴포넌트와 하위 훅에서 공통으로 사용되고 있었음.

```
PageContainer (최초 렌더링)
├── useDataQuery()                ← 직접 호출 (observer 1)
├── usePriceHook()
│   └── useDataQuery()            ← (observer 2)
├── useErrorHandleHook()
│   └── useDataQuery()            ← (observer 3)
├── useDefaultSettingHook()
│   └── useDataQuery()            ← (observer 4)
└── useTicketListQuery()
    └── useDataQuery()            ← (observer 5)
```

또한 하위 컴포넌트들은 `dynamic()` (Next.js 동적 임포트)로 비동기 로드되며, 각각 `useDataQuery()`를 사용하고 있었음.

```tsx
// 동적 임포트로 마운트 타이밍이 다른 컴포넌트들
const ComponentA = dynamic(() => import('./ComponentA')); // useDataQuery 사용
const ComponentB = dynamic(() => import('./ComponentB')); // useDataQuery 사용
const ComponentC = dynamic(() => import('./ComponentC')); // useDataQuery 사용
```

### 원인

#### 원인 1: 에러 상태를 캐시로 인식하지 못하는 `enabled` 조건

```tsx
// ❌ 문제 코드: getQueryData()는 성공 데이터만 반환
const cachedStates = useMemo(
  () =>
    new Map(
      ids.map((id) => [
        id,
        !!queryClient.getQueryData(['data', id]), // 에러 상태 → undefined → false
      ]),
    ),
  [ids, queryClient],
);

const queries = useQueries({
  queries: ids.map((id) => ({
    queryKey: ['data', id],
    queryFn: () => fetchData(id),
    // 에러 상태에서도 isCache=false → enabled=true
    enabled: ids.length > 0 && !cachedStates.get(id),
  })),
});
```

`queryClient.getQueryData()`는 **성공 데이터만** 반환함. 에러 상태에서는 `undefined`를 반환하기 때문에 `isCache = false`, `enabled = true`가 됨.

#### 원인 2: React Query `willFetchOnMount` 동작

React Query는 새 observer(새 `useQueries` 인스턴스)가 구독할 때 아래 조건을 만족하면 re-fetch를 트리거함.

```
willFetchOnMount = true 조건:
  - enabled: true
  - stale 상태 (staleTime 기본값 0, 에러 상태는 dataUpdatedAt=0이라 항상 stale)
  - dataUpdateCount === 0 (성공한 데이터가 없음)
  - isFetching === false (현재 요청 중이 아님)
```

에러 상태의 쿼리는 위 조건을 모두 만족함. Chrome API 차단 환경에서는 실패가 즉시 발생하기 때문에:

```
t=0  PageContainer 마운트 (5개 observer 동시 구독)
     → React Query deduplicate → API 호출 #1 → 즉시 실패
     → isFetching = false

t=1  ComponentA (dynamic) 마운트 → 새 observer 구독
     → willFetchOnMount: enabled=true, stale, dataUpdateCount=0, isFetching=false → true
     → API 호출 #2 → 즉시 실패

t=2  ComponentB 마운트 → API 호출 #3 → 실패

t=3  ComponentC 마운트 → API 호출 #4 → 실패

결과: API 4번 호출
```

> **핵심:** `retry: false`는 같은 observer의 자동 재시도만 막음. **새 observer 마운트 시 re-fetch는 막지 못함.**

### 해결 방안

`cachedStates`에서 `getQueryState()`로 에러 상태도 체크하여, 에러 발생 이후 새 observer가 마운트될 때 `enabled: false`가 되도록 함.

```tsx
// ✅ 해결 코드
const cachedStates = useMemo(
  () =>
    new Map(
      ids.map((id) => {
        const queryKey = ['data', id];
        const hasData = !!queryClient.getQueryData(queryKey);
        // getQueryState()로 에러 상태도 체크
        const isError = queryClient.getQueryState(queryKey)?.status === 'error';
        return [id, hasData || isError]; // 에러 상태 = 캐시된 것으로 처리
      }),
    ),
  [ids, queryClient],
);

const queries = useQueries({
  queries: ids.map((id) => ({
    queryKey: ['data', id],
    queryFn: () => fetchData(id),
    // 에러 상태이면 enabled=false → 새 observer가 마운트되어도 re-fetch 안 함
    enabled: ids.length > 0 && !cachedStates.get(id),
  })),
});
```

**동작 흐름:**

```
t=0  PageContainer 마운트 → 에러 없음 → enabled=true → API 호출 #1 → 실패
     → queryClient에 error 상태 저장

t=1  ComponentA (dynamic) 마운트
     → getQueryState().status === 'error' → isCache=true → enabled=false
     → re-fetch 안 함 ✅

t=2  ComponentB 마운트 → 동일 → re-fetch 안 함 ✅
t=3  ComponentC 마운트 → 동일 → re-fetch 안 함 ✅

결과: API 1번만 호출
```

---

## 문제 3: onError 콜백도 여전히 여러 번 호출됨 (5번)

### 문제

문제 1을 `onError` 콜백으로 해결했지만, 여전히 Sentry 로그가 **5번** 찍히는 현상이 남았음.

### 원인

`onError` 콜백은 **observer마다 독립적으로 실행**됨.

React Query는 네트워크 요청은 1번으로 deduplicate하지만, 해당 쿼리를 구독 중인 모든 observer의 `onError`가 각각 호출됨.

```
API 호출 1번 → 에러 발생
  → observer 1의 onError 실행 → Sentry 로그 (1)
  → observer 2의 onError 실행 → Sentry 로그 (2)
  → observer 3의 onError 실행 → Sentry 로그 (3)
  → observer 4의 onError 실행 → Sentry 로그 (4)
  → observer 5의 onError 실행 → Sentry 로그 (5)
```

PageContainer 최초 렌더링 시 동시에 구독하는 5개 observer 모두에서 `onError`가 실행됨.

### 해결 방안

`QueryCache.onError`를 사용함. 이는 observer 수에 관계없이 **쿼리당 1번만** 호출됨.

`meta` 옵션으로 어떤 쿼리를 Sentry에 로깅할지 제어함.

```tsx
// _app.tsx 또는 QueryClient 초기화 파일
import { QueryCache, QueryClient } from '@tanstack/react-query';

const queryClient = new QueryClient({
  queryCache: new QueryCache({
    // observer 수와 관계없이 쿼리당 1번만 실행
    onError: (error, query) => {
      if (!query.meta?.logSentry) return; // meta로 대상 쿼리 필터링
      if (axios.isAxiosError(error)) {
        loggingException(error);
      }
    },
  }),
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: false,
      retry: false,
    },
  },
});
```

```tsx
// useDataQuery.ts - onError 제거, meta 추가
const queries = useQueries({
  queries: ids.map((id) => ({
    queryKey: ['data', id],
    queryFn: () => fetchData(id),
    enabled: ids.length > 0 && !cachedStates.get(id),
    meta: { logSentry: true }, // QueryCache.onError에서 필터링용
  })),
});
```

---

## React Query v5 대응

React Query v5에서는 쿼리 옵션의 `onError`, `onSuccess`, `onSettled` 콜백이 **모두 제거**됨.

### 변경 사항 요약

| | v4 | v5 |
|---|---|---|
| 쿼리 옵션 `onError` | ✅ 지원 | ❌ 제거 |
| `QueryCache.onError` | ✅ 지원 | ✅ 지원 |
| `useEffect`로 에러 감지 | ✅ 가능 | ✅ 가능 |
| `throwOnError` + ErrorBoundary | ✅ 가능 | ✅ 가능 |

### 방법 1: QueryCache.onError (권장)

문제 3의 해결책과 동일하며, v5 공식 권장 방식임. 별도 변경 없이 그대로 사용 가능함.

```tsx
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      if (!query.meta?.logSentry) return;
      if (axios.isAxiosError(error)) {
        loggingException(error);
      }
    },
  }),
});
```

### 방법 2: useEffect로 에러 감지

특정 컴포넌트에서만 에러 처리가 필요한 경우.

```tsx
const queries = useQueries({ queries: [...] });

useEffect(() => {
  queries.forEach((query) => {
    if (query.isError && axios.isAxiosError(query.error)) {
      loggingException(query.error);
    }
  });
}, [queries]);
```

단, `queries` 배열의 참조가 변경될 때마다 실행되므로 에러 상태가 안정된 이후에도 리렌더링이 발생하면 반복 실행될 수 있음. 중복 방지를 위해 별도 처리가 필요함.

### 방법 3: throwOnError + ErrorBoundary

에러를 ErrorBoundary로 전파하여 처리하는 방식. 에러 로깅을 React 렌더링 트리와 함께 관리하고 싶을 때 적합함.

```tsx
const queries = useQueries({
  queries: ids.map((id) => ({
    queryKey: ['data', id],
    queryFn: () => fetchData(id),
    throwOnError: true, // v4: useErrorBoundary
  })),
});
```

```tsx
<ErrorBoundary
  fallback={<ErrorFallback />}
  onError={(error) => loggingException(error)}
>
  <Component />
</ErrorBoundary>
```

### 결론

이 글에서 다룬 문제들의 v5 최종 해결책은 **문제 3의 `QueryCache.onError`와 동일**함. v4에서 중간 단계로 사용했던 쿼리 옵션의 `onError` 콜백만 v5에서 제거된 것으로, 최종 해결책은 버전에 관계없이 동일하게 적용됨.

---

## 전체 요약

| 문제 | 원인 | 해결 |
|---|---|---|
| Sentry 로그 수십 번 | `useMemo` 안에서 부수효과(로깅) 실행 → 리렌더링마다 중복 호출 | `onError` 콜백으로 이동 |
| API 4번 호출 | 에러 상태를 캐시로 인식 못함 → dynamic 컴포넌트 마운트마다 re-fetch | `getQueryState()`로 에러 상태도 `enabled: false` 처리 |
| Sentry 로그 5번 | `onError`는 observer마다 실행 | `QueryCache.onError`로 쿼리당 1번 보장 |

## 교훈

1. **`useMemo`는 순수 계산 전용** — 부수효과(로깅, API 호출, DOM 조작 등)는 `useEffect` 또는 이벤트 핸들러에서 처리해야 함.

2. **`retry: false`는 완전한 재호출 방지가 아님** — 같은 observer의 자동 재시도만 막음. 새 observer 마운트 시 re-fetch는 `enabled` 조건으로 제어해야 함.

3. **`getQueryData()`와 `getQueryState()`의 차이** — `getQueryData()`는 성공 데이터만 반환함. 에러 상태 여부는 `getQueryState()?.status`로 확인해야 함.

4. **`onError` vs `QueryCache.onError`** — 동일 쿼리를 여러 컴포넌트에서 구독하는 구조라면, observer 단위인 `onError`보다 쿼리 단위인 `QueryCache.onError`가 적합함.
