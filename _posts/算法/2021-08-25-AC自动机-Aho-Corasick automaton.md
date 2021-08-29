---
layout:     post
title:      "AC自动机-Aho-Corasick automaton"
subtitle:   ""
date:       2021-08-25
author:     "CHuiL"
header-img: "img/algorithm-bg.png"
tags:
- 算法
---



AC自动机的算法融合了trie前缀树和KMP的思路来实现在O(n)时间复杂进行多模匹配

Trie的特点是，在一颗trie树中包含有多个关键词，并且利用公共前缀来减少不必要的字符比较，提高查询效率。 但是用在一个字符串匹配多模的场景下，依然需要对原串循环查询，这里减少的只是每次查询时拥有公共前缀的那部分子串。  

而KMP算法利用next数组，来实现O(n)时间复杂对一个串和子串的匹配。next数组记录了模串上每个字符匹配失败时，模串应该移动到的位置，使得主串在失败时可以不动，不断的往前走达到o(n)的复杂度。 但是对于多模的场景，我们就需要为各个摸串创建next数组，并且注意匹配，时间复杂度也会来到o(n*m)

AC自动在trie的基础上，融入了KMP的思路，即遍历一遍主串，对于每个字符，在trie进行状态转移判断，如果成功则顺着树节点往下走，如果失败了，则每个节点都有一个fail指针，指向失败后应该跳转到哪个状态（节点）。从而可以在O(n)时间复杂度内，遍历一遍主串找到所有匹配的模串。  

以经典的ushers为例，模式串是he、she、his、hers，文本为“ushers”。构建的自动机如图：
![image](/chuil/img/algorithm/aca-1.png)

我们试着从u开始输入，然后一步一步走
1. u不匹配，root的fail指向自己，取下一个字符s
2. s匹配，走到3，取下一个字符
3. h匹配，走到4，取下一个字符
4. e匹配，并且匹配到单词she。取下一个字符r
5. 5已经结束，其实没有下一个节点，匹配失败，走向2，2的位置匹配到单词he，然后继续匹配
6. r匹配，走到8，取下一个字符
7. s匹配，并且匹配到单词hers，此时文本走完，结束

可以看到，ac自动机与trie最大的区别，就是每个状态都多了一个fail指针，用来在匹配失败时应该走到哪个节点。

而fail指针构造的逻辑如下

1. 如果是root节点，则指向自己
2. 如果父节点是root节点，则指向root节点
3. 找到自己父节点fail指针指向的那个节点，然后输入自己节点的字符，看是否能匹配到其子节点，可以的话，就指向其子节点
4. 否则的话，则不断沿着fail指针寻找，直到根节点还没找到就指向根节点


## 参考
- [经典算法--Aho-Corasick automaton](https://carlos9310.github.io/2020/01/01/Aho-Corasick/#%E5%A4%9A%E6%A8%A1%E5%8C%B9%E9%85%8Dac%E8%87%AA%E5%8A%A8%E6%9C%BA)
- [Aho-Corasick 多模式匹配算法、AC自动机详解](https://www.huaweicloud.com/articles/ebc6af4670607c1585cf6ac05fb302b8.html)
- [深入理解Aho-Corasick自动机算法](https://blog.csdn.net/lemon_tree12138/article/details/49335051)