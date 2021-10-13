---
title: "Class Component Input Bug"
author: [jacobhsu]
---

函數式組件和類組件之間有一個非常重要的區別：**函數式組件捕獲了渲染所使用的值**

函數式組件可以理解為一個能返回React元素的函數，其接收一個代表組件屬性的參數`props`。

在React16.8之前，也就是沒有React Hooks之前，函數式組件只作為UI組件，其輸出完全由參數props控制，沒有自身的狀態沒有業務邏輯代碼，是一個純函數。函數式組件沒有實例，沒有生命週期，稱為無狀態組件。

在React Hooks出現後，可以用Hook賦予函數式組件狀態和生命週期，於是函數式組件也可以作為業務組件。

開發過程中，類組件和函數式組件都有使用，經過六個月的開發，感覺還是函數式組件比類組件好用一些，感受最深的是以下兩點：

* 不用去學習class，不用去管煩人的this指向問題；
* 復用性高，很容易就把共同的抽取出來，寫出自定義Hook，來替代高階組件。

BUG的場景：一個輸入框，輸入完內容，點擊按鈕搜索，搜索時先請求一個接口，獲取一個類型，再用類型和輸入框值去請求搜索接口

```js
import React, { Component } from "react";
import * as API from 'api/list';
class SearchComponent extends Component {
  constructor() {
    super();
    this.state = {
      inpValue: ""
    };
  }
  
  getType () {
    const param = {
      val:this.state.inpValue
    }
    return API.getType(param);
  }
  
  getList(type){
    const param = {
      val:this.state.inpValue,
      type,
    }
    return API.getList(param);
  }
  
  async handleSearch() {
    const res = await this.getType();
    const type = res?.data?.type;
    const res1 = await this.getList(type);
    console.log(res1);
  }
  
  render() {
    return (
      <div>
        <input
          type="text"
          value={this.state.inpValue}
          onChange={(e) => {
            this.setState({ inpValue: e.target.value });
          }}
        />
        <button
          onClick={() => {
            this.handleSearch();
          }}
        >
          搜索
        </button>
      </div>
    );
  }
}
export default SearchComponent;
```


BUG：在輸入框輸入要搜索的內容後，點擊搜索按鈕開始搜索，然後很快在輸入框中又輸入內容，結果搜索接口getList報錯。查一下原因，發現是獲取類型接口getType和搜索接口getList接受的參數val不一致。明明兩次請求中val都是讀取this.state.inpValue的值。

函數式組件就可解決這個BUG

```js
import React, { Component } from "react";
import * as API from 'api/list';
class SearchComponent extends Component {
  const [inpValue,setInpValue] = useState(''); // add

  constructor() {
    super();
    this.state = {
      inpValue: ""
    };
  }
  
  getType () {
    const param = {
      val:inpValue // val:this.state.inpValue
    }
    return API.getType(param);
  }
  
  getList(type){
    const param = {
      val:inpValue, // val:this.state.inpValue,
      type,
    }
    return API.getList(param);
  }
  
  async handleSearch() {
    const res = await this.getType();
    const type = res?.data?.type;
    const res1 = await this.getList(type);
    console.log(res1);
  }
  
  render() {
    return (
      <div>
        <input
          type="text"
          value={this.state.inpValue}
          onChange={(e) => {
            // this.setState({ inpValue: e.target.value });
            setInpValue(e.target.value);
          }}
        />
        <button
          onClick={() => {
            this.handleSearch();
          }}
        >
          搜索
        </button>
      </div>
    );
  }
}
export default SearchComponent;
```

函數式組件中的事件的state和props所獲取的值是事件觸發那一刻頁面渲染所用的state和props的值。
當點擊搜索按鈕後，`val`的值就是那一刻輸入框中的值，無論輸入框後面的值在怎麼改變，不會捕獲最新的值。

那為啥類組件中，能獲取到最新的state值呢？關鍵在於類組件中是通過`this`去獲取state的，而`this`永遠是最新的組件實例。

類組件的改法

```js
import React, { Component } from "react";
import * as API from 'api/list';
class SearchComponent extends Component {
  constructor() {
    super();
    this.state = {
      inpValue: ""
    };
  }
  
  getType () {
    const param = {
       val, // val:this.state.inpValue
    }
    return API.getType(param);
  }
  
  getList(type){
    const param = {
      val, // val:this.state.inpValue,
      type,
    }
    return API.getList(param);
  }
  
  async handleSearch() {
    const inpValue = this.state.inpValue; // add 
    const res = await this.getType(inpValue);
    const type = res?.data?.type;
    const res1 = await this.getList(inpValue, type);
    console.log(res1);
  }
  
  render() {
    return (
      <div>
        <input
          type="text"
          value={this.state.inpValue}
          onChange={(e) => {
            this.setState({ inpValue: e.target.value });
          }}
        />
        <button
          onClick={() => {
            this.handleSearch();
          }}
        >
          搜索
        </button>
      </div>
    );
  }
}
export default SearchComponent;
```

在搜索事件`handleSearch`觸發時，就把輸入框的值this.state.inpValue存在`inpValue`變量中，後續執行的事件用到輸入框的值都去inpValue變量取，
後續再往輸入框輸入內容也不會影響到inpValue變量的值，除非再次觸發搜索事件`handleSearch`。這樣修改也可以解決這個BUG。

References: 
[Function Component 與 Class Component](https://zh-hant.reactjs.org/docs/components-and-props.html)
[關於函數式組件的收獲](https://juejin.cn/post/7018328359742636039)