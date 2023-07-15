<center><font face="黑体" size=24 >react-router的核心介绍</font></center>

##### 在React中通常我们会使用react-router-dom作为我们的路由导航工具，我们可以做到改变了浏览器的url而不会去重新发起请求来更新我们的页面，其核心原理主要在于其中用到的一个history对象，react-router-dom会引入react-router所以我们只有安装react-router-dom。

**本文介绍过程中涉及到的源码版本是6.4.1**

react-router-dom提供了几种路由模式，其中最常用的就是history（BrowserRouter）和hash（HashRouter）路由，以BrowserRouter为例，通常我们在react中使用react-router-dom进行导航是这样写的(**目前react-router-dom最新版本是v6**)：

```javascript
import { BrowserRouter，Routes, Route, Link } from "react-router-dom"
const App = () => {
    return (
    	<BrowserRouter>
        	<Link to={'/page'}>go page</Link>
        	<Routes>
         		<Route path='/page' element={<Childs />}>
       		</Routes>
        </BrowserRouter>
    )
}
// Routes代替了之前版本的Switch
```

* BrowserRouter主要工作：通过createBrowserHistory**创建**一个**history**对象，再创建一个组件内部状态，通过useLayoutEffect把**状态更新函数绑定**在history对象上，当路由发生变化时候触发状态更新，状态更新后匹配到对应组件进行切换做到url改变浏览器不触发请求而切换页面。

  <details> 
  <summary><strong size="4">BrowserRouter源码</strong></summary> 
  <pre><code class="language-javascript">// 位置node_modules/react-router-dom/dist/index.js
  function BrowserRouter(_ref) {
    let {
      basename,
      children,
      window
    } = _ref;
    let historyRef = React.useRef();
    if (historyRef.current == null) {
      historyRef.current = createBrowserHistory({window,v5Compat: true}); // 创建history
    }
    let history = historyRef.current;
    // 创建组件状态
    let [state, setState] = React.useState({
      action: history.action,
      location: history.location
    });
    React.useLayoutEffect(() => history.listen(setState), [history]); // 绑定history变化监听，这里就是核心怎么做到url改变浏览器不触发请求而切换页面
    // 返回一个基于Router的react组件
    return React.createElement(Router, {
      basename: basename,
      children: children,
      location: state.location,
      navigationType: state.action,
      navigator: history
    });
  }
  </code>
  </pre> </details>

* 通过createBrowserHistory创建history：createBrowserHistory调用**getUrlBasedHistory创建history**。

  <details> 
  <summary><strong size="4">createBrowserHistory源码</strong></summary> 
  <pre><code class="language-javascript"> // 位置node_modules/@remix-run/router/history.ts
  export function createBrowserHistory(
    options: BrowserHistoryOptions = {}
  ): BrowserHistory {
    // 该函数产生一个localtion对象，我们在useLocation里面会用到
    function createBrowserLocation(
      window: Window,
      globalHistory: Window["history"]
    ) {
      let { pathname, search, hash } = window.location;
      return createLocation(
        "", //current
        { pathname, search, hash }, // to
        (globalHistory.state && globalHistory.state.usr) || null,
        (globalHistory.state && globalHistory.state.key) || "default"
      );
    }
    function createBrowserHref(window: Window, to: To) {
      return typeof to === "string" ? to : createPath(to);
    }
    return getUrlBasedHistory(
      createBrowserLocation,
      createBrowserHref,
      null,
      options
    );
  }
  </code></pre> 
  </details>
  
