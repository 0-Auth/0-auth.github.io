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

## 작성된 코드 이면의 차이 (해석 측면)

 지난시간에 살펴본 resolve 함수를 자세히 들여다 보자.

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

    let queue = [];
    let replace = false;
    const updater = {
      isMounted: function(publicInstance) {
        return false;
      },
      enqueueForceUpdate: function(publicInstance) {
        if (queue === null) {
          warnNoop(publicInstance, 'forceUpdate');
          return null;
        }
      },
      enqueueReplaceState: function(publicInstance, completeState) {
        replace = true;
        queue = [completeState];
      },
      enqueueSetState: function(publicInstance, currentPartialState) {
        if (queue === null) {
          warnNoop(publicInstance, 'setState');
          return null;
        }
        queue.push(currentPartialState);
      },
    };

    let inst;
    if (isClass) {
      inst = new Component(element.props, publicContext, updater);

      if (typeof Component.getDerivedStateFromProps === 'function') {
        if (__DEV__) {
          if (inst.state === null || inst.state === undefined) {
            const componentName =
              getComponentNameFromType(Component) || 'Unknown';
            if (!didWarnAboutUninitializedState[componentName]) {
              console.error(
                '`%s` uses `getDerivedStateFromProps` but its initial state is ' +
                  '%s. This is not recommended. Instead, define the initial state by ' +
                  'assigning an object to `this.state` in the constructor of `%s`. ' +
                  'This ensures that `getDerivedStateFromProps` arguments have a consistent shape.',
                componentName,
                inst.state === null ? 'null' : 'undefined',
                componentName,
              );
              didWarnAboutUninitializedState[componentName] = true;
            }
          }

          // If new component APIs are defined, "unsafe" lifecycles won't be called.
          // Warn about these lifecycles if they are present.
          // Don't warn about react-lifecycles-compat polyfilled methods though.
          if (
            typeof Component.getDerivedStateFromProps === 'function' ||
            typeof inst.getSnapshotBeforeUpdate === 'function'
          ) {
            let foundWillMountName = null;
            let foundWillReceivePropsName = null;
            let foundWillUpdateName = null;
            if (
              typeof inst.componentWillMount === 'function' &&
              inst.componentWillMount.__suppressDeprecationWarning !== true
            ) {
              foundWillMountName = 'componentWillMount';
            } else if (typeof inst.UNSAFE_componentWillMount === 'function') {
              foundWillMountName = 'UNSAFE_componentWillMount';
            }
            if (
              typeof inst.componentWillReceiveProps === 'function' &&
              inst.componentWillReceiveProps.__suppressDeprecationWarning !==
                true
            ) {
              foundWillReceivePropsName = 'componentWillReceiveProps';
            } else if (
              typeof inst.UNSAFE_componentWillReceiveProps === 'function'
            ) {
              foundWillReceivePropsName = 'UNSAFE_componentWillReceiveProps';
            }
            if (
              typeof inst.componentWillUpdate === 'function' &&
              inst.componentWillUpdate.__suppressDeprecationWarning !== true
            ) {
              foundWillUpdateName = 'componentWillUpdate';
            } else if (typeof inst.UNSAFE_componentWillUpdate === 'function') {
              foundWillUpdateName = 'UNSAFE_componentWillUpdate';
            }
            if (
              foundWillMountName !== null ||
              foundWillReceivePropsName !== null ||
              foundWillUpdateName !== null
            ) {
              const componentName =
                getComponentNameFromType(Component) || 'Component';
              const newApiName =
                typeof Component.getDerivedStateFromProps === 'function'
                  ? 'getDerivedStateFromProps()'
                  : 'getSnapshotBeforeUpdate()';
              if (!didWarnAboutLegacyLifecyclesAndDerivedState[componentName]) {
                didWarnAboutLegacyLifecyclesAndDerivedState[
                  componentName
                ] = true;
                console.error(
                  'Unsafe legacy lifecycles will not be called for components using new component APIs.\n\n' +
                    '%s uses %s but also contains the following legacy lifecycles:%s%s%s\n\n' +
                    'The above lifecycles should be removed. Learn more about this warning here:\n' +
                    'https://reactjs.org/link/unsafe-component-lifecycles',
                  componentName,
                  newApiName,
                  foundWillMountName !== null
                    ? `\n  ${foundWillMountName}`
                    : '',
                  foundWillReceivePropsName !== null
                    ? `\n  ${foundWillReceivePropsName}`
                    : '',
                  foundWillUpdateName !== null
                    ? `\n  ${foundWillUpdateName}`
                    : '',
                );
              }
            }
          }
        }

        const partialState = Component.getDerivedStateFromProps.call(
          null,
          element.props,
          inst.state,
        );

        if (__DEV__) {
          if (partialState === undefined) {
            const componentName =
              getComponentNameFromType(Component) || 'Unknown';
            if (!didWarnAboutUndefinedDerivedState[componentName]) {
              console.error(
                '%s.getDerivedStateFromProps(): A valid state object (or null) must be returned. ' +
                  'You have returned undefined.',
                componentName,
              );
              didWarnAboutUndefinedDerivedState[componentName] = true;
            }
          }
        }

        if (partialState != null) {
          inst.state = assign({}, inst.state, partialState);
        }
      }
    } else {
      if (__DEV__) {
        if (
          Component.prototype &&
          typeof Component.prototype.render === 'function'
        ) {
          const componentName =
            getComponentNameFromType(Component) || 'Unknown';

          if (!didWarnAboutBadClass[componentName]) {
            console.error(
              "The <%s /> component appears to have a render method, but doesn't extend React.Component. " +
                'This is likely to cause errors. Change %s to extend React.Component instead.',
              componentName,
              componentName,
            );
            didWarnAboutBadClass[componentName] = true;
          }
        }
      }
      const componentIdentity = {};
      prepareToUseHooks(componentIdentity);
      inst = Component(element.props, publicContext, updater);
      inst = finishHooks(Component, element.props, inst, publicContext);

      if (__DEV__) {
        // Support for module components is deprecated and is removed behind a flag.
        // Whether or not it would crash later, we want to show a good message in DEV first.
        if (inst != null && inst.render != null) {
          const componentName =
            getComponentNameFromType(Component) || 'Unknown';
          if (!didWarnAboutModulePatternComponent[componentName]) {
            console.error(
              'The <%s /> component appears to be a function component that returns a class instance. ' +
                'Change %s to a class that extends React.Component instead. ' +
                "If you can't use a class try assigning the prototype on the function as a workaround. " +
                "`%s.prototype = React.Component.prototype`. Don't use an arrow function since it " +
                'cannot be called with `new` by React.',
              componentName,
              componentName,
              componentName,
            );
            didWarnAboutModulePatternComponent[componentName] = true;
          }
        }
      }

      // If the flag is on, everything is assumed to be a function component.
      // Otherwise, we also do the unfortunate dynamic checks.
      if (
        disableModulePatternComponents ||
        inst == null ||
        inst.render == null
      ) {
        child = inst;
        return;
      }
    }

    inst.props = element.props;
    inst.context = publicContext;
    inst.updater = updater;

    let initialState = inst.state;
    if (initialState === undefined) {
      inst.state = initialState = null;
    }
    if (
      typeof inst.UNSAFE_componentWillMount === 'function' ||
      typeof inst.componentWillMount === 'function'
    ) {
      if (typeof inst.componentWillMount === 'function') {
        // In order to support react-lifecycles-compat polyfilled components,
        // Unsafe lifecycles should not be invoked for any component with the new gDSFP.
        if (typeof Component.getDerivedStateFromProps !== 'function') {
          if (__DEV__) {
            if (
              warnAboutDeprecatedLifecycles &&
              inst.componentWillMount.__suppressDeprecationWarning !== true
            ) {
              const componentName =
                getComponentNameFromType(Component) || 'Unknown';

              if (!didWarnAboutDeprecatedWillMount[componentName]) {
                console.warn(
                  // keep this warning in sync with ReactStrictModeWarning.js
                  'componentWillMount has been renamed, and is not recommended for use. ' +
                    'See https://reactjs.org/link/unsafe-component-lifecycles for details.\n\n' +
                    '* Move code from componentWillMount to componentDidMount (preferred in most cases) ' +
                    'or the constructor.\n' +
                    '\nPlease update the following components: %s',
                  componentName,
                );
                didWarnAboutDeprecatedWillMount[componentName] = true;
              }
            }
          }

          inst.componentWillMount();
        }
      }
      if (
        typeof inst.UNSAFE_componentWillMount === 'function' &&
        typeof Component.getDerivedStateFromProps !== 'function'
      ) {
        // In order to support react-lifecycles-compat polyfilled components,
        // Unsafe lifecycles should not be invoked for any component with the new gDSFP.
        inst.UNSAFE_componentWillMount();
      }
      if (queue.length) {
        const oldQueue = queue;
        const oldReplace = replace;
        queue = null;
        replace = false;

        if (oldReplace && oldQueue.length === 1) {
          inst.state = oldQueue[0];
        } else {
          let nextState = oldReplace ? oldQueue[0] : inst.state;
          let dontMutate = true;
          for (let i = oldReplace ? 1 : 0; i < oldQueue.length; i++) {
            const partial = oldQueue[i];
            const partialState =
              typeof partial === 'function'
                ? partial.call(inst, nextState, element.props, publicContext)
                : partial;
            if (partialState != null) {
              if (dontMutate) {
                dontMutate = false;
                nextState = assign({}, nextState, partialState);
              } else {
                assign(nextState, partialState);
              }
            }
          }
          inst.state = nextState;
        }
      } else {
        queue = null;
      }
    }
    child = inst.render();

    let childContext;
    if (disableLegacyContext) {
      if (__DEV__) {
        const childContextTypes = Component.childContextTypes;
        if (childContextTypes !== undefined) {
          console.error(
            '%s uses the legacy childContextTypes API which is no longer supported. ' +
              'Use React.createContext() instead.',
            getComponentNameFromType(Component) || 'Unknown',
          );
        }
      }
    } else {
      if (typeof inst.getChildContext === 'function') {
        const childContextTypes = Component.childContextTypes;
        if (typeof childContextTypes === 'object') {
          childContext = inst.getChildContext();
          for (const contextKey in childContext) {
            if (!(contextKey in childContextTypes)) {
              throw new Error(
                `${getComponentNameFromType(Component) ||
                  'Unknown'}.getChildContext(): key "${contextKey}" is not defined in childContextTypes.`,
              );
            }
          }
        } else {
          if (__DEV__) {
            console.error(
              '%s.getChildContext(): childContextTypes must be defined in order to ' +
                'use getChildContext().',
              getComponentNameFromType(Component) || 'Unknown',
            );
          }
        }
      }
      if (childContext) {
        context = assign({}, context, childContext);
      }
    }
  }
  return {child, context};
}
```

```tsx
// packages/react-dom/src/server/ReactPartialRenderer.js

