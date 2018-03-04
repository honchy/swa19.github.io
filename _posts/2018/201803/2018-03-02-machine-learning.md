---
layout: post
title:  "机器学习相关算法"
date:   2018-03-02 15:13:55 +0800
categories: 其他
tags: python
---

# 朴素贝叶斯
1. 将要处理的数据转换为数字
2. 分别计算待测试的文档中的每个单词，出现在不同类别的概率。
对于一个包含字符串的数组：`test_list=['problems', 'buying', 'him', 'has', 'quit', 'how', 'steak', 'help', 'please', 'flea', 'mr', 'ate', 'garbage', 'worthless', 'love', 'maybe', 'I', 'posting', 'licks', 'stupid', 'not', 'food', 'to', 'dog', 'so', 'is', 'cute', 'dalmation', 'park', 'my', 'take', 'stop']`
假定有两个类别：0和1.要判断该一个字符串数组属于类别0，会给出字符串数组。
~~~
    sample_list = [['my', 'dog', 'has', 'flea', 'problems', 'help', 'please'],
                    ['maybe', 'not', 'take', 'him', 'to', 'dog', 'park', 'stupid'],
                    ['my', 'dalmation', 'is', 'so', 'cute', 'I', 'love', 'him'],
                    ['stop', 'posting', 'stupid', 'worthless', 'garbage'],
                    ['mr', 'licks', 'ate', 'my', 'steak', 'how', 'to', 'stop', 'him'],
                    ['quit', 'buying', 'worthless', 'dog', 'food', 'stupid']]
    class_vec = [0, 1, 0, 1, 0, 1]
~~~
对给定的字符串数组test_list,需要判定的是其中的每个单词分别出现在类别0和1中的概率。
具体怎么计算呢？在第1步中，test_list和sample_list做比对后，获得一个矩阵：
~~~
matrix=
[[1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0], 
[0, 1, 0, 0, 0, 0, 1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0], 
[0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 1], 
[0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0], 
[0, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 1, 0], 
[0, 0, 0, 0, 0, 1, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0]]
~~~
python：
~~~
def set_of_words2_vec(vocab_list, input_set):
    return_vec = [0] * len(vocab_list)
    for word in input_set:
        if word in vocab_list:
            return_vec[vocab_list.index(word)] = 1
        else:
            print("the wojrd:%s is not in my vocabulary", word)
    return return_vec

train_mat = []
for postin_doc in sample_list:
    train_mat.append(set_of_words2_vec(test_list, postin_doc))
~~~

这个矩阵和数组class_vec中的类别对应，以matrix[0]为例，matrix[0]对应class_vec[0],即matrix[0]中，被标记为1的字符串被判定为在类别0中出现。
接下来计算针对类别0，计算test_list中的每个单词出现在类别0中的概率。
取出matrix[0],matrix[2],matrix[4],计算其中的1出现的总次数`p0_num=24`，
`matrix[0]+matrix[2]+matrix[4]`获得每个单词在不同的类别0样本中出现的次数和`p0_vec=[0. 0. 1. 0. 1. 1. 0. 1. 0. 1. 0. 1. 1. 3. 1. 0. 1. 1. 1. 0. 1. 1. 1. 1. 0. 2. 1. 0. 1. 1. 0. 1.]`
那么要计算其中的每个单词属于类别0的概率就是`p0_vec/p0_num`,即：

~~~
[0.05263158 0.05263158 0.         0.05263158 0.05263158 0.
 0.05263158 0.05263158 0.05263158 0.05263158 0.05263158 0.05263158
 0.         0.05263158 0.05263158 0.         0.         0.05263158
 0.05263158 0.05263158 0.         0.05263158 0.15789474 0.
 0.05263158 0.         0.05263158 0.         0.10526316 0.
 0.05263158 0.        ]
~~~
同样的方式计算test_list中每个单词属于类别1的概率：

~~~
[0.         0.         0.05263158 0.05263158 0.         0.05263158
 0.         0.10526316 0.         0.         0.         0.
 0.05263158 0.         0.         0.05263158 0.05263158 0.
 0.         0.05263158 0.10526316 0.         0.         0.15789474
 0.         0.05263158 0.         0.05263158 0.05263158 0.05263158
 0.         0.05263158]
~~~

python:
~~~
def train_nb_0(train_mat, class_vec):
    num_train_docs = len(train_mat)
    num_words = len(train_mat[0])
    p_abusive = sum(class_vec) / float(num_train_docs)  # 代表划分为类别1的样本占总样本的比例（0.5）
    p0_num = zeros(num_words)
    p1_num = zeros(num_words)
    p0_denom = 0.0
    p1_denom = 0.0
    for i in range(num_train_docs):
        if class_vec[i] == 1:
            p1_num += train_mat[i]
            p1_denom += sum(train_mat[i])
        else:
            p0_num += train_mat[i]
            p0_denom += sum(train_mat[i])
    print(p0_num)
    print(p1_num)
    p1_vect = p1_num / p1_denom
    p0_vect = p0_num / p0_denom
    print(p0_vect)
    print(p1_vect)
    print(p_abusive)
    return p0_vect, p1_vect, p_abusive
~~~

算法修正
1. 为了计算概率的乘积，消除概率为0的情况
2. 为了避免过小的浮点数做乘积运算时丧失运算精度，将乘积换做对数的相加。即 a*b<=>loga+logb



