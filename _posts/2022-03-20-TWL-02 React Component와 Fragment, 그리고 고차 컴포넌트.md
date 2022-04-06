---
layout: post
title: TWL-02 React Component와 Fragment, 그리고 고차 컴포넌트
subtitle: React의 컴포넌트 선언 방식에 따른 차이와 Fragment, HOC에 대한 설명
categories: [TWL, React]
tags: [TWL, React, Fragment, Component, HOC, AOP, mixin]
---
# Class vs Function Component

리액트 컴포넌트를 작성하는 방법에는 두 가지 방법이 있다.

첫번째는 리액트의 초기 버전부터 존재했고 가장 많이 채택되어 왔던 컴포넌트 선언 방식이면서, 자바스크립트 ES6의 class 문법을 활용한 클래스형 컴포넌트이며,

두번째는 function 키워드 혹은 arrow function을 활용한 함수 선언 방식과 동일하게 사용할 수 있어 간단하지만 컴포넌트로서의 기능은 부족해 널리 사용되지 못하다가 리액트 16.8버전 이후 리액트에서 Hook을 제공하기 시작하면서 가장 권장되는 컴포넌트 작성 방식으로 등극한 함수형 컴포넌트이다.

그렇다면 두 작성 방식에는 어떠한 차이가 있을지 알아보자.

## 겉으로 보이는 차이 (사용 측면)

1. **선언 방식**
    - 클래스형
        1. class 키워드로 작성한다.
        2. React.Components를 반드시 상속해야 한다.
        3. render() 메서드를 반드시 작성해야 렌더링이 가능하다.
        4. 함수형보다 메모리 자원을 더 사용한다.
        5. 클래스 내부에 임의의 멤버 메소드를 정의할 수 있다.
    - 함수형
        1. 클래스형보다 메모지 자원을 덜 사용한다.
        2. 컴포넌트 선언이 보다 명료하고 편리하다.
        3. function 키워드 또는 화살표 함수 표현을 통해 작성한다.
2. **State**
    - 클래스형
        - 클래스의 객체 멤버 변수 (this.state)로 작성할 수 있으며, React.Components 내부의 setState 함수를 접근하여 state값(각 상태 = this.state의 각 필드)을 변경할 수 있다.
    - 함수형
        - class와 달리 State를 사용하고 변경하기 위해서는 하나의 객체가 아닌 각 상태마다 별도의Hook(useState)를 사용해서 명시적으로 작성해줘야 한다.
3. **Props**
    - 클래스형
        - 전달받은 prop을 가져오기 위해 this를 통해 접근한다.
    - 함수형
        - 함수의 인자로 전달받기 때문에 직접 접근이 가능하다.
4. **이벤트 핸들링**
    - 클래스형
        - 클래스 내부의 멤버 메서드로 핸들러를 작성할 수 있고 `this.핸들러메서드이름` 을 통해 접근할 수 있다.
    - 함수형
        - 컴포넌트 내,외부에 일반적인 함수 선언 방식으로 작성하여 활용할 수 있다.
5. **생명주기 (LifeCycle)**

![life-cycle.png](/assets/2022-03-20-TWL-02%20React%20Component와%20Fragment,%20그리고%20고차%20컴포넌트/01-react-life-cycle.png)

- 클래스형
  - 각 생명주기에 해당하는 메서드를 사용할 수 있다.
- 함수형
  - 생명주기 메서드를 직접 사용할 수 없고, Hook을 사용해야 한다.

## 작성된 코드 이면의 차이 (처리 측면)

React에서 클래스형 컴포넌트와 함수형 컴포넌트는 앞서 살펴본 것처럼 JSX와 함께 컴포넌트 기반으로 코드를 작성하는 과정에서 나타나는 차이뿐만 아니라 우리가 작성한 코드를 프로세싱하는 과정에서도 차이가 발생한다.

먼저, 지난 시간에도 살펴본 resolve 함수를 조금 더 살펴보자.

