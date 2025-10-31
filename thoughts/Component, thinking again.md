### FEConf 컴포넌트, 다시 생각하기

#### 컴포넌트의 변화는 대부분 Remote Data Schema(API 데이터 구조)의 변화에 따라 발생함

### 문제 정의: 컴포넌트의 4가지 의존성

컴포넌트는 케이크가 밀가루, 설탕, 계란에 의존하듯 여러 요소에 의존 발표에서는 이 의존성을 4가지로 분류

- Style: CSS, SCSS 파일, CSS-in-JS 등

- Custom Logic: useXXX와 같은 커스텀 훅

- Global State: 로그인 상태, 테마 등 전역 상태

- Remote Data Schema: API 서버로부터 받는 데이터의 구조 (가장 중요)

이 중 가장 문제를 많이 일으키는 것은 Remote Data Schema 의존성

'숨은 의존성'의 문제 (Props Drilling)
특정 하위 컴포넌트(예: ArticleItem)에 새로운 데이터(예: authorName)를 추가해야 하는 시나리오를 가정

일반적인 방식(Props Drilling)에서는 다음과 같은 파일들을 모두 수정해야함

- 최상위 페이지 컴포넌트: useEffect 등에서 API 호출 로직을 수정해 authorName을 추가로 받아옴

- 중간 리스트 컴포넌트: ArticleList의 props 타입을 수정하고, 받아온 authorName을 ArticleItem에 전달

최하위 아이템 컴포넌트: ArticleItem의 props 타입을 수정하고, authorName을 렌더링

단지 ArticleItem을 수정하기 위해 상위 모든 컴포넌트가 변경되어야 했음. 이는 ArticleItem이 상위 컴포넌트 및 루트 컴포넌트의 데이터 페칭 로직에 **강하게 결합(Tightly Coupled)**되어 있음을 의미 이것이 바로 '숨은 의존성'입니다

### 해결책: 컴포넌트 설계 4가지 방식

#### 함께 두기 (Co-location)

"비슷한 관심사는 가까운 곳에 둔다."

Style/Logic: 컴포넌트에서만 쓰이는 스타일(CSS-in-JS)과 로직(커스텀 훅)은 컴포넌트 파일 내부에 함께 두는 것이 좋음, 수정이 필요할 때 한 파일만 보면 되기 때문

Data: 가장 중요한 데이터 의존성도 컴포넌트와 함께 둠 즉, 데이터를 Props로 받지 말고, 컴포넌트가 직접 데이터를 가져오게 하기

#### 데이터를 ID 기반으로 정리하기 (Abstraction by Normalization)

핵심은 데이터의 구조를 컴포넌트의 필요에 맞게 추상화
컴포넌트가 다루는 원격 데이터(Remote Data)는 대개 서버에서 받아온 **중첩된 구조**를 가지고 있음.

- 문제점: 중첩된(Nested) 데이터 구조

```javascript
{
  "id": "123",
  "author": {
    "id": "1",
    "name": "Paul"
  },
  "title": "My awesome blog post",
  "comments": [
    {
      "id": "324",
      "commenter": {
        "id": "2",
        "name": "Nicole"
      }
    }
  ]
}
```

- 이 구조는 `article` 객체 안에 `author` 객체가, 또 그 안에 `comments` 배열과 `commenter` 객체가 중첩되어 있습니다. 만약 ID가 '2'인 'Nicole' 유저 정보에 접근하려면 `data.comments[0].commenter`처럼 깊숙이 파고들어가야 하며, ID만으로 특정 데이터를 한 번에 찾기가 매우 어려움

```javascript
{
  "result": "123",
  "entities": {
    "articles": {
      "123": {
        "id": "123",
        "author": "1",
        "title": "My awesome blog post",
        "comments": [ "324" ]
      }
    },
    "users": {
      "1": { "id": "1", "name": "Paul" },
      "2": { "id": "2", "name": "Nicole" }
    },
    "comments": {
      "324": { "id": "324", "commenter": "2" }
    }
  }
}
```

- 모든 데이터가 entities라는 객체 안으로 펼쳐짐 (flatten)
- 데이터는 모델 유형(articles, users, comments)별로 분류
- 각 데이터는 자신의 고유 ID를 키(Key)로 하여 객체(Map) 형태로 저장

### 정규화의 이점: Global ID 와 GOI

