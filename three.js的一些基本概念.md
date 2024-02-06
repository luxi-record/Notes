## three.js的一些重要概念

1. **虚拟场景**  ***const scene =  new THREE.Scene()***

   * 在three.js里面我们首先需要创建一个虚拟空间，然后在这个虚拟空间里面添加虚拟物体Mesh，three.js提供各种各样的物体模型类。

   * 我们可以通过 ***const box = new  THREE.BoxGeometry(x,y,z)***创建一个正方体实例，通过参数创建想要的长宽高等。

   * 创建完物体后，我们可以给这个物体添加材质 ***const material = new THREE.MeshBasicMaterial({color: 0xff0000})***，也就是物体的一些外表设置等。three.js提供各种各样的材质（基础网格，漫反射网格，点材质，线材质等）。

   * 基于创建的物体和**材质**我们可以把两者结合组成一个有材质的虚拟物体，***const mesh  = new THREE.Mesh(box,material)***

   * 此时我们可以设置这个虚拟物体在虚拟空间的位置等属性，***mesh.position.set(x,y,z)***

   * 有了虚拟物体，我们就可以把虚拟物体放在我们的虚拟空间中，***scene.add(mesh)***

   * ### 我们创建的虚拟的物体都需要通过scene.add()添加到我们的虚拟空间中
   * ### const axesHelper = new THREE.AxesHelper(150); scene.add(axesHelper)添加开发时候辅助坐标

2. **虚拟相机**

   * 虚拟相机，实质就是模拟人眼去观察我们创建的虚拟物体，但是和我们正真人眼去看是有区别的，three.js有正投影相机***OrthographicCamera***和透视投影相机***PerspectiveCamera***。通常用的是透视相机。

   * 在创建相机前，我们需要先了解一个概念叫做***视锥体***，只有在这个视锥体里面的虚拟物体才会被绘制到我们的页面上。在我们定义相机时候会传入四个参数***const camera = new THREE.PerspectiveCamera(30, width / height, 1, 3000)***，第一个参数是观察的角度大小，第二个是输出画布（渲染器render返回的canvas）的宽高比，第三个参数是近裁剪面到相机的距离，第四个参数是远裁剪面到相机的距离。近裁剪面加上远裁剪面和观察角度会形成一个空间，这个空间就是***视锥体***。（想象下你只能观察一定的角度，此时离你近端有一个长方形面，那么你的视线和面的四个点会形成一个发散的空间，这个空间可以无限远，此时我们的远裁剪面就规定了我们最远可以发散到多远，此时近裁剪面和远裁剪面之间就会形成一个空间）。**如果虚拟物体没有在我们的视锥体内是绘制不出来的**

     ![视锥体](http://www.webgl3d.cn/threejs/%E8%A7%86%E9%94%A5%E4%BD%93.png)

   * 创建了虚拟相机我们可以设置相机的拍照位置和拍照的方向。***camera.position.set(x,y,z)***。***camera.lookAt(x,y,z)***，如果想把拍照的方向设置为你创建的虚拟物体的方向，可以通过***camera.lookAt(mesh.position)***控制

3. **渲染器**

   * 渲染器就相当于我们在相机中执行了拍照操作。***const renderer = new THREE.WebGLRenderer()***
   * 有了要观察到物体和相机，此时我们就可以绘制我们的观察结果。***renderer.render(scene,camera)***，render会绘制结果，然后生成一个canvas画布在其domElement属性上。我们可以通过***renderer.setSize(width, height)***设置画布大小，**通常就是在创建相机时候设置的宽高比来的**
   * 得到绘制结果了我们就可以把画布展示在我们的页面上。***document.body.appendChild(renderer.domElement)***

4. **光源**

   * 光源对我们创建的虚拟物体有着很大的影响，但是不同的**材质**受到的影响也不一样，**MeshBasicMaterial 基础材质不受光源影响**，受影响的有漫反射材质***MeshLambertMaterial***，高光材质，物理材质等。在three.js中有不同的光源，包括点光源（模拟成一个发光的点会向四周发散），平行光源（不发散的一组平行线），聚光灯光源（一个点往某个方向发散的光），环境光源等。
   * ***const pointLight = new THREE.PointLight(0xffffff, 1.0)***（点光源），创建光源时候可以设置光的颜色，光照强度。还可以设置光是否随着距离而衰减，***pointLight.decay = 0.0***，0表示不会。***pointLight.position.set(x, y, z)***设置光在虚拟空间的位置。***scene.add(pointLight)***添加进我们的虚拟空间让它生效。
