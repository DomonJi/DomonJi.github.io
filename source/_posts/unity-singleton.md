---
title: Unity 单例模式
tags:
  - unity
  - 架构
id: 44
categories:
  - Unity
date: 2016-01-16 15:07:06
---

[dropcap]U[/dropcap]**nity单例模式是一个基本的工具类**，在大多数游戏开发中都需要用到。在unity游戏开发过程中很多东西要进行单例化，但是在unity中直接进行单例化是很麻烦的，因为所有的脚本都必须挂载在gameobject上和场景关联起来。一旦场景销毁，脚本也会被清空，所以我们要创建一个单例。让它能够在unity中永久存在。
<!--more-->

* * *

<pre class="toolbar-overlay:false decode-attributes:false lang:default decode:true ">public class AllSceneSingleton&lt;T&gt; : MonoBehaviour where T: Component{
}</pre>
&nbsp;

首先让它成为一个模板，限制类型为`Component`，保证传进来的都是组件。
<pre class="toolbar-overlay:false decode-attributes:false lang:default decode:true">static T _Instance;
public static T Instance {
        get {
            if (_Instance == null) {
                _Instance = FindObjectOfType&lt;T&gt; ();
                if (_Instance == null) {
                    GameObject obj = new GameObject ();
                    obj.hideFlags = HideFlags.HideAndDontSave;
                    _Instance = obj.GetComponent&lt;T&gt; () as T;
                }
            }
            return _Instance;
        }
    }</pre>
&nbsp;

接着创建一个静态属性`Instance`，在变量前加下划线表示这是私有变量，这是一个编程的好习惯。`HideFlag`这个属性就是告诉unity是如何隐藏自己，`HideAndDontSave`表示这个gamobject是一个最顶层的，在切换场景的时候不会被删掉，也不会被保存，且被隐藏。

* * *

接下来我们也需要一个Awake函数，让它成为虚函数。
<pre class="toolbar-overlay:false decode-attributes:false lang:default decode:true ">public virtual void Awake ()
    {
        DontDestroyOnLoad (this.gameObject);
        if (_Instance == null) {
            _Instance = this as T;
        } else {
            Destroy (gameObject);
        }
    }
</pre>
`DontDestroyOnLoad`让这个脚本在切换场景的时候不要被删掉。

在这之后我们首先要判断`_Instance`是否空，如果空，我们就将this直接复制给他，如果我们在unity单场景的单例情况下，不切换时，当前场景下的gameobject已经是单例，所以直接将这个脚本赋值给它。另外一种情况，不是空的时候，它仍然会调用`Awake`，所以我们要删除。因为有了`DontDestroyOnLoad`它就永远不会被删除了，所以我们总要有一个地方删除它，单例不可能永远存在，所以当`_Instance`不为空，就是已经存在的情况下，把它清除掉。

我们的unity单例模式就写好了

## 

有了多场景的单例模式，我们可能也会需要单场景的单例模式。单场景的单例模式就简单一些。
<pre class="toolbar-overlay:false decode-attributes:false lang:default decode:true ">public class SceneSingleton&lt;T&gt; : MonoBehaviour where T :Component
{

    static T _Instance;
    public static T Instance {
        get {
            if (_Instance == null) {
                _Instance = FindObjectOfType&lt;T&gt; ();
                if (_Instance == null) {
                    GameObject obj = new GameObject ();
                    obj.hideFlags = HideFlags.HideAndDontSave;
                    _Instance = obj.GetComponent&lt;T&gt; () as T;
                }
            }
            return _Instance;
        }
    }
}</pre>
&nbsp;

只要保证这个脚本在当前场景下是唯一且被隐藏，在场景切换时不会被保存。