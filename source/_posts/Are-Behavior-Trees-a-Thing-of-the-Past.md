---
title: Are Behavior Trees a Thing of the Past?
date: 2021-06-16 15:01:00
categories:
- [翻译, 行为树]
tags:
- 行为树
- Behavior Trees
---

原文：
[Are Behavior Trees a Thing of the Past?](https://www.gamasutra.com/blogs/JakobRasmussen/20160427/271188/Are_Behavior_Trees_a_Thing_of_the_Past.php)

## 介绍

基于游戏沉浸感、以及和NPC智能交互的需求，游戏开发者在游戏中使用的AI技术成为了新的焦点。曾经在游戏中普遍使用，也是游戏中AI决策的主要方案的行为树，已经显得乏力。当游戏开发者希望实现更为复杂的AI行为时，例如处理突发情况，行为树已经无法满足。因此，更为先进的AI技术——Utility AI，已经逐步取代行为树的位置，开始在游戏中展现出更为优越的AI性能，开启一个新的游戏AI时代。

## 行为树是如何屹立游戏AI界（以及它的衰落）

大多数AI开发者都知道有限状态机（FSM）是一种简单且实用AI方法。通过简单地定义状态、以及状态之间切换的条件，然后可以基于决策实现无限循环的AI行为。例如，FSM将持续保持某种状态，直到满足某种条件，才会从一种状态切换到另一种状态。

任何两个状态之间都可以在特定条件下实现切换，很容易设计FSM来实现AI行为。但是，有利也有弊。在一些大型游戏中，构建FSM需要上百个状态，这么多的状态导致异常分析变的非常困难。Damian Isla在2005 GDC论坛中针对Halo 2的AI系统，详细的指出了这一点。
![](https://gamasutra.com/db_area/images/blog/apexgametools05.png)

为了解决这些问题，FSM进一步演化出层级结构，使得它易于设计和异常分析。分层结构使得FSM能够更好的处理大量的状态，但是并没有从根本上解决这个问题，当状态数量进一步增加时，FSM依然会变得难以管理。
![](https://gamasutra.com/db_area/images/blog/apexgametools011.png)

最终，在这种层级组织的思想上，进一步演化出任务组织的树状结构——行为树。和状态机一样，行为树也是由多个状态节点组成，同一时刻只占据一种状态。不同的是，状态机中，任意两个状态之间的跳转都需要单独配置跳转条件，而行为树则是将这些跳转条件进行拆分，这些拆分后的最小条件单元称为条件装饰器。状态机中的跳转仅仅是对当前状态中的条件进行判断，如果满足条件则跳转，而行为树是从根节点开始，定时进行条件遍历，每经过一个装饰器，会进行次条件判断，然后选择不同的状态分支，以此类推。如果最终筛选出的状态和当前状态不同，那么执行状态跳转。因此，行为树解决了FSM中存在的很多问题，例如，FSM中可能发生条件满足却没有跳转的情况，或者其他跳转异常。在Unreal引擎中就实现了一套非常不错的行为树机制。
![](https://gamasutra.com/db_area/images/blog/271188/image02.png)

相较于层级FSM，行为树能够实现更为复杂、同时容易理解的AI系统。并且这种树结构也便于实现可视化的异常分析。
![](https://gamasutra.com/db_area/images/blog/271188/apexgametools00.png)

然而，行为树也有很多缺陷。很多人试图通过实现各种扩展来消除这些缺陷。

当行为树状态数量变得庞大时，每次条件遍历都需要花费大量的时间。因此，有人引入了子树方案，就是将整个行为树分割为多个嵌套的子树，每次条件遍历只需要遍历当前状态所处的子树就可以，这样可以减少遍历数量，只有满足特定条件，才能跳出子树。但是子树的存在，导致行为树在控制、异常处理方便，变得和FSM一样复杂。

## 顶级的AI需要满足什么条件

这需要我们重新回顾一下顶级AI的特性。

顶级AI需要具备更为复杂的、更深层次的、沉浸式的游戏世界的处理能力。例如在《Hitman》、《Witcher》、《Assassin's Creed》中，NPC的数量、定制化数量、生活化的行为数量都变得越来越多。在《Call of Duty》、《The Division》中，对手的数量、以及深度的要求，可变且复杂的策略行为爆炸式的增长。静态的、脚本化的游戏设计不再适用，需要支持动态的、甚至可演化的场景来满足当下的游戏设计。

这依赖于易于设计的、具有复杂行为表达能力的AI机制。例如，这样的AI能够提高突发处理能力，能够应对游戏设计者未曾预料的场景，能够像人类一样表达更为复杂的行为模式。

以上提到AI技术显然无法处理更为庞大的数据输入。AI设计者不可能使用行为树、或状态机来表示所有可能的情况。更别说设计一套保证AI正常使用的测试用例。因此，我们需要一种更好、且更稳定的AI技术。

处理复杂的行为可能是AI中主要的挑战。在游戏开发中的，有很多问题是由于数据量增加、而AI处理能力不足导致的。

1、 技术性遗留问题 - 如果在游戏开发过程中，扩展AI的难度很大，那么后面需要修复的问题、以及所花费的时间将会暴增。
2、 生产效率 - 如果不能快速实现满足游戏设计方案的AI系统，那么游戏设计者无法快速原型化其思路。最终结果是游戏缺乏内容，或者内容偏向程序化、没有趣味性。
3、 质量 - 最终很多游戏只能放弃、或者最终只是实现无聊的智障游戏。

## 一个新的AI范例

Killzone 2 就具有非常棒的AI设计。Guerilla 游戏使用了一个AI公共系统来实现动态决策，并且展示了如何使用简单的打分系统来实现AI决策，即便是在对游戏世界不了解的情况下，该AI决策系统也能达到一个良好的效果。同样的系统已经使用在其他游戏，例如《文明》。

这种公共系统实现方式是，识别可选项，然后基于当前状态，对每一种选项进行打分，然后选出评分最高的选项最为决策项。这种方案具有很多优势：
1、设计简单 - 
2、易于扩展 - 
3、质量高 - 