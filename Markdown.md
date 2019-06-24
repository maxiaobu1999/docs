# Markdown #

## 1、 编辑器

Typora todo 下载链接

## 2、语法

### 2-1 段落与换行

段尾两个空格加回车换行  



### 2-2引用

> item
>
> > item1

### 2-3列表

* 使用*作为标记

+ 使用+作为标记

- 使用-作为标记    

  


1. 有序列表以数字和 `.` 开始；

2. 数字的序列并不会影响生成的列表序列；

3. 但仍然推荐按照自然顺序（1.2.3…）编写  

   

1. 第一层
   + 1-1
   + 1-2
2. 无序列表和有序列表可以随意相互嵌套
   1. 2-1
   2. 2-2 

05\. 可以使用：数字\. 来取消显示为列表

### 2.4 代码

```
new Object()
```

行内代码`new Object()`

代码高亮(拓展语法)  ```JS

```JS
window.addEventListener('load', function() {
  console.log('window loaded');
});
```



### 2.5 分割线

***

---

___

<hr>

### 2.6 超链接

普通链接：[Baidu](www.baidu.com)

指向本地文件的链接: [icon.png](./images/icon.png)

包含 'title' 的链接:[Google](http://www.google.com/ "Google")

自动链接：<123@email.com>

### 2.7 图片

插入图片的语法和插入超链接的语法基本一致，只是在最前面多一个 `!`。也分为行内式和参考式两种。

- 行内式

![GitHub](https://avatars2.githubusercontent.com/u/3265208?v=3&s=100"GitHub,Social Coding")

- 参考式

  ![GitHub][github]

  

  [github]: https://avatars2.githubusercontent.com/u/3265208?v=3&s=100	"GitHub,Social Coding"



- 指定图片的显示大小

  Markdown 不支持指定图片的显示大小，不过可以通过直接插入`<img />`标签来指定相关属性：

  <img src="https://avatars2.githubusercontent.com/u/3265208?v=3&s=100" alt="GitHub" title="GitHub,Social Coding" width="50" height="50" />

### 2.8 强调

1. 使用 `* *` 或 `_ _` 包括的文本会被转换为 `<em></em>` ，通常表现为斜体：

   这是用来 *演示* 的 _文本_ 

2. 使用 `** **` 或 `__ __` 包括的文本会被转换为 `<strong></strong>`，通常表现为加粗：

   这是用来 **演示** 的 __文本__

3. 用来包括文本的 `*` 或 `_` 内侧不能有空白，否则 `*` 和 `_` 将不会被转换（不同的实现会有不同的表现）：
这是用来 ** 演示** 的 __文本  __

4. 如果需要在文本中显示成对的 `*` 或 `_`，可以在符号前加入 `\` 即可：

   这是用来 \*演示\* 的\ _文本\_

### 2.9 字符转译

反斜线（`\`）用于插入在 Markdown 语法中有特殊作用的字符。

```
\
`
*
_
{}
[]
()
#
+
-
.
!
```

### 3.0 删除线 

这就是~~删除线~~

### 3.1 表格

- 使用 `|` 来分隔不同的单元格，使用 `-` 来分隔表头和其他行：

  ```markdown
  name | age
  ---- | ---
  LearnShare | 12
  Mike |  32
  ```

- 为了美观，可以使用空格对齐不同行的单元格，并在左右两侧都使用 `|` 来标记单元格边界：

  ```markdown
  |    name    | age |
  | ---------- | --- |
  | LearnShare |  12 |
  | Mike       |  32 |
  ```

- 对齐

  - `:---` 代表左对齐
  - `:--:` 代表居中对齐
  - `---:` 代表右对齐

  ```markdown
  | left | center | right |
  | :--- | :----: | ----: |
  | aaaa | bbbbbb | ccccc |
  | a    | b      | c     |
  ```

- 表格中可以插入其他 Markdown 中的行内标记：

  ```markdown
  |     name     | age |             blog                |
  | ------------ | --- | ------------------------------- |
  | _LearnShare_ |  12 | [LearnShare](http://xianbai.me) |
  | __Mike__     |  32 | [Mike](http://mike.me)          |
  ```

### 3.2 锚点

- 任意 1-6 个 **#** 标注的标题都会被添加上同名的锚点链接
- 无论目标几个#，都只用一个#
- [参考资料](https://my.oschina.net/antsky/blog/1475173)

[锚点](#3.1 表格)