- Global ID (전역 ID): 이렇게 데이터가 정규화되면, 'Article' 모델의 ID 1과 'User' 모델의 ID 1이 충돌할 수 있음. 이를 해결하기 위해 모델명과 ID를 조합한 "Global ID" (또는 Node ID)라는 구조 도입 (예: Article:123, User:1)
- 범용 훅 (useNode): `useArticle`이라는 특정 모델에 의존하는 훅 대신 , `useNode(globalId)`라는 범용 훅을 사용할 수 있게 됨. 컴포넌트는 자신이 어떤 모델의 데이터를 사용할지 외부에서 주입받는 게 아니라, 스스로 결정하여 내부에 함께 둘 수(Co-locate) 있음
- GOI (Global Object Identification) : 이 전역 ID를 기반으로 서버에 GET /node/C#3처럼 특정 객체를 요청하는 API
  - 자동 데이터 페칭 (Cache Miss): 컴포넌트가 `useNode`로 특정 필드를 요청했는데 로컬 데이터 저장소에 해당 필드가 없으면(Cache Miss) , 저장소가 GOI API를 통해 서버에서 자동으로 데이터를 가져옴
  - 컴포넌트 단위 Refetch: useNode 훅이 refetch 함수를 반환 --> 상위 컴포넌트와 관계없이 특정 컴포넌트가 원하는 시점에 자신의 데이터만 새로고침 가능

### 이름 짓기 (Naming)

원칙의 핵심은 **"의존한다면 그대로 드러내기 (Make Explicit)"**

- 컴포넌트의 Props 이름을 leftImageThumbnailUrl, title, title2처럼 모호하게 지으면 , 이 컴포넌트가 어떤 데이터 모델(Schema)에 의존하는지 알기 매우 어려움
- userImageThumbnailUrl, userName, userNickname처럼 의존하는 모델을 이름에서 드러나게 함
  - 더 좋은 방법: Props의 구조 자체를 실제 데이터 모델의 관계(Graph)와 동일하게 만들어서 user 객체 안에 image 객체가 있는 구조를 그대로 Props 타입으로 정의
- 가장 좋은 예시: 의존하는 모델에 따라 컴포넌트를 분리 (UserCard, Avatar). 그리고 Global ID를 사용해, `UserCard`는 `userId만` 받고 `Avatar`는 `imageId`만 받도록 하여 컴포넌트 간의 의존성을 느슨하게(loosely coupled) 처리

### 재사용하기

원칙의 핵심은 "개발할 때 편리하기 위한" 재사용이 아니라 "변경할 때 편리하기 위한" 재사용을 해야함

- 컴포넌트의 변화(스타일, 로직, 전역 상태 등) 중 가장 큰 변화를 이끄는 것은 **"리모트 데이터 스키마"**
- UI가 비슷하다는 이유만으로 다른 모델의 컴포넌트를 재사용하려는 함정에 빠지기 쉬움 (재사용성 함정)
- 만약 User와 Page를 보여주기 위해 하나의 Profile 컴포넌트를 재사용했다면 "페이지의 프로필 사진만 둥근 사각형으로 바꿔주세요" 라는 요구사항이 왔을 때 문제 발생

  - Profile 컴포넌트를 수정하면 Page뿐만 아니라 User의 프로필까지 함께 바뀌는 부작용(Side effect)이 발생 이를 막으려면 `if (props.type === 'page')` 같은 분기 처리가 들어가 컴포넌트가 복잡해짐

- 해결책: 모델 기준으로 컴포넌트 분리하기:

  - User와 Page는 겉모습은 비슷해도 서로 다른 데이터 모델

  - 데이터 모델이 다르면, 미래의 **"변화의 방향성"**도 다름

  - 따라서 이 둘은 재사용하는 것이 아니라 별도의 컴포넌트로 분리해야 함

- So...

  - 같은 모델을 의존하는 컴포넌트: 재사용

  - 다른 모델을 의존하는 컴포넌트: 분리

## 정리

1. Keep Locality (함께 두기): 비슷한 관심사라면 가까운 곳에 둔다

2. Abstraction by Normalization (ID 기반 정리): 데이터를 ID 기반으로 정리한다

3. Make Explicit (드러내기): (데이터) 의존성이 있다면 그대로 드러낸다

4. Seperating Components by Model (모델 기준 분리): 모델 기준으로 컴포넌트를 분리한다

이러한 기술적 원칙들보다 중요한 것은 "관점"을 갖는것 -- 스스로의 코드를 다른 관점에서 바라보고 질문을 던지는 태도

### References

- [A3 Component, thinking again](https://youtu.be/HYgKBvLr49c?si=vrc_aojW_8vu8tbP)
