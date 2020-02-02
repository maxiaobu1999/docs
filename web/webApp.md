# Tomcat

## 1、Viewport

```html
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no">
```

- width：设置布局viewport的特定值("device-width"布局宽度=设备宽度)
- Initial-scale:设置页面的初始缩放（自带width=device-width效果）
- minimum-scale：最下缩放
- maximum-scale：最大缩放
- user-scalable：用户能否缩放

Document.body.clientWidth:布局宽度

window.innerWidth：度量宽度

Note:

## Flexbox弹性盒子模型
###属性
[ flex-direction ]方向
flex-direction: row;  水平
flex-direction: row-reverse;  水平反方向
flex-direction: column; 垂直
flex-direction: column-reverse; 垂直反方向
[ flex-wrap ] 子元素换行
[ justify-content ]ViewGroup使用，子元素排版，layout=“center”
[ align-self ]view使用，子元素排版
[ align-items ] align-items: stretch;填满整个容器
[ align-content ]子元素换行
[ order ]排序

###兼容性
建议使用旧版的flexbox方案，也无所谓了
- iOS无问题
- Android 4.4以下，只能兼容旧版的flexbox布局
- Android 4.4及以上，可以兼容最新的flex布局

### 不定宽高的水平垂直居中

```css
.myoff-wrapper {
  position:absolute; //绝对定位
  top:50%;
  left:50%;
  z-index:3;
  -webkit-transform:translate(-50%,-50%);
  border-radius:6px;
  background:#fff;
}

```

【flexbox版】不定宽高的水平垂直居中

```css
.parent {
  justify-content:center; //子元素水平居中
  align-items:center; //子元素垂直居中
  display:-webkit-flex;
}
```
###响应式布局
屏幕尺寸改变布局改变

####媒体查询

类似设置不同layout.xml

- 媒体类型

screen屏幕、print打印机、handheld手持设备、all通用

- 常用媒体查询参数：

width-视口宽度

height 视口高度

device-width 设备的宽度

device-height 设备的高度

orientation 检查设备处于横向（landscape）还是竖屏（portait）

```css
	// screen媒体类型
	// (max-width:1024px)条件
	@media screen and (max-width:1024px) {
			//pagewrap 样式体
 			#pagewrap {
            width: 95.5%;// 屏幕宽度小于1024，宽度为95.5%
        }
      #content {
            width: 62%;
        }
      #content .article .hr{
            width: 66%;
            margin-left: 34%;
        }
	}
```

####设计点1:百分比布局

设计点2:弹性图片，图片百分比

设计点3:重新布局，显示与隐藏

## 移动web特别样式处理

高清图片：使用dp

1像素边框：Y轴缩放：sacleY(0.5)

相对单位：em：根据父节点的font-size为相对单位；rem：根据html的font-size为相对单位;类似设置demains.xml

文本溢出

```css
//单行文本溢出
.inaline {
  overflow:hidden;
  white-space:nowrap;
  text-overflow:ellipsis;
}
// 多行文本溢出
.intwoline{
  display:-webkit-box !important;
  overflow:hidden;
  text-overflow:ellipsis;
  word-break:break-all;
  
  -webkit-box-orient:vertical;
  -webkit-line-clamp:2;
}
```

