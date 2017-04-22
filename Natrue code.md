---
title: Natrue code
tags: javascript,visual,perlin noise,p5,code train,Force
grammar_cjkRuby: true
---

# perlin noise

> perlin noise  [WIKI][1] , 相对随机函数,noise生成的数据是互相关联的

## 一维例子

```javascript
	let width =900,height = 900
	function setup(){
		createCanvas(width,height)
	}
	
	let yoff = 0
	let inc = 0.02
	let start  = 0 
	let i = 0
	function draw(){
		
		stroke(255,0,0)
		noFill();
	// 开始收集坐标
		beginShape()
		let noiseY,sinY,y
		for( i = 0 ; i<width;i++){
			yoff +=0.02;
			noiseY= map(noise(yoff),0,1,-400,400) 
			y = height/2 + noiseY
			vertex(i + start ,y)
		}
		// 将收集到的坐标进行线性连接
		endShape()
		
		start + = inc // 更新
	}
```


## 二维

```javascript

let colorOff = 0;
let colorOffY = 0;
let pixelStart = 0;

function noise2d() {
  //   noLoop();
  background(255);
  noFill();
  loadPixels();
  let pink;
  colorMode(HSB);
  let d = pixelDensity();
  //   let total = width * d * (height * d) * 4;
  // 4 个像素点 构成一个 视点
  for (let k = 0; k < height * d; k++) {
    colorOff = 0;
    for (let i = 0; i < width * d; i++) {
      let index = (k * width + i) * 4;
      pink = noise(colorOff, colorOffY) * 255; // noise 给定2个参数,colorOffY,表示返回的数还要和其有联系;
      pixels[index] = pink; //red
      pixels[index + 1] = pink; //grren
      pixels[index + 2] = pink; //blue
      pixels[index + 3] = 255; // alpha
      colorOff += inc;
    }
    colorOffY += inc;
  }
  colorOffY = 0;
  pixelStart += inc;
  updatePixels();
}
```

## Air and Fluid Resistance

> 当对象进入空气或者液体的时候收到的阻力 

阻力计算公式 : 

<math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><msub><mrow><mi> F </mi></mrow><mrow><mi> d </mi></mrow></msub><mo> = </mo><mo> - </mo><mfrac><mrow><mn> 1 </mn></mrow><mrow><mn> 2 </mn></mrow></mfrac><mi> ρ </mi><msup><mrow><mi> v </mi></mrow><mrow><mn> 2 </mn></mrow></msup><mi> A </mi><msub><mrow><mi> C </mi></mrow><mrow><mi> d </mi></mrow></msub><mover><mrow><mi> v </mi></mrow><mrow><mo> ∧ </mo></mrow></mover></mstyle></math>


- Fd 代表阻力,最终用来提供给 `applyForce()`的参数
- - 1/2 是一个常数.这个在我们的程序中无关紧要,和其他的常量一起会被替代.这个 **负号**是重要的,它告诉说这个力是跟加速度相反方向的
- <math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><mi> ρ </mi></mstyle></math> 念 `rho` , 指代液体的密度,这也是不需要考虑的. 可以简单的将其视作1;
- <math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><mi>v</mi></mstyle></math> 指代 液体 的速度. 这个相当于 进入 的物体的速度的大小  = `velocity.magnitude()`.  <math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><msup><mrow><mi> v </mi></mrow><mrow><mn> 2 </mn></mrow></msup></mstyle></math> 意味着 要平方;(重力加速度 = 1/2 g (t)2)
- A 指代要通过的液体的区域面积,简单起见,不考虑这个因素
- <math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><msub><mrow><mi> C </mi></mrow><mrow><mi> d </mi></mrow></msub></mstyle></math> 是 拉力的因子,和fraction(<math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><mi> ρ </mi></mstyle></math> )一样也是常数,改变其来改变拉力的大小
- <math xmlns="http://www.w3.org/1998/Math/MathML"><mstyle displaystyle="true"><mover><mrow><mi> v </mi></mrow><mrow><mo> ∧ </mo></mrow></mover></mstyle></math> velocity的速度矢量.


简化后的公式:
![formalu][2]



  [1]: https://www.wikiwand.com/zh-hans/Perlin%E5%99%AA%E5%A3%B0
  [2]: ./images/1492835018049.jpg "1492835018049"