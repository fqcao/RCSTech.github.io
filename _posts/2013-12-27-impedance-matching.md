---
layout: post
title: "OLED显示异常调试记录"
author: 魏闻
mail: w.wei@rcstech.org
star: 3
description: "记录了一次关于阻抗匹配的调试过程"
category: 经验总结
tags: 
    - 电路
    - 阻抗匹配
---
{% include JB/setup %}

　　很少像这样全程比较完整地参与解决了一个电路问题，感觉很有意义，所以花点时间决定好好记录一下。

<!--more-->

###调试记录

>调试时间：2013/4/30  
　  
>问题描述：使用串行协议的OLED显示器接约两米长导线到主控板上时显示不能。同样的线直接接到STM32F407开发板I/O口时，可以正常显示。使用较短的导线接到主控板上时可以显示。开发板I/O与主控板上接口的唯一区别在于：主控板上I/O口经过一个74LVC4245A缓冲器（B>A）再连接到接口。  
　  
>工作环境：龙丘OLED模块串行5V、第三代主控接线板（以下简称主控板）  
　  
>原因分析：导线接触问题>缓冲器驱动能力问题>I2C传输速率与传输长度关系问题>长导线信号干扰问题>缓冲器电容问题>长导线阻抗匹配问题  
　  
>处理方案：重新做接头（失败）->拆掉缓冲器直接短接AB端（未使用，因为不算解决问题）修改传输速率，并测试稳定传输长度（失败）->将导线拆分成单独导线互不干扰（失败）->在OLED的D0端和GND之间接示波器（X10,可以使用，原因不明）->在PE8’和GND之间接入1000pf或47pf电容（可以使用，原因不明，怀疑阻抗匹配问题）->在PE8’和缓冲器A0端之间加100欧姆电阻（可以使用，测试完成）  
　  
>处理结果：完成  
　  
>问题总结：见最后  

###电路图

![原理图]({{ site.img_path }}/2013-12-27 schematic_diagram.png)

###过程详解

　　发现问题：在花了好长时间仔细完善OLED的各种显示函数，以及图形库后，用STM32F407开发板I/O口直接测试了超长导线，可以与原本短导线的测试一样完美显示图形。于是继续放心开发。在要用到手柄进行绑定具体功能的测试时突然发现，OLED挂了！

　　表示不理解，于是用开发板+短线测试了代码，显示没问题；再测试了数据线的通断，没问题；再用开发板+长线测试，没问题；再用主控板+断线测试，也没问题。所以说是只有在主控板+长线是有问题的。

　　猜测和实验开始：主控和OLED的数据传输是依靠的I/O口模拟I2C总线协议进行通讯的。而串行通信的信号是有距离限制的。

　　为什么长线会有问题呢？首先想到的是接触问题，毕竟就算用万用表测量通断时是通的，在插线的时候还是有可能会不稳的，当即要求重新做线的接头，理所当然的失败了。

　　那么难道是驱动能力的问题么？专门的缓冲器的驱动能力还不如主控芯片I/O引脚的推挽输出？特意请教了汉平学长，他表示缓冲器里面都是门电路，驱动能力不足也是有可能的。好吧，查资料去。74LVC4245A的输出电流参数如下（表示没有怎么很系统地看过芯片资料但是应该是这个，可以达到50mA）：

![驱动电流]({{ site.img_path }}/2013-12-27 output_current.png)

　　而主控ARM芯片的资料太烦杂实在是看不懂，但是我没记错的话就算主控开启大部分外设的电流也就是一百多毫安的样子，IO口的输出功率肯定是不会大到50mA这个等级的。并且测试了那根长导线的电阻，两欧姆多点，不到三欧姆。所以这肯定也是在可以驱动的范围内，从原理上失败了。但是同时也注意到了缓冲器的另一个可疑的参数。

![缓冲器参数]({{ site.img_path }}/2013-12-27 4245_param.png)

　　暂时先不理会，继续下一步实验。

　　接着想到：主控和OLED的数据传输是依靠的I/O口模拟I2C总线协议进行通讯的。而串行通信的信号是有距离限制的，所以说是信号传输过程中发生了信号干扰或者丢失之类的事情么？两米对于串行通信确实也不算短吧，但是还是要计算测试一下模拟I2C的传输速率：计算得知2ms可以发送（3+6）*28=252b的数据。没算错的话也就是约为123kpbs吧，貌似还挺快。但是I2C资料上说的最大传输距离可以达到25英尺（7米多）的样子，所以还是试验一下手头的板子吧。

　　使用手头的几根排线加上排针来延长，发现3根排线的距离（每根30cm）传输还能稳定，4根就不行了；再试验了一下降速，在I/O模拟时，每个时钟加上两个延时，每个延时1ms，也就是大约降了两千倍的样子，堪称肉眼可见的奇慢，再测；居然传输只变成了4根的距离！！！这一定不科学啊！！！

