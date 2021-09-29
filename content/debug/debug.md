---
id: tutorial
title: "除錯指南"
layout: tutorial
sectionid: tutorial
permalink: debug/debug.html
# redirect_from:
#   - "docs/debug.html"
---

React Debug 筆記。

## React Hook useEffect has a missing dependency: 

[在 useEffect 中使用呼叫需被覆用的函式 - useCallback 的使用](https://ithelp.ithome.com.tw/articles/10225504)

>Tip
>
>如果某個函式不需要被覆用，那麼可以直接定義在 `useEffect` 中，但若該方法會需要被共用，則把該方法提到 useEffect 外面後，記得用 `useCallback` 進行處理後再放到 useEffect 的 dependencies 中

這個錯誤提示是由 `ESLint` 發出的，ESLint 是用來檢查 JavaScript 程式碼中有無語法錯誤或是撰寫風格不符的工具，在這個工具中可以根據專案或團隊的需要設定不同的規則，而這裡之所以會跳出錯誤提示，是因為在 CodeSandbox 上是透過 create-react-app 這個官方工具來建立的 React 專案，因此預設會根據 React 官方的建議來安裝與設定 ESLint。

這個 ESLint 的錯誤提示是由名為 react-hooks 的 ESLint Plugin 顯示，告訴我們在 useEffect 中似乎遺漏了 dependencies，它認為應該要把 fetchData 放到 useEffec 的 dependencies 的陣列中。

這個錯誤提示之所以會產生，是因為先前當我們把 fetchData 定義在 useEffect 中時，React Hooks ESLint Plugin 可以很清楚的知道在 fetchData 這個函式中，並沒有相依到任何和 React 組件有關的資料狀態（即，state 或 props），因此允許我們在 dependencies 陣列中不帶入任何元素。

但是當我們把 fetchData 搬到 useEffect 外之後，React Hooks ESLint Plugin 不確定 fetchData 中是否有使用到 React 內部的資料狀態（即，state 或 props），如果 fetchData 有相依到 state 或 props 但在 dependencies 中卻沒把相依的資料放入陣列時，就可能使得 fetchData 沒辦法適時的重新被呼叫到而產生問題，因此 React Hooks ESLint Plugin 才會建議我們把 fetchData 放到 dependencies 中。

useEffect 內函式被呼叫的原則是：
> 「組件渲染完後，如果 dependencies 有改變，才會呼叫 useEffect 內的 function」

每一次 Function Component 被呼叫時，都會再定義一次新的 fetchData（但函式的內容都相同），因此雖然對我們來說，因為 fetchData 內做的事是一樣，所以我們覺得它是相同的；但對 useEffect 的 dependencies 來說每次的 fetchData 卻都是不同的，而這也就是為什為會導致無窮迴圈的緣故

## 避免 useEffect 內的函式不斷執行 - useCallback 的使用

在 React Hooks 則提供了 useCallback 這樣的方法，在有需要時，它可以幫我們把這個函式保存下來，讓它不會隨著每次組件重新執行後，因為作用域不同而得到兩個不同的函式。

[useCallback](https://reactjs.org/docs/hooks-reference.html#usecallback)

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

```js
 const fetchData = async () => {
    const [currentWeather, weatherForecast] = await Promise.all([
      fetchCurrentWeather(),
      fetchWeatherForecast(),
    ]);

    setWeatherElement({
      ...currentWeather,
      ...weatherForecast,
    });
  };

```

```js
// STEP 1：從 react 中載入 useCallback
import React, { useState, useEffect, useCallback } from 'react';

  // STEP 2：使用 useCallback 並將回傳的函式取名為 fetchData
  const fetchData = useCallback(() => {
    // STEP 3：把原本的 fetchData 改名為 fetchingData 放到 useCallback 的函式內
    const fetchingData = async () => {
      const [currentWeather, weatherForecast] = await Promise.all([
        fetchCurrentWeather(),
        fetchWeatherForecast(),
      ]);

      setWeatherElement({
        ...currentWeather,
        ...weatherForecast,
      });
    };

    // STEP 4：記得要呼叫 fetchingData 這個方法
    fetchingData();
    // STEP 5：因為 fetchingData 沒有相依到 React 組件中的資料狀態，所以 dependencies 陣列中不帶入元素
  }, []);

  useEffect(() => {
    console.log('execute function in useEffect');

    fetchData();

    // STEP 6：把透過 useCallback 回傳的函式放到 useEffect 的 dependencies 中
  }, [fetchData]);
```
