---
title: Unity常用工具类之对象池
tags:
  - unity
id: 237
categories:
  - Unity
date: 2016-02-07 10:19:32
---

在游戏开发制作中，我们经常需要动态的`Instantiate`(实例化)和`Destroy`(销毁)大量的物体，但是每一次实例化游戏物体都会对系统有很大的开销，占用内存并且造成大量的内存碎片，影响游戏运行效率。尤其是在短时间内进行大量实例化操作，会造成明显的卡顿现象。于是利用对象池来管理游戏物体是游戏开发中必不可少的。

<!--more-->

* * *

**什么是对象池**

对象池，顾名思义，就是用来储存游戏对象的池子。在游戏场景加载的时候，预先实例化出游戏过程中需要用到的对象并隐藏起来。在真正需要用到对象的时候再激活。游戏对象销毁的时候不是直接销毁，而是将其隐藏起来放入对象池中，等到再需要用到的时候不需要再次实例化而直接从对象池中取出将其激活。减少了大量实例化对象的操作。

* * *

**具体实现**

首先这是一个工具类，并不需要用到`MonoBehavior`里的回调和各种方法，所以就不需要继承自它。我们需要用到列表，所以需要引用对应的命名空间：

    using System.Collections.Generic;
    `</pre>

    紧接着我们需要开一些变量。

    <pre>`public class ObjectPool{
        private int m_AllocNum = 0;
        private GameObject m_BaseGameObj;
        private List&lt;GameObject&gt; m_IdleList = new List&lt;GameObject&gt; ();
        private int m_ReAllocNum = 0;
        private List&lt;GameObject&gt; m_UsingList = new List&lt;GameObject&gt; ();
    }
    `</pre>

    `AllocNum`和`ReAllocNum`代表首次分配的空间(实例化出游戏对象的个数)和对象池内对象不够时重新分配的空间。

    `BaseGameObject`即该对象池所储存的对象。

    `IdleList`表示在对象池中已经实例化完成处于空闲状态即被隐藏未激活的游戏物体对象列表。`UsingList`是对象池中已被激活在场景中正在使用的游戏物体对象列表。

    <pre>`public ObjectPool ()
        {
            this.m_BaseGameObj = null;
            this.m_IdleList.Clear ();
            this.m_UsingList.Clear ();
        }
    `</pre>

    构造方法。不用多解释，清空所有列表，置空物体。

    <pre>`public void Init (GameObject baseobj, int allocnum, int reallocnum)
        {
            this.m_BaseGameObj = baseobj;
            this.m_AllocNum = allocnum;
            this.m_ReAllocNum = reallocnum;
            this.Alloc (this.m_AllocNum);
        }
    `</pre>

    初始化方法。也不用多解释。

    <pre>`public void Alloc (int allocnum)
        {
            for (int i = 0; i &lt; allocnum; i++) {
                GameObject item = UnityEngine.Object.Instantiate (this.m_BaseGameObj, Vector3.zero, Quaternion.identity) as GameObject;
                item.gameObject.SetActive (false);
                this.m_IdleList.Add (item);
            }
        }
    `</pre>

    分配空间的方法。实例化出指定个数的游戏对象，将其隐藏(实际上不仅是隐藏，是让所有组件和子物体都失效)，并加入到闲置列表中。

    <pre>`public GameObject Pop (Vector3 position)
        {
            if (this.m_IdleList.Count == 0) {
                if (this.m_ReAllocNum == 0) {
                    if (this.m_UsingList.Count &gt; 0) {
                        GameObject obj2 = this.m_UsingList [0];
                        this.m_IdleList.Add (obj2);
                        this.m_UsingList.Remove (obj2);
                    } else {
                    //debug
                    }
                } else {
                    this.Alloc (this.m_ReAllocNum);

                }
            }

            if (this.m_IdleList.Count &lt;= 0) {
                return null;
            }

            GameObject item = this.m_IdleList [0];
            item.gameObject.SetActive (true);
            this.m_UsingList.Add (item);
            this.m_IdleList.Remove (item);
            item.transform.position = position;
            Debug.Log (item.name + &quot;poped&quot;);
            return item;
        }
    `</pre>

    `Pop`方法。将对象池中的对象取出使用。先判断闲置的对象个数，如果没有闲置的对象，判断指定的重新分配的个数，如果重新分配个数为0，即不重新实例化需要的对象，并且存在正在使用的对象，将正在使用的对象的列表中第一个对象直接激活拿来使用。如果重新分配的个数大于0，则重新实例化一定的对象，取第一个激活使用。如果对象池中有闲置的对象，则取闲置列表中第一个对象激活使用。每次激活使用对象也要将其从闲置列表移到使用列表。

    <pre>`public bool Push (GameObject pushobj)
        {
            if (this.m_UsingList.Find (x =&gt; x == pushobj)) {
                pushobj.gameObject.SetActive (false);

                this.m_IdleList.Add (pushobj);
                this.m_UsingList.Remove (pushobj);

                return true;
            }

            return false;
        }
    `</pre>

    `Push`回收不需要的对象。我们不使用到的对象，将其失效。移入闲置列表中。成功返回true。

    <pre>`public int GetAllocNum ()
        {
            return this.m_AllocNum;
        }

        public int GetReAllocNum ()
        {
            return this.m_ReAllocNum;
        }
    `</pre>

    两个方法获取到相应的两个字段。

    <pre>`public void Clean ()
        {
            for (int i = 0; i &lt; this.m_UsingList.Count; i++) {
                UnityEngine.Object.DestroyImmediate (this.m_UsingList [i]);
            }

            for (int j = 0; j &lt; this.m_IdleList.Count; j++) {
                UnityEngine.Object.DestroyImmediate (this.m_IdleList [j]);
            }
        }

场景结束的时候清理对象池。将两个列表中的游戏物体对象立即销毁。

* * *

_最基本的对象池就到此结束了_

一般情况下，我们创建一个游戏对象管理器来统一管理游戏中的各类物体。如角色，怪物，粒子，动画，特效等。使用`Dictionary&lt;int string or other,ObjectPool&gt;`字典形式来存储和引用对象池。