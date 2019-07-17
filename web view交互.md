# web view交互

## 1native调用js

- loadUrl

  ```java
  // 注意调用的JS方法名要对应上
  // 调用javascript的callJS()方法
  mWebView.loadUrl("javascript:callJS()");	
  // Android需要调用的方法
     function callJS(){
        alert("Android调用了JS的callJS方法");
     }
  ```

  

- evaluateJavascript

  - 该方法的执行不会使页面刷新，而第一种方法（loadUrl ）的执行则会。
  - Android 4.4 后才可使用

  ```java
  // 只需要将第一种方法的loadUrl()换成下面该方法即可
      mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
          @Override
          public void onReceiveValue(String value) {
              //此处为 js 返回的结果
          }
      });
  }
  ```

  ##2、双向通信

  ​	js调用native方法，native执行完得返回值，

  ​	native调js，js接收返回值

  ## 3、jsbridge 原理

  消息队列处理事件

  Map<句柄，callback>保存，通过句柄