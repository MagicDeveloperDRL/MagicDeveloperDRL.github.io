---

layout:     post
title:      「算法刷题」LeetCode中的C++常用API
subtitle:   C++11
date:       2022-03-26
author:     MRL Liu
header-img: img/post-bg-hacker.png
catalog: true
tags:
    - 算法刷题
---

[TOC]

​		本文记录LeetCode刷题中C++中常见的API。

# 一、STL

## 1、vector

```c++
#include<vector>//一段可以动态扩充的顺序存储的数组容器
#include<algorithm>
#include<iostream>
using namespace std;

int test_Vector(){
    vector<int> vct; //构建一个空的vector
    // vector<int> vct(26);//构建一个含有n个元素的vector
    // vector<int> vct(26,0);//构建一个含有n个元素、每个元素都是val的vector
    // vector<int> vct={2,3,4};//使用某个vector的区间[first, last)构造一个新的vector，用来裁剪vector
    // vector<int> vct(arr.begin(), arr.begin()+5);//构建一个含有指定元素的vector
    // 遍历方式一
    for(int a:vct) cout<<a;
    // 遍历方式二
    for (int i = 0; i < vct.size(); i++) cout<<vct[i];
    // 遍历方式二
    for (auto it = vct.begin(); it != vct.end(); it++) cout<<*it;
    // 反向遍历
    for (auto it = vct.rbegin(); it != vct.rend(); it++) cout<<*it;
    // 在尾部插入\删除元素
    vct.push_back(1); //在末尾插入一个数
    vct.pop_back();// 删除末尾的数
    // 在任意位置插入\删除元素
    vct.insert(vct.begin(),2); //在指定位置插入一个数
    vct.erase(vct.begin()); //删除指定位置的数
    // 获取首尾元素
    int first = vct.front();//获取第一个元素
    int last = vct.back(); // 获取最后一个元素
    // 反转
    reverse(vct.begin(), vct.end());
    // 分割（中间节点）
    int left=0,right=vct.size();
    int mid = left+(right-left)/2;
    vector<int> leftArr(vct.begin(),vct.begin()+mid); //左半部分
    vector<int> rightArr(vct.begin()+mid,vct.end());  //右半部分
    // 排序
    sort(vct.begin(), vct.end());//默认从小到大排序
    sort(vct.begin(), vct.end(), comp);//自定义排序
}

```

## 2、queue

```C++
#include<queue>//一段可以单向扩充的顺序存储的单向队列容器
using namespace std;
queue<int> que;
// 入队
que.push(1);
// 出队
int front=que.front();
que.pop();
```

## 3、stack

```c++
#include<stack>//一段可以单向扩充的顺序存储的栈容器
using namespace std;
stack<int> stk;
// 进栈
stk.push(1);
// 出栈
int top=stk.top();
stk.pop();
```

## 4、deque

```c++
#include<deque>//一段可以双向扩充的顺序存储的双端队列容器
using namespace std;
deque<int> deq;
deq.begin();
deq.end();
// 获取队列的头尾元素
deq.front();
deq.back();
// 在队首队尾加入元素
deq.push_front(1);
deq.push_back(1);
// 从队首队尾取出元素
deq.pop_front();
deq.pop_back();
```

## 5、list

```C++
#include<list>//一段可以双向遍历的链式存储的双向链表容器
using namespace std;
list<int> lst;// 初始化一个空链表
list<int> lst(3);// 初始化一个含有3个元素且每个元素是0的链表
list<int> lst(3，4);// 初始化一个含有3个元素且每个元素是4的链表
list<int> lst={1,2,3};// 初始化一个含有指定元素的链表
// 遍历链表
list<int>::iterator iter=lst.begin();
while(iter!=lst.end()){
    cout<<*iter;
    iter++;
}
// 反向遍历链表
list<int>::reverse_iterator iter=lst.rbegin();
while(iter!=lst.rend()){
    cout<<*iter;
    iter++;
}
// 获取首尾元素
int front=lst.front();
int back=lst.back();
// 在尾部添加\删除元素
lst.push_back(1);
lst.pop_back();
// 在头部添加\删除元素
lst.push_front(1);
lst.pop_front();
// 反转链表
lst.reverse();
// 排序链表
lst.sort();//默认从小到大
// 将链表的第3个元素移动到头部
auto iter=lst.begin();
int num=2;
while(num--) iter++;
lst.splice(lst.begin(),lst,iter);//将iter移动到lst的begin()位置
```

## 6、unordered_map

```c++
#include<unordered_map>// 包含unordered_map/unordered_multimap
unordered_map<int,int> dict;
// 向字典中插入key-value
dict[1]=3;
dict.insert(make_pair(2,3));
// 遍历字典的key-value
for(auto iter=dict.begin();iter!=dict.end();iter++){
    cout<<"key: "<<iter->first<<" value: "<<iter->second<<endl;
}
// 查找字典中是否存在某个key
if(dict.count(1)==1) cout<<"找到了1"<<endl;
if(dict.find(2)!=dict.end()) cout<<"找到了2"<<endl;
// 从字典中删除key为2的键值对
dict.erase(2);
// 查找字典中是否存在某个key
if(dict.count(1)==0) cout<<"没有找到1"<<endl;
if(dict.find(2)==dict.end()) cout<<"没有找到2"<<endl;
```

## 7、priority_queue

```C++
#include<queue>//包含优先级队列
priority_queue<int> big_heap;//默认构造最大堆
priority_queue<int,vector<int>,greater<int>> small_heap;//构造最小堆
big_heap.empty();
big_heap.size();
// 入队
big_heap.push(1);
big_heap.push(2);
big_heap.push(3);
// 出队
int a=big_heap.top();//3
big_heap.pop();
```

自定义堆元素类型和排序方式：

```C++
//自定义堆结构
class mycomp{
public:
    bool operator()(const pair<int,int>& a,const pair<int,int>& b){
        return a.second>b.second;
    }
};
priority_queue<pair<int,int>,vector<pair<int,int>>,mycomp> my_big_heap;
```

# 二、string

```C++
#include<string>
string str;// 本质是个字符数组，其大部分API和vector相同
string.length();// 和vector不同的地方：vector.size();
to_string(2);//数字转字符串
stoi("2")//字符串转数字
// 获取字符串的子串
string.substr(3,5)//表示索引3开始的5个字符，即[3,3+5)。
```

# 三、排序

## 将vector从大到小排序

```c++
// 自定义排序比较函数，参数必须为const修饰的vector成员类型变量
bool comp(const int a, const int b)
{
    return a > b;//从大到小排序
}
// 排序
sort(vct.begin(), vct.end());//默认从小到大排序
sort(vct.begin(), vct.end(), comp);//自定义排序
```

