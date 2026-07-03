# 高性能计算与 FreeBSD

作者：Johannes M. Dieterich

                 高性能计算涵盖科学         通用计算中广泛使用的重要基础技术来源，也是
                    和工程到金融和社会研究等多个     创新技术的熔炉，例如基础数值
                领域。共同点在于聚合          库以及最近的深度学习应用。其次，更深入地看会揭示一个 H             计算能力，使其      更复杂的生态系统，还包含
                                                通信和前述数值
         能够交付远高于典型桌面       库、工具链和编程语言，
         计算机或工作站的性能           以及网络
         以解决大型问题 [1]。更具体的例子      交换机和存储等辅助硬件解决方案。最后，还有许多
           按计算份额降序排列包括：         多样的系统，如开发者工作站，这些系统
           材料设计、药物发现、         更具多样性，有不同的能力要求
           气候建模、材料设计、计算         ，对操作系统替代品更易接受；例如，Ubuntu Linux 是开发者
            流体力学、经济模拟和大数据      工作站的首选，但不太适合集群安装。
            分析。                      我们生活在一个 HPC 操作系统和微
             可以说，我们生活在一个以      架构 consolidated 的世界中，从粗略一瞥看，当代
         操作系统和微      HPC 安装看起来非常相似。
         架构 consolidated 的 HPC 世界中，      典型安装将包含一个由数千个节点组成的 Beowulf 集群，                                                            38.93% 材料科学
         通过快速      15.49% 计算流体力学
         互连连接。本文撰写时，TOP500      11.79% 气候与海洋
         世界上最快超级计算机的      4.56% 生物化学
         名单包含 91.5% 的 amd64 处理器和 99.6% 的 Linux 操作      1.44% 分子化学        系统（OSs）[3]。                                  0.77% 燃烧
           那么 FreeBSD 在这个 monoculture 中           图 1：典型超级计算机上 HPC 中按领域的 FLOP 使用情况。参考文献 [2]
          有何空间，FreeBSD 社区为何应该
          关心 HPC？首先，HPC 仍然是
  在某种程度上，甚至可以认为云计算         所有可能的架构（包括 amd64）使用
  是为传统上没有或无法      基于 LLVM 的 clang 编译器作为基础
  负担大型      编译器，原因在于许可，
  投资或访问      并因此，
  大型安装的客户重新打包或分解的 HPC。       作为默认的 ports 编译器。与其整理
  尽管 FreeBSD 已经作为设备      混合编译器环境，往往
  解决方案表现出色，但其他功能（虚拟机管理程序选择、存储      更简单，完全依赖 GNU 编译器来处理
  、安全、网络等）比      混合语言程序。这种情况略令人不满。
  计算 OS 更重要。我将在此关注      是否有更根本的改进可能？也许有。flang 是一个基于 LLVM 的开放、新的
  HPC 功能，无论是计算 OS 还是开发者 OS，如何      Fortran 编译器。[4] 其前端源自著名的
  映射到更通用的计算需求，并      Portland Group 的编译器，由
  尝试强调 FreeBSD 如何从      NVIDIA 开源并继续维护。它后来也得到了 AMD
  针对 HPC 的改进中受益。      在其 AOCC 包中的支持和
  作为 FreeBSD 用户和开发者，我们 aware      使用。flang 仅支持
  并重视 FreeBSD 的固有优势：      Fortran 2003 语言特性，限于
  ports 系统中应用和库的一致体验，      64 位架构。FreeBSD 现在包含
  自定义包的轻松构建和部署，       devel/flang 作为预览。当然，需要一些时间
  可能带有架构       和努力才能使此 port 与 gfortran 竞争，并正确 vet
  特定的优化，以及总体上非常稳定      用于 HPC
  的操作系统体验。最重要的是，      用途。就在最近，NVIDIA 公布了
  统一的 ports 体验使我们能够在与用户或开发者交互的 OS 的所有部分一致传播更改。      重写 flang 大部分以改善
                                                                             其特性集和被官方 LLVM 接受
  语言                                 的可能性。在 HPC 社区内的采用
                                        将是有趣的观察，但
  HPC 工具——研究者使用的应用，将          增加的竞争无论如何都应证明对
  FLOPS 转化为知识——由什么组成？如果我们记得 HPC 是      GNU 编译器有益。
  计算机最初用途之一，      让我们超越单核思考。如上
  大部分计算时间      所述，典型集群包含数万
  用于 Fortran 代码      个核心，作业通常需要使用数十
       不足为奇。超过      到数百个并行。扩展 HPC 有两种主要
  65% 的计算周期用于典型      方法：OpenMP
  安装上的 Fortran 代码      [5] 和消息传递接口（MPI）[6]。
  。如果      OpenMP 主要针对共享内存
  考虑混合语言代码中使用的数值 Fortran 库，      机器，后来的标准支持
  这个份额还会增加。不幸的是，这表明      加速器 offloading，下面讨论。这是一种
  Fortran 仍然活跃，由于      相对简单、基于 pragma 的方法，支持
  现有代码库的规模和复      C/C++ 和 Fortran，通常用于以数据并行方式
  杂性，在中短期内不太可能消失。因此，HPC 支持      并行化循环。
  绝对需要 Fortran 支持。由于      目前，我们的 ports 树将再次默认使用
  FreeBSD 上通常使用的商业      lang/gcc port 如果遇到 USES= compiler:openmp
  编译器实际缺席，我们回退到开源      。因此，ports 通常不会
   替代方案。      默认启用 OpenMP（包括一些常
  目前，如果 port 指定 USES=fortran，所有      用的，如 graphics/ImageMagick），因此
  架构将使用 lang/gcc 中的 gfortran 编译器。      将限于单核。
  gfortran 是一个好的 Fortran 编译      amd64 和
  器，产生稳定、快速的二进制文件，支持      i386 有更好的替代方案，即 LLVM
  所有相关的 Fortran 标准。目前，使用      项目自 Intel 最初开源以来
  gfortran 需要显式指定      的 libomp 库。它尚未导入基础系统，但存在
  rpath 给链接器以搜索 GNU 库，用于      较旧库版本的
  该 port 以及（如果是库）所有依赖      review。在其令人
  ports（或 ports 树外的应用，      充满希望地暂时缺席期间，ports 系统应
  就此而言）。虽然简单，但显著增加了      使用包含
  porter 和开发者的维护负担。      libomp 的 devel/llvm port 之一。已进行多次集成测试
  正在讨论补救措施，但      ，该功能应该很快就能落地。
  尚未有补丁进入树。还请记住      希望这也会激励其他开发者
        为其他架构添加必要的 FreeBSD 位，      提供厂商调优的、汇编优化的 BLAS
        上游支持这些架构。我的测试表明      和 LAPACK 库。对于 FreeBSD，我们有多个
      libomp 在 LLVM 中的集成从      通过 blaslapack
       性能角度看不理想；特别是，LLVM 的      选择器公开的选项。默认情况下，此选择器将使用
      向量化器在 OpenMP 的 simd      math/blas 和 math/lapack。这些是
      pragma 与标准      Fortran 源码的参考实现，作为
      OpenMP 线程并行化（例如 parallel      此类，无法与优化替代方案
       for）一起使用时      竞争，例如 math/blis 和 math/libflame。两者
        时无法（良好）工作。但当然，次优并行化      参考实现和 math/open-
        优于无并行化。       blas 有 Fortran 依赖，不同于 FLAME
        另一方面，MPI 通过函数      项目的 BLIS 和 libflame [7]。
        调用操作 MPI 通信库。          如图 3 所示，无论是
            它具有简单的发送/接收/广播      OpenBLAS 还是 BLIS 实现，在
        功能以及更高级的操作      从小型到中型矩阵-矩阵乘法中，如果
         如归约，都显示出显著的性能领先。MPI 用于基于进程的节点内和节点间      使用其 CPU 架构优化内核。
         并行化，在硬件级别，典型      更重要的是，即使使用 BLIS 的仅源参考
         使用快速互连如 InfiniBand。      实现，其实现
        通常，启用 MPI 的软件包      和阻塞方案也带来高达 80% 的性能
      使用 mpicc/mpif90 包装脚本编译      优势。此外，BLIS 提供
      ，配置底层编译器以查找      使用 pthread 并行化的能力，在
      MPI 头文件并链接 MPI 库。      此测试案例中影响较小。FLAME 项目
      在 ports 树中，存在多个 MPI 选择      表示有兴趣与我们合作，接受
      ，我们仅受限于底层      FreeBSD 更改的 pull request，并
      Fortran 的编译器工具链。      实现我们通用包所需的功能，如运行时内核选择
                                                      。因此，BLIS
                                                                                           是我们默认 BLAS 实现
    数值库                     的良好候选，并使我们摆脱低级 Fortran
     HPC 中的高性能很大一部分不      依赖。
  来自应用代码，而是来自      math/libflame port 最近已
  谨慎和大量使用高度优化的      更新到最近的开发快照，并
  数值库，实现标准化 API。      配置为公开
  可以说最重要的 API 是 BLAS，一组      amd64 和 i386 CURRENT 的 LAPACK 接口。在 FLAME 库可以
  基本密集线性代数运算      作为近期版本的 blaslapack 选项添加
  （如矩阵-矩阵乘法），LAPACK，一组      之前，需要更多
  更复杂的求解器      审查。关于
  （如 Cholesky 或 LU 分解），以及快速      这方面的工作正在进行中。
  傅里叶变换（FFT），通常在 FFTW3 API      其他数值、工程和
  中体现。它们的根本性质也使      科学库在 FreeBSD 上的状态已经非常优秀：
  它们成为我们 ports 系统中的常见依赖，      math/fftw3 port 状态良好；我们还
  例如音频和图形应用。      拥有来自
  实现这些 API 的库的选择      各领域（包括量子
  远不如单一集合的性能和      物理/化学）的开发库，可直接使用，并有多个活跃的
  特性重要。大多数 HPC 集群      该领域的 ports 提交者。


   20                                                                                                          图 3. BLIS 和 OpenBLAS 相对于
             netlib gfortran
                                                                                            参考净库 BLAS 在不同大小矩阵-矩阵乘法（dgemm）上的加速
             carrizo BLIS g++
                                                                                            M/N/K 在 AMD A12-8800B CPU 上。数据netlib          carrizo BLIS clang++
                                                                          在 FreeBSD HEAD 上平均 1,000 次评估，编译使用 -O2，所有库使vs
            reference BLIS clang++                                                -mavx -mavx2 -msse -msse2
   10                                                                                                 -msse3 -msse4 -msse4a -msse4.1 -
                                                                           -msse4.2 -mmmx -maes -mbmi -mbmi2-
                                                                           -mf16c -mfsgsbase -mtune=bdver4, BLISspeedup                                                                                     使用 clang++  5.0.0 编译，
                                                                                         netlib/OpenBLAS 使用 g++ 6.4.0。

    0
      128      256         512        768            1,024         1,280         1,536
                          M/N/K

