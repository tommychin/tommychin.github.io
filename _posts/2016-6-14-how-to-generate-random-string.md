---
tags: []
layout: post
title: 关于生成随机字符串的那点事儿
category: null
keywords: 随机生成不重复的字符串
description: 如何生成随机不重复的字符串，一个原创的实现方法
---
在开发过程中，经常会遇到生成一个随机字符串（有时候要求不能重复），例如生成一个ID，或者生成订单编号等等。
这里只提供一下个人的思考思路，希望可以对后面的人有所帮助。

#### 从最简单的一种情况说起 

我们先看看不那么随机的时间戳。
在JAVA中我们可以使用如下方式获取时间戳

```java
System.currentTimeMillis()
// 1465900786571
```

在这里，我们获取到的这一长串数字，表示自1970-01-01至当前系统时间的毫秒数。

其实，这个时间戳我们可以直接拿来用，没什么不妥的地方，就是有点LOW。

程序猿们一看就知道这TM是个啥玩意儿。

所以我们最好弄得高大上一点儿，这样会显得有逼格。

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

只有字符串最后一位是自增变化的，前面的都是重复的，看起来还是很LOW。

#### 另外一种替换方式

上面的例子使用的是固定位移49（字符1到字符a的偏移），我们将剩下未使用的字符（k-z）也加入到这个游戏，方法如下：

我们先简单想一下，有大约多少种替换规则？
如果只是简单按照顺序以此按位移替换（大家可以想一下，其实还有很多更加复杂的替换规则，我们在这里只举个最简单的例子），计算方法如下：

第1种：0->a      1->b      2->c     ...      9->j       位移49
第2种：0->b      1->c      2->d     ...      9->k       位移50
....
第17种：0->q      1->r      2->s     ....     9->z       位移65

具体替换策略如下图

[![替换策略流程图](http://7xvhwc.com1.z0.glb.clouddn.com/random-str-demo.gif "替换策略流程图")](http://7xvhwc.com1.z0.glb.clouddn.com/random-str-demo.gif "替换策略流程图")

我们对替换的方法进行改进，每次使用49~65之间的随机位移

```java
    // 使用49~65之间的随机位移进行字符串替换
    public static String num2str(long num){
        int shift = new Random().nextInt(16)+49;
        StringBuffer buffer = new StringBuffer();
		buffer.append((char)(shift+48));//第一个字符表示位移,防止重复
        buffer.append(num);
        for(int i=1;i<buffer.length();i++){
            buffer.setCharAt(i,(char) (buffer.charAt(i)+shift));
        }
        return buffer.toString();
    }
```

再次连续打印5次

```java
    public static void main(String[] args){
        for(int i=0;i<5;i++){
            long t=1465900786571L+i;
            System.out.println(num2str(t));
        }
    }

///////////// out put //////////////////
fgjlkoffmnlkmg
cdgihlccjkihje
jknposjjqrpoqm
opsutxoovwutvs
efikjneelmkjlj
```

哈哈，是不是马上逼格很高了。


#### 还能再短些吗
当然可以短。
缩短这个字符串有2个方案：
1. 缩短时间戳数字长度；
2. 改进字符串替换算法，尽量使一个字符表示多个数字；

第一种方案看起来很简单，我们在这里先只考虑第一种方案。

我们再来回顾一下时间戳的定义：表示自1970-01-01至当前系统时间的毫秒数；
为什么一定要从1970年开始呢，其实我们也可以从2016年或者干脆今天开始计算毫秒数也一样。
甚至，其实我们也没有必要真的一定使用时间戳。例如我们随机产生一个随机的5位数字

```java
        int start = new Random().nextInt(10000)+10000;
        for(int i=0;i<5;i++){
            System.out.println(num2str(start+i));
        }

//////////// out put /////////////////
nonvsp
mnmurp
efemji
pqpxuu
bcbjgh
```

看起来清爽很多。

#### 如何连续产生不重复的随机字符串
经过上面的一些思考，我们应该对产生不重复字符串有些想法了，主要在2点：

**不重复**
我们在这里使用自然增长的时间戳，或者一直累加的数字，所以不会重复；
**随机**
剩下的工作就是对有一定规律的数字进行随机替换工作，重点是替换算法要保证替换后一定不能重复

写一个完整的例子

```java
import java.util.Random;
import java.util.concurrent.atomic.AtomicInteger;


public class RandomString {

    private Random random = new Random();
    private AtomicInteger atomicInteger = new AtomicInteger();
    public RandomString(int seed){
        random = new Random();
        atomicInteger = new AtomicInteger(random.nextInt(seed)+seed);
    }

    public synchronized String next(){
       return num2str(atomicInteger.incrementAndGet());
    }
	
    private  String num2str(long num){
        int shift =random.nextInt(16)+49;
        StringBuffer buffer = new StringBuffer();
		buffer.append((char)(shift+48));//第一个字符表示位移,防止重复
        buffer.append(num);
        for(int i=1;i<buffer.length();i++){
            buffer.setCharAt(i,(char) (buffer.charAt(i)+shift));
        }
        return buffer.toString();
    }

    public static void main(String[] args){
        RandomString randomString = new RandomString(10000);
        for(int i=0;i<5;i++){
            System.out.println(randomString.next());
        }
    }
}
```


#### 到底可以多短

首先，我们自己要做一个评估，要生成几位的字符串。

举个栗子，如果我们要生成2位的字符串，如果只用0-9数字表示，那么最多只能生成100种不重复的字符串，如果用a-z表示，则最多可以生成26*26=676种字符串。用a-zA-Z区分大小写共52个字母表示，则可以最多生成52*52=2704种字符串。

有句话怎么说的来着，**抛开剂量谈毒性都是耍流氓**。

