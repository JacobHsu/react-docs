---
title: "React Hooks: useMemo vs useEffect"
author: [gaearon,jacobhsu]
---

## useEffect

**useEffect 一般用於處理狀態更新導致的 side effects**。雖然說不提倡面向生命週期函數編程，但是在沒有熟練掌握 useEffect 的時候，類比 Class Component 的生命週期函數最能幫助我們快速上手了。useEffect 可以看成 componentDidMount / componentDidUpdate / componentWillUnmount 這 3 個生命週期函數的替代。

這裡貼一個官網的例子，可以非常全面的展示 useEffect 的使用方式：

```js
import React, { useState, useEffect } from 'react';

// 該組件定時從服務器獲取好友的在線狀態
function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }
    // 在瀏覽器渲染結束後執行
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    
    // 在每次渲染產生的 effect 執行之前執行
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
    
   // 只有 props.friend.id 更新了才會重新執行這個 hook
  }, [props.friend.id]);

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

## useLayoutEffect

`useEffect` 是官方推薦拿來代替 `componentDidMount` / `componentDidUpdate` / `componentWillUnmount` 這 3 個生命週期函數的，但其實他們並不是完全等價，useEffect 是在瀏覽器渲染結束之後才執行的，而這三個生命週期函數是在瀏覽器渲染之前同步執行的，React 還有一個官方的 hook 是完全等價於這三個生命週期函數的，叫 `useLayoutEffect`。

這兩者的區別可以看一下這個例子（[codePen](https://link.zhihu.com/?target=https%3A//codepen.io/Lxylona/pen/xxOgzoV)）

這個例子可以很明顯看出 useEffect 和 useLayoutEffect 之間的區別，useEffect 是在瀏覽器重繪之後才異步執行的，所以點擊按鈕之後按鈕上的數字會先變成 0，再變成一個隨機數；而 useLayoutEffect 是在瀏覽器重繪之前同步執行的，所以**兩次 setCount 合並到 300 毫秒後的重繪裡了**。

因為 useEffect 不會阻塞瀏覽器重繪，而且平時業務中我們遇到的絕大多數場景都是時機不敏感的，比如取數、修改 dom、事件觸發/監聽…… 

所以首推用 `useEffect` 來處理 side effects，性能上的表現會更好一些。

## ComponentWillReceiveProps

ComponentWillReceiveProps 是在組件接收到新 props 時執行的，和 useEffect 的執行時機完全不一致，事實上它和 useMemo 才是執行時機一致的，但是為什麼卻推薦用 useEffect 而不是 useMemo 來替代它呢？

我們來看看一個典型的 Class Component 可能會在 willReceiveProps 裡做什麼事情：

```js
componentWillReceiveProps(nextProps) {
  
  if (nextProps.queryKey !== this.props.queryKey) {
    // 觸發外部狀態變更
    nextProps.setIsLoading(true);
    // 取數
    this.reFetch(nextProps.queryKey);
  }
  
  if (nextProps.value !== this.props.value) {
    // state 更新
    this.setState({
      checkList: this.getCheckListByValue(nextProps.value);
    })
  }
  
  if (nextProps.instanceId !== this.props.instanceId) {
    // 事件 / dom
    event.emit('instanceId_changed', nextProps.instanceId);
  }
  
}
```


這些代碼是不是很眼熟？ ComponentWillReceiveProps 經常被拿來：

1. 觸發回調，造成外部狀態變更
2. 事件監聽和觸發、dom 的變更
3. 重新取數
4. state 更新

很明顯前 3 種情況是時機不敏感的，為什麼我們習慣在 ComponentWillReceiveProps 中做這些事情呢？因為 ComponentWillReceiveProps 可以第一時間拿到 props 和 nextProps ，方便我們做對比，而現在 React 已經接管了這個對比的工作，我們完全可以使用 useEffect 來替代，不阻塞瀏覽器重渲染，用戶會覺得頁面更加流暢。像取數這種經常涉及到復雜計算的場景，更是如此。

對於第 4 種情況我們需要思考一下，在組件更新期間更新狀態是否是一個恰當的行為？歸根到底組件需要動態根據某個 prop 來生成某個數據，
如果在 Class Component 中，直接在 render 方法中生成即可，
完全不需要 setState；如果是在 Function Component 中，確實是一個適合使用 useMemo 的場景，但是注意我們不是想要“更新狀態”，而是因為“依賴改變了所以對象更新了”。

```js
// 當 props.params 更新時，重新生成 newParams
const checkList = React.useMemo(() => {
 
  // 復雜的計算之後得到新的 checkList
  const newCheckList = props.value.map(each => ...)
  
  return newCheckList
}, [props.value])
```

## useMemo

**useMemo 是拿來保持一個對象引用不變的。** useMemo 和 useCallback 都是 React 提供來做性能優化的。比起 classes， Hooks 給了開發者更高的靈活度和自由，但是對開發者要求也更高了，因為 Hooks 使用不恰當很容易導致性能問題。

比如我有這樣一段 JSX：

```js
<LineChart 
  dataconfig={{ // 取數配置
    ...dataConfig,
    datasetId: getDatasetId(queryId)
  }} 
  fetchData={(newDataConfig) => { // fetcher
    realFetchData(newDataConfig);
  }} 