* getUrlBasedHistory核心作用创建一个包含listen、push、replace和go等方法属性的history对象：

  1. listen：

     我们在BrowserRouter的useLayoutEffect时候调用history.listen(setState)，把BrowserRouter组件状态更新方法传入进来，这时listen会**注册**window的**popstate监听事件**（该事件是在**浏览器**某些**行为**下**触发**，比如点击后退按钮**或**者调用**history.back()，history.go()**才会触发），当监听到**事件触发**后就会**调用**listener({ action, location: history.location })，也就是我们的**setState**方法，把BrowserRouter组件状态更新为我们传入的action和location，**触发组件重新渲染**（重新渲染匹配页面逻辑在Router里后面会讲）

  2. push和replace原理一样，只是调用API不同：

     push和replace会**分别调用window.history.pushState()**和**window.history.replaceState()**方法传入对应的historyState, "", url，它们会向浏览器历史里面添加或者替代一条记录，并把**浏览器**的**url**地址栏会**变更**成我们传入的url但是并**不会触发请求**，调用这两个方法**再去调用**listener({ action, location })也就是**setState**，**触发BrowserRouter组件更新渲染**

  3. go原理和他们一样只不过是触发浏览器的popstate事件再调用setState，这里就不展开了，具体看源码。

  4. go push replace会在react-router-dom的Link组件或者useNavigate这些钩子里面调用（后面会具体展开）

     <details> 
     <summary><strong size="4">getUrlBasedHistory源码</strong></summary> 
     <pre><code class="language-javascript"> // 位置node_modules/@remix-run/router/history.ts
     // Action = { Pop = "POP", Push = "PUSH", Replace = "REPLACE"}
     function getUrlBasedHistory(
       getLocation: (window: Window, globalHistory: Window["history"]) => Location,
       createHref: (window: Window, to: To) => string,
       validateLocation: ((location: Location, to: To) => void) | null,
       options: UrlHistoryOptions = {}
     ): UrlHistory {
       let { window = document.defaultView!, v5Compat = false } = options;
       let globalHistory = window.history;
       let action = Action.Pop;
       let listener: Listener | null = null;
       function handlePop() { // popstate事件处理
         action = Action.Pop;
         if (listener) {
           listener({ action, location: history.location }); // 触发setState
         }
       }
       function push(to: To, state?: any) { // 调用window.history.pushState()改变url和UI
         action = Action.Push;
         let location = createLocation(history.location, to, state);
         if (validateLocation) validateLocation(location, to);
         let historyState = getHistoryState(location);
         let url = history.createHref(location);
         try {
           globalHistory.pushState(historyState, "", url);
         } catch (error) {
           window.location.assign(url);
         }
         if (v5Compat && listener) {
           listener({ action, location }); // 触发setState
         }
       }
       function replace(to: To, state?: any) { // 调用window.history.replaceState()改变url和UI
         action = Action.Replace;
         let location = createLocation(history.location, to, state);
         if (validateLocation) validateLocation(location, to);
         let historyState = getHistoryState(location);
         let url = history.createHref(location);
         globalHistory.replaceState(historyState, "", url);
         if (v5Compat && listener) {
           listener({ action, location: location }); // 触发setState
         }
       }
       let history: History = {
         get action() {
           return action;
         },
         get location() {
           return getLocation(window, globalHistory);
         },
         listen(fn: Listener) {
           if (listener) {
             throw new Error("A history only accepts one active listener");
           }
           window.addEventListener(PopStateEventType, handlePop); // popstate
           listener = fn; // 把setState绑定到listener上
           return () => {
             window.removeEventListener(PopStateEventType, handlePop);
             listener = null;
           };
         },
         createHref(to) {
           return createHref(window, to);
         },
         push,
         replace,
         go(n) {
           return globalHistory.go(n);
         },
       };
       return history;
     }
     </code>
     </pre> </details>

**小结**：BrowserRouter组件会把状态更新函数也就是setState绑定在getUrlBasedHistory创建的history对象上，当调用history里的方法的时候或者点击浏览器的前进后退按钮就会触发BrowserRouter组件更新来切换渲染页面。