```jsx
// packages/react-dom/src/server/ReactPartialRenderer.js

function resolve(
  child: mixed,
  context: Object,
  threadID: ThreadID,
): {|
  child: mixed,
  context: Object,
|} {
  while (React.isValidElement(child)) {
    // Safe because we just checked it's an element.
    const element: ReactElement = (child: any);
    const Component = element.type;
    if (__DEV__) {
      pushElementToDebugStack(element);
    }
    if (typeof Component !== 'function') {
      break;
    }
    processChild(element, Component);
  }

  // Extra closure so queue and replace can be captured properly
  function processChild(element, Component) {
    const isClass = shouldConstruct(Component);
    const publicContext = processContext(Component, context, threadID, isClass);

  // ...

    let inst;
    if (isClass) {
      inst = new Component(element.props, publicContext, updater);

   // ...

    } else {

   //...

      const componentIdentity = {};
      prepareToUseHooks(componentIdentity);
      inst = Component(element.props, publicContext, updater);
      inst = finishHooks(Component, element.props, inst, publicContext);

   // ...
    }

    inst.props = element.props;
    inst.context = publicContext;
    inst.updater = updater;

    // ...

    child = inst.render();

  // ...
  }
  return {child, context};
}
```

`resolve` 함수는 입력된 `child` 가 올바른 리액트 엘리먼트일 경우 `processChild` 함수를 통해 자식 요소를 처리한다.

![typeof-class.png](/assets/2022-03-20-TWL-02%20React%20Component와%20Fragment,%20그리고%20고차%20컴포넌트/03-typeof-class.png)

> while문 블록 내의 `if (typeof Component !== 'function')` 을 보고 순간 당황했다면 걱정하지 않아도 괜찮다. 클래스 또한 함수 타입이다.
>

`processChild` 함수 내에서는 `shouldConstruct` 함수를 활용해 컴포넌트를 호출하여 우리가 여기서 궁금한 클래스 컴포넌트인지, 함수형 컴포넌트인지에 대한 결과를 받아온다.

클래스 컴포넌트일 경우, 생성자를 `new` 키워드로 호출하여 인스턴스를 생성한 뒤에 처리를 시작하고, 함수형 컴포넌트일 경우 컴포넌트 내부에서 hook을 사용할 수 있도록 처리해주어야 하고, class가 아니기 때문에 `new` 키워드 없이 인스턴스를 생성하며, 그 이후로 컴포넌트 공통적인 처리 로직이 수행됨을 알 수 있다.

이처럼 클래스형/함수형 여부에 따라 자식 요소를 처리할 때 다른 방식으로 수행되는 것을 위 함수를 통해 알아볼 수 있었다.

```tsx
// packages/react-dom/src/server/ReactPartialRenderer.js

function shouldConstruct(Component) {
  return Component.prototype && Component.prototype.isReactComponent;
}
```

위에서 언급한 `shouldConstruct` 함수는 Component의 프로토타입을 살펴보게 되는데, 둘 다 함수 타입이기 때문에 프로토타입은 클래스형, 함수형 모두 존재하지만 `isReactComponent` 속성은 바로 이어서 살펴볼 수 있듯이, 클래스형 컴포넌트만 가지고 있다.

```jsx
// packages/react/src/React.js

function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

Component.prototype.isReactComponent = {};
```

앞서 언급한 것처럼 우리가 다루는 Javascript class는 함수 타입이고, 리액트 컴포넌트를 생성하기 위해 상속하는 Component 또한 함수이다. `shouldConstruct` 함수에서 살펴본 `isReactComponent` 를 프로토타입에 가지고 있는 것을 확인할 수 있고, 때문에 class형 컴포넌트일 경우 참값을 반환하게 된다.

```jsx
// packages/react-dom/src/server/ReactPartialRenderer.js

class ReactDOMServerRenderer {

 // ...

  read(bytes: number): string | null {
    // ...
    try {
      // ...
      while (out[0].length < bytes) {
        if (this.stack.length === 0) {
          this.exhausted = true;
          freeThreadID(this.threadID);
          break;
        }
        const frame: Frame = this.stack[this.stack.length - 1];
        // ...
        const child = frame.children[frame.childIndex++];

        let outBuffer = '';
        // ...
        try {
          outBuffer += this.render(child, frame.context, frame.domNamespace);
        } catch (err) {
          // ...
        } 
      // ...
      }
      return out[0];
    } finally {
      ReactCurrentDispatcdher.current = prevDispatcher;
      setCurrentPartialRenderer(prevPartialRenderer);
      resetHooksState();
    }
  }

  render(
    child: ReactNode | null,
    context: Object,
    parentNamespace: string,
  ): string {
    if (typeof child === 'string' || typeof child === 'number') {
      // ...
    } else {
      let nextChild;
      ({child: nextChild, context} = resolve(child, context, this.threadID));

     // ...
    }
  }

 // ...
}
```

