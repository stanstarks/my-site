---
title: 综述：分布式平台的机器学习算法
---


介绍
===========

由于现代数据集的大小和复杂度的爆炸性增长，如何解决超大规模数据的特征提取和训练问题越发重要。因此，这类数据集的收集和存储过程，经常伴随着分布式处理的需要。本文以两个算法为例，介绍并行计算在机器学习算法实现中的应用现状。其中一个算法是非常有效也非常适合并行的Boosting，介绍这个实例的目的是，说明一些算法在设计时已经有分布式的想法在，从而与并行计算可以十分自然地结合。另外一个算法是ADMM，这是一个解决数学优化基本问题的方法。介绍这个算法的并行方式可以说明，并行计算在机器学习领域具有十分广阔的应用前景。

分布式凸优化(ADMM)
===========

最近统计和机器学习领域关心的很多问题可以使用凸优化的结构提出。[Boyd, et al][4]提出，alternating direction method of multipliers (ADMM)适合解决大规模数据的分布式凸优化问题。这个方法根植于上世纪50年代，发展于70年代，和很多其他算法有者紧密关系甚至等价性，如dual decomposition、乘子法、Douglas–Rachford splitting、Spingarn's method of partial inverses, Dykstra's alternating projections, Bregman的$\mathcal{l}_1$问题迭代算法、proximal methods和其他一些。这个算法可以解决很广泛的统计学习问题，包括lasso、细数logistic回归、basis pursuit、相关系数selection、支持向量机和很多其他问题。这个方法还可以推广到非凸的问题上，并可以通过分布式技术有效实现。

为了简明起见，我们集中在标准化的全局问题上：
$$\min\,\,\,\,\sum_{i=1}^Nf_i(x_i)+g(z)$$
$$\mathrm{s.t.}\,\,\,\,x_i-z=0$$
其中$f_i$为目标函数第$i$项，$g$为全局标准化函数。这个模型可以很简单推广到其他应用领域中。我首先描述一种抽象的实现方法，然后将其推广到多种软件框架中。

抽象实现
--------
称$x_i$和$u_i$为储存在子系统$i$中的*局部变量*，$z$为*全局变量*。对于分布式实现方式，一般可以很自然地将局部计算（计算$x_i$和$u_i$的更新）分组。我们可以将ADMM写为
$$u_i:=u_i+x_i-z$$
$$x_i:=\mathrm{argmin}(f_i(x_i)+(\rho/2)\|x_i-z+u_i\|_2^2)$$
$$z:=\mathbf{prox}_{g,N_\rho}(\bar x+\bar u)$$
由于新迭代中可以将变量重写，迭代指标可以省略。

实现ADMM需要实现以下几个重要特征：

* 可变状态：每个子系统$i$必须记录$x_i$和$u_i$当前的值。
* 局部计算：每个子系统必须能够解一个小型凸优化问题。其中‘小’的意思是问 题可以用顺序执行的算法解决。另外，每个局部处理中必须可以访问所需数据。
* 全局聚合：必须可以分布局部变量并将结果通知返回一个中心的收集者。或者通过[distributed averaging][6]等方法。如果计算包括近似步骤，既可以在计算节点中完成，也可以在中心完成。
* 同步处理：所有局部变量必须在开始全局聚合前更新。全局更新同步的实现方法有系统检查点barrier等。

另外，可以不必通过中央结点储存$z$，而令每个子系统可以相互通信，这样我们可以将ADMM看作图上的消息传递算法。

消息传递接口
------------

[消息传递接口][5]（MPI）是一个并行算法使用的语言无关的消息传递机制，是当今被最广泛使用的高性能并行计算模型。各种分布式平台都有很多MPI的实现。MPI的接口可以通过多种语言调用，比如C，C++和Python。

