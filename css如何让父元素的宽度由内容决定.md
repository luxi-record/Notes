<center><font face="黑体" size=24 >css如何让父元素的宽度由内容决定</font></center>

##### 在css的盒子模型中，我们一般用到的盒子分为：块级元素（block），内联元素（inline）， 内联块级元素（inline-block）等。

* **块级元素** *让块级元素宽度脱离父元素影响*  **通过内联块级元素实现**

  ```html
  <div class='gf'>
      <div class='father'>
          <span>text</span>
          <span>text2</span>
      </div>
  </div>
  <style>
  .gf {
      width: 400px;
      height: 200px;
  }
  /* white-space阻止换行，father宽度为两个span之和 */
  .father {
      display: inline-block;
      white-space: nowrap;
  }
  .father span {
      display: inline-block;
  }
  </style>
  // 由于div默认是一个块级元素，而块级元素它的宽度默认会和父元素的一样，所以father的宽度为400px。
  // 此时我们只需要把div以及他所有一级子元素设置为内联块级元素即可，内联元素宽度不受父元素影响。 
  ```

* **块级元素**  *让块级元素宽度脱离父元素影响*  **通过弹性盒子实现**

  ```html
  <div class='gf'>
      <div class='father'>
          <span>text</span>
          <span>text2</span>
      </div>
  </div>
  <style>
  .gf {
      width: 400px;
      height: 200px;
  }
  /* father为一个弹性子元素宽度为类容宽度 */
  .gf {
      display: flex;
  }
  </style>
  // 把祖父元素设置为弹性盒子，这样div就是一个弹性子元素，其宽度不会默认和父元素一样不受父元素影响。
  ```
  
* **块级元素**  *让块级元素宽度脱离父元素影响*  **通过float实现**

  ```html
  <div class='gf'>
      <div class='father'>
          <span>text</span>
          <span>text2</span>
      </div>
  </div>
  <style>
  .gf {
      width: 400px;
      height: 200px;
      position: relative;
  }
  /* father设置为浮动 */
  .father {
      float: left;
  }
  </style>
  // 把元素设置为浮动元素，这样宽度也不会受到gf元素的影响，只不过如果father元素允许换行的话那么father最宽和gf元素一样
  ```

* **块级元素**  *让块级元素宽度脱离父元素影响*  **通过绝对定位实现**

  ```html
  <div class='gf'>
      <div class='father'>
          <span>text</span>
          <span>text2</span>
      </div>
  </div>
  <style>
  .gf {
      width: 400px;
      height: 200px;
      position: relative;
  }
  /* father设置为浮动 */
  .father {
      position: absolute;
  }
  </style>
  // 把元素设置为绝对定位，这样其宽度也不会受父元素gf的影响
  ```

* **内联元素或者内联块级元素的宽度为内容宽度**

#### 总结：

​		1. 想让元素的**宽度由内容决定**，可以把元素设置为**内联**元素或者**内联块级**元素。

​		2. 如果是**块级元素**想要**宽度由内容决定**，可以让块级元素的宽度**脱离父元素的影响**（**浮动，决对/固定定位，弹性盒子，内联块级**等）。

