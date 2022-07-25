.. cemc-ecflow-tutorial documentation master file, created by
   sphinx-quickstart on Sun Jul 24 19:53:09 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

CEMC ecFlow 教程
================================================

本教程用于 CEMC 2022 培训的 ecFlow 上机实习环节。

本教程以 **简化版** CMA-TYM 为例说明如何使用 ecFlow 搭建并运行数值预报模式流程。

教程源码：https://github.com/perillaroc/cemc-ecflow-tutorial-2022

在线访问：https://cemc-ecflow-tutorial-2022.readthedocs.io

为什么不用虚拟工作流介绍？
--------------------------

ecFlow 官网教程使用虚拟工作流，用非常简单的脚本模拟工作流运行情况。

考虑到虚拟工作流与中心使用 ecFlow 的实际情况有所脱节，本次上机实习尝试选择实际的工作流来介绍 ecFlow 的用法，期望能给大家留下更直观的印象。

如果大家对使用 ecFlow 感兴趣，可以访问 ECMWF 的官网查看最新教程（介绍最新的 v5 版本）：

https://confluence.ecmwf.int/display/ECFLOW/Tutorial

也可以访问我之前翻译的中文版教程（仅介绍 v4 版本）：

https://perillaroc.github.io/ecflow-tutorial-cn/

为什么选择 CMA-TYM 模式？
--------------------------

目前 CEMC 有 5 个主要数值天气预报模式业务系统，均使用 ecFlow 实时运行并维护：

- CMA-GFS
- CMA-MESO
- CMA-TYM
- CMA-GEPS
- CMA-REPS

因为本轮培训有专题介绍 CMA-GFS 和 CMA-MESO 的运行脚本，ecFlow 实战环节不再重复介绍。
集合预报系统 CMA-GEPS 和 CMA-REPS 都有众多集合成员，系统流程比较复杂，不适合初学者。

CMA-TYM 模式流程清晰明了，包含数值天气预报模式系统常见的几类模块：

- 观测资料获取：从观测资料数据库检索台风报文
- 资料预处理：处理 NCEP GFS 数据
- 同化：CMA-TYM 本身没有同化模块，可以将云分析看成一种同化
- 模式积分：grapes.exe 程序
- 后处理：处理模式输出的二进制数据，生成制作产品需要的数据
- 产品制作：生成图片产品

同时，CMA-TYM 模式的大部分模块都进行了封装，脚本比较简单，非常适合 ecFlow 的初学者。

因此，本次培训选择 CMA-TYM 模式作为示例讲解如何使用 ecFlow 搭建模式流程。
这也是第一次尝试使用实际系统来介绍 ecFlow。

教程内容
---------

本教程由三部分组成：

- 开始使用：介绍如何启动 ecFlow 服务，并创建只有一个任务的工作流。
- 构建流程：添加更多任务，创建从报文获取到云分析的多个任务，介绍如何使用触发器设定运行流程，以及如何将作业提交到 Slurm 队列中。
- 进阶：添加模式积分、产品后处理等任务，形成一个完整的 CMA-TYM 工作流并运行。

.. toctree::
   :maxdepth: 2
   :hidden:
   :caption: 开始使用

   started/introduction
   started/prepare
   started/start-ecflow-server
   started/create-a-suite
   started/load-the-suite
   started/create-includes
   started/create-task-script

.. toctree::
   :maxdepth: 2
   :hidden:
   :caption: 构建流程

   future/add-another-task
   future/add-family
   future/add-more-tasks
   future/use-script-to-ignore-task
   future/create-parallel-task

.. toctree::
   :maxdepth: 2
   :hidden:
   :caption: 进阶

   advance/create-model-task
   advance/create-post-tasks
   advance/create-archive-task
   advance/run-the-flow