　　难道导线之间干扰的影响真的那么大么？于是乎，拆导线，将两米长的排线全部拆成单条的，并且拉开距离，再测试，还是失败了。

　　然后在小渊撒嘛的指导下，拿出了陀螺仪用的屏蔽线，做线！好线总能有用吧，可是测试完还是没有用啊坑爹！

　　简直要放弃了，苦逼地继续漫无目的查资料，心想着难道这么久折腾出来的OLED代码就打水漂了？

　　这时科长撒嘛出现了，详细的描述了之前的出现的问题、现象和各种实验后，我们开始了科长带你做实验活动。先做了一些重复的测试，并且终于想起还有示波器这样的神器存在了。

　　简直是不能忍受的谐波！抖成这样能用就奇怪了嘛。

　　很可惜的是，在这时科长建议尝试接上个100欧姆电阻在数据线上，结果现在分析应该是因为接触问题失败了（因为最后证明这个才是对的，只能怪运气不好）。

　　再次受挫，科长让我自己解决去吧！好的！自己解决！

　　怒盯了半小时资料，无果，想要放弃了，问科长，要不把缓冲器拆了，直接短过去吧。科长说：没找到问题，不要拆缓冲器，肯定是用的方式不对（忘记原话了）。

　　好吧，继续搞。

　　这次再次注意到了缓冲器的电容参数：

![缓冲器电容参数]({{ site.img_path }}/2013-12-27 4245_param2.png)

　　由此，按照我们的电路，在主控到Bn端，电容为10.4pf，在An到OLED端电容27.9pf，就是觉得很大可是没有对比也就没有依据，就是搞不懂啊。

　　于是再次搬出神奇的示波器，再次看波形，一如既往的渣，就在这时候，我保持示波器连接按了一下复位键，等了几秒钟，居然显示出来了！！！

　　拔掉示波器就不显示了！！！

　　示波器你丫不应该是相当于啥都没有的吗？坑爹呐！

　　不过至少能显示了，火速去查示波器引入了什么，并且向科长报告。

　　度娘告诉我示波器探头X1是1M欧，X10是10M欧，所以莫非我拿个大电阻串上去就好了？

　　这时，科长拿来了一个1000pf的瓷片电容，扎上去，测试，居然活了！！！虽然依旧是不明觉厉的状态，但是至少活了啊。

　　因为时间比较晚了，我们讨论了下，大概应该是阻抗匹配的问题，在这一块我完全不清楚。似乎是有很多公式来计算这个匹配的阻抗关系，可以是数据传输和其他许多方面能有更加精确的表现。而且在阻抗匹配这个方面更多使用的应该是电阻而不是电容。

　　所以第二天继续完善实验吧，联系昨晚示波器的等效电阻，拿了一枚1M欧姆的电阻接在D0-GND，可是居然失败了。

　　和汉平学长讨论，然后和汉平学长一起再次做实验。

　　在看完无操作的三种波形后，汉平学长表示这些波形都带有震荡，应该是阻抗匹配没做好，产生了RC震荡。并且我们凭资料了解到这个应该更多的是容性而产生的较大震荡影响了数据传输。而在导线末端加上电容，相当于对这个带震荡的波形进行积分，减缓了波形的震荡，使其数据传输在可以接受的范围。

　　而对于这样的容性负载，应该做的操作应该是在缓冲器输出端加电阻匹配，先让输出端的震荡减少。于是加上了一个100欧姆的电阻，果然缓冲器输出端的震荡减缓了，但是还是不够明显，再换了一个220欧姆的电阻，缓冲器输出端的波形已经基本趋近于完美的方波了。而且OLED的显示也正常了。实验基本完成。

　　再次测试了一下OLED输入端的波形，居然还是有震荡，分析得知应该是电阻大了，再次更换为100欧姆的电阻，得出最终波形，基本没有震荡，应该算完成整个实验及修改了。

　　最终方案是在缓冲器及长导线接口间加入一个100欧姆的电阻，可以完美显示。


###附表：波形图表

![波形图]({{ site.img_path }}/2013-12-27 wave.png)

###最终结论

* 感谢科长和汉平的指导！有学长带着做实验并且解决了问题真的很爽很有成就感！
* 示波器真心是神器！遇到这种问题确实示波器应该早点使用并且用数据来分析找到根本原因，而不是大呼不科学胡思乱想乱做实验。乱实验就算成功了也肯定不完美！
* 遇到问题，猜想原因，实验检测，得出结论。这真心是个很好很科学的过程！
* 我真心太水了，一个这种瞎问题折腾了两天！要谦虚要加油！