* **useNavigate**

  **useNavigate**其实就是一个useCallback，里面主要调用了一个navigator，通过navigator.go(),navigator.push(),navigator.replace()等方法实现路由导航，而navigator是从useContext(NavigationContext)中获取的。

  <details> 
  <summary><strong size="4">useNavigate源码</strong></summary> 
  <pre><code class="language-javascript">// useNavigate在node_modules/react-router/dist/index.js
  function useNavigate() {
    // 进入函数会做一些是否是在router组件内使用useNavigate之类的校验
    let {
      basename,
      navigator
    } = React.useContext(NavigationContext);
    let {
      matches
    } = React.useContext(RouteContext);
    let {
      pathname: locationPathname
    } = useLocation();
    let routePathnamesJson = JSON.stringify(getPathContributingMatches(matches).map(match => match.pathnameBase));
    let activeRef = React.useRef(false);
    React.useEffect(() => {
      activeRef.current = true;
    });
    let navigate = React.useCallback(function (to, options) {
      if (options === void 0) {
        options = {};
      }
      process.env.NODE_ENV !== "production" ? warning(activeRef.current, "You should call navigate() in a React.useEffect(), not when " + "your component is first rendered.") : void 0;
      if (!activeRef.current) return;
      if (typeof to === "number") {
        navigator.go(to);
        return;
      }
      let path = resolveTo(to, JSON.parse(routePathnamesJson), locationPathname, options.relative === "path"); 
      if (basename !== "/") {
        path.pathname = path.pathname === "/" ? basename : joinPaths([basename, path.pathname]);
      }
      (!!options.replace ? navigator.replace : navigator.push)(path, options.state, options); // 判断是替换还是追加
    }, [basename, navigator, routePathnamesJson, locationPathname]);
    return navigate;
  }
  </code>
  </pre> </details>

  可以从useNavigate的源码中看到，useNavigate主要是调用NavigationContext提供的navigator对象里面的方法，而对于NavigationContext的赋值是在Router组件里，而Router组件会在BrowserRouter组件中调用。NavigationContext主要代码如下：

  ```javascript
  function Router(_ref4) {
    let {
      basename: basenameProp = "/",
      children = null,
      location: locationProp,
      navigationType = Action.Pop,
      navigator, // 在BrowserRouter组件里面会把history赋值给navigator
      static: staticProp = false
    } = _ref4;
    let navigationContext = React.useMemo(() => ({
      basename,
      navigator,
      static: staticProp
    }), [basename, navigator, staticProp]);
    return React.createElement(NavigationContext.Provider, { 
      value: navigationContext 
    },React.createElement(LocationContext.Provider, {
      children: children,
      value: {
        location,
        navigationType
      }
    }));
  }
  ```

  结合BrowserRouter的源码可以得知：**useNavigate到最后调用的navigator对象实际就是我们通过getUrlBasedHistory创建的history对象，所以useNavigate间接调用history改变浏览器url地址触发最外层BrowserRouter组件的更新渲染来达到页面切换效果**