开发和移植到             包括优秀的自由替代方案。dtrace FreeBSD         和 hwpmc 与 benchmarks/                                                     make 分析应用性能，从用户到内核级别
有了语言和库在 FreeBSD 上的总体积极状态，      简单。           和     可能devel/gdb                                                                 lldb没有      相同级别的图形用户
移植 HPC 应用到 FreeBSD 和在其上开发有多难？      界面支持      ，但能完成
典型挑战来自非标准      工作。尽管大多数 HPC 代码仍使用      或       现代      vi                                                          编辑器/IDE 替代make 系统（通常是 shell 和其他脚本      ，但 java/eclipse，以及      java/netbeans，      linuxulator-using存在      编辑器
以及其他脚本以及 make 文件的组合），      ，linux-sublime3 可用。最近，我发现      hardocded      provides an invalu-                                                           bhyve系统路径和其他假设（      等代码通常不符合      标准      ，为 HPC                                                                                  开发者工具包提供了宝贵的补充。无
例如只在      特定      编译器      [版本]      下编译并产生正确结果），以及我们自己的      论 bug 是仅      BSD-isms 如 rpath。      在特定平台配置上出现，需要支持特定
          除了最后一个挑战外的所有      Linux 发行版，还是      完整、连续      的      都是移植的论据：代码库      集成设置，      bhyve通常改进，潜伏的      bug 会暴露      为所有这些      用例提供出色的解决方案。
  和修复。如果      做得正确，移植到 FreeBSD 有切实      优势：      加速器
