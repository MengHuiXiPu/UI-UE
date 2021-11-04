# Three.js 建立入门思维20211104

## Three.js的基础知识

想象一下，在一个虚拟的3D世界中都需要什么？**首先，要有一个立体的空间，其次是有光源，最重要的是要有一双眼睛**。下面我们就看看在three.js中如何创建一个3D世界吧！

1. 创建一个场景
2. 设置光源
3. 创建相机，设置相机位置和相机镜头的朝向
4. 创建3D渲染器，使用渲染器把创建的场景渲染出来

此时，你就通过three.js创建出了一个可视化的3D页面，很简单是吧！

### 关于场景

你可以为场景添加背景颜色，或创建一个盒模型（球体、立方体），给盒模型的内部贴上图片，再把相机放在这个盒模型内部以达到模拟场景的效果。盒模型的方式多用于360度全景，比如房屋vr展示

【登陆页面】创建场景的例子：

```
const scene = new THREE.Scene()
// 在场景中添加雾的效果，Fog参数分别代表‘雾的颜色’、‘开始雾化的视线距离’、刚好雾化至看不见的视线距离’
scene.fog = new THREE.Fog(0x000000, 0, 10000)
// 盒模型的深度
const depth = 1400
// 在场景中添加一个圆球盒模型
// 1.创建一个立方体
const geometry = new THREE.BoxGeometry(1000, 800, depth)
// 2.加载纹理
const texture = new THREE.TextureLoader().load('bg.png')
// 3.创建网格材质（原料）
const material = new THREE.MeshBasicMaterial({map: texture, side: THREE.BackSide})
// 4.生成网格
const mesh = new THREE.Mesh(geometry, material)
// 5.把网格放入场景中
scene.add(mesh)
复制代码
```

### 关于光源

为场景设置光源的颜色、强度，同时还可以设置光源的类型（环境光、点光源、平行光等）、光源所在的位置

【登陆页面】创建光源的例子：

```
// 1.创建环境光
const ambientLight = new THREE.AmbientLight(0xffffff, 1)
// 2.创建点光源，位于场景右下角
const light_rightBottom = new THREE.PointLight(0x0655fd, 5, 0)
light_rightBottom.position.set(0, 100, -200)
// 3.把光源放入场景中
scene.add(light_rightBottom)
scene.add(ambientLight)
复制代码
```

### 关于相机（重要）

很重要的一步，相机就是你的眼睛。**这里还会着重说明一下使用透视相机时可能会遇到的问题**，我们最常用到的相机就是正交相机和透视相机了。

正交相机：无论物体距离相机距离远或者近，在最终渲染的图片中物体的大小都保持不变。**用于渲染2D场景或者UI元素是非常有用的**。如图：

**图注解：**

1. 图中红色三角锥体是视野的大小
2. 红色锥体连着的第一个面是摄像机能看到的最近位置
3. 从该面通过白色辅助线延伸过去的面是摄像机能看到的最远的位置

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

透视相机：被用来模拟人眼所看到的景象。**它是3D场景的渲染中使用得最普遍的投影模式**。如图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

我们在使用透视相机时，可能会遇到这种情况：边缘处的物体会产生一定程度上的形变，原因是：**透视相机是鱼眼效果，如果视域越大，边缘变形越大。为了避免边缘变形，可以将fov角度设置小一些，距离拉远一些**

关于透视相机的几个参数，`new THREE.PerspectiveCamera(fov, width / height, near, far)`

- fov（field of view） — 摄像机视锥体垂直视野角度
- aspect（width / height） — 摄像机视锥体长宽比
- near — 摄像机视锥体近端面
- far — 摄像机视锥体远端面

```
/**
 * 为了避免边缘变形，这里将fov角度设置小一些，距离拉远一些
 * 固定视域角度，求需要多少距离才能满足完整的视野画面
 * 15度等于(Math.PI / 12)
 */
const container = document.getElementById('login-three-container')
const width = container.clientWidth
const height = container.clientHeight
const fov = 15
const distance = width / 2 / Math.tan(Math.PI / 12)
const zAxisNumber = Math.floor(distance - depth / 2)
const camera = new THREE.PerspectiveCamera(fov, width / height, 1, 30000)
camera.position.set(0, 0, zAxisNumber)
const cameraTarget = new THREE.Vector3(0, 0, 0)
camera.lookAt(cameraTarget)
复制代码
```