* **Link**

  **Link**是一个通过**React.forwardRef**基于a标签**创建的**一个**组件**，这里我们只关心当我们**点击Link**组件后做了些什么。当我们给Link组件传入**reloadDocument**属性后默认点击时候会**触发**我们传入的**onClick**方法，如果**没有reloadDocument**属性就会判断我们**是否**传入**onClick**方法，如果**有**就**调用**onClick就会调用我们传入的onClick事件，如果**没有**就会调用**useLinkClickHandler**，**useLinkClickHandler**是一个hooks主要**调用**我们熟悉的**useNavigate**钩子方法主要代码如下：

  ```javascript
  function useLinkClickHandler(to, _temp) {
    let {
      target,
      replace: replaceProp,
      state,
      preventScrollReset,
      relative
    } = _temp === void 0 ? {} : _temp;
    let navigate = useNavigate();
    let location = useLocation();
    let path = useResolvedPath(to, {
      relative
    });
    return React.useCallback(event => {
      if (shouldProcessLinkClick(event, target)) {
        event.preventDefault();
        let replace = replaceProp !== undefined ? replaceProp : createPath(location) === createPath(path);
        navigate(to, {
          replace,
          state,
          preventScrollReset,
          relative
        });
      }
    }, [location, navigate, path, replaceProp, state, target, to, preventScrollReset, relative]);
  }
  ```
  
  <details> 
  <summary><strong size="4">Link组件源码</strong></summary> 
  <pre><code class="language-javascript">// 位置node_modules/react-router-dom/dist/index.js
  const Link = React.forwardRef(function LinkWithRef(_ref4, ref) {
    let {
      onClick,
      relative,
      reloadDocument,
      replace,
      state,
      target,
      to, // 路径
      preventScrollReset
    } = _ref4,
      rest = _objectWithoutPropertiesLoose(_ref4, _excluded);
    let href = useHref(to, {
      relative
    });
    let internalOnClick = useLinkClickHandler(to, {
      replace,
      state,
      target,
      preventScrollReset,
      relative
    });
    function handleClick(event) {
      if (onClick) onClick(event);
      if (!event.defaultPrevented) {
        internalOnClick(event);
      }
    }
    return (
      React.createElement("a", _extends({}, rest, {
        href: href,
        onClick: reloadDocument ? onClick : handleClick,
        ref: ref,
        target: target
      }))
    );
  });
  </code>
  </pre> </details>

* **扩展**：其实react-router的核心就在于history方法调用于React状态改变进行绑定和一些匹配规则算法，react-router创建的history和[history库]('https://github.com/remix-run/history/blob/dev/packages/history/index.ts')几乎是一样的，history库的功能要比react-router多一些。我们发现新版本的react-router中没有实现像之前的Prompt组件，用于路由跳转前做一些操作，对比之前版本的源码可以发现Prompt组件其实就是在history触发路由变化前注册了一个阻塞事件。而新版本react-router实现的history没有注册阻塞事件的方法，所以我们可以结合旧版本源码来模拟简单实现Prompt的功能。

  这里只是一个简单的例子抛砖引玉实际工作中还需要**考虑浏览器行为**导致的页面切换，（**6.7版本已实现**）
  
  ```javascript
  import React, { useContext, useLayoutEffect, useState, createContext } from "react";
  // react-router-dom暴露了一个UNSAFE_NavigationContext的React Context，也就是NavigationContext
  import { UNSAFE_NavigationContext } from 'react-router-dom'
  import { Modal } from "antd";
  const Nav = createContext(null)
  export { Nav }
  export default function App(){
      // 只要组件包含在NavigationContext的Provide组件中就可以通过useContext获取
      // NavigationContext.navigator就是我们熟悉的history
      const { navigator: history } = useContext(UNSAFE_NavigationContext) 
      const wrap = (nav) => {
      	const history = nav
      	const push = nav.push
      	history.blockTasks = []
      	history.block = (fn) => { // 增加阻塞，返回去除阻塞的函数
        		history.blockTasks.push(fn)
              return () => history.blockTasks = history.blockTasks.filter((val) => val !== fn) 
          }
          history.push = function (...args) { // 重构push，先执行阻塞回调，
            if (history.blockTasks.length) {
              history.blockTasks.forEach((fn) => {
                fn(() => push(...args))
              })
            } else {
              push(...args)
            }
          }
          return history
    }
      return(
          <Nav.Provider value={{history: wrap(navigator)}}>
          	// 路由，布局
          </Nav.Provider>
      )
  }
  // 应用
  import React, { useContext, useEffect, useState } from "react"
  import { Nav } from "./App" 
  export default function App4 () {
      const {history} = useContext(Nav)
      useEffect(() => {
          const deleteCallback = history.block((rety) => {
              // 这里执行页面跳转前的一些操作
          })
      }, [])
      return(
          <></>
      )
  }
  ```
  
  原理：其实逻辑很简单，就是把react-router里面的history对象取出来重构一下，把push，replace，go，等方法重构一下，在执行这些方法前，先执行阻塞回调，由阻塞回调决定是否切换页面并把阻塞回调删除。

