---
title: 1.学习不迷茫：Ansible要如何学至精通
p: ansible/1_how_mastering_ansible.md
date: 2021-07-11 10:37:29
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。


# 1.学习不迷茫：Ansible要如何学至精通

## 1.1 三分钟内我要Ansible的所有资料

我去百度上Google了一下Ansible的资料，对它做个简介。

Ansible是一个基于Python开发的配置管理和应用部署工具，现在也在自动化管理领域大放异彩。它融合了众多老牌运维工具的优点，Pubbet和Saltstack能实现的功能，Ansible基本上都可以实现。

Ansible能批量配置、部署、管理一大堆的主机。比如以前需要切换到每个主机上执行的一或多个操作，使用Ansible只需在固定的一台Ansible控制节点上去完成所有主机的操作。

例如，下面的操作(看不懂没关系)表示将Ansible所在主机上的/etc/my.cnf文件拷贝到mysql主机组内的所有主机的/etc/目录下，mysql主机组可以是单台主机，也可以是很多台主机，这可以由用户自己来定义。
```shell
$ ansible mysql -m copy -a "src=/etc/my.cnf dest=/etc/"
```

其实，固定在一台主机上去控制其它主机，通过ssh工具或一些基于ssh二次开发的简单工具也能实现，但Ansible吸引人的地方在于它提供的playbook能批量整合不同主机上执行的不同任务，同时还提供一些额外的机制让用户可以去协调这些任务的执行策略。

![](/img/ansible/1576393267029.png)

Ansible是基于模块工作的，它只是提供了一种运行框架，它本身没有完成任务的能力，真正执行操作的是Ansible的模块，比如copy模块用于拷贝文件到远程主机上，service模块用于管理服务的启动、停止、重启等。这有点像Shell，Shell自身提供的操作是很有限的，但是它提供了一个很友好的平台让实现各种功能的命令得以执行。Ansible自身可以类比于Shell自身，Ansible的各种模块可以类比于Shell下各种命令。

Ansible其中一个比较鲜明的特性是Agentless，即无Agent的存在，它就像普通命令一样，并非C/S软件，也只需在某个作为控制节点的主机上安装一次Ansible即可，通常它基于ssh连接来控制远程主机，远程主机上不需要安装Ansible或其它额外的服务。

Ansible的另一个比较鲜明的特性是它的绝大多数模块都具备**幂等性**(idempotence)。所谓幂等性，指的是多次操作或多次执行不影响结果。比如算术运算时数值加0是幂等的，无论加多少次结果都不会改变，而数值加1是非幂等的，每次加1结果都会改变。再比如执行`systemctl stop xxx`命令来停止服务，当发现要停止的目标服务已经处于停止状态，它什么也不会做，所以多次停止的结果仍然是停止，不会改变结果，它是幂等的，而`systemctl restart xxx`是非幂等的。Ansible的很多模块在执行时都会先判断目标节点是否要执行任务，所以，可以放心大胆地让Ansible去执行任务，重复执行某个任务绝大多数时候不会产生任何副作用。

## 1.2 如何学习并学好Ansible

Ansible作为一个自动化管理工具，它的用法可以非常简单，只需学几个基本的模块，就像ssh命令一样去使用，就能完成一些简单的批量配置管理功能。

它的用法也可以非常难，需要学习大量Ansible自身的知识，而且还需要学习Ansible中涉及到的额外知识。比较悲催的是Ansible的知识体系比较庞大，它的知识板块也比较零散，想要构建一个比较完善的Ansible知识体系确实稍有难度，这也往往会让我们对深入Ansible无处下手甚至产生迷茫感，相信不少学习过Ansible的人对此应该都有所体会。

其实，Ansible的知识虽多，但这些知识点本身是死的，需要什么功能学什么功能就可以，总有一天可以吞下绝大多数的知识点。但这并不够，实际的环境是非常灵活且复杂的，使用Ansible去管理配置时，需要关注很多任务流程和逻辑，如果编写的任务没有逻辑，使用Ansible很可能会让你知道一口大锅从天而降是一种什么样的体验，所以需要去协调Ansible中的各个任务，协调各个任务是Ansible的另一难点。

最后一个难点，我个人认为是写出适合自己公司环境的可复用(即一次编写多次使用)的playbook或role。这要求熟悉如何应对各种需求和逻辑，对知识点也了然于心，这需要很强的综合能力。

所以要掌握Ansible，需要：  
1. **系统性地学习Ansible，从而构建自己的Ansible知识体系**  
    - (1).学基本用法和常用模块  
    - (2).学Ansible涉及到的额外知识，如YAML和JinJa2  
    - (3).学习零散但却重要的边角知识点，比如`delegate_to`、`run_once`  
    - (4).了解一些高级玩法或很少用的上的功能  
2. **真实应用Ansible去配置管理各类服务并协调好任务逻辑，保证Ansible的执行逻辑不会影响业务**  

总结起来，就是一句在座的各位听了估计会很想打人的废话：**理论和实践相结合**。

虽然确实是废话，但却是真理，学习总是离不开理论和实践的，缺了任何一环，都会遇到难点和瓶颈。对Ansible的学习也如此，想要通过Ansible来释放自己，并让Ansible完美地执行各个任务，不仅要求Ansible的功力要达到一定层次，还需要实际的经验来理解任务逻辑。

为了达到理论和实践相结合的目标，需要你我双方共同的努力。

对于我而言，为了让各位不打我，我会尽最大的努力，安排好知识点的出场顺序(正如人生的出场顺序，知识点的出场顺序也很重要)，同时辅以一些我精心挑选的经典案例来解释用法、适用场景并为大家详细解说任务之间的逻辑，从而循序渐进地逐步掌握Ansible的知识点，最终系统性地掌握Ansible，并熟悉Ansible的正确使用姿势。

对于各位读者而言，要深入掌握Ansible，需要自己去练习、测试、并记下属于自己的笔记(杜绝完全拷贝粘贴)，我能提供给各位的是知识的展现和引导，缺少了各位**自己的练习**和**自己的笔记**，这一切都是空话。