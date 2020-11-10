# 了解防抖与节流



在前端开发的过程中，我们经常需要绑定一些持续触发的事件，如 resize、scroll、mousemove 等，然而有些时候我们并不希望在事件持续触发的过程中频繁地去执行函数，这时候就会用到函数防抖(Debounce)与节流(Throttle)：

- 防抖：在触发事件 n 秒后才执行函数，如果在 n 秒内又触发了事件，则重新计算时间。

- 节流：连续触发事件时在 n 秒中只执行一次函数，节流的目的是稀释函数的执行频率。

下面我们简单地了解这两种限制函数执行次数的具体实现方式。首先看这个例子：

```html
<button type="submit" id="btn">提交</button>
<script>
    var btn = document.getElementById('btn');
    btn.addEventListener('click', submit, false);
    function submit(){
        console.log('submit');
    }
</script>
```

每次 button 的点击事件都会执行 submit 函数，我们如何限制 submit 的执行次数？

## Debounce

防抖是在事件多次触发时让函数只执行一次，有非立即执行和立即执行两种实现。

### 闭包实现防抖

用闭包可以实现一个简单的 debounce 函数来包装 submit 函数，实现防抖效果。

```js
var btn = document.getElementById('btn');
btn.addEventListener('click', debounce(submit, 1000), false);
function submit(){
    console.log('submit');
}
function debounce(fn, timer){     
    let t = null;    
    return function(){
    // 计时未到 timer 的定时器会被清理，函数就不会执行        
        if(t){             
            clearTimeout(t);         
        };        
        t = setTimeout(fn, timer);     
    };
}
```

连续点击时，始终从最新一次点击开始计时。直到不再点击的 1s 后，才会执行一次 submit。

### 传递 this 和 event

当我们在事件监听中绑定 submit 时，this 指向 button，并且可以拿到 MouseEvent 事件。

```js
var btn = document.getElementById('btn');
btn.addEventListener('click', submit, false);
function submit(){
    console.log('submit');
    console.log(this);       // <button type="submit" ...
    console.log(arguments);  // Arguments [MouseEvent ...
}
```

现在我们的 submit 被 debounce 包装，此时 this 和事件参数要从 debounce 传递给 submit。

```js
var btn = document.getElementById('btn');
btn.addEventListener('click', debounce(submit, 1000), false);
function submit(){
    console.log('submit');
    console.log(this);       // <button type="submit" ...
    console.log(arguments);  // Arguments [MouseEvent ...
}
function debounce(fn, timer){     
    let t = null;    
    return function(){    
        let context = this;
        let args = arguments;    
        if(t) clearTimeout(t);                 
        t = setTimeout(()=>{
            fn.apply(context, args); 
        }, timer);     
    }
}
```

这就是一个非立即执行的防抖实现，缺陷是第一次点击也需要等待 1s 后才会有反馈。

### 立即执行版本

第一次点击时让函数立即执行。不再点击的 1s 后，新的点击将成为第一次点击，再次执行函数。

```js
function debounce(fn, timer){     
    let t = null;  
    return function(){    
        let context = this;
        let args = arguments;    
        let firstClick = !t;
        if(t) clearTimeout(t);     
        if(firstClick) {
            fn.apply(context, args); 
        };   
        t = setTimeout(()=>{
            t = null;
        }, timer);     
    }
}
```

## Throttle

节流的实现方式是减少函数的触发频率，同样有非立即执行和立即执行两种实现。

### 定时器版本

非立即执行我们自然想到用定时器实现：

```js
function throttle(fn, timer) {
    let t = null;
    return function() {
        let context = this;
        let args = arguments;
        // 计时未到 timer 时不会执行函数
        if (!t) {
            t = setTimeout(() => {
                    fn.apply(context, args)
                    t = null;
                }, timer)
        }
    }
}
```

### 时间戳版本

立即执行我们要计算时间差，所以用时间戳实现：

```js
function throttle(fn, timer){
    let begin = 0;
    return function(){
        let now = new Date().getTime();
        if (now - begin > timer) {
            fn.apply(this, arguments);
            begin = now;
        }
    }
}
```

实际上，结合非立即执行和立即执行的两种实现方式，你可以构造出具有更多功能的防抖函数和节流函数，满足业务需求。

**参考资料**

- [性能优化之防抖和节流](https://juejin.im/post/6844903888978444296)