위에서 살펴본 `resolve` 함수와 같은 파일의 메인인 ReactDOM의 Server Renderer 클래스 중 일부이다.  

핵심적인 내용과 거리가 멀고 읽기 복잡한 코드는 많이 날려서 오히려 내용이 너무 생략된 면이 없잖아 있지만 `read` 함수를 보면, 스택에 담긴 프레임들을 읽으면서 해당 프레임으로부터 자식 노드를 가져와 `render` 함수를 호출하는데 바로 이 함수에서 우리가 살펴본 resolve 함수를 호출하면서 자식 노드를 처리한 뒤, 다음 자식 노드를 반환받아오는 방식으로 render 함수가 수행된다.  

이러한 식으로 React DOM Server에서 우리가 작성한 컴포넌트 기반의 요소들을 해석하고, 처리하고, 렌더링하기 위해 위와 같은 절차로 요소들을 처리하고, 그 과정에서 클래스형 컴포넌트와 함수형 컴포넌트가 분기되어 다른 방식으로 처리되고 있음을 알 수 있었다.

```jsx
// packages/react-server/src/ReactFizzServer.js
function renderElement(
  request: Request,
  task: Task,
  type: any,
  props: Object,
  ref: any,
): void {
  if (typeof type === 'function') {
    if (shouldConstruct(type)) {
      renderClassComponent(request, task, type, props);
      return;
    } else {
      renderIndeterminateComponent(request, task, type, props);
      return;
    }
  }
  if (typeof type === 'string') {
    renderHostElement(request, task, type, props);
    return;
  }

 // ...
}
```

번외로, FizzServer의 구체적인 역할은 잘 모르겠으나, 이 서버에서도 element를 렌더링하기 위해 분기 처리를 하고 있는 모습을 볼 수 있다.

위에서 살펴본 것과 동일하게 타입이 함수(함수형, 클래스형 모두 포함)일 때, `shouldConstruct` 가 참이면 `renderClassComponent` 를 수행하고, 그렇지 않으면 다른 렌더 함수를 호출하는 아주 유사한 방식을 가지고 있는 것을 확인할 수 있다.

# Fragment

Fragment는 같은 depth(or level)에 있는 여러개의 자식 엘리먼트를 한 번에 묶어 반환하기 용이하도록 하기 위한 패턴으로, React 16.2 이후부터 지원되는 기능이다.

```jsx
Some text.
<h2>A heading</h2>
More text.
<h2>Another heading</h2>
Even more text.
```

위와 같은 HTML을 반환해야한다고 가정했을 때, React 16.0 이전에는 아래처럼 `span` 태그나 `div` 태그를 활용해 감싼 뒤 반환하는 것이 유일한 방법이었다.

```jsx
render() {
  return (
    // Extraneous div element :(
    <div>
      Some text.
      <h2>A heading</h2>
      More text.
      <h2>Another heading</h2>
      Even more text.
    </div>
  );
}
```

불필요한 DOM 요소를 추가적으로 반환할 필요 없이 이 문제를 해결하기 위해 React 16.0에서는 DOM 요소 대신에 배열로 묶어 아래와 같이 반환할 수 있도록 지원하게 되었다.

```jsx
render() {
 return [
  "Some text.",
  <h2 key="heading-1">A heading</h2>,
  "More text.",
  <h2 key="heading-2">Another heading</h2>,
  "Even more text."
 ];
}
```

그러나 위와 같이 작성하게 되면 몇 가지 문제가 발생하게 된다.

- 배열 형태이기 때문에 자식 요소들을 반드시 `,` 를 사용해서 구분해야 한다.
- 배열 형태를 반환하기 때문에 자식 요소에 key prop을 반드시 전달해야 한다.
- 텍스트 요소가 문자열의 형태로 반드시 따옴표로 묶여있어야 한다.

