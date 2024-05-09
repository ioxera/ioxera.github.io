---
title: 语义分析与中间代码生成
date: 2023-11-30T16:26:30+08:00
draft: true
math: true
tags:
  - 编译原理笔记
toc: true
bold: true
---

## 属性文法
![Alt text](image.png)

**综合属性和继承属性**:  
用继承属性来表示程序设计语言结构中的上下文依赖关系很方便。  
![Alt text](image-1.png)
![Alt text](image-2.png)
![Alt text](image-3.png)
![Alt text](image-4.png)
![Alt text](image-5.png)

**属性计算方法**:  
依赖图：  
![Alt text](image-6.png)
![Alt text](image-7.png)
![Alt text](image-9.png)
![Alt text](image-8.png)
![Alt text](image-10.png)

树遍历:  
![Alt text](image-11.png)
![Alt text](image-12.png)

一遍扫描： 
要求语法分析方法和属性文法类型要对应。  
![Alt text](image-13.png)

## S属性文法  
仅使用综合属性的文法称为S属性文法。S属性文法适合一遍扫描的由下至上语法分析方法，在对非终结符进行归约时进行属性计算。

## L属性文法
L属性文法适合一遍扫描的由上至下分析。  
![Alt text](image-14.png)

## 语法制导翻译  
由源程序的语法结构所驱动的处理方法就是语法制导翻译法。

**翻译模式**:  
![Alt text](image-15.png)
![Alt text](image-16.png)
![Alt text](image-17.png)
![Alt text](image-18.png)

翻译模式示例:  
![Alt text](image-19.png)
![Alt text](image-20.png)
![Alt text](image-21.png)

**统一语义动作执行时机**:  
翻译模式中有些语义动作在产生式推导过程中执行，有些语义动作在产生式推导完成后执行，语义动作不同的执行时机给语法分析器的设计带来了困难。经过转换把所有语义动作放在产生式的末尾可统一执行时机。  
![Alt text](image-22.png)

**消除翻译模式中的左递归**:  
![Alt text](image-23.png)
![Alt text](image-24.png)
![Alt text](image-25.png)
![Alt text](image-26.png)

**递归下降翻译器实现设计**:  
![Alt text](image-27.png)
![Alt text](image-28.png)
![Alt text](image-29.png)

# 中间语言
![Alt text](image-30.png)
后缀式:  
![Alt text](image-31.png)
![Alt text](image-32.png)
有向无环图:  
![Alt text](image-33.png)
![Alt text](image-34.png)
三地址码:  
![Alt text](image-35.png)
![Alt text](image-36.png)
![Alt text](image-37.png)
![Alt text](image-38.png)
![Alt text](image-39.png)
![Alt text](image-40.png)
三元式不利于代码优化，衍生出间接三元式，既方便优化又节省空间，优化时仅调整间接码表。。  
![Alt text](image-41.png)
![Alt text](image-42.png)

## 常见语句到三地址码的翻译模式
