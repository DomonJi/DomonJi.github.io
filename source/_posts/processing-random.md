---
title: Processing中的随机
tags:
  - Processing
id: 7
categories:
  - Processing
date: 2016-01-17 19:47:29
---

<script type="text/javascript" src="http://artcoder.me/wp-content/uploads/2016/01/processing.min.js"> </script>
这个学期开始学计算机图形学和数据可视化，正好有机会回顾一下之前学的processing语言，这门语言非常适合设计师，艺术生用代码来画图和设计，具体可以百度或者Google一下。

这个系列的博文围绕《The Nature Of Code》书中所给的案例进行拓展和探讨。<!--more-->

## 第一章 引言 随机函数

**书中举了一个随机游走的例子来说明随机函数**

下面是基础的完全随机的游走类对象代码

    Walker w;

    void setup() {
        size(512, 512);
        w = new Walker();
        background(255);
    }

    void draw() {
        w.setup();
        w.display();
    }

    class Walker{
        int x;
        int y;
        Walker(){
            x = width /2;
            y = height /2;
        }

        void setup() {
        int choice =int(random( 4));    
        if (choice ==0) {
            x++;
        }else if (choice==1) {
            x--;
        }else if (choice==2) {
            y++;
        }else {
            y--;
        }}
        void display(){
            stroke(0);
            point(x, y);
        }
    }
    `</pre>

    具体效果演示：
    <canvas class="aligncenter size-full" data-processing-sources="http://artcoder.me/wp-content/uploads/sketchs/RandomWalkerBase.pde"> </canvas>

    * * *

    **拓展：创建一个有动态移动趋势的Walker对象。让它有50%几率向鼠标所在方向移动。**

    在这个地方，我们使用向量会有更好的效果，因为之前的代码只会朝着上下左右四个方向运动，使用向量之后便可以朝任意方向运动。
    <pre>`Walker w;

    void setup() {
    	size(480, 270);
    	w = new Walker();
    	background(255);
    }

    void draw() {
    	w.setup();
    	w.display();
    }

    class Walker{
    	PVector location;
    	PVector velocity;

    	Walker(){
    		location = new PVector(width/2,height/2);
    	}
    	void setup() {
    		if ((int)random(2)>0) {
    			velocity = PVector.random2D();
    		} else {
    			velocity = new PVector(mouseX-location.x,mouseY-location.y);
    			velocity = velocity.normalize();
    		}
    		location.add(velocity);
    	}
    	void display() {
    		stroke(180*sin(6*second()),180*cos(6*second()),180*sin(-6*second()),100);
    		point(location.x, location.y);
    	}
    }
    `</pre>
    并且我稍微修改了一下draw方法，让它每秒钟变换一下颜色，使画面更好看一些。
    具体效果演示：
    <canvas class="aligncenter size-full" data-processing-sources="http://artcoder.me/wp-content/uploads/sketchs/RandomWalker.pde"> </canvas>

    * * *

    ## 高斯随机数

    利用高斯随机数在屏幕中生成随机位置的圆。
    <pre>`float sd = 60;
    float mean = 320;
    void setup() {
        size(650, 650);
        background(255);
    }

    void draw() {
        noStroke();
        fill(255*sin(3*second()),255*cos(3*second()),255*sin(-3*second()),100);
        ellipse(creatRandom(), creatRandom(), 4, 4);
    }

    float creatRandom(){
        float num = (float) randomGaussian();
        return sd*num+mean;
    }
    `</pre>
    具体效果演示：
    <canvas class="aligncenter size-full" data-processing-sources="http://artcoder.me/wp-content/uploads/sketchs/gaussian.pde"> </canvas>
    点击鼠标可以清屏

    ## 用perlin二维噪声生成地形图

    首先，生成立体的四边形网格的框架：
    <pre>`    for (int i = 0; i &lt; 40; ++i) {
            beginShape(QUAD_STRIP);
            for (int j = 0; j &lt; 41; ++j) {
                vertex(5*j, 5*i,0);
                vertex(5*j, 5*(i+1),0);
            }
            endShape();
        }
    `</pre>
    只要把Z轴改成noise函数生成的值就可以了，这里要注意的是noise函数中必须传入相应维数的参数，这相应的参数映射到具体的噪声值上，相同参数的噪声值是完全一样的，所以我们要在每一个网格的xy坐标传入递增的二维噪声的参数。
    <pre>`void setup() {
        size(600, 400,P3D);
        background(255);
    }

    void draw() {
        rotateX(PI/4);
        mesh();
    }

    void mesh(){
        float xoff = 0.0;
        translate(200, 120, 0);
        for (int i = 0; i &lt; 40; ++i) {
            float yoff = 0.0;
            beginShape(QUAD_STRIP);
            for (int j = 0; j &lt; 41; ++j) {
                if (i&gt;0) {
                    vertex(5*j, 5*i, map(noise(xoff-0.2, yoff), 0, 1, -6, 6));
                }else {
                    vertex(5*j, 5*i, map(noise(xoff, yoff), 0, 1, -6, 6));
                }
                vertex(5*j, 5*(i+1), map(noise(xoff, yoff), 0, 1, -6, 6));
                yoff+=0.2;
            }
            endShape();
            xoff+=0.2;
        }
    }
    `</pre>
    结果如下

    ![BC546A48-030B-4960-A137-A3E5DE086AA3.png](http://artcoder.me/wp-content/uploads/2016/01/BC546A48-030B-4960-A137-A3E5DE086AA3.png)

    _拓展：将该地形图的xy值也改成噪声值_

    在xy坐标上也加上一定的噪声即可
    <pre>`void mesh(){
        float xoff = 0.0;
        translate(200, 120, 0);
        for (int i = 0; i &lt; 40; ++i) {
            float yoff = 0.0;
            beginShape(QUAD_STRIP);
            for (int j = 0; j &lt; 41; ++j) {
                if (i&gt;0) {
                    vertex(5*j+map(noise(xoff-0.3, yoff), 0, 1, -5, 5), 5*i+map(noise(xoff-0.3, yoff), 0, 1, -5, 5), map(noise(xoff-0.3, yoff), 0, 1, -6, 6));
                }else {
                    vertex(5*j+map(noise(xoff, yoff), 0, 1, -5, 5), 5*i+map(noise(xoff, yoff), 0, 1, -5, 5), map(noise(xoff, yoff), 0, 1, -6, 6));
                }
                vertex(5*j+map(noise(xoff, yoff), 0, 1, -5, 5), 5*(i+1)+map(noise(xoff, yoff), 0, 1, -5, 5), map(noise(xoff, yoff), 0, 1, -6, 6));
                yoff+=0.3;
            }
            endShape();
            xoff+=0.3;
        }
    }

![284A966F-2888-40B0-9087-5588062680FF.png](http://artcoder.me/wp-content/uploads/2016/01/284A966F-2888-40B0-9087-5588062680FF.png)