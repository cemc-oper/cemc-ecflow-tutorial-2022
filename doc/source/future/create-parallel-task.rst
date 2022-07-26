创建并行任务
============

接下来我们创建两个并行任务：

- psi：生成侧边界及初始场
- gcas：云分析

并行任务与之前创建的串行任务类似，只不过要额外设置运行节点数和每个节点使用的 CPU 个数。

更新工作流定义
--------------

更新 ``${TUTORIAL_HOME}/def`` 中的工作流定义文件 **cma_tym.py**：

.. code-block::
    :linenos:
    :emphasize-lines: 15-23,87-93

    import os

    import ecflow


    def slurm_serial(class_name="serial"):
        variables = {
            "ECF_JOB_CMD": "slsubmit6 %ECF_JOB% %ECF_NAME% %ECF_TRIES% %ECF_TRYNO% %ECF_HOST% %ECF_PORT%",
            "ECF_KILL_CMD": "slcancel4 %ECF_RID% %ECF_NAME% %ECF_HOST% %ECF_PORT%",
    	    "CLASS": class_name,
        }
        return variables


    def slurm_parallel(nodes, tasks_per_node=32, class_name="normal"):
        variables = {
            "ECF_JOB_CMD": "slsubmit6 %ECF_JOB% %ECF_NAME% %ECF_TRIES% %ECF_TRYNO% %ECF_HOST% %ECF_PORT%",
            "ECF_KILL_CMD": "slcancel4 %ECF_RID% %ECF_NAME% %ECF_HOST% %ECF_PORT%",
            "NODES": nodes,
            "TASKS_PER_NODE": tasks_per_node,
    	    "CLASS": class_name,
        }
        return variables


    current_path = os.path.dirname(__file__)
    tutorial_base = os.path.abspath(os.path.join(current_path, "../"))
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

        suite.add_limit("total_tasks", 10)
        suite.add_inlimit("total_tasks")

        with suite.add_task("copy_dir") as tk_copy_dir:
            pass

        with suite.add_task("get_message") as tk_get_message:
            tk_get_message.add_trigger("./copy_dir == complete")
            tk_get_message.add_variable(slurm_serial("serial"))
            tk_get_message.add_event("arrived")
            tk_get_message.add_event("peaceful")

        with suite.add_family("get_ncep") as fm_get_ncep:
            fm_get_ncep.add_trigger("./get_message == complete")
            fm_get_ncep.add_variable(slurm_serial("serial"))
            for hour in range(0, 120 + 1, 3):
                hour_string = "{hour:03}".format(hour=hour)
                with fm_get_ncep.add_task(hour_string) as tk_hour:
                    tk_hour.add_variable("FFF", hour_string)
                    tk_hour.add_variable(
                        "ECF_SCRIPT_CMD",
                        "cat {def_path}/ecffiles/getgmf_ncep.ecf".format(def_path=def_path)
                    )

        with suite.add_task("extgmf") as tk_extgmf:
            tk_extgmf.add_trigger("./get_ncep == complete")
            tk_extgmf.add_variable(slurm_serial("serial"))

        with suite.add_task("pregmf") as tk_pregmf:
            tk_pregmf.add_trigger("./extgmf == complete")
            tk_pregmf.add_variable(slurm_serial("serial"))

        with suite.add_task("dobugs") as tk_dobugs:
            tk_dobugs.add_trigger("./pregmf == complete")
            tk_dobugs.add_variable(slurm_serial("serial"))

        with suite.add_task("psi") as tk_psi:
            tk_psi.add_trigger("./dobugs == complete")
            tk_psi.add_variable(slurm_parallel(4, 32, "normal"))

        with suite.add_task("gcas") as tk_psi:
            tk_psi.add_trigger("./psi == complete")
            tk_psi.add_variable(slurm_parallel(4, 32, "normal"))


    print(defs)
    def_output_path = str(os.path.join(def_path, "cma_tym.def"))
    defs.save_as_defs(def_output_path)

新增代码解析：

- 15-23 行定义 ``slurm_parallel`` 函数，定义提交 Slurm 并行作业需要的一些变量。
- 87-93 行定义两个并行任务，psi 和 gcas，使用并行队列 normal 运行，每个任务需要 4 个节点

挂起 cma_tym 节点，更新 ecFlow 上的工作流：

.. code-block:: bash

    cd ${TUTORIAL_HOME}/def/ecffiles
    python cma_tym.py
    ecflow_client --port 43083 --replace /cma_tym cma_tym.def


创建头文件
----------

在 ``${TUTORIAL_HOME}/def/include`` 中创建头文件 **slurm_parallel.h**：