이와 같은 문제들을 해결하기 위해 v16.2부터 Fragment를 본격적으로 도입하여 지원하고 있다.

```jsx
const Fragment = React.Fragment;

... 

render() {
  return (
    <Fragment>
      Some text.
      <h2>A heading</h2>
      More text.
      <h2>Another heading</h2>
      Even more text.
    </Fragment>
  );
}
```

React는 E4X (Ecmascript for XML)의 `XMLList() <></>` 생성자 등으로부터 영감을 받아 한 쌍의 빈 태그를 사용함으로써 DOM에 실제 요소를 추가하지 않는다는 아이디어를 통해 Fragment 문법의 간소화된 버전을 제시하고 JSX에서 사용 가능하도록 지원하고 있다.

```jsx
render() {
  return (
    <>
      Some text.
      <h2>A heading</h2>
      More text.
      <h2>Another heading</h2>
      Even more text.
    </>
  );
}
```

다만, 빈 태그는 속성값을 가질 수 없기 때문에 map을 통한 렌더링과 같이 리스트 형태로 DOM 요소를 Fragment로 묶어 반환할 필요가 있을 경우에는 반드시 명시적인 `React.Fragment` 요소를 사용해서 작성해줘야 한다는 점을 양지할 필요가 있다.

## Fragment의 본체

Fragment의 내부 동작 방식을 알아보기 위해 조사해본 결과 대단한 것은 없었고, 그저 React 라이브러리 내에서 React Element와 유사한 다른 타입들과 함께 전역 심볼로서 레지스트리에 등록되어 있는 것을 확인할 수 있었다.

```jsx
export { 
 ...
 REACT_FRAGMENT_TYPE as Fragment,
 ...
}
```

```jsx
// The Symbol used to tag the ReactElement-like types.
export const REACT_ELEMENT_TYPE = Symbol.for('react.element');
export const REACT_PORTAL_TYPE = Symbol.for('react.portal');
export const REACT_FRAGMENT_TYPE = Symbol.for('react.fragment');
...
```

# HOC (Higher Order Comonents)

고차 컴포넌트(HOC)는 컴포넌트 로직을 재사용하기 위한 기술일 뿐, React API의 일부가 아니라 구성적 특성에서 발생하는 일련의 패턴이라고 할 수 있다.

요약하면, 고차 컴포넌트는 컴포넌트를 가져와 새 컴포넌트를 반환하는 **함수**이다.

간단한 예로, 아래와 같이 사용할 수 있다.

```jsx
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```

고차 컴포넌트에 대해 소개하기에 앞서서, 이전에는 횡단 관심사 문제를 제어하기 위해 mixin 사용을 권장했지만 mixin을 사용하는 것이 더 많은 문제를 일으키는 것으로 밝혀졌기 때문에 현재에는 더 이상 사용이 권장되지 않으며, 고차 컴포넌트를 사용하는 방식으로 발전하게 되었다.

## 횡단 관심사에 대하여

이 파트에 대해 조사하기 시작하면서 **횡단 관심사** 라는 단어를 처음 접하게 되었고 이 부분에 대해서 고차 컴포넌트와 함께, 짧게 다루고 넘어가면 좋겠다는 생각이 들었다.