function shouldConstruct(Component) {
  return Component.prototype && Component.prototype.isReactComponent;
}
```

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

```jsx
// packages/react-dom/src/server/ReactPartialRenderer.js

class ReactDOMServerRenderer {
  threadID: ThreadID;
  stack: Array<Frame>;
  exhausted: boolean;
  // TODO: type this more strictly:
  currentSelectValue: any;
  previousWasTextNode: boolean;
  makeStaticMarkup: boolean;
  suspenseDepth: number;

  contextIndex: number;
  contextStack: Array<ReactContext<any>>;
  contextValueStack: Array<any>;
  contextProviderStack: ?Array<ReactProvider<any>>; // DEV-only

  constructor(children: mixed, makeStaticMarkup: boolean) {
    const flatChildren = flattenTopLevelChildren(children);

    const topFrame: Frame = {
      type: null,
      // Assume all trees start in the HTML namespace (not totally true, but
      // this is what we did historically)
      domNamespace: HTML_NAMESPACE,
      children: flatChildren,
      childIndex: 0,
      context: emptyObject,
      footer: '',
    };
    if (__DEV__) {
      ((topFrame: any): FrameDev).debugElementStack = [];
    }
    this.threadID = allocThreadID();
    this.stack = [topFrame];
    this.exhausted = false;
    this.currentSelectValue = null;
    this.previousWasTextNode = false;
    this.makeStaticMarkup = makeStaticMarkup;
    this.suspenseDepth = 0;

    // Context (new API)
    this.contextIndex = -1;
    this.contextStack = [];
    this.contextValueStack = [];

    if (__DEV__) {
      this.contextProviderStack = [];
    }
  }

