---
title: 字节决赛北京游
date: 2024/11/23 20:00:00
updated: 2024/12/23 01:01:00
tags:
    - non-ctf
    - offline
thumbnail: /assets/trueblog/bytectf.jpg
excerpt: 决赛的形式是作品答辩，字节给了一系列选题，我们Emmmmmm2024选了蓝军：隐秘的通道，因为这个选题看起来最像传统的C2。但是大家都没什么时间做，最后只是糊弄糊弄。但是对我来说，这是极好的公费旅游机会！直接飞北京，见识见识其他队的成员，其他队的作品，与北京。
---

## 赛前

决赛的形式是作品答辩，字节给了一系列选题，我们 **Emmmmmm2024** 选了
**蓝军：隐秘的通道**，因为这个选题看起来最像传统的C2。于是我们选完题后开会讨论，
才发现要求能用来通信的域名只有GitHub，飞书等。那意味着我们只能利用厂家开放的api来做。
并且跨平台、实现多个域名的通信、实现socks5代理等都可以加分。我们一开始打算用go写，
然而，队里会go的人并不多，大家还各有事，北邮的选手纷纷退出，这个项目也一拖再拖。

在比赛开始前一个星期，我想着还没去过北京呢，这么好的公费旅游机会怎么能错过！
就算是只实现命令执行，做`git pull/push`也要把项目做出来！所幸派一个人完成了后端，
*pid* 也帮助完成了前端。

{% note green fa-heart %}
感谢 *pankas* 和 *Pid* 对项目做出的卓越贡献！！
{% endnote %}

最后赶在出发前做了一下ppt，然后就飞北京了。

{% folding green::飞机上拍的照片 %}
<img src="/assets/trueblog/bytectf/oncloud.jpg" height="90%" width="90%">
<p align="center"><em>拍云，但是超宽屏</em></p>
<img src="/assets/trueblog/bytectf/sunset.jpg" height="30%" width="30%">
<p align="center"><em>舷窗旁拍的落日</em></p>
<img src="/assets/trueblog/bytectf/daxing.jpg" height="60%" width="60%">
<p align="center"><em>大兴机场</em></p>
{% endfolding %}

> 飞机上无聊到在手机上看sed的文档了

当晚入住了字节安排的 **京仪大酒店** ，真是一点也不含糊，看了一下，一晚上就是大500的价格。
一进到大厅，那种豪华的感觉直入眼帘。

{% folding grey::下飞机后拍的 %}
你好，北京！

<img src="/assets/trueblog/bytectf/daxing2downtown.jpg" height="30%" width="30%">
<p align="center"><em>大兴开往市区的地铁</em></p>
<img src="/assets/trueblog/bytectf/jyButtom.jpg" height="30%" width="30%">
<p align="center"><em>1楼看大酒店内部</em></p>
<img src="/assets/trueblog/bytectf/jyTop.jpg" height="30%" width="30%">
<p align="center"><em>12楼往下看</em></p>
{% endfolding %}

休整了一下第二天就正式开始答辩了。

## 答辩第一天

字节比赛的时间还是很宽松的，10点才开始，可以睡个懒觉，好评！到了之后领了队牌，
见到了 *PID* ，他此时正在长亭实习。

第一个上场的是 **W&M** ，讲越权漏洞解决方案。第二个是 **大吉北** ，跟我们一样是C2，
但是显然比我们实现得好多了。总而言之，第三个就是我们，我负责上去讲ppt。
线上接入了派，大部分问题都由他来回答了。

然后一个一个过，也没有留下啥深刻的印象，午餐是盒饭，下午有茶歇，有咖啡，水果啥的，
好评！

不过下午的有一个实现tty显示的思路很有意思：通过将tmux的session导出到文件，然后将文件发送出去，
再在服务端上解析，就能拿到tmux的输出了。感觉实现起来估计不会很简单。

{% folding blue::又一些照片 %}
<img src="/assets/trueblog/bytectf/lunch.jpg" height="30%" width="30%">
<p align="center"><em>中午的盒饭，挺丰盛的</em></p>
<img src="/assets/trueblog/bytectf/banner.jpg" height="60%" width="60%">
<p align="center"><em>另一个牌子</em></p>
<img src="/assets/trueblog/bytectf/building.jpg" height="30%" width="30%">
<p align="center"><em>夜晚的抖音集团大楼</em></p>
{% endfolding %}

在休息的时候和好多大佬互换了联系方式，积累一下人脉。还有工作人员跟我们交谈待在北京的经历。
北京人多，车多，房价高，但是机会也多，让她见识到了与自己家乡完全不同的一面。

晚上本想去天安门看看的，问了问朋友，才知道只是看看也必须提前预约！
**而且至少得提前7天！** （因为人多）我只得作罢，在房间里哪也没去。

## 答辩第二天 & 颁奖

第二天ByteAI也来了，在另一个房间。还在我们的会场上见到了 *纯真* 。
答辩中有点无聊，我还顺便研究了一下[邮件补丁](/2024/11/23/PRbyMail/)。

最令人眼前一亮的当属清华✌️的项目。 **RedBud** 队，依靠仅仅 **1人** ，做了好几个通信方式，
并且只花了 **1天** 左右，太强了orz。下午颁奖也是毫不意外直接拿下第一名。

颁奖完了还有抽奖和合照，抽充电宝，键盘，鼠标和switch。我一看我抽中了一个“机械键盘”，
心想还不错，结果一摸手感，不对啊！淘宝搜了一下，原来是“机械手感薄膜键盘”...
再一搜充电宝的价格，好家伙，和我的键盘只差20块，搞了半天白高兴一场...

{% folding blue::再来些照片 %}
<img src="/assets/trueblog/bytectf/rest.jpg" height="60%" width="60%">
<p align="center"><em>茶歇一角</em></p>
<img src="/assets/trueblog/bytectf/top5.jpg" height="60%" width="60%">
<p align="center"><em>3-5名</em></p>
<img src="/assets/trueblog/bytectf/2nd.jpg" height="60%" width="60%">
<p align="center"><em>第二名——L3H Sec</em></p>
<img src="/assets/trueblog/bytectf/1st.jpg" height="60%" width="60%">
<p align="center"><em>实至名归</em></p>

什么？我们在哪？别问

晚上去高中朋友那边参观他的学校：北京农业大学。我们聊了聊大学的基础设施，
看了看大学夜景，了解了一下他在北京的学习生活。他还说他也没去过天安门，一方面因为没空，
另一方面因为人多。

<img src="/assets/trueblog/bytectf/statue.jpg" height="60%" width="60%">
{% endfolding %}

## 尾声

周一我们就离开了，这次我们在首都机场乘飞机离开。

{% folding orange::最后一组照片 %}
<img src="/assets/trueblog/bytectf/danger.jpg" height="40%" width="40%">
<p align="center"><em>能忍住不笑的是神人</em></p>
<img src="/assets/trueblog/bytectf/captain.jpg" height="90%" width="90%">
<p align="center"><em>首都机场</em></p>
<img src="/assets/trueblog/bytectf/dinner.jpg" height="30%" width="30%">
<p align="center"><em>国航的飞机餐，精致得很，机上还有能连地面的wifi</em></p>

北京，再见！
{% endfolding %}
