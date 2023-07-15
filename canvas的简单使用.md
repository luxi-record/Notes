<center><font face="黑体" size=24 >关于Canvas的一些使用</font></center>

##### Canvas画布，我们可以通过Canvas画出各种各样的东西，也可以通过canvas实现图片裁切，图片像素转化，以及一些视频处理操作。

* **Canvas常用的方法**

  ```html
  <!DOCTYPE html>
  <html>
  <head>
      <meta charset="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>canvas</title>
  </head>
  <body>
      <canvas id="canvas" width="200" height="200"></cacanvasnvas>
  </body>
  <script type="text/javascript">
      const canvas = document.getElementById('canvas'), ctx = cas.getContext('2d')
  	ctx.beginPath（）//开始新的绘画路径（相当于保存之前的绘画，可以把之前绘画的一些配置（fillstyle这些）进行修改以绘制新的图画）
  	ctx.moveTo（x，y）//把画笔抬起来离开我的画布移动到x，y（理解这个抬起来）
  	ctx.lineTo（x，y）	//把画笔移动到x，y（不用抬起画笔所以就会形成一条线，此时画笔点在x，y）
  	ctx.stroke（）	//开始画线（把我刚才移动画笔时候形成线画出来）
  	ctx.closePath（）//形成封闭，比如画三角形，我们可以先画两条线，最后调用closePath（）会自动绘制最后一条线形成闭合
  	ctx.fill()		//填充形成的封闭区域
  	ctx.strokeStyle = ‘’ 	//画笔颜色
  	ctx.fillStyle =‘’	//填充颜色，fillRect（），fillArc（）填充一个正方形或者圆，fillText（txt，x,y）画字
  	ctx.rect（x，y，w，h）	//起点x，y画一个宽w和高h的正方形
  	ctx.fillRect（x,y,w,h）	//填充一个矩形区域
  	ctx.arc（x, y, r, start, end，Boolean）//画⚪, 圆心x，y，半径，开始角，结束角度，是否逆时针
  	ctx.clearRect（x，y，w，h）	//清除画布的一个正方形区域起点事x，y，宽w和高h
  	ctx.lineWidth = ‘’	// 线条宽
  	ctx.scale（w，h）	//缩放当前绘制的宽和高
  	ctx.save()		//保存当前状态到栈里面（状态包括设置的stokeStyle，filleStyle和一些转化等，不包含绘画内容）
  	ctx.restore()		// 返回上一次save（）保存的状态
      ctx.rotate(angle) // 图像以原点旋转多少度
      ctx.scale(x, y) // 图像水平和竖直缩放多少
      ctx.translate(x, y) // 图像水平和垂直移动多少位置
  </script>
  </html>
  ```