### 关于渲染器

用**WebGL**[1]渲染出你精心制作的场景。它会创建一个canvas进行渲染

【登陆页面】创建渲染器的例子：

```
// 获取容器dom
const container = document.getElementById('login-three-container')
// 创建webgl渲染器实例
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
// 设置渲染器画布的大小
renderer.setSize(width, height)
// 把画布实例（canvas）放入容器中
container.appendChild(renderer.domElement)
// 渲染器渲染场景
renderer.render(scene, camera)
复制代码
```

需要注意，这样创建出来的场景并没有动效，原因是这次渲染的仅仅只是这一帧的画面。为了让场景中的物体能动起来，我们需要使用requestAnimationFrame，所以我们可以写一个loop函数

```
//动画刷新
const loopAnimate = () => {
    requestAnimationFrame(loopAnimate)
    scene.rotateY(0.001)
    renderer.render(scene, camera)
}
loopAnimate()
复制代码
```

## 完善效果

### 创建一个左上角的地球

```
// 加载纹理
const texture = THREE.TextureLoader().load('earth_bg.png')
// 创建网格材质
const material = new THREE.MeshPhongMaterial({map: texture, blendDstAlpha: 1})
// 创建几何球体
const sphereGeometry = new THREE.SphereGeometry(50, 64, 32)
// 生成网格
const sphere = new THREE.Mesh(sphereGeometry, material)
// 为了单独操作球体的运动效果，我们把球体放到一个组中
const Sphere_Group = new THREE.Group()
const Sphere_Group.add(sphere)
// 设置该组（球体）在空间坐标中的位置
const Sphere_Group.position.x = -400
const Sphere_Group.position.y = 200
const Sphere_Group.position.z = -200
// 加入场景
scene.add(Sphere_Group)
// 使球能够自转，需要在loopAnimate中加上
Sphere_Group.rotateY(0.001)
复制代码
```

### 使地球自转

```
// 渲染星球的自转
const renderSphereRotate = () => {
    if (sphere) {
      Sphere_Group.rotateY(0.001)
    }
}
// 使球能够自转，需要在loopAnimate中加上
const loopAnimate = () => {
    requestAnimationFrame(loopAnimate)
    renderSphereRotate()
    renderer.render(scene, camera)
}
复制代码
```

### 创建星星

```
// 初始化星星
const initSceneStar = (initZposition: number): any => {
    const geometry = new THREE.BufferGeometry()
    const vertices: number[] = []
    const pointsGeometry: any[] = []
    const textureLoader = new THREE.TextureLoader()
    const sprite1 = textureLoader.load('starflake1.png')
    const sprite2 = textureLoader.load('starflake2.png')
    parameters = [
      [[0.6, 100, 0.75], sprite1, 50],
      [[0, 0, 1], sprite2, 20]
    ]
    // 初始化500个节点
    for (let i = 0; i < 500; i++) {
      /**
       * const x: number = Math.random() * 2 * width - width
       * 等价
       * THREE.MathUtils.randFloatSpread(width)
       * _.random使用的是lodash库中的生成随机数
       */
      const x: number = THREE.MathUtils.randFloatSpread(width)
      const y: number = _.random(0, height / 2)
      const z: number = _.random(-depth / 2, zAxisNumber)
      vertices.push(x, y, z)
    }

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))

    // 创建2种不同的材质的节点（500 * 2）
    for (let i = 0; i < parameters.length; i++) {
      const color = parameters[i][0]
      const sprite = parameters[i][1]
      const size = parameters[i][2]

      materials[i] = new THREE.PointsMaterial({
        size,
        map: sprite,
        blending: THREE.AdditiveBlending,
        depthTest: true,
        transparent: true
      })
      materials[i].color.setHSL(color[0], color[1], color[2])
      const particles = new THREE.Points(geometry, materials[i])
      particles.rotation.x = Math.random() * 0.2 - 0.15
      particles.rotation.z = Math.random() * 0.2 - 0.15
      particles.rotation.y = Math.random() * 0.2 - 0.15
      particles.position.setZ(initZposition)
      pointsGeometry.push(particles)
      scene.add(particles)
    }
    return pointsGeometry
}
const particles_init_position = -zAxisNumber - depth / 2
let zprogress = particles_init_position
let zprogress_second = particles_init_position * 2
const particles_first = initSceneStar(particles_init_position)
const particles_second = initSceneStar(zprogress_second)
复制代码
```

