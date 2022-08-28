
# MoSh: Motion and Shape Capture from Sparse Markers


基于marker的运动捕捉技术是一中精准的动作捕捉技术，被广泛的应用在游戏和影视当中的角色动画中。一般情况下，需要在人的身体上贴上一些主动发光和被动反光的物质点，这些点被称作markers，然后通过搭建复杂的多相机系统来捕捉这些markers的三维位置，最后再根据这些三维点来获取对应的skleton, 从而来驱动三维模型做出各种各样的动画。

那如何从这些marker点中获取我们需要的人体骨架呢，在这里介绍德国马普所一系列对应的工作。德国马普所有很多专门做人体重建、动态捕捉的的经典论文，比如学界经常使用的人体参数化模型SMPL极其变种smplx。笔者在研究生三年期间就是看他们的论文慢慢成长起来的。


## 总体思路


Mosh 从名字中就可以看出，他是根据三维marker点同时估计人体形状和姿态的一套方法。他大致包含四个核心部分：1、使用参数化的3D人体模型对人体的形状、姿态进行建模。在Mosh的论文中使用的参数化模型叫做SCAPE，而在后续的Mosh++的论文中使用的参数化模型则是SMPL。2、marker点相对于三维参数化模型的放置位置，因为不同的人的marker点的放置位置绝对不可能是在同一点上的，所以需要设计出一种方法来解决这种问题，在论文里叫做marker点的参数化。3、它可以求解出人体的形状从而能够最优的满足已观测的marker点。4、有了已经估算的人体形状和已经参数化的merker点，就可以估算人体姿态了。


## 参数化marker
为什么要参数化marker点呢？因为是无法保证在不同的人身上能够准确地放置在每个marker点该在的位置，因此我们是无法提前知道每个marker点相对于参数化模型的准确位置的。因此需要参数化marker。

假定已知marker点的数量和每个marker点相对于参数化模型的大致位置，需要手动解决的一个问题就是找到每个marker点到参数化模型中顶点的映射$h(i)$，除此这之外还需要指定每个marker点到参数化模型的距离$d_i$。这是唯一需要手动操作的部分。如果我们的marker点的语义定义和数量一直都是相同的，那这些操作只需定义一次即可。

有了上述的定义之后，就可以参数化marker点相对位置了。在这里作者定义了latent marker同时将人体参数化模型的pose、shape和marker点引入进来，这个坐标系可以同时建立起人体参数化模型和marker的关系，而且这种关系与姿态和位移均无关，因为marker点的定义是在neutral pose下定义的。

$v_i(\beta) =S_{h_(i)}(\beta, \theta_0, \gamma_0 ) + d_i N_{h(i)}(\beta, \theta_0, \gamma_0)$
$N_k(\beta, \that, \gamma)$是对应顶点的法向量。


有了neutral pose下的latent marker后，那么该如何建立起不同pose下的latent marker在世界坐标系下的位置呢？在这里定义了 \widetilde(m_i)为第i个marker点的位置， 然后定义了一个映射函数$\hat(m(\widetilde(m_i), \beta, \theta, \gamma))$来将latent marker映射到特定shape、特定pose下，从而最小化到观测marker点的距离来求解人体的shape和pose。这个过程首先需要找到每个latent marker的在surface表面的局部坐标系，主要是根据位置首先找到三个距离latent marker最近的非共线的顶点，然后以此来建立局部坐标系，从而计算出这个latent marker在这个局部坐标系下的位置。而当在在不同的shape和pose下面，在这个局部坐标系将会变换到观测空间下，从而可以计算出观测空间下的latent marker点的位置。这种变换方式在论文可以表达成如下公式：
$\hat{m}(\widetilde{m_i}, \beta, \theta, \gamma)) = B_{\tau(\widetilde{m})}(\beta, \theta, \gamma)B^{-1}_{\tau(\widetilde{m})}(\beta, \theta_0, \gamma_0)\widetilde{m}$
在这个过程中涉及到找最近的顶点、建立局部坐标系、投影和最终变换这几个步骤。通过这一系列过程就可以将latent marker变化到观测空间下。


## 求解

有了观测空间下的latent marker和实际观测的marker点后，就可以建立目标函数求解。

在这里有以下几个目标函数：
1 $E_D(\widetilde{M}, \Beta,\Theta) = \sum||\hat{m}(\widetilde(m_i), \beta, \theta, \gamma) - m_i||^2$

2 $E_S(\beta, \widetilde(M)) = \sum||r(\widetilde(m_i), S(\beta, \theta+0, \gamma_0)) -d_i ||^2$


3 $E_i(\beta, \widetilde(M)) = \sum||\widetilde(m_i) - v_i(\beta)||^2$

4 $E_{\beta}(\beta) = (\beta-\mu_{\beta})^T{\sum}^{-1}_{\beta}(\beta-\mu_{\beta})$
用来防止latent marker距离原始定义的位置太远

$E_{\theta}(\Theta) = \sum(\theta - \mu_{\theta}{\sum}^-1_{\theta}(theta-mu_{\theta}))$定义了shape和pose prior来规则化估计的shape和pose参数，放置出现很奇怪的pose和shape

5 $E_{\mu}(\theta) = ||\theta_t - 2\theta_{t-1}+\theta_{t-2}||$
用来平滑marker点的噪声

