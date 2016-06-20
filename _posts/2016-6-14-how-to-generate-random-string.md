---
tags: []
layout: post
title: 关于生成随机字符串的那点事儿
category: null
keywords: 随机生成不重复的字符串
description: 如何生成随机不重复的字符串，一个原创的实现方法
---
在开发过程中，经常会遇到生成一个随机字符串（有时候要求不能重复），例如生成一个ID，或者生成订单编号等等。
关于具体的生成方式有很多种，我在这里提供一个不太一样的的思路，目前在网上还没有见到有人写过这个种方法，故整理此文，希望可以对后面的人有所帮助。

#### 核心思路整理
其实，这个问题困难的地方在于：生成随机字符串的时候要保证不能重复。
例如，生成一个随机字符串，不考虑是否重复，这个比较简单；
再如，生成一个不重复的字符串，不考虑是否随机，这个也比较简单；
问题在于，将两个约束条件整合到一起，就显得比较困难了。

所以，我们的思路是，寻找一种算法，在生成看似随机的字符串时，要保证字符串的唯一性。

而我们下面要介绍的算法，基本步骤如下：
1.生成一个唯一的数字（最简单的方法就是时间戳自增数字）
2.设计一个算法将数字转换为字符串，此算法要保证：
* 字符串和数字必须是一对一关系，以保证替换后的字符串的唯一性
* 字符串必须看起来是随机的


#### 如何生成一个唯一的数字 

天然生成一个不重复唯一数字的方法就是时间戳。

* *注：其实时间戳也并不能保证一定不重复，例如Java中的时间戳精确到毫秒，那么如果1ms内要连续产生2个随机数，那么肯定会重复。同理，如果要求连续产生10万个4位的数字不重复，也是不可能的，因为4位的10进制的不重复数字最多只有1万个，所以根据业务评估数量级是很有必要的。*



在JAVA中我们可以使用如下方式获取时间戳

```java
System.currentTimeMillis()
// 1465900786571
```

在这里，我们获取到的这一长串数字，表示自1970-01-01至当前系统时间的毫秒数。

时间戳优点是在业务量不大的情况下，后面的毫秒数看起来是随机的。
缺点是，有点长，而且在业务量大的情况下不能保证一定是唯一的。

另外一种数字生成方案就是，定义一个初始值，每次调用累加1，这样可以保证数字唯一性。而且数字可以根据业务定义的比较短。


#### 怎么把数字转换乘字符串

首先，0-9共10个数字，最简单的想法就是，找10个字符，建立对应关系，然后数字替换为字符。在这里，我们举例，0-9 对应 a-j，那么我们可以搞一个方法，自动替换之，如下：

```java
    // 0->a 1->b 2->c ... 9->i
    public static String num2str(long num){
        StringBuffer buffer = new StringBuffer();
        buffer.append(num);
        for(int i=0;i<buffer.length();i++){
            buffer.setCharAt(i,(char) (buffer.charAt(i)+49));//49=字符1到字符a的位移
        }
        return buffer.toString();
    }
```

方法造好了，测试一下

```java
System.out.println(num2str(1465900786571L));
//begfjaahigfhb
```
嗯嗯，暂时看起来还不错。只是不知在连续调用下如何呢？

上面的例子，只用到了a-j这10个字符，我们发现还有很多剩余的字符没有用到，是不是感觉有些浪费？而且，字符串看起来不怎么随机呀，比如，我们连续打印5次

```java
        for(int i=0;i<5;i++){
            long t=1465900786571L+i;
            System.out.println(num2str(t));
        }
////// out put //////
begfjaahigfhb
begfjaahigfhc
begfjaahigfhd
begfjaahigfhe
begfjaahigfhf
```

只有字符串最后一位是自增变化的，前面的都是重复的。
怎样才能在连续调用下看起来是随机的呢？

#### 改进的替换方式

上面的例子使用的是固定位移49（字符1到字符a的偏移），我们将剩下未使用的字符（k-z）也加入到这个游戏，方法如下：

我们先简单想一下，有大约多少种替换规则？26个字母应该有26种。
如果只是简单按照顺序以此按位移替换（大家可以想一下，其实还有很多更加复杂的替换规则，我们在这里只举个最简单的例子），计算方法如下：

* 第1种：0->a      1->b      2->c     ...      9->j       位移49
* 第2种：0->b      1->c      2->d     ...      9->k       位移50
* ....
* 第17种：0->q      1->r      2->s     ....     9->z       位移65
* ....
* 第26种：0->z      1->a      2->b     ....     9->i       位移74

