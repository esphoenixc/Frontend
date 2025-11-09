![Rendering Process Image - web.dev](./images/crp.png)

## Critical Rendering Path

### 1. DOM 생성 과정

브라우저가 HTML을 받으면 바이트 → 문자 → 토큰 → 노드 → DOM 순서로 변환

- **토큰화**: HTML 마크업을 개별 토큰으로 분해함 (`<html>`, `<div>`, `</div>` 등)
- **노드 생성**: 각 토큰을 노드 객체로 변환
- **트리 구축**: 노드들 간의 부모-자식 관계를 파악해 트리 구조를 만듦

이 과정은 **점진적**으로 진행 전체 HTML을 다 받지 않아도 파싱이 시작

### 2. CSSOM 생성 과정

CSS도 DOM과 유사한 과정을 거침

- 바이트 → 문자 → 토큰 → 노드 → CSSOM
- 하지만 CSS는 **렌더 블로킹 리소스**임. 전체 CSS를 파싱해야만 다음 단계로 넘어갈 수 있음
- 이유: CSS 규칙은 서로 덮어쓸 수 있어서(cascading) 일부만 적용하면 잘못된 스타일이 표시될 수 있기 때문

### 3. JavaScript의 영향

```html
<script src="app.js"></script>
```

JS를 만나면

- **파서 블로킹**: HTML 파싱이 멈춤
- **다운로드 대기**: JS 파일을 다운로드할 때까지 기다림
- **실행**: JS가 실행되는 동안 파싱이 중단됨
- **CSSOM 대기**: JS는 CSSOM에 접근할 수 있어서, CSSOM 생성이 완료될 때까지 JS 실행도 대기함

이래서 `<script defer>` 또는 `<script async>` 사용이 중요함.

### 4. 렌더 트리 구축

DOM + CSSOM → 렌더 트리

- **포함되는 것**: 화면에 보이는 요소들만
- **제외되는 것**:
  - `display: none` 요소
  - `<head>`, `<script>`, `<meta>` 같은 메타데이터
  - `visibility: hidden`은 포함됨 (공간은 차지하니까)

각 노드에는 계산된 스타일 정보가 포함됨.

### 5. 렌더링 4단계

#### **Style (스타일 계산)**

- 각 요소의 최종 스타일을 계산함
- CSS 우선순위(specificity) 적용
- 상속 가능한 속성들을 처리함

#### **Layout (레이아웃 / Reflow)**

- 각 요소의 **정확한 위치와 크기**를 계산함
- 뷰포트 기준으로 %나 vh 같은 상대값을 절대값으로 변환함
- Box Model 계산 (content, padding, border, margin)
- 가장 비용이 큰 작업 중 하나임

#### **Paint (페인트 / Repaint)**

- 레이어별로 픽셀을 채워넣음
- 텍스트, 색상, 이미지, 그림자 등을 그림
- 여러 페인트 레이어로 나뉘어 처리됨

#### **Composite (합성)**

- 각 레이어를 올바른 순서로 합성함
- GPU를 활용해 최종 화면을 생성함
- `transform`, `opacity` 같은 속성은 이 단계에서만 처리 --> 성능 👍

### 6. 주요 성능 Metrics

**FCP (First Contentful Paint)**

- 첫 번째 콘텐츠(텍스트, 이미지 등)가 화면에 그려지는 시점

**LCP (Largest Contentful Paint)**

- 가장 큰 콘텐츠 요소가 보이는 시점

**TTI (Time to Interactive)**

- 페이지가 완전히 상호작용 가능해지는 시점

### 7. 최적화 전략

**CSS 최적화**

```html
<!-- 중요한 CSS는 인라인으로 -->
<style>
  /* Critical CSS */
</style>

<!-- 나머지는 비동기로 -->
<link
  rel="preload"
  href="styles.css"
  as="style"
  onload="this.rel='stylesheet'"
/>
```

**JS 최적화**

```html
<!-- defer: HTML 파싱 후 실행 -->
<script src="app.js" defer></script>

<!-- async: 다운로드 후 즉시 실행 -->
<script src="analytics.js" async></script>
```

- `async`: 다운로드 병렬 + 도착 즉시 실행 → DOM 파싱 중단 가능 순서 보장 안 됨

- `defer`: 다운로드 병렬 + DOM 파싱 끝난 뒤 실행 → 실행 순서 보장 렌더 지연 최소화에 유리

**리소스**

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="dns-prefetch" href="https://api.example.com" />
<link rel="prefetch" href="next-page.js" />
```

**Code Splitting**

- 필요한 코드만 먼저 로드
- Dynamic import 활용
- 번들 사이즈 최소화

### 8. Reflow와 Repaint 트리거

**Reflow를 일으키는 것들** (비용이 큼):

- 요소의 크기나 위치 변경
- 폰트 크기 변경
- `offsetWidth`, `clientHeight` 같은 속성 읽기
- 윈도우 리사이즈

**Repaint만 일으키는 것들** (상대적으로 가벼움):

- 색상 변경
- `visibility` 변경
- 배경 이미지 변경

**Composite만 일으키는 것들** (가장 가벼움):

- `transform`, `opacity`
- `will-change` 사용

## References

https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Critical_rendering_path

https://web.dev/learn/performance/understanding-the-critical-path
