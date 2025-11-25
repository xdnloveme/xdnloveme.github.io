---
layout:     post
title:      "基于C++动态链接库（.dll）的多业务应用共享架构"
subtitle:   "ts的动态链接库"
date:       2025-09-30
author:     "Leon"
header-img: "img/2025110116035317619842331798.jpg"
tags:
    - DLL动态链接库
    - Typescript
    - 微应用
    - 架构设计
---

# 基于C++动态链接库的多业务共享架构

## 前言

写在前面的话，如果你有下面的这个**巨石应用**的业务架构，业务B从业务A中孵化，你的需求是从业务A的孵化中拆分出业务B，加快B的开发（因为B在业务A中非常庞大，开发效率低，独立部署开发的需求很急），那么你的解决办法是什么？

![](/img/assets/2025-11-25-01-59-53-image.png)

你的解决办法可能是如下的架构：

![](/img/assets/2025-11-25-02-03-42-image.png)

如果业务是可被枚举的，api是可以进行抽象化导出的。进行上述改造是可行的，

**那如果业务B直接依赖业务A的模型层，渲染层，业务逻辑层呢**？**且业务B内有大量`import xx from MODULEA`那么你的工作量是翻倍级别的，巨量的抽象公共逻辑，且业务层是无法抽象的**

聪明的你可能会想到，那我用依赖倒置的原则解决这个问题，用sdk封装起来。这样可以解决打包太大的问题，但是这样又有新的问题了：

1. **import引用语法全部都要改动，枚举等语法也用不了了，且需要定义大量影子对象进行映射**

2. **原来耦合在一起的类型也因为解耦无法推断出来了**

3. **无法规范边界，缺乏上下游监控和边界冲突，路径依赖等**

聪明的你看到了c++的动态链接库架构，你灵机一动，想出了下面的改良架构

## 为什么是这个架构

这个架构主要为了解决什么问题？

#### 一、无法解决模块引用问题，如import，且有些ts语法类型会失效（如枚举）

如题所述，原有的import代码会随着改动而失效，有些枚举会失效。

目标：像原来耦合的时候那样丝滑地引用import语法，使用ts enum枚举等语法

#### 二、抽象的难度大，改造工程巨量

如果你import或者依赖某个方法，如果有500处代码，你就需要定义500个方法或者模块或者变量，且不论是否规范，且经常需要改动，这是roi非常低的操作

目标：减少改造工作量

#### 三、缺乏上下游监控和引用隔离，无法进行有效管控，有极大的风险

如果由上游定义对外暴露，会造成可能污染或者改动到原有引用的风险，且下游业务引用会造成黑箱风险，无法获取错误问题等。

目标：范式输出，有效隔离，提升安全性

## 从C++动态链接库开始

### C++静态库

我们先分析一下C++静态库的实现流程

![](/img/assets/2025-11-25-03-01-02-image.png)

第二步，进行link静态库流程

![](/img/assets/2025-11-25-03-01-30-image.png)

我们可以从C++静态库中得到以下的启发：

#### 优点

1. 静态库的实现本质是一项地址修正和内存分配的过程

2. 静态库的流程比较简单和直接，是从N到1的过程，且可以满足很多资源调用和归一化的目的。

3. 静态库复杂度较低，维护简单。

#### 缺点

1. 从上述地址修正和内存分配的流程来看数据体现都是绝对的，可执行文件是线性的，且都是静态过程生成，没有动态加载的情况，会导致可执行文件体积过于庞大且难以更新。

2. 冗余代码较多，复杂场景难以支撑

所以C++为了解决以上缺点，引出了C++动态链接库

### C++动态库

为了解决以上痛点，动态库原理实现如下

![](/img/assets/2025-11-25-03-05-20-image.png)

如何解决共享包的外部引用问题：

![](/img/assets/2025-11-25-03-06-09-image.png)

我们可以从以上C++静态/动态链接库学到以下几个点

#### 从Compiler编译的视角

Header头文件和Section分离，即声明与实现分离

#### 从Link链接角度

代码抛弃符号地址固定化，主张动态寻址，可以利用中间层转发（容器）的形式，进行引用的重定向和动态链接的流程

#### 从Load加载的角度

主张按需加载使用，摒弃硬编码，高复用，低耦合

我们从上面几个思考中，可以进阶得出我们JS的动态链接库架构基本思路如下：

## 巨石应用的业务共享架构

以下是设计的基础

