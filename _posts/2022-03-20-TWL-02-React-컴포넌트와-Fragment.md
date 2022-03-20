---
layout: post
title: TWL-02 React Fragment 문법
subtitle: React의 Fragment에 대한 설명
categories: [TWL, React]
tags: [TWL, React, Fragment]
---

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

# References

- [https://ko.reactjs.org/docs/react-api.html](https://ko.reactjs.org/docs/react-api.html)
- [https://ko.reactjs.org/docs/fragments.html#short-syntax](https://ko.reactjs.org/docs/fragments.html#short-syntax)
- [https://ko.reactjs.org/blog/2017/11/28/react-v16.2.0-fragment-support.html](https://ko.reactjs.org/blog/2017/11/28/react-v16.2.0-fragment-support.html)