MPI中实现ADMM有多种方法，其中最简单的方法如下:

	initialize N个过程中的变量x_i, u_i, r_i, z
	repeat
		1 u_i := u_i + x_i -z
		2 x_i := argmin_x (f_i(x)+(rho/2) * norm_2( x-z+u_i)^2)
		3 w := x_i + u_i; t := norm_2(r_i)^2
		4 Allreduce w和t
		5 z_p := z, z := prox_{g, Nvarrho}(w/N)
		6 exit if rho*sqrt(N)*norm_2(z-z_p) <= epsilon and sqrt(t) <= epsilon^f
		7 r_i := z_i - z

我们假设有$N$个处理器，每个处理器$i$这局部变量$x_i$和$u_i$，和一个冗余的全局变量$z$，只处理目标函数项$f_i$的局部数据。

在第4步中，*Allreduce*意为使用**MPI_Allreduce**操作来计算所有处理器中向量$w$的全局和并且将结果在所有处理器上存储为$w$；同样的方法也适用于$t$的更新。在第4步后，在每个处理器上，$w=\sum_{i=1}^n(x_i+u_i)=N(\bar{x}+\bar{u})$，且$t=\|r\|_2^2=\sum_{i=1}^n\|r_i\|_2^2$。我们使用*Allreduce*的原因为这个实现方法比令每个子系统直接将结果传送给中心收集要可扩展很多。

接下来，在第5步和第6步中，所有处理器（冗余地）计算z的更新并且进行终结测试。这个测试可以只在一个处理器上做，并将结果广播给其他处理器。但是这样做会让代码更加复杂且不会上程序执行变快。

图计算框架
----------

由于ADMM可以看作图上的消息传递，用图计算框架实现ADMM十分自然。概念上，这个实现方法和MPI方案比较相似，只是中心收集者的任务总会被系统抽象地处理，而不是通过一个外显的中心收集过程。另外，更高等级的图处理框架提供了一定数量的内在服务，可以现成使用而不用另外实现，比如容错机制。

很多现代的图框架基于或者受启发于Valiant的[bulk-synchronous parallel][7] （BSP）并行计算模型。一个BSP计算机由一组处理器网络一起构成。一个BSP计算但愿由一系列全局*supersteps*构成。每个superstep分三个阶段：
*	并行计算，处理器在这一阶段进行局部计算。ADMM的局部变量更新在这一步完成。
*	信息交流，处理器在这一阶段相互传递信息。将新的局部变量值广播给一个中央收集结点。
*	界限同步，每个过程等待其他所有进程完成通信。这个过程可以保证每个处理器在中央结点开始处理数据重新发布之前已经更新了他们的初始变量。

