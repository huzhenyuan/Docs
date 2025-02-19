# 安全多方计算

<style>
 hr { color:  !important; background-color: skyblue !important; }
 </style>

隐私问题是目前区块链技术所要解决的其中一重要问题，隐私保护不仅在区块链领域成为了重点研究内容之一，而且近几年不断出现的数据隐私泄露问题，也让公众对隐私保护有日益深入的认知。

PlatON基于安全多方计算密码学算法实现一种隐私合约的解决方案，主要思想是将隐私计算算法通过合约进行发布，并由隐私保护需求的数据提供方和计算节点配合执行MPC协议，以实现数据的协同计算。

## 安全多方计算介绍

安全多方计算，英文全称为Secure Multi-Party Computation，简称MPC。它指的是用户在无需进行数据归集的情况下，完成数据协同计算，同时保护数据所有方的原始数据隐私。具体来说，有$n$个计算参与方，分别持有私有数据$x_1,x_2,\dots ,x_n$，共同计算既定函数$f(x_1,\dots ,x_n)=y$。计算完成，得到正确计算结果$y$，且参与各方除了自己的输入数据和输出结果外，无法获知任何额外有效信息。

MPC协议满足的基本性质是：

- **输入隐私性：** 协议执行过程中的中间数据不会泄露双方原始数据的相关信息；
- **健壮性：** 协议执行过程中，参与方不会输出不正确的结果。