/>
```

LineChart 會在 dataConfig 發生變化時重新取數，如果 LineChart 是一個 Class Component，那他的代碼一般會這麼寫：

```js
// Class Component
class LineChart extends React.Component {
  
  componentWillReceiveProps(nextProps) {
    // 當 dataConfig 發生變化時重新取數
    if (nextProps.dataConfig !== this.props.dataConfig) {
      nextProps.fetchData(nextProps.dataConfig);
    }
  }
  
}
```

如果用 Hooks 來實現，那麼代碼就變成了這樣：

```js
// Function Component
function LineChart ({ dataConfig, fetchData }) {
  
  React.useEffect(() => {
    fetchData(dataConfig);
  }, [dataConfig, fetchData])
  
}
```

從上面的代碼中很明顯看出 Class Component 和 Function Component 在開發心智上的區別，在 Class Component 中我們需要自己管理依賴。

比如上面的例子我們會手動判斷前後 dataConfig 是否發生了變化，如果發生了變化再重新取數；而在 Function Component 中我們把依賴交給 React 自動管理了，雖然減少了手動做 diff 的工作量，但也帶來了副作用：因為 React 做的是淺比較( Object.is() )，所以**當 fetchData 的引用變化了，也會導致重新取數。**

但這個重取數邏輯上其實是合理的， 因為對於 React 來說，**任何一個依賴項改變了都應該重新處理 hooks 中的邏輯**，如果一個依賴的函數改變了，有可能是確實是函數體已經改變了。這和 React 的 callback ref 的處理方法是一致的: 如果每次傳一個變化的 callback，那麼 React 認為你需要重新處理這個 ref，因此他會重新初始化 ref。

雖然 React 對於依賴的處理是合理的，但是也需要解決引用變化導致的性能問題，這時候解法：
想辦法讓 fetchData 的引用不變化。官方提供了一個 hooks —— `useCallback` 來解決函數引用的問題。

```js
const fetchData = React.useCallback((newDataConfig) => {
    realFetchData(newDataConfig);
  }, [realFetchData]);

return <LineChart 
  dataconfig={{ // 取數配置
    ...dataConfig,
    datasetId: getDatasetId(queryId)
  }} 
  fetchData={fetchData} 
/>
```

這時候還沒有徹底解決問題，因為**只要 props 更新**，LineChart 還是每次都會重新取數，你應該已經發現了，
dataConfig 也是一個每次都會引用變化的 prop。memo 是 Hooks 中最容易被忽略的了，即使大家有意不在 JSX 中做計算，也經常會出現這種情況：

```js
const fetchData = React.useCallback((newDataConfig) => {
    realFetchData(newDataConfig);
  }, [realFetchData]);