  destroy() {
    if (!this.exhausted) {
      this.exhausted = true;
      this.clearProviders();
      freeThreadID(this.threadID);
    }
  }

  /**
   * Note: We use just two stacks regardless of how many context providers you have.
   * Providers are always popped in the reverse order to how they were pushed
   * so we always know on the way down which provider you'll encounter next on the way up.
   * On the way down, we push the current provider, and its context value *before*
   * we mutated it, onto the stacks. Therefore, on the way up, we always know which
   * provider needs to be "restored" to which value.
   * https://github.com/facebook/react/pull/12985#issuecomment-396301248
   */

  pushProvider<T>(provider: ReactProvider<T>): void {
    const index = ++this.contextIndex;
    const context: ReactContext<any> = provider.type._context;
    const threadID = this.threadID;
    validateContextBounds(context, threadID);
    const previousValue = context[threadID];

    // Remember which value to restore this context to on our way up.
    this.contextStack[index] = context;
    this.contextValueStack[index] = previousValue;
    if (__DEV__) {
      // Only used for push/pop mismatch warnings.
      (this.contextProviderStack: any)[index] = provider;
    }

    // Mutate the current value.
    context[threadID] = provider.props.value;
  }

  popProvider<T>(provider: ReactProvider<T>): void {
    const index = this.contextIndex;
    if (__DEV__) {
      if (index < 0 || provider !== (this.contextProviderStack: any)[index]) {
        console.error('Unexpected pop.');
      }
    }

    const context: ReactContext<any> = this.contextStack[index];
    const previousValue = this.contextValueStack[index];

    // "Hide" these null assignments from Flow by using `any`
    // because conceptually they are deletions--as long as we
    // promise to never access values beyond `this.contextIndex`.
    this.contextStack[index] = (null: any);
    this.contextValueStack[index] = (null: any);
    if (__DEV__) {
      (this.contextProviderStack: any)[index] = (null: any);
    }
    this.contextIndex--;

    // Restore to the previous value we stored as we were walking down.
    // We've already verified that this context has been expanded to accommodate
    // this thread id, so we don't need to do it again.
    context[this.threadID] = previousValue;
  }