> [객체 지향 소프트웨어 개발](https://ko.wikipedia.org/w/index.php?title=%EA%B0%9D%EC%B2%B4_%EC%A7%80%ED%96%A5_%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4_%EA%B0%9C%EB%B0%9C&action=edit&redlink=1)에서 **횡단 관심사** 또는 **크로스커팅 관심사**(cross-cutting concerns)는 다른 관심사에 영향을 미치는 [프로그램](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%93%A8%ED%84%B0_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%A8)의 [애스펙트](https://ko.wikipedia.org/w/index.php?title=%EC%95%A0%EC%8A%A4%ED%8E%99%ED%8A%B8_(%EC%BB%B4%ED%93%A8%ED%84%B0_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D)&action=edit&redlink=1)이다. 이 관심사들은 디자인과 구현 면에서 시스템의 나머지 부분으로부터 깨끗이 [분해](https://ko.wikipedia.org/wiki/%EB%AA%A8%EB%93%88%EC%84%B1_(%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D))되지 못하는 경우가 있을 수 있으며 분산([코드 중복](https://ko.wikipedia.org/wiki/%EC%BD%94%EB%93%9C_%EC%A4%91%EB%B3%B5))되거나 얽히는(시스템 간의 상당한 의존성 존재) 일이 일어날 수 있다. ([위키백과](https://ko.wikipedia.org/wiki/%ED%9A%A1%EB%8B%A8_%EA%B4%80%EC%8B%AC%EC%82%AC))
>

간단히 말해서 우리가 개발하고자 하는 하나의 프로그램의 목적과 방향성을 고려할 때, 핵심이 되는 기능이나 역할들을 하는 모듈이나 로직들이 있을 텐데, 이를 **핵심 관심사**라고 하며, 이 핵심 관심사들의 로직에서 공통적으로, 혹은 반복적으로 호출되거나 쓰임이 있는 코드들의 역할을 두고 횡단 관심사라고 정리할 수 있을 것 같다.

![cross-cutting.png](/assets/2022-03-20-TWL-02%20React%20Component와%20Fragment,%20그리고%20고차%20컴포넌트/02-cross-cutting-concerns.png)

예를 들어, 은행 애플리케이션을 개발한다고 했을 때, 계좌이체, 입출금, 이자계산 등과 같이 애플리케이션의 본질적인 목적과 역할에 맞게 제 기능을 하기 위한 로직들은 핵심 관심사이고, 로깅, 보안 등과 같은 모듈들은 모든 핵심 관심사를 관통해 공통적인 쓰임이 있는 코드로, 이를 횡단 관심사라고 한다.

이러한 내용을 바탕으로, 횡단 관심사 또한 프로그래밍 관점에서 보았을 때 중복되어 재작성된다는 문제가 존재한다. 이러한 문제를 해결하기 위해 이러한 관심사를 기반으로 프로그램을 설계하고 개발하는 [AOP(Aspect Oriented Programming, 관점 지향 프로그래밍)](https://ko.wikipedia.org/wiki/%EA%B4%80%EC%A0%90_%EC%A7%80%ED%96%A5_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D) 기법이 등장하게 되었다고 한다.

따라서, 이 챕터에서 횡단 관심사라는 용어는 고차 컴포넌트의 등장 배경과 목적도 그렇듯, 리액트 컴포넌트를 작성할 때 공통/반복적으로 작성되는 코드들에 대한 재사용 방법을 설명하기 위해 등장했다고 생각된다.

## mixin에 대하여

횡단 관심사와 마찬가지로, 이 챕터에서 언급되지만 익숙치 않은 용어인 믹스인에 대해서 살펴봤다.

이 또한 그 자체로 나름 복잡한 하나의 영역이므로 자세히는 다루지 않고 본 챕터에서 언급을 위해 필요한 정도로만 설명하도록 하겠다.

> 객체 지향 프로그래밍 언어에서 믹스인(또는 믹스인)은 다른 클래스의 부모 클래스일 필요 없이 다른 클래스에서 사용할 메서드를 포함하는 클래스입니다. 다른 클래스가 mixin의 메서드에 액세스하는 방법은 언어에 따라 다릅니다. 믹스인은 때때로 "상속"이 아니라 "포함"되는 것으로 설명됩니다. 믹스인은 코드 재사용을 권장하며 다중 상속으로 인해 발생할 수 있는("다이아몬드 문제") 상속 모호성을 피하거나 언어에서 다중 상속에 대한 지원 부족을 해결하는 데 사용할 수 있습니다. 믹스인은 구현된 메서드가 있는 인터페이스로 볼 수도 있습니다. 이 패턴은 종속성 반전 원칙을 적용하는 예입니다. ([wikipedia](https://en.wikipedia.org/wiki/Mixin) - 번역)
>

이를 간단히 요약하면 다른 클래스를 상속받을 필요 없이 이들 클래스에 구현되어 있는 메서드를 담고 있는 클래스라고 설명할 수 있다.

믹스인은 자바스크립트에서만 존재하는 패턴은 아니고 객체지향 프로그래밍에서 등장하는 개념이다.

자바스크립트는 그 중에서도 단일 상속만을 허용하는 언어인데, 객체에는 단 하나의 `[[Prototype]]` 만 존재할 수 있고, 클래스는 하나의 클래스만 상속받을 수 있다. 이러한 제약 속에서 각기 다른 역할을 하는 두 개 이상의 클래스를 상속받아 혼합된 기능을 개발하고 싶다고 한다면, 이 때 등장하는 것이 믹스인이다.

다시 말해 믹스인은 특정 행동을 실행해주는 메서드를 제공하지만, 단독으로 쓰이지 않고 다른 클래스에 행동을 더해주는 용도로 사용된다고 설명할 수 있다.

**믹스인 예시**

자바스크립트에서 간단하게 믹스인을 구현하려면 여러 메서드가 담긴 객체를 한 개 만드는 것이다. 그렇게 하면 다수의 메서드를 원하는 클래스의 프로토타입에 쉽게 병합할 수 있다.

```jsx
// 믹스인
let sayHiMixin = {
  sayHi() {
    alert(`Hello ${this.name}`);
  },
  sayBye() {
    alert(`Bye ${this.name}`);
  }
};

// 사용법:
class User {
  constructor(name) {
    this.name = name;
  }
}

// 메서드 복사
Object.assign(User.prototype, sayHiMixin);

// 이제 User가 인사를 할 수 있습니다.
new User("Dude").sayHi(); // Hello Dude!
```

만약, `User` 클래스가 다른 클래스를 상속받고 있다고 해도 믹스인에 구현된 추가 메서드도 함께 사용할 수 있다.

```jsx
class User extends Person {
  // ...
}

Object.assign(User.prototype, sayHiMixin);
```

믹스인에 대한 소개는 이러한데, 왜 React에서 mixin을 도입했다가 이를 걷어내고 고차 컴포넌트를 도입하게 되었는지에 대한 자세한 이야기는 [리액트의 공식 블로그](https://ko.reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)에도 작성되어 있기 때문에 해당 내용을 참조하는 것이 작성에는 글을 줄이는 데 도움이 될 것 같고, 간략하게 요약된 내용으로 대표적인 것들만 설명하자면 아래와 같다.

- 왜 믹스인이 해로울까?
  - 리액트가 개발되고 Facebook 내부에서 점차 많이 사용되기 시작할 때, 혼란을 야기한다고 지목되는 코드베이스의 대부분에 믹스인이 사용되고 있었고, 리액트를 만들던 개발자들은 결국 믹스인을 사용하는 것은 끔찍하다는 결론을 지었다.
  - 잘 쓰면 좋고, Facebook 내부에서도 믹스인으로 덕을 본 프로젝트도 있지만 React에서는 불필요하며 문제가 있다는 점은 명백하다고 한다.
- 믹스인은 암시적 의존성을 만든다.
  - 믹스인은 한 컴포넌트 파일에 존재하는 코드 아니기 때문에 종속성이 생기며, 문서화하기 어렵기 때문에 이 컴포넌트를 처음 파악하는 개발자 입장에서는 파편화된 모든 믹스인 코드를 살펴야 하는 문제가 발생한다.
- 믹스인은 이름 충돌을 야기한다.
  - 타 패키지나 타 믹스인, 심지어 믹스인이 적용되는 컴포넌트까지 메서드 명이 충돌될 수 있다.

## HOC의 역할 및 쓰임새

예를 들어, 외부 데이터 소스로부터 데이터를 구독해 댓글 목록을 렌더링하는 `CommentList` 컴포넌트와, 마찬가지로 블로그 포스트를 구독해 포스트 목록을 렌더링하는 `BlogPost` 컴포넌트가 있다고 가정해보자.

위에서 언급한 두가지 컴포넌트는 동일하지 않다. 두 컴포넌트는 다른 데이터소스를 구독하고, 다른 메서드를 호출하며 다른 렌더링 결과를 보여준다. 하지만 대부분의 구현체가 작성된 로직이 동일하다.

큰 규모의 애플리케이션에서 위와 같이 데이터 소스를 구독하면서 상태 변경이 호출되는 동일한 패턴이 여러 컴포넌트에서 반복적으로 발생한다고 했을 경우에 이러한 공통 로직을 추출하여 한 곳에서 정의하고, 이를 각 컴포넌트에서 공유하며 서용할 수 있도록 하는 추상화 과정이 필요하게 된다. 이럴 때 활용할 수 있는 것이 고차 컴포넌트이다.

위의 댓글 목록과 블로그 포스트 목록 컴포넌트의 예를 활용하면, 아래와 같은 고차 컴포넌트를 작성할 수 있다.

```jsx
// 이 함수는 컴포넌트를 매개변수로 받고..
function withSubscription(WrappedComponent, selectData) {
  // ...다른 컴포넌트를 반환하는데...
  return class extends React.Component {
    constructor(props) {
      super(props);
      this.handleChange = this.handleChange.bind(this);
      this.state = {
        data: selectData(DataSource, props)
      };
    }

    componentDidMount() {
      // ... 구독을 담당하고...
      DataSource.addChangeListener(this.handleChange);
    }

    componentWillUnmount() {
      DataSource.removeChangeListener(this.handleChange);
    }

    handleChange() {
      this.setState({
        data: selectData(DataSource, this.props)
      });
    }

    render() {
      // ... 래핑된 컴포넌트를 새로운 데이터로 랜더링 합니다!
      // 컴포넌트에 추가로 props를 내려주는 것에 주목하세요.
      return <WrappedComponent data={this.state.data} {...this.props} />;
    }
  };
}

const ComponentWithSub = withSubscription(SomeComponent, dataSelector)
```

고차 컴포넌트는 입력된 컴포넌트를 수정하지 않으며, 상속을 사용해 동작을 복사하지도 않으면서, 오히려 고차 컴포넌트는 원본 컴포넌트를 컨테이너 컴포넌트로 포장하고, 조합한다. 고차 컴포넌트는 사이드 이펙트가 없는 순수함수이다. 위 함수는 일반 함수로서 동작하기 때문에 목적에 따라 원하는 개수만큼 인수를 추가할 수 있다.

고차 컴포넌트 패턴을 잘 활용하기 위해서는 몇 가지 지켜야 할 작성 규칙(컨벤션)과 유념해야할 주의사항이 있다.

### 컨벤션

- 원본 컴포넌트를 변경하지 말고, 조합(Compose)하라.
- 래핑된 컴포넌트에만 필요하고 HOC와는 관련 없는 Props를 분리하라.
- 조합 가능성을 끌어올려라. (고차 컴포넌트는 여러 방식으로 작성될 수 있기 때문에)
  - 단일 인수 ex) `const NavbarWithRouter = withRouter(Navbar);`
  - 추가 인수 ex) `const CommentWithRelay = Relay.createContainer(Comment, config);`
  - 고차 컴포넌트를 반환하는 고차함수 - 일반적을 많이 사용되는 패턴

      ```jsx
        // React Redux의 `connect`
        const ConnectedComment = connect(commentSelector, commentActions)(CommentList);
      ```

- 디버깅을 위해 디스플레이 네임은 `고차함수명(래핑된 컴포넌트)` 와 같이 작성하라.

### 주의사항

- render 메서드 내에서 고차 컴포넌트를 사용하지 마라.
- 정적 메서드는 반드시 별도로 복사하라.
- ref는 전달되지 않는다.

# References

- [https://born-dev.tistory.com/27](https://born-dev.tistory.com/27)
- [https://ko.reactjs.org/docs/react-api.html](https://ko.reactjs.org/docs/react-api.html)
- [https://ko.reactjs.org/docs/fragments.html#short-syntax](https://ko.reactjs.org/docs/fragments.html#short-syntax)
- [https://ko.reactjs.org/blog/2017/11/28/react-v16.2.0-fragment-support.html](https://ko.reactjs.org/blog/2017/11/28/react-v16.2.0-fragment-support.html)
- [https://choi3950.tistory.com/32](https://choi3950.tistory.com/32)
- [https://ko.javascript.info/mixins](https://ko.javascript.info/mixins)
- [https://ko.reactjs.org/blog/2016/07/13/mixins-considered-harmful.html](https://ko.reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)
- [https://itmining.tistory.com/124](https://itmining.tistory.com/124)
- [https://ui.toast.com/weekly-pick/ko_20160624](https://ui.toast.com/weekly-pick/ko_20160624)
- [https://ko.reactjs.org/docs/higher-order-components.html](https://ko.reactjs.org/docs/higher-order-components.html)