几个现成的基于BSP模型的框架包括[Parallel BGL](http://osl.iu.edu/research/pbgl/)，[GraphLab](http://graphlab.org/)，[Pregel](http://kowshik.github.io/JPregel/pregel_paper.pdf)和其他一些。

MapReduce
---------
MapReduce是一个可处理多种数据集的分布式分批计算流行程序模型。在工程界和学术界都有广泛的应用。

MapReduce计算包括一系列的Map任务，将输入数据处理为可以并行的子集，随后Reduce过程将Map任务的结果整合起来。Map和Reduce函数都由用户创建，并处理一个key-value对。Map函数进行如下变换：
$$(k,v)\rightarrow[(k_1',v_1'),...,(k_m',v_m')]$$
即将一个key-value对分割为一列中间结果的key-value对。引擎将收集到的向量对应每个输出$k'$，传递给Reduce函数。Reduce函数进行如下变换：
$$(k',[v_1',\cdots,v_r'])\rightarrow(k'',R(v', ...,v_r'))$$
其中$R$为一个满足交换和结合的函数。比如，R可能为一个简单的加法运算。在Hadoop中，Reducer可以提供一组key-value对。

ADMM的MapReduce算法如下：

	function map(key i, dataset Di)
		1 在HBase表中读取(xi, ui, zh)
		2 z := prox_{g, Nvarrho}(1/N)*zh
		3 ui := x_i + u_i -z; t := norm_2(r_i)^2
		4 xi := argmin_x (fi(x)+(rho/2) * norm_2( x-z+ui)^2)
		5 输出 (key CENTRAL, record (xi,ui))
		
	function reduce(key CENTRAL, records (x1, u1), ..., (xN, uN))
		1 zh := sum_{i=1}^N(xi + ui)
		2 输出 (key j, record (xj, uj, zh)到HBase for j = 1, ..., N)

主要的困难是，MapReduce任务并不是按迭代结构设计的，在各次迭代的
Mapper中不保留状态，从而实现一个像ADMM的迭代算法，需要一些底层的基础设施的理解。Hadoop包含了许多的支持大型的，容错的分布式计算应用程序的组件。比如基于谷歌的GFS分布式文件系统HDFS和基于谷歌的BigTable的分布式数据库HBase。

HDFS是一个分布式文件系统，这意味着它在整个机器集群的数据存储管理。它的设计时为了某些情况下，一个典型的文件可能需要在规模和高速流的读访问是GB或TB。存储在HDFS中的基本单位是块，这是一个典型的配置中64 MB到128 MB的大小。HDFS上存储的文件是由在特定机器上（虽然冗余的，是在多台机器上的每个块的副本），但是同一文件中的不同的块在不需要被存储在同一机器上甚至附近。出于这个原因，任何HDFS上存储的数据进行处理的任务（例如，本地数据集Di）应一次处理一个单一的数据块，由于一个块被保证全驻留在同一台机器上，否则，可能会导致不必要的网络的数据传输

在一般情况下，每个Map任务的输入是存储在HDFS上的数据，并映射器不能直接访问本地磁盘或执行任何状态的计算。调度器运行每个映射器越好，最好在同一节点上接近到它的输入数据，以最大限度地减少网络的数据传输。为了帮助保护数据局部性，每一个Map任务应该也被分配一个数据块的值周围。要注意的是，这和MPI的实现中的情况是非常不同的，MPI中每个进程可以被告知获取运行在任何机器上的本地数据。

由于每个映射只处理一个单个数据块，通常会有一些映射器在同一台机器上运行。为了减少在网络上传输的数据量，Hadoop支持combiners的使用，这实质上是降低给定节点上的所有Map任务，所以结果只有一组中间键值对需要在Reduce时跨机器转移。换句话说，Reduce步骤应被看作是一个两步过程：首先，每个节点上的所有映射器的结果通过Combiner进行Reduce，然后在每个机器的记录再次进行Reduce。这是Reduce函数必须是交换律和结合的一个主要原因。

由于一个Mapper的输入值是一个数据块，我们还需要一种机制让Mapper读取局部变量，并让Reducer为下一次迭代存储更新后的变量。在这里，我们使用HBase，一个建立在HDFS基础上的分布式数据库，它可以提供快速的随机读写访问。类似BigTable，HBase提供了一个分布式的多维度排序的映射表。映射表通过行键，列键和时间戳索引。HBase的表中每个单元可以包含相同的数据的多个版本，通过时间戳区分。在我们的例子中，我们可以使用迭代计数作为时间戳来存储和访问以前的迭代的数据；在检查终止条件时，这是非常有用的。在一个表中的行键是字符串，HBase按行键字典顺序维护数据。这意味着，字典序相邻的行将被存储在同一台机器或附近机器上。在我们的例子中，变量应该存储与子系统标识符的开头的行的键，所以对同一子系统的信息一起被存储，并且是高效的访问。
	

Boosting和Random Forest
=========== 

很多机器学习算法为了获得最精确的结果，经常需要仔细地调控不同问题学习算法的参数。参数优化的计算难度随着超参数数量的增多而更具挑战性。非参数方法可以自动调整参数，但是效率和效果仍没有太好保证。这就衍生出了使用高性能计算来进行模型选择的方法。[Ganjisaffar, et al][1] 指出MapReduce集群非常适合这种参数计算。他们使用MapReduce进行boosted trees和random forest应用与文档问题的标准化参数优化，并给出来模型准确度随参数空间的变化趋势。

Boosted Trees基本的想法是通过对弱分类器的组合来构造一个强分类器。这个想法可以追溯到由Leslie Valiant教授（2010年图灵奖得主）在80年代提出的probably approximately correct learning (PAC learning) 理论。不过很长一段时间都没有一个切实可行的办法来实现这个理想。Schapire在1996年提出一个有效的算法真正实现了这个夙愿，它的名字叫AdaBoost。AdaBoost把多个不同的决策树用一种非随机的方式组合起来，表现出惊人的性能。第一，把决策树的准确率大大提高，可以与SVM媲美。第二，速度快，且基本不用调整参数。第三，几乎不会过度拟合。1999年，Friedman的一篇[技术报告][2]解释了大部分的疑惑。基于此，Friedman提出了他的GBM（Gradient Boosting Machine，也叫MART或者TreeNet）。几乎在同时，Breiman另辟蹊径，结合他的Bagging (Bootstrap aggregating) 提出了Random Forest，今天[微软的Kinect][3]里面就采用了Random Forest。

实现
----

Ganjisaffar等人利用MapReduce对这个随机算法进行了大量的实验。对每种参数组合，均进行$K\times S$次实验，其中$K$为数据Cross Validation所分的次数，$S$为使用的随机种子的个数。由于实验相互独立，Boosting的效率也比较高，可以使用一个集群来并行进行实验，再进行结果归并。MapReduce集群非常适合这个结构的实验。主结点可以通过*Map*分别将$N\times K\times S$个任务初始化。每个映射任务根据自己分配到的参数进行实验，再将结果通过自己的那份Cross Validation的数据比较。MapReduce的框架首先自动将通过同一组参数计算得到的$K\times S$次实验结果整合，再将结果传递给*Reducer*。*Reducer*只需计算每组数据的均值和标准差，并通过比较得出结果。

总结
=======

目前的分布式是一把双刃剑，一方面它让大规模参数优化成为可能，这在几年前是无法想像的，但同时，也带来了模型过度拟合的可能，从而影响模型的精确性。处理这个矛盾，不仅需要研究算法的可并行性和灵敏度，也需要设计更适合处理并行计算的分布式框架和更多像MapReduce一样的项目出现。

[1]: http://www.ganjisaffar.com/papers/2011-KDD-LDMTA.pdf "Ganjisaffar, Yasser, et al. Distributed tuning of machine learning algorithms using MapReduce clusters. Proceedings of the Third Workshop on Large Scale Data Mining: Theory and Applications. ACM, 2011."

[2]: http://www.jstor.org/stable/2674028 "Friedman, Jerome, Trevor Hastie, and Robert Tibshirani. Additive logistic regression: a statistical view of boosting (With discussion and a rejoinder by the authors). The annals of statistics 28.2 (2000): 337-407."

[3]: http://sistemas-humano-computacionais.wdfiles.com/local--files/capitulo:modelagem-e-simulacao-de-humanos/BodyPartRecognition%20MSR%2011.pdf "Shotton, Jamie, et al. Real-time human pose recognition in parts from single depth images. Communications of the ACM 56.1 (2013): 116-124."

[4]: https://www.stanford.edu/~boyd/papers/pdf/admm_distr_stats.pdf "Boyd, Stephen, et al. Distributed optimization and statistical learning via the alternating direction method of multipliers. Foundations and Trends® in Machine Learning 3.1 (2011): 1-122."

[5]: http://www.mpi-forum.org/docs/mpi-2.2/mpi22-report-book.pdf "M. Forum, MPI: A Message-Passing Interface Standard, version 2.2. High-Performance Computing Center: Stuttgart, 2009."

[6]: http://www.mit.edu/~jnt/Papers/PhD-84-jnt.pdf "J. N. Tsitsiklis, Problems in decentralized decision making and computation. PhD thesis, Massachusetts Institute of Technology, 1984."

[7]: http://dl.acm.org/citation.cfm?id=79181 "L. G. Valiant, A bridging model for parallel computation, Communications of the ACM, vol. 33, no. 8, p. 111, 1990"