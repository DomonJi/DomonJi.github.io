---
title: Unity之有限状态机
tags:
  - unity
  - 架构
id: 112
categories:
  - Unity
date: 2016-01-24 09:54:18
---

## 状态机

状态机也是游戏中一个非常有用的概念。它可以用于管理游戏不同状态下的逻辑，也可以用于人工智能体的状态机。

在简单逻辑的情况下，用最基础的枚举变量和`switch`，`case`的有限状态机就比较合适。但是这种模式不适合扩展，逻辑一旦复杂起来将会变得非常混乱。
<!--more-->

_一种更好的方式是将状态也转变为对象。_状态对象拥有自己的属性和方法，这样一来就不用在状态枚举中不断添加枚举，也不需要在用容易混乱的`switch`，`case`语句来判断状态，增强程序的维护性和可读性。

### State

    public abstract class State : MonoBehaviour 
    {
        public virtual void Enter ()
        {
            AddListeners();
        }

        public virtual void Exit ()
        {
            RemoveListeners();
        }

        protected virtual void OnDestroy ()
        {
            RemoveListeners();
        }

        protected virtual void AddListeners ()
        {

        }

        protected virtual void RemoveListeners ()
        {

        }
    }
    `</pre>
    将其定义为`abstract` 是为了必须创建具体子类来使用该类。基本用例是`State Machine` 会决定哪个是当前状态（如果存在）——以及在调用`Exit` 和`Enter` 之后会分别切换到哪个状态。然后使用`Enter` 和`Exit` 方法来添加和移除事件监听器。

    ### StateMachine

    <pre>`public class StateMachine : MonoBehaviour 
    {
        public virtual State CurrentState
        {
            get { return _currentState; }
            set { Transition (value); }
        }
        protected State _currentState;
        protected bool _inTransition;

        public virtual T GetState&lt;T&gt; () where T : State
        {
            T target = GetComponent&lt;T&gt;();
            if (target == null)
                target = gameObject.AddComponent&lt;T&gt;();
            return target;
        }

        public virtual void ChangeState&lt;T&gt; () where T : State
        {
            CurrentState = GetState&lt;T&gt;();
        }

        protected virtual void Transition (State value)
        {
            if (_currentState == value || _inTransition)
                return;

            _inTransition = true;

            if (_currentState != null)
                _currentState.Exit();

            _currentState = value;

            if (_currentState != null)
                _currentState.Enter();

            _inTransition = false;
        }
    }

这个脚本也相当简单。它拿到一个`State` 实例作为当前状态。该实例是一个叫做 `_currentState`的保护字段，并且可以通过属性`CurrentState`来访问。该属性支持赋值和获取两种操作，赋值操作调用`Transition` 方法有一些其它的逻辑：

1.  如果设置当前状态传入的值已经是当前状态了，就直接返回（而不是退出当前状态再重新进入）。
2.  你不能在`transition`过程中设置状态（例如，不能存在那种调用Exit或Enter方法后导致其它状态变为当前状态的状态）。我选择避免这种问题以防我想在状态改变时抛出事件。否则，在切换状态之前transition过程还未完成这种情况下，会看到抛出很多的事件都把最新创建的事件作为当前事件（而不是每个状态一个事件），这样会导致意想不到的bug。
3.  标记`transition`开始。。。
4.  如果之前状态不为空，通知其退出。
5.  在`setter`中将保护字段 `_currentState`设为传入的状态。
6.  如果新状态不为空，通知其调用`Enter`。
7.  标记`transition`结束。
除此还添加了几个简便的方法。想以很简单的方式通知`StateMachine`切换状态，根据当前状态的类型调用泛型方法。这样一来，就不用对将切换到的状态实例的引用进行硬编码了。我是通过`ChangeState` 方法来实现的，约束条件是其泛型参数类型只能是`State`。在方法内部调用了另外一个泛型方法`GetState` ，其参数就是泛型类型。`GetState`方法尝试通过Unity的`GetComponent` 方法来获取状态，如果获取不到，就调用`AddComponent`添加。

* * *

以上引用自[游戏蛮牛](http://www.manew.com)
原文作者：Jonathan Parham
原文链接：[https://theliquidfire.wordpress.com/2015/06/01/tactics-rpg-state-machine/](https://theliquidfire.wordpress.com/2015/06/01/tactics-rpg-state-machine/)