![](/img/assets/2025-11-25-03-25-53-image.png)

我们可以通过 Linker/Lib/Monitor/Runtime/Framework结构来构建我们的架构

1. **Linker模块处理基座和库的链接关系，通过注入实现**

2. **Loader模块处理Section实现的内存载入问题**

3. **TS编译器处理头文件（声明）的生成，包括如何dead_strip，类似tree-shaking（api-extractor）**

4. **容器中间件（类似GOT）处理映射寻址问题（import问题）**

#### 如何解决模块引用问题？

我们知道ES模块化的本质还是对象的require，对于ESModule，常常是通过__webpack__require进行引用

思路是可以将插件import的语义等价于require函数调用对象，import xx语义 = #include xx.h语义，而实际的load是通过链接器将模块对象映射到对应的require中去达到目的，这里我们可以利用TS的namespace命名空间作为数据载体，进行引用，对我们的Lib库进行声明

#### 如何保证import的寻址问题

C++的做法是地址无关，利用中间层，我们的做法是保持相同的引用，利用namespace进行约束，对于导出的《声明》和《实现》模块，必须是相同的命名空间，库就是命名空间和中间层的具象化

import module =  ‘Namespace.module’，命名空间的一致性保证了导出一体化，也保证了导入的一体化，解决了import寻址问题。

#### 如何解决框架安全性和隔离问题

我们的解决方案是Proxy，利用代理我们模块（万物皆对象）的方式进行get/set代理或者是属性访问的原子化监听，通过代理的形式进行权限设置（比如禁止引用方进行改动），通过代理的模式进行监听数据收集（知道下游引用情况）和出错情况，

### 动态链接流程如下

整体加载流程如下

![](/img/assets/2025-11-25-16-14-40-image.png)

我们利用承载页的模板渲染逻辑依赖确定加载顺序，整体流程是

1. 加载A业务承载页模板，后加载B业务承载页模板

2. A业务微应用记载（不启动），立即执行代码进行执行

3. 动态链接包静态链接（立即执行的时机）

4. 业务B插件加载（不启动）

5. 业务B立即执行代码加载（load）动态包静态模块

6. A微应用入口进行启动

7. 链接A微应用的运行时bridge桥接

8. B微应用入口进行启动，加载动态包运行时模块桥接

### 案例使用方式

```
// 假设我有一个项目A和一个项目B，项目B依赖项目A
// 原来的用法
import moduleName from '@private/A'
moduleName.doSomething() // 这是B项目
// 在B内试用

/**
* -------------------改造后---------------------
*/

// @private/lab是一个容器包，没有实现，只有类型
import NAMESPACE_LIB from '@private/lib';

// 通过此方式进行静态库的引用
import Module = NAMESPACE_LIB.moduleName;
moduleName.doSomething() // 逻辑同上

// 通过运行时进行运行时桥接
import RUNTIME from '@private/runtime'

// 桥接
RUNTIME.bridge.sync(somethings);
// 响应更新
RUNTIME.bridge.on('update', somethings);
// 异步执行
RUNTIME.bridge.async().then(somethings)
```

#### 具体案例细节实现如下
```
// 声明包的定义方式
// 我们首先在我们的容器层，定义我们需要导出的包，比如这里是一个class的分类

// Container.ts 容器层
class DemoLib extends ClassLib<Interface> implements SomeThingDecl {
    protected bizId = YOUR_BIZ_NAME;

    get ModuleClass() {
        return this.getApi(FUN_KEY.ModuleClass);
    }
}

export let DEMO_LIB: SomeThingDecl;

registerLib<ClassLib<any, any>>(() => new DemoLib());

registerLibProxy(framework => {
    DEMO_LIB = framework.getLib<SomeThingDecl>(YOUR_BIZ_NAME);
});

/**
* -------------------分割线---------------------
*/

// .h文件的声明层
// 这里会映射到这个类型DemoLib
import type DemoLib from 'somewhere'

export type {
    DemoLIB
}

/**
* -------------------业务B使用---------------------
*/
import { DEMO_LIB } from '@private/lib';

import ModuleClass = DEMO_LIB.ModuleClass;

// 业务B开发如此使用
// 可以继承从import引入的某个class
class PlanBClass extends ModuleClass {
    // dosomething
}

```

### 如何解决沙箱隔离问题和版本管理，监控？

#### 沙箱问题：Proxy代理解决

待续

#### 版本管理

待续

#### 监控旁路

待续
