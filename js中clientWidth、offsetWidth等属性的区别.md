<center><font face="黑体" size=6 >js中clientWidth和offsetWith等一些空间属性的区别</font></center>

以下列html样式代码为例：(默认情况下所有盒子都是content-box：盒子设置的宽高为内容的宽高，盒子真实的宽高还需要加上padding和边框的宽度)

```html
<html>
    <div class='viewbase'>
        <div class='views'>
        </div>
    </div>
    <style>
    .views{
        margin-top: 20px;
        height: 300px;
        width: 300px;
        border: solid 1px black;
        padding: 10px;
        margin: 10px;
    }
    .viewbase {
        height: 200px;
        width: 200px;
        border: solid 2px red;
        padding: 10px;
        margin: 10px;
        overflow: auto;
    }
    </style>
</html>
```

<center><font face="黑体" size=6 >在盒子模型为content-box条件下</font></center>

**clientHeight/clientWidth**

无滚动条：

clientHeight/clientWidth：height/width + (padding-top + padding-bottom)/(padding-left + padding-right) =  200 + 10 + 10 = 220

有滚动条:

clientHeight/clientWidth：height/width + (padding-top + padding-bottom)/(padding-left + padding-right) - y/x滚动条高度 = 200 + 10 + 10 - 17 = 203

**clientLeft/clientTop**

可以理解为左边框或者上边框的宽度（就是左/上padding和左/上margin的距离）

**offsetHeight/offsetWidth**（有无滚动条都一样）***ps: 不知为何border设置为1px时候算出来不是222，而是221***

offsetHeight/offsetWidth：height/width + (padding-top + padding-bottom)/(padding-left + padding-right)  +( 左右/上下)border = 200 + 10 + 10 + 2 + 2 = 224

**offsetTop/offsetLeft**

分别代表盒子距离**offsetParent**上边和左边的距离，而不是文档根元素。（offsetparent：**离元素最近，是元素的祖先元素，定位不为static**）position默认static

***`offsetParent`** 是一个指向最近的（指包含层级上的最近）包含该元素的定位元素或者最近的 `table`, `td`, `th`, `body` 元素。当元素的 `style.display` 设置为 "none" 或者position为fixed时，offsetparent为null*

**scrollTop/scrollLeft**

代表有滚动条的盒子向下或者向右**滚动了**多少距离

**scrollWidth/scrollHeight**（内容实际宽高加上padding）

代表盒子内容实际的宽度和高度（包含自身padding），例如上面例子：

scrollWidth（scrollHeight同理）：viewbase的宽 + viewbase的左右padding + viewbase的左右border宽 + viewbase的左右margin + views的左右padding

300 + 10 + 10 + 1 + 1 + 10 + 10 + 10 + 10 = 362

<center><font face="黑体" size=6 >在盒子模型为border-box条件下只列举有区别项</font></center>

**clientHeight/clientWidth**

无滚动条：

clientHeight/clientWidth：height/width - 上下/左右边框的宽  =  200 - 1 - 1 = 198

有滚动条:

clientHeight/clientWidth：height/width +上下/左右边框的宽 - y/x滚动条高度 = 200 + 10 + 10 - 17 = 203

**offsetHeight/offsetWidth**（有无滚动条都一样）

offsetHeight/offsetWidth：等于设置的宽高 offsetWidth = width

<center><font face="黑体" size=6 >坐标信息区别</font></center>

我们常用的鼠标事件中几乎都带有clientX/Y，pageX/Y，screenX/Y这一组坐标信息。

**clientX/Y**：相对于浏览器视口，其原点坐标在于**浏览器视口**的左上角

**pageX/Y**：相对于的是**整个页面（包括页面滚动后卷起的部分）**，原点坐标在于页面的左上角

**screenX/Y**：相对于我们的**屏幕**，原点坐标在屏幕左上角

<center><font face="黑体" size=6 >getBoundingClientRect</font></center>

**getBoundingClientRect()**会返回一个矩形盒子的一些相对于**浏览器视口**位置信息{ bottom，height ，left，right ，top，width，x，y}

width/height： 为盒子宽高（content-box和border-box有区别）

x/y = left/top： 表示矩形左上角相对于**浏览器视口**的坐标位置

bottom/right： 相当于矩形右下角对于**浏览器视口**的坐标位置

我们可以根据这个rect属性配合window.scrollX/window.scrollY计算出dom相对于文档页面的位置信息