与任何其他应用一样，更改应上游化、解释，      可能过去十年 HPC
和持续维护。同时，      景观中最具颠覆性的变化是引入      我们应考虑减少 BSD-isms 以减轻      HPC 主流中的加速器，以及其
  移植任务。                                                    随后渗透到工作站和主      FreeBSD 上的持续维护或开发      流市场。尽管大多数 TOP500 安装
  简单。尽管没有      商业      仍然
  用于分析、调试、      不包含加速器，但加速器
  等的工具链      ，基础系统和 ports 集合      对 HPC 和工作站


          的重要性不言而喻。存在各种
  应用的重要性      我们的 libm 和来自 LLVM。这些应该
  无法被夸大。      已经是一个有趣的 HPC 平台，供
  存在各种      开发者使用。希望从长远来看，我们能够
  利用其潜力的      改善 Fortran 情况，使 FreeBSD 成为
  方法，最重要的      真正引人注目的 HPC 替代方案。
  是通过      还需要认识到，为
  NVIDIA 的专有 CUDA 语言、开放      HPC 改进 FreeBSD 不会损害它作为服务器
  替代方案 OpenCL 或通过      或工作站系统。相反，它很可能
  OpenMP offloading      对这些用例大有裨益。
  进行。      致谢
  这些编程框架都依赖      我要向所有 FreeBSD
  三个基本部分：内核驱动支持、      开发者和用户表达感激之情，是他们让 FreeBSD 成为
  编译器或库，以及运行时环境。      今天这样的平台。特别地，我要感谢
  通过 FreeBSDDesktop 项目，我们      FreeBSDDesktop 团队和我的导师 Matthew
  现在可以使用更近期的 AMD      Macy、Niclas Zeising、Steve Wills 和 Rene Ladan。
  和 Intel GPU 的开放驱动程序。NVIDIA GPU 仍然      我还要向 BSDTW
  由 ports 中的      会议和组织者表达深深的感谢，本文内容
  二进制驱动支持。      最初作为讲座在那里      展示。 •
  FreeBSD 没有官方 CUDA 支持；但      参考列表
  有些人      过去曾      [1] InsideHPC: https://insidehpc.com/
  在 Linux 上编译 CUDA 应用程序并      hpc-basic-training/what-is-hpc/
  通过 Linuxulator      [2] 数据源：UK 超级计算机 ARCHER
  在 FreeBSD 上运行它们。          上月应用使用情况，
          http://www.archer.ac.uk/status/codes
  OpenCL 支持处于合理状态：我们      [3] 数据源：TOP500 列表，2017 年 6 月，
  包含来自      https://www.top500.org/statistics/ list/
  Mesa 项目的非官方 OpenCL clover 库，用于 AMD GPU。其性能      [4] Flang github: https://github.com/
  绝对      flang-compiler/flang
  无法竞争，但工作相当      [5] OpenMP https://www.openmp.org/
  可靠。对于 Intel，我们包含其官方 beignet      [6] MPI forum https://www.mpi-forum.org/
  实现；但 Intel GPU      [7] FLAME 项目 https://github.com/flame
  在计算任务上竞争力较弱。开发      [8] Linuxism 讨论
  OpenCL 应用程序得到良好支持；我们包含      https://wiki.freebsd.org/AvoidingLinuxisms
  CPU OpenCL 模拟器 lang/pocl 和      [9] Radeon Open Compute
  完整性检查器 devel/oclgrind。通过      https://github.com/RadeonOpenCompute/
  OpenMP      offloading 利用加速器
  目前不支持，但      总体上
  尚未普及。
  一项重大改进可能是纳入
  Radeon Open Compute（ROCm）项目 [9]。它
  需要一个开放配套内核驱动 amdkfd，
  用于常规开放 amdgpu 驱动，并提供
  以 LLVM      为中心的大型开放生态系统
  编译器技术，支持 OpenCL 和
  CUDA 的开放竞争者——AMD
  GPU 上的 HIP。FreeBSDDesktop 团队现在积极
  正在移植 amdkfd，大型 ROCm
  栈应该直接，虽然有些
  劳动密集，可以移植。

      接下来？                       JOHANNES DIETERICH 自 6.1 版本开始使用 FreeBSD，
  最高优先级应该是审查 FLAME      一年前成为 ports 提交者。
  的 BLAS 和 LAPACK 库，并将它们添加为      白天，他过去九年在学术研究
  blaslapack 选择器的选项。其次，      中从事高性能计算工作，
  libomp 应通过 devel/llvm 成为 amd64 的      解决从全局优化
  默认选项。随后，对这些重大更改进行一些稳定      到量子化学方法的一系列
  和扩展到      问题。最近，他
  非 amd64      开始为 AMD 从事 GPU 加速深度
  架构将需要。在      学习工作。
  中期，我希望我们将能够
  让 ROCm 在 FreeBSD 上工作。另一个
  重要事项是在
  libm 和 LLVM 中正确支持 SIMD/向量化。