const dataCOnfig = getDataConfig(queryid);

return <LineChart 
  dataconfig={dataConfig} 
  fetchData={fetchData} 
/>
```

函數式編程就這種習慣，大家已經習慣這種無狀態的寫法，但是組件就是有狀態的，狀態更新了就得重新處理相關邏輯、重新渲染。
我們得告訴 React 什麼時候應該重新處理這個狀態了

`useMemo` 就是拿來做這個的：

```js
const fetchData = React.useCallback((newDataConfig) => {
    realFetchData(newDataConfig);
  }, [realFetchData]);

const dataConfig = React.useMemo(() => ({
    ...dataConfig,
    datasetId: getDatasetId(queryId)
  }), [getDatasetId, queryId]);

return <LineChart 
  dataconfig={dataConfig} 
  fetchData={fetchData} 
/>
```

這樣 dataConfig 只有在 getDatasetId 或者 queryId 變化時才會重新生成，LineChart 只會在必要的時候才會重新取數。

useMemo example : pancake-frontend/src/[index.tsx](https://github.com/pancakeswap/pancake-frontend/blob/develop/src/index.tsx)

### memo

你可能會發現你已經很注意用 useMemo 和 useCallback 來進行性能優化了，但是效果卻不如人意。

只用 useMemo 和 useCallback 來做性能優化可能是無法得到預期效果的，原因是如果 props 引用不會變化，子組件不會重新渲染，但它依然會重新執行，看下面這個例子（[codePen](https://link.zhihu.com/?target=https%3A//codepen.io/Lxylona/pen/dyXNBEe)）：

```js
function Counter({ count }) {
  console.log('Counter 重新執行了！', count);
  
  // ...進行了一堆很復雜的計算！

  return <span>{count}</span>;
}

function App() {
  const [count, setCount] = React.useState(0);
  const [stateAutoChange, setStateAutoChange] = React.useState(0);

  React.useEffect(() => {
    setInterval(() => {
      setStateAutoChange(s => s + 1);
    }, 500);
  }, []);

  return (
    <div>
      <div>{stateAutoChange}</div>
      <div>
        {/* count 是不會變化的 */}
        <Counter count={count} />
      </div>
    </div>
  );
}
```

如果 Counter 計算量很大，那瓶頸就不是重渲染而是重執行的這個過程了。如果想要阻斷 Counter 重新執行，React 提供了一個 API：memo，它相當於 PureComponent，是一個高階組件，默認對 props 做一次淺比較，如果 props 沒有變化，則子組件不會重新執行。

那麼給 Counter 套上 memo：

```js
const Counter = React.memo(({ count }) => {
  console.log('Counter 重新執行了！', count);

  // ...進行了一堆很復雜的計算！
  
  return <span>{count}</span>;
});
```

什麼時候應該用 memo 和 useMemo？
我們可能只有在復雜應用中才會關注到性能問題（某些簡單的應用可能永遠都不會出現性能問題- -），我的看法是：

用 useMemo 和 useCallback 來控制子組件 props 的引用，和 memo 一起使用效果是最佳的，原因上面的例子也呈現了，子組件會有重新執行的開銷，沒有配套 memo 的話還可能出現反效果，這幾個 API 各司其職，性能優化是一個整體的過程，不是單獨在某一個組件裡面做一些操作就可以得到改善的

具體策略也是根據具體情況來，只要記住幾個 API 的功能就可以了：
`useMemo` 避免頻繁的昂貴計算，`useCallback` 讓 shouldComponentUpdate 可以正常發揮作用，memo 就是 shouldComponentUpdate。

References: [React Hooks: 深入剖析 useMemo 和 useEffect](https://zhuanlan.zhihu.com/p/268802571) 