.. code-block:: bash

    ## This is a head file for Slurm serial job.
    #SBATCH -J GRAPES
    #SBATCH -p %CLASS%
    #SBATCH -N %NODES%
    #SBATCH --ntasks-per-node=%TASKS_PER_NODE%
    #SBATCH -o %ECF_JOBOUT%
    #SBATCH -e %ECF_JOBOUT%.err
    #SBATCH --comment=GRAPES
    #SBATCH -t 00:60:00
    #SBATCH --no-requeue


创建任务脚本
--------------

在 ``${TUTORIAL_HOME}/def/ecffiles`` 中创建 ecf 脚本 **psi.ecf**：

.. code-block:: bash
    :emphasize-lines: 3

    #!/bin/ksh
    %include <slurm_parallel.h>
    #SBATCH -t 00:15:00
    %include <head.h>
    %include <configure.h>
    #--------------------------------------

    do_static=0

    run_dir=${CYCLE_RUN_DIR}
    cd ${run_dir}

    cp -vr ${PROGRAM_CON_DIR}/grapes/run/* .

    rm -f bogus.dat
    rm -f namelist.input
    rm -f grapesinput grapesbdy

    echo "[INFO] use cma-ncep bckg"
    ${PROGRAM_SCRIPT_DIR}/do_grapesd01.csh \
      ${START_TIME} ${END_TIME} ${START_TIME} ${GMF_TINV} ${FORECAST_LENGTH} ${RMF_TINV}

    if [ -s ${MSG_DIR}/tc_report_${START_TIME}.txt -o -s ${MSG_DIR}/tc_message_global_${START_TIME} ] ;then
      ln -sf ${CYCLE_VTX_DIR}/bogus${START_TIME}000.dat bogus.dat
      sed -e "s#do_bogus.*=#do_bogus = .true., ! #" namelist.input > namelist.tmp
      mv -f namelist.tmp namelist.input
    fi

    if [ do_static -eq 1 -o ! -f static_data ] ; then
      ln -sf ${GEOGDIR} ./geog_data
      sed -e "s#do_static_data.*=#do_static_data = .true., ! #" namelist.input > namelist.tmp
      mv -f namelist.tmp namelist.input
    fi

    ulimit -s unlimited

    # module load compiler/intel/composer_xe_2017.2.174
    # module load mpi/intelmpi/2017.2.174
    export I_MPI_PMI_LIBRARY=/opt/gridview/slurm17/lib/libpmi.so

    srun ${PROGRAM_BIN_DIR}/psi.exe

    #---------------------------------------
    %include <tail.h>

注意标亮的第三行设置墙钟时间为 15 分钟，会覆盖 **slurm_parallel.h** 头文件中定义的 60 分钟墙钟时间。

.. note::

    定义精确的墙钟时间会加快作业的排队效率，短时限的任务没有限时的任务更容易排上队。

在 ``${TUTORIAL_HOME}/def/ecffiles`` 中创建 ecf 脚本 **gcas.ecf**：

.. code-block::

    #!/bin/ksh
    %include <slurm_parallel.h>
    #SBATCH -t 00:10:00
    %include <head.h>
    %include <configure.h>
    #--------------------------------------
    obs_dir="/g1/COMMONDATA/obs/OPER/NWPC"

    run_dir=${CYCLE_RUN_DIR}
    cd $run_dir

    #===========================#
    # check grapesinput namelist.input

    cp ${PROGRAM_SCRIPT_DIR}/gcasnamelist.sh .

    ln -sf grapesinput grapesinput${START_TIME}00
    rm -f qcqr_gcas_${START_TIME}00 radar_temperatue.dat

    ./gcasnamelist.sh  $START_TIME $obs_dir 1 .false.

    ulimit -s unlimited

    module load compiler/intel/composer_xe_2017.2.174
    module load mpi/intelmpi/2017.2.174
    export I_MPI_PMI_LIBRARY=/opt/gridview/slurm17/lib/libpmi.so

    srun ${PROGRAM_BIN_DIR}/gcas_new.exe

    #============================

    iret=$?
    if [ $iret -eq 0 -a -e qcqr_gcas_${START_TIME}00 ];then
       echo "gcas complete...."
    else
       echo "gcas failed ...."
       exit 1
    fi

    cat namelist.input | sed -e "s#warm_start .*=#warm_start = .T., ! #" | sed -e "s#do_cld .*=#do_cld = .T., ! #" > namelist.tmp
    mv namelist.tmp namelist.input

    #---------------------------------------
    %include <tail.h>

运行任务
-----------

在 ecFlowUI 中运行任务 psi 和 gcas，检查输出日志。

.. note::

    CMA-PI 计算资源比较紧张，作业较多，ecFlow 提交的并行作业可能会排队。
    下图中 gcas 作业就处于 submitted 状态，表明作业可能正在排队。

    .. image:: image/ecflow-ui-submitted-task.png

    调用 squeue 命令可以查看排队的作业：

    .. code-block:: bash

        squeue -u wangdp

    .. code-block::

           JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
        43420772    normal   GRAPES   wangdp PD       0:00      4 (Priority)
