定义一个工作流
===============

下面首先创建只有一个任务的工作流，在 ecFlow 中工作流被称为 suite。

创建 python 文件
-----------------

在 ``${TUTORIAL_HOME}/def`` 目录中创建文件 **cma_tym.py**：

.. code-block:: bash

    cd ${TUTORIAL_HOME}/def

.. code-block:: py

    import os

    import ecflow


    tutorial_base = "/g8/JOB_TMP/wangdp/tutorial/ecflow"
    def_path = os.path.join(tutorial_base, "def")
    ecfout_path = os.path.join(tutorial_base, "ecfout")
    program_base_dir = os.path.join(tutorial_base, "program/grapes-tym-program")
    run_base_dir = os.path.join(tutorial_base, "workdir")

    defs = ecflow.Defs()

    with defs.add_suite("cma_tym") as suite:
        suite.add_variable("PROGRAM_BASE_DIR", program_base_dir)
        suite.add_variable("RUN_BASE_DIR", run_base_dir)

        suite.add_variable("ECF_INCLUDE", os.path.join(def_path, "include"))
        suite.add_variable("ECF_FILES", os.path.join(def_path, "ecffiles"))

        suite.add_variable("USE_GRAPES", ".false.")
        suite.add_variable("FORECAST_LENGTH", 120)
        suite.add_variable("GMF_TINV", 3)
        suite.add_variable("RMF_TINV", 3)
        suite.add_variable("USE_GFS", 12)

        suite.add_variable("ECF_DATE", "20220704")
        suite.add_variable("HH", "00")

        with suite.add_task("copy_dir") as tk_copy_dir:
            pass

    print(defs)
    def_output_path = str(os.path.join(def_path, "cma_tym.def"))
    defs.save_as_defs(def_output_path)

上述脚本主要完成如下操作：

- 定义名为 ``cma_tym`` 的工作流 (suite)
- 为 suite 定义多个变量 (variable)，包括目录、模式配置、运行日期和时次等
- 定义名为 ``copy_dir`` 的任务 (task)
- 将工作流定义写入到文件 cma_tym.def 中

生成 def 文件
-------------

运行 Python 脚本 **cma_tym.py**，生成工作流定义文件 **cma_tym.def**：

.. code-block:: bash

    cd ${TUTORIAL_HOME}/def
    python cma_tym.py

**cma_tym.def** 文件内容如下：

.. code-block::

    # 4.11.1
    suite cma_tym
      edit PROGRAM_BASE_DIR '/g8/JOB_TMP/wangdp/tutorial/ecflow/program/grapes-tym-program'
      edit RUN_BASE_DIR '/g8/JOB_TMP/wangdp/tutorial/ecflow/workdir'
      edit ECF_INCLUDE '/g8/JOB_TMP/wangdp/tutorial/ecflow/def/include'
      edit ECF_FILES '/g8/JOB_TMP/wangdp/tutorial/ecflow/def/ecffiles'
      edit USE_GRAPES '.false.'
      edit FORECAST_LENGTH '120'
      edit GMF_TINV '3'
      edit RMF_TINV '3'
      edit USE_GFS '12'
      edit ECF_DATE '20220704'
      edit HH '00'
      task copy_dir
    endsuite
    # enddef