### 使星星运动

```
// 渲染星星的运动
const renderStarMove = () => {
    const time = Date.now() * 0.00005
    zprogress += 1
    zprogress_second += 1

    if (zprogress >= zAxisNumber + depth / 2) {
      zprogress = particles_init_position
    } else {
      particles_first.forEach((item) => {
        item.position.setZ(zprogress)
      })
    }
    if (zprogress_second >= zAxisNumber + depth / 2) {
      zprogress_second = particles_init_position
    } else {
      particles_second.forEach((item) => {
        item.position.setZ(zprogress_second)
      })
    }

    for (let i = 0; i < materials.length; i++) {
      const color = parameters[i][0]

      const h = ((360 * (color[0] + time)) % 360) / 360
      materials[i].color.setHSL(color[0], color[1], parseFloat(h.toFixed(2)))
    }
}
复制代码
```

星星的运动效果，实际就是沿着z轴从远处不断朝着相机位置移动，直到移出相机的位置时回到起点，不断重复这个操作。我们使用上帝视角，从x轴的左侧看去。

### 创建云以及运动轨迹

```
// 创建曲线路径
const route = [
    new THREE.Vector3(-width / 10, 0, -depth / 2),
    new THREE.Vector3(-width / 4, height / 8, 0),
    new THREE.Vector3(-width / 4, 0, zAxisNumber)
]
const curve = new THREE.CatmullRomCurve3(route, false)
const tubeGeometry = new THREE.TubeGeometry(curve, 100, 2, 50, false)
const tubeMaterial = new THREE.MeshBasicMaterial({
  opacity: 0,
  transparent: true
})
const tube = new THREE.Mesh(tubeGeometry, tubeMaterial)
// 把创建好的路径加入场景中
scene.add(tube)
// 创建平面几何
const clondGeometry = new THREE.PlaneGeometry(500, 200)
const textureLoader = new THREE.TextureLoader()
const cloudTexture = textureLoader.load('cloud.png')
const clondMaterial = new THREE.MeshBasicMaterial({
  map: cloudTexture,
  blending: THREE.AdditiveBlending,
  depthTest: false,
  transparent: true
})
const cloud = new THREE.Mesh(clondGeometry, clondMaterial)
// 将云加入场景中
scene.add(cloud)
复制代码
```

现在有了云和曲线路径，我们需要将二者结合，让云按着路径进行运动

### 使云运动

```
let cloudProgress = 0
let scaleSpeed = 0.0006
let maxScale = 1
let startScale = 0
// 初始化云的运动函数
const cloudMove = () => {
  if (startScale < maxScale) {
    startScale += scaleSpeed
    cloud.scale.setScalar(startScale)
  }
  if (cloudProgress > 1) {
    cloudProgress = 0
    startScale = 0
  } else {
    cloudProgress += speed
    if (cloudParameter.curve) {
      const point = curve.getPoint(cloudProgress)
      if (point && point.x) {
        cloud.position.set(point.x, point.y, point.z)
      }
    }
  }

}
复制代码
```

### 完成three.js有关效果

最后，把cloudMove函数放入loopAnimate函数中即可实现云的运动。至此，该登录页所有与three.js有关的部分都介绍完了。剩下的月球地面、登录框、宇航员都是通过定位和层级设置以及css3动画实现的，这里就不进行深入的讨论了。

上面的每个部分的代码在连贯性并不完整，并且同登录页的完整代码也有些许出入。上面更多是为了介绍每个部分的实现方式。完整代码，我放在github上了，每行注释几乎都打上了，希望能给你入坑three.js带来一些帮助，地址：https://github.com/Yanzengyong/threejs-login-view

## 结语

之前用react+three.js写过一个3D可视化的知识图谱，如果这篇对大家有帮助并且反响好的话，后续我会写一篇如何使用three.js创建一个3D知识图谱。

最后，我认为3D可视化的精髓其实在于设计，有好的素材、好的建模，能让你的页面效果瞬间提升N倍

