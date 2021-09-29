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