* **使用Canvas处理图片像素（灰阶处理或者图片裁剪）**，原理：主要用到canvas的drawImage、getImageData、putImageData、toDataURL这几个方法

  1. **ctx.drawImage()**（x，y坐标系中原点坐标都在左上角）

     该方法可以用于在canvas里绘制图片和视频帧等，它有三种使用方式：

     ctx.drawImage(img, x, y)  表示以canvas的x和y为起点绘制图片img（是Image对象）
  
     ctx.drawImage(img, x, y,width,height )表示以canvas的x和y为起点绘制一个宽为width高为height的图片img(绘制的图片是完整的只是会压缩)
  
     ctx.drawImage(img, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight);表示先以图片sx和sy为起点裁剪一个宽为swidth高为sheight的图片绘画在以canvas的dx和dy为起点宽为width高为height的区域（图片也会压缩）
  
  2. **ctx.getImageData(x,y,w,h)**
  
     表示获取以canvas的x和y为起点宽为w高为h区域的图像的二进制数据，返回一个数组，数组每四位代表一个像素点，这四位数分别是rgba的参数
  
  3. **ctx.putImageData(data, x, y)**
  
     表示把data数据绘制在canvas上以x和y为起点，data为数组和getImageData获取的数据格式一样
  
  4. **canvas.toDataURL(file.type)**
  
     把canvas图转化为一个url，类似于base64格式
  
  ```html
  <!DOCTYPE html>
  <html>
  <head>
      <meta charset="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>图片</title>
  </head>
  <body>
      <canvas hidden id="canvas" width="200" height="200"></canvas>
      <input type="file" id="file" onchange="fileChange(event)" />
  </body> 
      <script type='text/javascript'>
          function showGrayImage(files) {
              const canvas = document.getElementById('canvas'), ctx = canvas.getContext("2d")
              const fm = document.createDocumentFragment()， translateData = []
              while (files.length) {
                  const img = new Image(), file = files.shift(), imageItem = document.createElement('img')
                  if (file.size === 0) continue
                  img.src = URL.createObjectURL(file)
                  img.onload = function () {
                      ctx.clearRect(0, 0, canvas.width, canvas.height) // 清理画布
                      ctx.drawImage(img, 0, 0, (canvas.height / img.height) * img.width, canvas.height) // 绘图，按比例不然图像要变形
                      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height) // 获取像素数据
                      const data = imageData.data;
                      for (let i = 0; i < data.length; i += 4) { // 改变像素
                          const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
                          data[i] = avg;
                          data[i + 1] = avg;
                          data[i + 2] = avg;
                      }
                      ctx.putImageData(imageData, 0, 0); // 把改变的像素绘制在canvas上
                      translateData.push({ [file.name]: canvas.toDataURL(file.type) })
                      imageItem.src = canvas.toDataURL(file.type) // 把绘制后的图转化为url
                      imageItem.alt = imageItem.title = file.name
                      fm.appendChild(imageItem)
                      if (files.length === 0) {
                          document.body.appendChild(fm)
                      }
                  }
              }
          }
          function file_Change(e) {
              const input_file = []
              const event = e || window.event
              if (event?.target?.files) {
                  for (let file of event.target.files) {
                      if (file.type === 'image/png' || file.type === 'image/jpeg') {
                          input_file.push(file)
                      }
                  }
              }
              if (input_file.length) {
                  showGrayImage(input_file)
              }
      	}
      </script>
  </html>
  ```
  
* **一些有意思的canvas方法**

  1. **ctx.clip()**,裁剪绘画区域，**裁剪后绘制的图像**只有和这个区域相交才会显示，注意裁剪区域必须是闭合的

     ```javascript
     // 探照灯效果
     var canvas = document.getElementById('canvas'); 
     var ctx = canvas.getContext('2d'); 
     var w = canvas.width = 1080; 
     var h = canvas.height = 800; 
     var image = new Image(); 
     image.src = 'https://farm3.staticflickr.com/2908/32764885503_1a04915b11_k.jpg'; 
     canvas.addEventListener('mousemove', mouseMove, false);
     function draw(cx, cy, radius) { 
     	ctx.clearRect(0, 0, w, h);
     	ctx.save();
       	ctx.beginPath();
     	ctx.fillStyle = '#FFF'; 
     	ctx.fillRect(0, 0, w, h); 
     	ctx.fill(); 
     	ctx.save(); 
     	ctx.beginPath();
       	ctx.fillStyle="#FFF";
     	ctx.arc(cx, cy, radius, 0, Math.PI * 2, false); 
     	ctx.fill(); 
     	// 将上面的区域作为剪辑区域
       	ctx.clip(); 
     	// 由于使用clip()，画布背景图片会出现在clip()区域内 
     	ctx.drawImage(image, 0, 0); 
     	ctx.restore(); 
     }
     function mouseMove(e) {
       	e.preventDefault();
     	e.stopPropagation();
     	mouseX = parseInt(e.clientX);
       	mouseY = parseInt(e.clientY);
       	draw(mouseX, mouseY, 100);
     }
     draw(w / 2, h / 2, 100);
     ```