  clearProviders(): void {
    // Restore any remaining providers on the stack to previous values
    for (let index = this.contextIndex; index >= 0; index--) {
      const context: ReactContext<any> = this.contextStack[index];
      const previousValue = this.contextValueStack[index];
      context[this.threadID] = previousValue;
    }
  }

  read(bytes: number): string | null {
    if (this.exhausted) {
      return null;
    }

    const prevPartialRenderer = currentPartialRenderer;
    setCurrentPartialRenderer(this);
    const prevDispatcher = ReactCurrentDispatcher.current;
    ReactCurrentDispatcher.current = Dispatcher;
    try {
      // Markup generated within <Suspense> ends up buffered until we know
      // nothing in that boundary suspended
      const out = [''];
      let suspended = false;
      while (out[0].length < bytes) {
        if (this.stack.length === 0) {
          this.exhausted = true;
          freeThreadID(this.threadID);
          break;
        }
        const frame: Frame = this.stack[this.stack.length - 1];
        if (suspended || frame.childIndex >= frame.children.length) {
          const footer = frame.footer;
          if (footer !== '') {
            this.previousWasTextNode = false;
          }
          this.stack.pop();
          if (frame.type === 'select') {
            this.currentSelectValue = null;
          } else if (
            frame.type != null &&
            frame.type.type != null &&
            frame.type.type.$$typeof === REACT_PROVIDER_TYPE
          ) {
            const provider: ReactProvider<any> = (frame.type: any);
            this.popProvider(provider);
          } else if (frame.type === REACT_SUSPENSE_TYPE) {
            this.suspenseDepth--;
            const buffered = out.pop();

            if (suspended) {
              suspended = false;
              // If rendering was suspended at this boundary, render the fallbackFrame
              const fallbackFrame = frame.fallbackFrame;

              if (!fallbackFrame) {
                throw new Error(
                  'ReactDOMServer did not find an internal fallback frame for Suspense. ' +
                    'This is a bug in React. Please file an issue.',
                );
              }

              this.stack.push(fallbackFrame);
              out[this.suspenseDepth] += '<!--$!-->';
              // Skip flushing output since we're switching to the fallback
              continue;
            } else {
              out[this.suspenseDepth] += buffered;
            }
          }

          // Flush output
          out[this.suspenseDepth] += footer;
          continue;
        }
        const child = frame.children[frame.childIndex++];

        let outBuffer = '';
        if (__DEV__) {
          pushCurrentDebugStack(this.stack);
          // We're starting work on this frame, so reset its inner stack.
          ((frame: any): FrameDev).debugElementStack.length = 0;
        }
        try {
          outBuffer += this.render(child, frame.context, frame.domNamespace);
        } catch (err) {
          if (err != null && typeof err.then === 'function') {
            if (enableSuspenseServerRenderer) {
              if (this.suspenseDepth <= 0) {
                throw new Error(
                  // TODO: include component name. This is a bit tricky with current factoring.
                  'A React component suspended while rendering, but no fallback UI was specified.\n' +
                    '\n' +
                    'Add a <Suspense fallback=...> component higher in the tree to ' +
                    'provide a loading indicator or placeholder to display.',
                );
              }

              suspended = true;
            } else {
              throw new Error('ReactDOMServer does not yet support Suspense.');
            }
          } else {
            throw err;
          }
        } finally {
          if (__DEV__) {
            popCurrentDebugStack();
          }
        }
        if (out.length <= this.suspenseDepth) {
          out.push('');
        }
        out[this.suspenseDepth] += outBuffer;
      }
      return out[0];
    } finally {
      ReactCurrentDispatcher.current = prevDispatcher;
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
      const text = '' + child;
      if (text === '') {
        return '';
      }
      if (this.makeStaticMarkup) {
        return escapeTextForBrowser(text);
      }
      if (this.previousWasTextNode) {
        return '<!-- -->' + escapeTextForBrowser(text);
      }
      this.previousWasTextNode = true;
      return escapeTextForBrowser(text);
    } else {
      let nextChild;
      ({child: nextChild, context} = resolve(child, context, this.threadID));
      if (nextChild === null || nextChild === false) {
        return '';
      } else if (!React.isValidElement(nextChild)) {
        if (nextChild != null && nextChild.$$typeof != null) {
          // Catch unexpected special types early.
          const $$typeof = nextChild.$$typeof;

          if ($$typeof === REACT_PORTAL_TYPE) {
            throw new Error(
              'Portals are not currently supported by the server renderer. ' +
                'Render them conditionally so that they only appear on the client render.',
            );
          }

          // Catch-all to prevent an infinite loop if React.Children.toArray() supports some new type.
          throw new Error(
            `Unknown element-like object type: ${($$typeof: any).toString()}. This is likely a bug in React. ` +
              'Please file an issue.',
          );
        }
        const nextChildren = toArray(nextChild);
        const frame: Frame = {
          type: null,
          domNamespace: parentNamespace,
          children: nextChildren,
          childIndex: 0,
          context: context,
          footer: '',
        };
        if (__DEV__) {
          ((frame: any): FrameDev).debugElementStack = [];
        }
        this.stack.push(frame);
        return '';
      }
      // Safe because we just checked it's an element.
      const nextElement = ((nextChild: any): ReactElement);
      const elementType = nextElement.type;

      if (typeof elementType === 'string') {
        return this.renderDOM(nextElement, context, parentNamespace);
      }

      switch (elementType) {
        // TODO: LegacyHidden acts the same as a fragment. This only works
        // because we currently assume that every instance of LegacyHidden is
        // accompanied by a host component wrapper. In the hidden mode, the host
        // component is given a `hidden` attribute, which ensures that the
        // initial HTML is not visible. To support the use of LegacyHidden as a
        // true fragment, without an extra DOM node, we would have to hide the
        // initial HTML in some other way.
        case REACT_LEGACY_HIDDEN_TYPE:
        case REACT_DEBUG_TRACING_MODE_TYPE:
        case REACT_STRICT_MODE_TYPE:
        case REACT_PROFILER_TYPE:
        case REACT_SUSPENSE_LIST_TYPE:
        case REACT_FRAGMENT_TYPE: {
          const nextChildren = toArray(
            ((nextChild: any): ReactElement).props.children,
          );
          const frame: Frame = {
            type: null,
            domNamespace: parentNamespace,
            children: nextChildren,
            childIndex: 0,
            context: context,
            footer: '',
          };
          if (__DEV__) {
            ((frame: any): FrameDev).debugElementStack = [];
          }
          this.stack.push(frame);
          return '';
        }
        case REACT_SUSPENSE_TYPE: {
          if (enableSuspenseServerRenderer) {
            const fallback = ((nextChild: any): ReactElement).props.fallback;
            const fallbackChildren = toArray(fallback);
            const nextChildren = toArray(
              ((nextChild: any): ReactElement).props.children,
            );
            const fallbackFrame: Frame = {
              type: null,
              domNamespace: parentNamespace,
              children: fallbackChildren,
              childIndex: 0,
              context: context,
              footer: '<!--/$-->',
            };
            const frame: Frame = {
              fallbackFrame,
              type: REACT_SUSPENSE_TYPE,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '<!--/$-->',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
              ((fallbackFrame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            this.suspenseDepth++;
            return '<!--$-->';
          } else {
            throw new Error('ReactDOMServer does not yet support Suspense.');
          }
        }
        // eslint-disable-next-line-no-fallthrough
        case REACT_SCOPE_TYPE: {
          if (enableScopeAPI) {
            const nextChildren = toArray(
              ((nextChild: any): ReactElement).props.children,
            );
            const frame: Frame = {
              type: null,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            return '';
          }
          throw new Error(
            'ReactDOMServer does not yet support scope components.',
          );
        }
        // eslint-disable-next-line-no-fallthrough
        default:
          break;
      }
      if (typeof elementType === 'object' && elementType !== null) {
        switch (elementType.$$typeof) {
          case REACT_FORWARD_REF_TYPE: {
            const element: ReactElement = ((nextChild: any): ReactElement);
            let nextChildren;
            const componentIdentity = {};
            prepareToUseHooks(componentIdentity);
            nextChildren = elementType.render(element.props, element.ref);
            nextChildren = finishHooks(
              elementType.render,
              element.props,
              nextChildren,
              element.ref,
            );
            nextChildren = toArray(nextChildren);
            const frame: Frame = {
              type: null,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            return '';
          }
          case REACT_MEMO_TYPE: {
            const element: ReactElement = ((nextChild: any): ReactElement);
            const nextChildren = [
              React.createElement(
                elementType.type,
                assign({ref: element.ref}, element.props),
              ),
            ];
            const frame: Frame = {
              type: null,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            return '';
          }
          case REACT_PROVIDER_TYPE: {
            const provider: ReactProvider<any> = (nextChild: any);
            const nextProps = provider.props;
            const nextChildren = toArray(nextProps.children);
            const frame: Frame = {
              type: provider,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }

            this.pushProvider(provider);

            this.stack.push(frame);
            return '';
          }
          case REACT_CONTEXT_TYPE: {
            let reactContext = (nextChild: any).type;
            // The logic below for Context differs depending on PROD or DEV mode. In
            // DEV mode, we create a separate object for Context.Consumer that acts
            // like a proxy to Context. This proxy object adds unnecessary code in PROD
            // so we use the old behaviour (Context.Consumer references Context) to
            // reduce size and overhead. The separate object references context via
            // a property called "_context", which also gives us the ability to check
            // in DEV mode if this property exists or not and warn if it does not.
            if (__DEV__) {
              if ((reactContext: any)._context === undefined) {
                // This may be because it's a Context (rather than a Consumer).
                // Or it may be because it's older React where they're the same thing.
                // We only want to warn if we're sure it's a new React.
                if (reactContext !== reactContext.Consumer) {
                  if (!hasWarnedAboutUsingContextAsConsumer) {
                    hasWarnedAboutUsingContextAsConsumer = true;
                    console.error(
                      'Rendering <Context> directly is not supported and will be removed in ' +
                        'a future major release. Did you mean to render <Context.Consumer> instead?',
                    );
                  }
                }
              } else {
                reactContext = (reactContext: any)._context;
              }
            }
            const nextProps: any = (nextChild: any).props;
            const threadID = this.threadID;
            validateContextBounds(reactContext, threadID);
            const nextValue = reactContext[threadID];

            const nextChildren = toArray(nextProps.children(nextValue));
            const frame: Frame = {
              type: nextChild,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            return '';
          }
          // eslint-disable-next-line-no-fallthrough
          case REACT_LAZY_TYPE: {
            const element: ReactElement = (nextChild: any);
            const lazyComponent: LazyComponent<any, any> = (nextChild: any)
              .type;
            // Attempt to initialize lazy component regardless of whether the
            // suspense server-side renderer is enabled so synchronously
            // resolved constructors are supported.
            const payload = lazyComponent._payload;
            const init = lazyComponent._init;
            const result = init(payload);
            const nextChildren = [
              React.createElement(
                result,
                assign({ref: element.ref}, element.props),
              ),
            ];
            const frame: Frame = {
              type: null,
              domNamespace: parentNamespace,
              children: nextChildren,
              childIndex: 0,
              context: context,
              footer: '',
            };
            if (__DEV__) {
              ((frame: any): FrameDev).debugElementStack = [];
            }
            this.stack.push(frame);
            return '';
          }
        }
      }

      let info = '';
      if (__DEV__) {
        const owner = nextElement._owner;
        if (
          elementType === undefined ||
          (typeof elementType === 'object' &&
            elementType !== null &&
            Object.keys(elementType).length === 0)
        ) {
          info +=
            ' You likely forgot to export your component from the file ' +
            "it's defined in, or you might have mixed up default and " +
            'named imports.';
        }
        const ownerName = owner ? getComponentNameFromType(owner) : null;
        if (ownerName) {
          info += '\n\nCheck the render method of `' + ownerName + '`.';
        }
      }

      throw new Error(
        'Element type is invalid: expected a string (for built-in ' +
          'components) or a class/function (for composite components) ' +
          `but got: ${
            elementType == null ? elementType : typeof elementType
          }.${info}`,
      );
    }
  }

  renderDOM(
    element: ReactElement,
    context: Object,
    parentNamespace: string,
  ): string {
    const tag = element.type;

    let namespace = parentNamespace;
    if (parentNamespace === HTML_NAMESPACE) {
      namespace = getIntrinsicNamespace(tag);
    }

    let props = element.props;

    if (__DEV__) {
      if (namespace === HTML_NAMESPACE) {
        const isCustomComponent = isCustomComponentFn(tag, props);
        // Should this check be gated by parent namespace? Not sure we want to
        // allow <SVG> or <mATH>.
        if (!isCustomComponent && tag.toLowerCase() !== element.type) {
          console.error(
            '<%s /> is using incorrect casing. ' +
              'Use PascalCase for React components, ' +
              'or lowercase for HTML elements.',
            element.type,
          );
        }
      }
    }

    validateDangerousTag(tag);

    if (tag === 'input') {
      if (__DEV__) {
        checkControlledValueProps('input', props);

        if (
          props.checked !== undefined &&
          props.defaultChecked !== undefined &&
          !didWarnDefaultChecked
        ) {
          console.error(
            '%s contains an input of type %s with both checked and defaultChecked props. ' +
              'Input elements must be either controlled or uncontrolled ' +
              '(specify either the checked prop, or the defaultChecked prop, but not ' +
              'both). Decide between using a controlled or uncontrolled input ' +
              'element and remove one of these props. More info: ' +
              'https://reactjs.org/link/controlled-components',
            'A component',
            props.type,
          );
          didWarnDefaultChecked = true;
        }
        if (
          props.value !== undefined &&
          props.defaultValue !== undefined &&
          !didWarnDefaultInputValue
        ) {
          console.error(
            '%s contains an input of type %s with both value and defaultValue props. ' +
              'Input elements must be either controlled or uncontrolled ' +
              '(specify either the value prop, or the defaultValue prop, but not ' +
              'both). Decide between using a controlled or uncontrolled input ' +
              'element and remove one of these props. More info: ' +
              'https://reactjs.org/link/controlled-components',
            'A component',
            props.type,
          );
          didWarnDefaultInputValue = true;
        }
      }

      props = assign(
        {
          type: undefined,
        },
        props,
        {
          defaultChecked: undefined,
          defaultValue: undefined,
          value: props.value != null ? props.value : props.defaultValue,
          checked: props.checked != null ? props.checked : props.defaultChecked,
        },
      );
    } else if (tag === 'textarea') {
      if (__DEV__) {
        checkControlledValueProps('textarea', props);
        if (
          props.value !== undefined &&
          props.defaultValue !== undefined &&
          !didWarnDefaultTextareaValue
        ) {
          console.error(
            'Textarea elements must be either controlled or uncontrolled ' +
              '(specify either the value prop, or the defaultValue prop, but not ' +
              'both). Decide between using a controlled or uncontrolled textarea ' +
              'and remove one of these props. More info: ' +
              'https://reactjs.org/link/controlled-components',
          );
          didWarnDefaultTextareaValue = true;
        }
      }

      let initialValue = props.value;
      if (initialValue == null) {
        let defaultValue = props.defaultValue;
        // TODO (yungsters): Remove support for children content in <textarea>.
        let textareaChildren = props.children;
        if (textareaChildren != null) {
          if (__DEV__) {
            console.error(
              'Use the `defaultValue` or `value` props instead of setting ' +
                'children on <textarea>.',
            );
          }

          if (defaultValue != null) {
            throw new Error(
              'If you supply `defaultValue` on a <textarea>, do not pass children.',
            );
          }

          if (isArray(textareaChildren)) {
            if (textareaChildren.length > 1) {
              throw new Error('<textarea> can only have at most one child.');
            }

            textareaChildren = textareaChildren[0];
          }

          if (__DEV__) {
            checkPropStringCoercion(textareaChildren, 'children');
          }
          defaultValue = '' + textareaChildren;
        }
        if (defaultValue == null) {
          defaultValue = '';
        }
        initialValue = defaultValue;
      }

      if (__DEV__) {
        checkFormFieldValueStringCoercion(initialValue);
      }
      props = assign({}, props, {
        value: undefined,
        children: '' + initialValue,
      });
    } else if (tag === 'select') {
      if (__DEV__) {
        checkControlledValueProps('select', props);

        for (let i = 0; i < valuePropNames.length; i++) {
          const propName = valuePropNames[i];
          if (props[propName] == null) {
            continue;
          }
          const propNameIsArray = isArray(props[propName]);
          if (props.multiple && !propNameIsArray) {
            console.error(
              'The `%s` prop supplied to <select> must be an array if ' +
                '`multiple` is true.',
              propName,
            );
          } else if (!props.multiple && propNameIsArray) {
            console.error(
              'The `%s` prop supplied to <select> must be a scalar ' +
                'value if `multiple` is false.',
              propName,
            );
          }
        }

        if (
          props.value !== undefined &&
          props.defaultValue !== undefined &&
          !didWarnDefaultSelectValue
        ) {
          console.error(
            'Select elements must be either controlled or uncontrolled ' +
              '(specify either the value prop, or the defaultValue prop, but not ' +
              'both). Decide between using a controlled or uncontrolled select ' +
              'element and remove one of these props. More info: ' +
              'https://reactjs.org/link/controlled-components',
          );
          didWarnDefaultSelectValue = true;
        }
      }
      this.currentSelectValue =
        props.value != null ? props.value : props.defaultValue;
      props = assign({}, props, {
        value: undefined,
      });
    } else if (tag === 'option') {
      let selected = null;
      const selectValue = this.currentSelectValue;
      if (selectValue != null) {
        let value;
        if (props.value != null) {
          if (__DEV__) {
            checkFormFieldValueStringCoercion(props.value);
          }
          value = props.value + '';
        } else {
          if (__DEV__) {
            if (props.dangerouslySetInnerHTML != null) {
              if (!didWarnInvalidOptionInnerHTML) {
                didWarnInvalidOptionInnerHTML = true;
                console.error(
                  'Pass a `value` prop if you set dangerouslyInnerHTML so React knows ' +
                    'which value should be selected.',
                );
              }
            }
          }
          value = flattenOptionChildren(props.children);
        }
        selected = false;
        if (isArray(selectValue)) {
          // multiple
          for (let j = 0; j < selectValue.length; j++) {
            if (__DEV__) {
              checkFormFieldValueStringCoercion(selectValue[j]);
            }
            if ('' + selectValue[j] === value) {
              selected = true;
              break;
            }
          }
        } else {
          if (__DEV__) {
            checkFormFieldValueStringCoercion(selectValue);
          }
          selected = '' + selectValue === value;
        }

        props = assign(
          {
            selected: undefined,
          },
          props,
          {
            selected: selected,
          },
        );
      }
    }

    if (__DEV__) {
      validatePropertiesInDevelopment(tag, props);
    }

    assertValidProps(tag, props);

    let out = createOpenTagMarkup(
      element.type,
      tag,
      props,
      namespace,
      this.makeStaticMarkup,
      this.stack.length === 1,
    );
    let footer = '';
    if (omittedCloseTags.hasOwnProperty(tag)) {
      out += '/>';
    } else {
      out += '>';
      footer = '</' + element.type + '>';
    }
    let children;
    const innerMarkup = getNonChildrenInnerMarkup(props);
    if (innerMarkup != null) {
      children = [];
      if (
        newlineEatingTags.hasOwnProperty(tag) &&
        innerMarkup.charAt(0) === '\n'
      ) {
        // text/html ignores the first character in these tags if it's a newline
        // Prefer to break application/xml over text/html (for now) by adding
        // a newline specifically to get eaten by the parser. (Alternately for
        // textareas, replacing "^\n" with "\r\n" doesn't get eaten, and the first
        // \r is normalized out by HTMLTextAreaElement#value.)
        // See: <http://www.w3.org/TR/html-polyglot/#newlines-in-textarea-and-pre>
        // See: <http://www.w3.org/TR/html5/syntax.html#element-restrictions>
        // See: <http://www.w3.org/TR/html5/syntax.html#newlines>
        // See: Parsing of "textarea" "listing" and "pre" elements
        //  from <http://www.w3.org/TR/html5/syntax.html#parsing-main-inbody>
        out += '\n';
      }
      out += innerMarkup;
    } else {
      children = toArray(props.children);
    }
    const frame = {
      domNamespace: getChildNamespace(parentNamespace, element.type),
      type: tag,
      children,
      childIndex: 0,
      context: context,
      footer: footer,
    };
    if (__DEV__) {
      ((frame: any): FrameDev).debugElementStack = [];
    }
    this.stack.push(frame);
    this.previousWasTextNode = false;
    return out;
  }
}
```
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

이 파트에 대해 조사하기 시작하면서 **횡단 관심사** 라는 단어를 처음 접하게 되었고 이 부분에 대해서 고차 컴포넌트와 함께, 짧게 다루고 넘어가면 좋게다는 생각이 들었다.

> [객체 지향 소프트웨어 개발](https://ko.wikipedia.org/w/index.php?title=%EA%B0%9D%EC%B2%B4_%EC%A7%80%ED%96%A5_%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4_%EA%B0%9C%EB%B0%9C&action=edit&redlink=1)에서 **횡단 관심사** 또는 **크로스커팅 관심사**(cross-cutting concerns)는 다른 관심사에 영향을 미치는 [프로그램](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%93%A8%ED%84%B0_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%A8)의 [애스펙트](https://ko.wikipedia.org/w/index.php?title=%EC%95%A0%EC%8A%A4%ED%8E%99%ED%8A%B8_(%EC%BB%B4%ED%93%A8%ED%84%B0_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D)&action=edit&redlink=1)이다. 이 관심사들은 디자인과 구현 면에서 시스템의 나머지 부분으로부터 깨끗이 [분해](https://ko.wikipedia.org/wiki/%EB%AA%A8%EB%93%88%EC%84%B1_(%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D))되지 못하는 경우가 있을 수 있으며 분산([코드 중복](https://ko.wikipedia.org/wiki/%EC%BD%94%EB%93%9C_%EC%A4%91%EB%B3%B5))되거나 얽히는(시스템 간의 상당한 의존성 존재) 일이 일어날 수 있다. ([위키백과](https://ko.wikipedia.org/wiki/%ED%9A%A1%EB%8B%A8_%EA%B4%80%EC%8B%AC%EC%82%AC))
> 

간단히 말해서 우리가 개발하고자 하는 하나의 프로그램의 목적과 방향성을 고려할 때, 핵심이 되는 기능이나 역할들을 하는 모듈이나 로직들이 있을 텐데, 이를 **핵심 관심사**라고 하며, 이 핵심 관심사들의 로직에서 공통적으로, 혹은 반복적으로 호출되거나 쓰임이 있는 코드들의 역할을 두고 횡단 관심사라고 정리할 수 있을 것 같다.

![995FB13A6007BDB813.png](/assets/2022-03-20-TWL-02%20React%20Component와%20Fragment,%20그리고%20고차%20컴포넌트/02-cross-cutting-concerns.png)

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

고차 컴포넌트는 입력된 컴포넌트를 수정하지 않으며, 상속을 사용해 동작을 복사하지도 않는으며, 오히려 고차 컴포넌트는 원본 컴포넌트를 컨테이너 컴포넌트로 포장하고, 조합한다. 고차 컴포넌트는 사이드 이펙트가 없는 순수함수이다. 위 함수는 일반 함수로서 동작하기 때문에 목적에 따라 원하는 개수만큼 인수를 추가할 수 있다.

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