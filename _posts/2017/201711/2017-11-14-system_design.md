---
layout: post
title:  "系统设计感悟"
date:   2017-11-14 21:58:51 +0800
categories: 小结
tags: design
---

* TOC
{:toc}

最近在对系统做大修改，业务背景是：原本的只处理一个产品场景的系统，现在由于业务发展，需要支持两种产品。所以主要的问题就是实现原有系统的抽象和复用。

# 系统的可扩展性
系统扩展性强的前提是复用度高。否则一味的新增代码，只会徒增系统的复杂度，甚至在某种程度上，复杂度将会呈指数级增长。
1. 数据持久层封装
虽然新增了一种产品（CODE），但是对于关键的数据表不再复用，而是针对新的产品创建新表，且由于产品概念的不统一，两个产品对应的数据表的字段也存在差别；虽然业务流程一致，但由于底层数据表及其对应的实体的差异，导致所有持久层代码全部需要重新写，这是一个繁琐且极易出错的任务。
为了应对后续未知的需求变动，对数据持久层的上层再次做封装，负责屏蔽掉数据表中实体的差异，使得上层逻辑对底层的数据表的变更不再敏感。
2. 对接口的封装
对接口做抽象和封装的最大敌人就是CtrlC+CtrlV，一旦开始了CtrlC+CtrlV，原本两个逻辑一致的代码，后续可能发展成为不断交错的两条曲线，再想择干净就需要先细细地读上半天代码，真的有机会去改的话也要战战兢兢，免得有所遗漏。在业务开发中，不会有专门的机会去重构代码，看到一坨丑陋的代码，只能祈祷永远不要再让自己看到，或者涉及到的功能发生需求变更，趁此机会把它CLEAN UP，毕竟，修改的每一行代码都要通过测试人员去上线，而鲜有测试人员会为自己的洁癖买单。
3. 持久层尽量避免业务逻辑
在现有系统中，大量的业务逻辑透过Service直接到达dao持久层，导致对一个数据表的更新操作可能存在10个mapper方法，对本次修改造成极大的麻烦。如果能把持久层做出通用的接口，把业务逻辑和持久层做分离，那么持久层的表和字段变动对上层业务逻辑的影响会降至最低。

# 系统的可维护性
原有的系统为了避免修改影响原有功能，提倡需要修改原有逻辑的，新加接口，避免修改原有接口。也可能是团队同时开发，代码review形同虚设，系统代码无法维持一个统一的风格，导致现有系统中，一个简单的查询sql就有不计其数个，这给我的此次工作造成了极大的负担。
怎样的系统最好维护？当然是代码越少越好，一个需求，能用10行代码写完的，完全没必要拆成20行来写（当然，代码的可读性还是要保证的，不然所有的接口放在一行代码里传递，读起来晦涩难懂，也无从谈维护了）。后续需求变更，修改1个方法，总好过10个方法一一改过。
这次对所有的查询和更新sql，做了一个统一的实现，替换原有的sql
对重载方法，特别是重载的facade方法，一律删除重复的，把请求参数封装到一个实体中传递
对如method（String strList）和method（List<String>str）之类，只保留一个method（List<String>str），至于请求参数的不同，交给调用方去处理

# 持久层代码的通用性
在业务数据量比较大的情况，DBA总是会建议尽量避免联结查询。在这种情况下，对一个表的操作大部分情况下就是查询，更新操作。对于这种通用的情况，可以考虑使用通用的sql来完成。在现有的系统中发现的一个问题就是，对代码没有统一的管理，对同一个表的更新，添加了数个update。当对一个表添加字段，或者需要对代码做抽象整理时，这些重复的持久层代码将会造成极大的工作量。

# 业务惟一键
在自己的业务系统中，尽量使用自己的流水号，尽量避免使用上游的流水号作为自己的业务惟一键。一般来说，不同的业务系统，通常用来处理不同的业务流程，如订单系统中，会以订单号作为自己的惟一键，串联整个业务流程；而一笔订单到了账务系统中后，业务系统需要根据这个订单号生成自己的账务流水号来进行账务系统的业务串联。
在目前所做的催收系统中，存在一个问题：在业务处理中，催收系统没有自己重新生成新的对应于催收业务的流水号，而是沿用了贷后传递的贷款订单号；但是在催收系统中，同一个贷款订单号会存在多次逾期的情况。为了区分多次进来的贷款订单，只能使用在催表中的自增ID来标示唯一的催收记录。
在MySQL的设计中，主键一般建议使用自增ID，并且不应当具有业务含义。虽然在业务处理中，不会主动对在催表中的ID做修改操作，似乎不会对MySQL的性能造成什么不好影响，但在实际的业务流程中，这种设计方案还是给业务处理造成了很多不便，而这些不便其实都可以通过生成自己的业务流水号来避免。具体造成的业务问题，不再赘述。