构建通用MPC协议的典型范式之一是基于姚期智提出的针对两方计算的加密电路方法，并由Beaver、Micali 和Rogaway 进一步扩展到多方计算。目前MPC相关的协议或算法有Garbled Circuit、Oblivious Transfer、Secret Sharing、BGW、GMW、BMR等，同时工程实现上也有如[SPDZ](https://github.com/bristolcrypto/SPDZ-2)、[LEGO](https://eprint.iacr.org/2008/427.pdf)、Sharemind、Fairplay等不同的安全多方计算框架出来。典型的安全两方计算所使用的协议为加密电路（Garbled Circuit, GC）和不经意传输（Oblivious Transfer, OT）。

### Garbled Circuit

关于Garbled Circuit原理，可进一步参考[此页面](https://en.wikipedia.org/wiki/Garbled_circuit)所做的解释。

目前已经有很多对加密电路进行优化的方案。包括Free-XOR 技术，这意味着XOR 门几乎无需加密，row reduction 和half gate技术，将每个AND门所需要的密文从4个降为2个。最近使用认证加密电路的研究结果实现了针对两方和多方恶意敌手模型下非常有效的协议。

若有兴趣对Garbled Circuit的理论知识作更深入的了解，可参考[此论文](https://eprint.iacr.org/2012/265.pdf)。

### Oblivious Transfer

Oblivious Transfer，中文称为不经意传输，通常简写为OT，它指的是发送者从一个值集合中向接收者发送单个值的问题，这里发送者无法知道发送的是哪一个值，而且接收者也不能获知除了接收值之外的其它任何值。形式化描述为：发送者有由$N$个值组成的集合，接收者有索引$i$（$0\leq i<N$）。协议执行完成，接收者只知道$N_i$，不知道$N_j$，这里$j\neq i$，并且发送者不知道$i$，这称为1-out-of-N Oblivious Transfer。

对于$N=2$，即为1-out-of-2 Oblivious Transfer，接收者在计算加密电路之前，首先根据自己的输入数据获得与之对应的标签，由于标签是发送者定义的，因此接收者与发送者之间执行此OT协议。接收者按照其输入的每个比特，以$\sigma =0/1$为输入，发送者以标签$m_0$、$m_1$为输入。协议执行完成，接收者获得标签$m_\sigma$。图7为1-out-of-2 OT的示意图。协议执行的过程中，满足以下性质：

- 发送者不可获知接收者选择的是哪个标签；
- 接收者无法知道另一个标签$m_{1-\sigma}$。

![](./images/1_out_of_2_ot.png)

<center><font size="2">图1: 1-out-of-2 OT示意图</font><br /> </center>

## 两方安全计算的工作原理

两方计算的实现过程为：

两个计算参与方Alice和Bob，想共同计算$f(s,t)$，这里$s$为Alice所拥有的数据，$t$为Bob所拥有的数据，$f$为计算逻辑。首先，Alice将$f$转换为相应的布尔电路$C$，其中$C$的每个门都有一个真值表表示门的输入输出。然后，Alice对真值表进行加密处理，得到加密电路$\widetilde{C}$。同时，Alice也对其输入进行加密，然后将加密后的输入与加密电路$\widetilde{C}$一同发送给Bob。那么此时Bob就拥有了$\widetilde{C}$和Alice的加密输入（对应于Alice输入的标签），但却未被告知Alice的加密过程，因此Bob就无法获知该如何使用自己的输入。这时，Bob通过与Alice之间执行1-out-of-2 Oblivious Transfer协议来获得加密输入（对应于自己输入的标签）。之后，Bob使用两方的加密输入对加密电路逐个门进行解密，获得电路计算结果。

总体分为如下五个步骤，其详细过程解释如下：

### 1. 布尔电路生成

假设$f(s,t)$是需要安全计算的函数，将此函数转换为布尔电路$C$，满足对任意$s,t$，有$f(s,t)=C(s,t)$。理论上任意函数均可表示成布尔电路。布尔电路的格式如图1所示：

![](./images/function_to_boolean_circuit.png)

<center><font size="2">图2: 函数转换为布尔电路</font><br /> </center>

### 2. 加密电路生成

Alice将函数转换为布尔电路$C$后，将电路的每个门表示成如图2所示的真值表，接着通过加密这些真值表将$C$转化为加密电路$\widetilde{C}$。

![](./images/gates_to_true_false_table.jpg)

<center><font size="2">图3: 将电路中的每个门表示成真值表</font><br /> </center>

\
接着Alice给电路中的每条线选取两个长度为$k$的随机数标签$A_0$、$A_1$（这里$k$为安全参数，通常为128 bits）来代表0、1。如$A_0$表示0，$A_1$表示1，对应关系只有Alice知道，依次逐个门进行替换，以图3第1、2两个AND、XOR门分别为例，其替换过程如图4所示：

![](./images/label_to_true_false_table_values.png)

<center><font size="2">图4: 标签替换真值表中的0/1</font><br /> </center>

\
所有门替换完成，生成如图5所示的表：

![](./images/choose_random_replace_0_1.jpg)

<center><font size="2">图5: 选取随机数替代每个门中的0/1</font><br /> </center>

\
然后Alice使用两个输入标签对每个门的输出标签进行加密，以第一个标签表的第一行为例，Alice以两个输入标签$A_0$、$B_0$为输入进行双重AES加密输出标签$E_0$，得到$AES_{A_0}(AES_{B_0}(E_0))$，整个电路的所有其它电路门均以此形式进行处理，得到最终如图6所示的加密表，生成加密电路$\widetilde{C}$。可看出只有在获得输入标签的前提下，才能解密出加密表：

![](./images/use_input_wire_label_encrypt_output_wire.jpg)

<center><font size="2">图6: 使用输入线标签加密输出线标签</font><br /> </center>

### 3. Alice发送与自己输入相对应的标签

生成加密电路后，Alice也需要对其输入$s$进行加密。首先Alice将$s$转换为对应于电路$C$输入的布尔值，然后将布尔值的每一比特都用对应于$\widetilde{C}$输入的标签$A_0$、$A_1$等来替换，替换完成$s$的每一比特后，生成相应的标签。然后Alice将这些标签以及加密电路$\widetilde{C}$一同发送给Bob。

假设Alice的输入为$s=s_0s_1=10$，则Alice根据其输入，发送给Bob的标签为$A_1$、$B_0$。

### 4. Bob接收与其输入相对应的标签

Bob此时已有加密电路$\widetilde{C}$和Alice的标签，要想解密出加密电路，Bob还需要对应于自己输入的标签来计算出加密电路。如前所述的加密过程，Alice已经构造出Bob输入的标签，但却不知Bob的实际输入。Bob知道自己的输入，但不能确定与其输入值相对应的标签。这里采用1-out-of-2 Oblivious Transfer（见上文解释）来实现。对于Bob的每一个输入比特，他通过与Alice执行Oblivious Transfer来获得对应其输入的标签。这里，Alice是发送者，Bob是接收者，Alice输入为标签，Bob输入为0或1，取决于实际的输入数据。这样Bob就可获得与自己输入所对应的标签，同时，Bob无法获知关于电路任何其它有效的信息，Alice也无从可知Bob实际的输入数据。

假设Bob的输入为$t=t_0t_1=01$，与Alice执行完Oblivious Transfer后，获得与其输入相对应的标签$C_0$、$D_1$，此时Bob就可完成对整个加密电路的解密。

### 5. 电路计算

Bob收到对应于自己输入的标签后，Bob此时拥有的标签有$A_1$、$B_0$、$C_0$、$D_1$，然后逐个门依次进行解密，并将每个门解密后的输出值传递给下一个门，作为下一个门解密的输入。整个电路执行结束，Bob获得输出标签，如图7所示。

对于第一个加密表，Bob以$A_1$、$B_0$作为key输入，解密得到$E_0$；对于第二个加密表，以$A_1$、$B_0$作为key输入，解密得到$F_1$；对于第三个加密表，以$C_0$、$D_1$作为key输入，解密得到$G_1$；对于第四个加密表，以$F_1$、$G_1$作为key输入，解密得到$H_0$；最后以$E_0$、$H_0$作为key输入，解密最后一个加密表，得到最后的输出标签$I_0$。而标签的实际对应关系Bob知道，因此Bob可获得最终计算结果，并将结果发送给Alice，这样双方获得计算输出结果。

![](./images/gates_decrypt_output_label.jpg)

<center><font size="2">图7: 按门依次解密获得输出标签</font><br /> </center>

## 基本的两方安全计算模式

![](./images/MPC-basic-flow.png)

<center><font size="2">图8：基本的两方安全计算实现模式</font><br /> </center>

1、参与计算的其中一方编写业务算法，编写语言包括C++、Java、Python等高级编程语言(具体依赖于使用的多方全计算框架)
2、通过MPC编译器将业务算法编译成布尔电路（由AND、XOR、NOT门组成，常见格式为Bristol）文件
2、参与计算的两方将各自拥有的原始数据进行预处理，转换为所编译布尔电路所需的输入数据格式
4、双方通过部署在各自本地的应用程序解释执行布尔电路，并相互通讯进行安全计算
5、计算完成，双方获得计算结果，并且通讯过程中的数据不会泄露双方原始数据的任何信息

## PlatON中的安全多方计算

安全多方计算为PlatON实现隐私数据的协同计算提供了根本性的技术手段，在PlatON中结合MPC实现隐私合约，为具有多方参与数据作为输入的应用提供隐私保护，实现真正的隐私计算。

PlatON的两方安全计算架构如下：

![](./images/PlatON_two_parties_infrastructure.png)

<center><font size="2">图9: PlatON的两方安全计算架构</font><br /> </center>

PlatON的隐私合约同样支持高级语言编程，但不是编译成庞大的布尔电路文件，而是编译成更高效的LLVM IR字节码，并部署到PlatON网络上，在MPC计算节点内置的MPC虚拟机中以JIT方式执行。隐私合约的输入数据保存在数据节点本地，由数据节点在链下以安全多方计算方式进行隐私计算，并提交计算结果到链上。

关于隐私合约详细使用，请参考：[隐私合约开发指南](https://platonnetwork.github.io/Docs/#/zh-cn/development/[Chinese-Simplified]-%E9%9A%90%E7%A7%81%E5%90%88%E7%BA%A6%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97)，[深入理解隐私合约编程](https://platonnetwork.github.io/Docs/#/zh-cn/development/[Chinese-Simplified]-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E9%9A%90%E7%A7%81%E5%90%88%E7%BA%A6)

PlatON会持续优化当前已实现版本的MPC性能，并实现更先进的MPC协议，在以下几个方面进行改进：

- 结合同态加密（HE）来降低MPC的通信复杂度
- 从两方安全计算实现到三方以上的多方计算，以满足更加复杂多样化的应用场景