---
title: Unity学习心得
date: 2018-10-29 22:54:14
tags: Unity
mathjax: true
---

近些天总算下定决心开始学习Unity，所以开篇博客来督促自己，记录每天的成果。  
[Unity学习笔记](https://dynalist.io/d/xddnSO6WHWpqSFV0Yau20K-R)  

### 2018.10

#### 28&29

开始学习Unity，没找到什么入门资料，只能去官方网站看教程了。没想到教程质量还算可以，只不过大部分都是英文版，这时候才发现学英语是真的有用（逃。  
开始按照[roll a ball教程](https://unity3d.com/cn/learn/tutorials/s/roll-ball-tutorial)学习。29日晚先看完了第一个视频，过几天也不一定再有这么多时间了，不知道多久才能把这个做完。总之，加油吧！  

#### 31

本来考虑买Unity3D和C#的书，后来上网看了一眼，实在没有什么有用的，不如硬啃官方文档。  
跟着教程写了个控制球移动的小脚本，虽然大部分是一知半解，但总算是写出来了。  
顺带着学了点C#的基础语法，浅显地理解了一点面向对象  

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerController : MonoBehaviour 
{

    public float speed; //通过将每帧移动的距离乘speed，可以控制移动速度，
                        //定义成public即可在Unity中修改其值
    private Rigidbody rb;
    void Start ()//脚本运行的第一帧会调用（通常是游戏运行的第一帧
    {
        rb = GetComponent<Rigidbody>(); //引用一个刚体组件，从而对这个刚体进行操作
    }
    void FixedUpdate ()//有关物理计算时会调用
    {
        float moveHorizontal = Input.GetAxis("Horizontal");//横轴移动的距离
        float moveVertical = Input.GetAxis("Vertical");//纵轴移动的距离
        Vector3 movement = new Vector3(moveHorizontal, 0.0f, moveVertical);//三维向量
        rb.AddForce(movement * speed);//此处省略了第二个变量，力的作用方式
    }
}
//由于开始没有理解其工作方式，所以我弄了两个球体并都使用了这个脚本，结果是两个球都可以响应键盘的操作。
//所以这个GetComponent引用这个脚本“所在”的GameObject的某个Component
```