具体替换策略如下图

[![替换策略流程图](http://7xvhwc.com1.z0.glb.clouddn.com/random-str-demo.gif "替换策略流程图")](http://7xvhwc.com1.z0.glb.clouddn.com/random-str-demo.gif "替换策略流程图")

代码实现如下

```java
    // 使用a~z之间的随机位移进行字符串替换
    public static String num2str(long num){
        char[] dic = "abcdefghigklmnopqrstuvwxyz".toCharArray();
        int shift = new Random().nextInt(dic.length);

        StringBuffer buffer = new StringBuffer();
        buffer.append(dic[shift%dic.length]);
        buffer.append(num);
        for(int i=1;i<buffer.length();i++){
            int pos = (buffer.charAt(i)-48+shift)%dic.length;
            buffer.setCharAt(i,dic[pos]);
        }
        return buffer.toString();
    }
```

再次连续打印5次

```java
        int start=35270;//一个初始的随机数，可随意填写
        for(int i=0;i<5;i++){
            System.out.println(num2str(start+i));
        }

///////////// out put //////////////////
ehggle
fikhmg
gmolql
ehgglh
begdif
```

哈哈，It works。


#### 算法的进一步改进
上面的例子只是使用a~z小写的字母，如果区分大小写的话，还可以将大写字母也进一步纳入到游戏规则，改进方法很简单，直接将字典数组更新即可，（这里将大小写字母交叉混合以降低规律性）



```java
       char[] dic = "aBcDeFgHiGkLmNoPqRsTuVwXyZAbCdEfGhIgKlMnOpQrStUvWxYz".toCharArray();

//////////// out put /////////////////
NqsPuN
cFHeGD
VyAXCX
qTVsXT
xaczeB
```

字符串看起来更随机了。

#### 此算法的优缺点
**优点：**
* 跟传统算法比，不需要维护一个大的数组来产生唯一的随机数，速度较快
* 由于根据字符串可以反向推出对应的数字，故在程序异常退出后，可以根据最近一次生成的字符串继续根据算法生成并确保不重复

**缺点**
* 某些情况下不是那么随机，需要再针对做优化处理。
例如在一个整数附近，例如10000左右，生成的随机数如下：

```java
       int start=10000;
        for(int i=0;i<5;i++){
            System.out.println(num2str(start+i));
        }
//////////// out put /////////////////
vwvvvv
pqpppq
gkgggl
opooor
yzyyyc
```

#### 总结
经过上面的一些思考，我们应该对产生不重复字符串有些想法了，其实核心主要在2点：

**不重复**
我们在这里使用自然增长的数字或时间戳，确保不会重复；
**随机**
剩下的工作就是对有一定规律的数字进行随机替换工作，重点是替换算法要保证替换后一定不能重复

根据上面的思想，写一个完整的例子

```java
package random.string;

import java.util.concurrent.atomic.AtomicInteger;


public class RandomString {
    private static char[] dic = "aBcDeFgHiGkLmNoPqRsTuVwXyZAbCdEfGhIgKlMnOpQrStUvWxYz".toCharArray();
    private AtomicInteger atomicInteger = new AtomicInteger(0);
    private int length ;
    public RandomString(int length){
        this.length = length;
    }

    public synchronized String next(){
       return num2str(atomicInteger.getAndIncrement());
    }

    private  String num2str(int num){
        //这里的shift可以随机生成也可以自增轮替
        int shift = atomicInteger.intValue()%dic.length;
        StringBuffer buffer = new StringBuffer();
        buffer.append(dic[shift%dic.length]);
        buffer.append(num);

        //补零
        while(buffer.length()<length){
            buffer.insert(1,'0');
        }
        //根据映射将数字替换为字符
        for(int i=1;i<buffer.length();i++){
            int pos = (buffer.charAt(i)-48+shift)%dic.length;
            buffer.setCharAt(i,dic[pos]);
        }

        //针对重复的字符进行优化,将重复的字符替换之
        if(buffer.length()>2){
            for(int i=2;i<buffer.length();i++){
                if(buffer.charAt(i) == buffer.charAt(i-1)){
                    //替换的字符要确保不在当前的映射范围之内
                    buffer.setCharAt(i-1,dic[(shift+dic.length-i)%dic.length]);
                }
            }
        }

        return buffer.toString();
    }

    public static void main(String[] args){
        RandomString randomString = new RandomString(5);
        for(int i=0;i<100;i++){
            System.out.println(randomString.next());
        }
    }
}

```
