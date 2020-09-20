---
title: js的节流、反抖以及通过Vue自定义指令实现
date: 2020-07-20 13:39:04
comments: true
categories: 
- 记录
tags: 
- js
- vue
---



js的节流、反抖函数和通过Vue自定义指令实现节流、反抖

<!-- more -->

### 节流、反抖目的  

反抖和截流都是为了在指定时间内避免函数的重读执行，区别在于这个指定时间的等待是在函数执行之前还是函数执行之后下一次函数执行之前。

* 节流-throttle    

  函数执行后，在指定时间内不重复执行。就跟水龙头滴水一样，为了防止大量的水流流出造成浪费，将水龙头拧紧，第一滴水滴落下后要多一段时间第二滴水滴才会凝聚然后下落，若把水滴下落看作一次函数执行，那么第一滴水滴到第二滴水滴下落的时间就是节流函数不重复执行的指定时间。

  

* 反抖-debounce  

  在最后一次触发函数的指定时间后才执行函数，若在这段时间内再次触发函数，则重新等待指定时间后才执行函数。就像一个不稳定的机器，在他抖动不稳定时不能执行任务，需要观察一定时间稳定后才执行任务，若在观察时间内出现不稳定的情况则重新进行观察，等到稳定后才执行。

    

  ### js实现  

  
  
  * 节流实现    
  
```javascript 
        function(func, delay) {  
                  var prev = Date.now();
                  return function() {
                         var context = this;
                         var args = arguments;                
                         var now = Date.now();                
                         if (now - prev >= delay) {                    
                         		func.apply(context, args);                    
                         		prev = Date.now();                
                         		}            
                         	}        
                    }  
                    
    
```
 
通过将目标函数作为参数传入，在记录上一次执行时间的情况下判断当前时间与上一次执行时间之差是否超过指定的间隔时间来决定目标函数是否执行  
    
      
      
    
  
  * 反抖实现  
  
    
    
```javascript 
  			 function debounce(fn, wait) {
  			     var timeout = null;
  			     return function() {        
  			     		if(timeout !== null)                 
  			     		clearTimeout(timeout);        
  			     		timeout = setTimeout(fn, wait);
  			     		    }
  			      }  
  			      
  
```

通过对timeout的重置来实现目标函数的延迟执行，设置一个timeout来处理目标函数的延迟执行，若在目标函数执行之前再一次触发该函数，则将timeout重置，重新计算延迟时间  
  
### Vue自定义指令实现反抖、节流  
  
  通过Vue的directive方法定义自定义指令后在组件上实现点击事件的反抖和节流 ，通过binding对象获得目标函数即eventCb和等待时间，通过监听click时间来操作目标函数的执行与否
  
  Vue自定义指令的实现可以参考[官方文档](https://cn.vuejs.org/v2/guide/custom-directive.html)
  
  * 反抖  
  
   
  
```javascript
    Vue.directive('antiShake',{
      inserted:function (el,binding){
        const {eventCb, timeOut} = binding.value
        el.eventCb = eventCb
        el.timeOut = timeOut
        el.timerCall = null;
        el.addEventListener('click', ()=>{
          clearTimeout(el.timerCall)
          el.timerCall = setTimeout(() =>{
            el.eventCb()
          },el.timeOut||500)
    
        })
    
      },
      update:function (el,binding){
        console.log('update')
      }
    })  
    
```

   
  
  * 节流  
  
```javascript
	  Vue.directive('throttle',{
	    inserted:function (el,binding){
	      const{ eventCb, timeOut } = binding.value
	      el.eventCb = eventCb
	      el.timeOut = timeOut
	      el.addEventListener('click',() =>{
	        const nowTime = new Date().getTime()
	        if (!el.preTime || nowTime - el.preTime > el.timeOut){
	          el.preTime = nowTime
	          el.eventCb()
	        }
	      })
	    },
	    update:function (el,binding){
	      console.log('update')
	    }
	  })  
	  
```
在组件上定义一个目标函数对象绑定到自定义指令v-antishake和v-throttle  

```javascript

v-throttle={
	eventCb:function(){xxx},
	timeOut:1000
}

```


实现效果：  


![anti-shake-throttle](/images/imageForPost/vue/anti-shake-throttle.png)