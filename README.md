# 前言

对于Promise用法就还不是很清楚的同学可以先看一下[这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)，此外官方给出的[Promise/A+规范](https://promisesaplus.com/)，[中文版](https://segmentfault.com/a/1190000002452115)也需要事先进行了解。事实上在Promise/A+规范推行之前，社区已经有了实际使用的Promise类库，[Q](https://github.com/kriskowal/q/tree/v1)就是其中使用最广泛的类库之一，本文也着重对Q作者的[设计思路](https://github.com/kriskowal/q/tree/v1/design)做一些翻译和补充，供大家共同学习。

***
本文旨在渐进的对Promise的设计思路进行阐述，从而帮助读者更深刻的理解promise的原理和使用，并从中得到启发。
***