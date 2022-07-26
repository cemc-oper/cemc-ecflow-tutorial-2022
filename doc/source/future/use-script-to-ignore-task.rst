使用脚本跳过任务
================

接下来我们创建 dobugs 任务，该任务在有台风时，进行涡旋初始化，没有台风时则跳过。

类似 get_message 中的代码，我们通过检测台风报文文件是否存在来判断是否有台风。
我们一般将这类通用代码放到 include 头文件中。

更新工作流定义
--------------

更新 ``${TUTORIAL_HOME}/def`` 中的工作流定义文件 **cma_tym.py**：

.. code-block:: bash
    :linenos:
    :emphasize-lines: 72-74

    import os

    import ecflow


    def slurm_serial(class_name="serial"):
        variables = {
            "ECF_JOB_CMD": "slsubmit6 %ECF_JOB% %ECF_NAME% %ECF_TRIES% %ECF_TRYNO% %ECF_HOST% %ECF_PORT%",
            "ECF_KILL_CMD": "slcancel4 %ECF_RID% %ECF_NAME% %ECF_HOST% %ECF_PORT%",
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


    print(defs)
    def_output_path = str(os.path.join(def_path, "cma_tym.def"))
    defs.save_as_defs(def_output_path)


72-74 行添加 dobugs 任务，使用串行队列 serial 运行。

挂起 cma_tym 节点，更新 ecFlow 上的工作流：

.. code-block:: bash

    cd ${TUTORIAL_HOME}/def/ecffiles
    python cma_tym.py
    ecflow_client --port 43083 --replace /cma_tym cma_tym.def

创建头文件
-----------

在 ``${TUTORIAL_HOME}/def`` 中的创建头文件 **check_message.h**：

.. code-block:: bash

    #----this file is just for check TC exist or not -----#
    if [ ! -s ${MSG_DIR}/tc_report_${START_TIME}.txt -a ! -s ${MSG_DIR}/tc_message_global_${START_TIME} ] ;then
      echo " "
      echo " No TC exists, Skip...  "
      echo " "

      date
      ecflow_client --complete  # Notify SMS of a normal end
      trap 0       # Remove all traps
      exit 0       # End the shell
    fi

上述代码检查两种台风报文是否存在，如果都不存在，则调用 ``ecflow_client --complete`` 和 ``exit 0`` 结束脚本运行。

创建任务脚本
-------------

在 ``${TUTORIAL_HOME}/ecffiles`` 中的创建 ecf 脚本文件 **dobugs.ecf**：

.. code-block:: bash

    #!/bin/ksh
    %include <slurm_serial.h>
    %include <head.h>
    %include <configure.h>
    #--------------------------------------

    %include <check_message.h>

    #===========================#
    RUN_DIR=${CYCLE_RUN_DIR}
    cd ${RUN_DIR}

    #===========================#
    rm -f xbfile.dat tc_report_${START_TIME}.txt tc_report_${LAST_TIME}.txt tc_message_global_${START_TIME} tc_message_global_${LAST_TIME}
    ln -sf ${CYCLE_VTX_DIR}/xb${START_TIME}000.dat xbfile.dat

    test -s ${MSG_DIR}/tc_message_global_${START_TIME} && ln -sf ${MSG_DIR}/tc_message_global_${START_TIME} ./
    test -s ${MSG_DIR}/tc_message_global_${LAST_TIME} && ln -sf ${MSG_DIR}/tc_message_global_${LAST_TIME} ./

    test -s ${MSG_DIR}/tc_report_${START_TIME}.txt && ln -sf ${MSG_DIR}/tc_report_${START_TIME}.txt ./
    test -s ${MSG_DIR}/tc_report_${LAST_TIME}.txt && ln -sf ${MSG_DIR}/tc_report_${LAST_TIME}.txt ./

    rm -f namelist.bogus
    rm -f bogus.dat envir.dat spevtx.dat vtxdom*.txt
    ${PROGRAM_SCRIPT_DIR}/bogusinput.pl ${START_TIME}
    ${PROGRAM_BIN_DIR}/bogus.exe < namelist.bogus

    mv -f bogus.dat ${CYCLE_VTX_DIR}/bogus${START_TIME}000.dat
    test -f env00.dat  && mv -f env00.dat  ${CYCLE_VTX_DIR}/envir${START_TIME}000.dat
    test -f envtx.dat  && mv -f envtx.dat  ${CYCLE_VTX_DIR}/envtx${START_TIME}000.dat
    test -f spvtx.dat  && mv -f spvtx.dat  ${CYCLE_VTX_DIR}/spvtx${START_TIME}000.dat
    test -f envir.dat  && mv -f envir.dat  ${CYCLE_VTX_DIR}/envir${START_TIME}000.dat
    test -f spevtx.dat && mv -f spevtx.dat ${CYCLE_VTX_DIR}/spvtx${START_TIME}000.dat

    if [ -f vtxdom*.txt ] ; then
      for file in vtxdom*.txt ; do
        prefix=$(echo $file | cut -c1-10)
        suffix=$(echo $file | cut -c12-18)
        mv -f $file ${CYCLE_VTX_DIR}/${prefix}_${START_TIME}${suffix}
      done
    fi

    rm -f xbfile.dat tc_report_${START_TIME}.txt tc_report_${LAST_TIME}.txt
    #===========================#
    ${PROGRAM_SCRIPT_DIR}/xbctl.csh -C 0 bogus ${START_TIME} 000
    test -f bogus${START_TIME}000.ctl && mv -f bogus${START_TIME}000.ctl ${CYCLE_VTX_DIR}/

    #---------------------------------------
    %include <tail.h>

运行任务
----------

在 ecFlowUI 上运行 dobugs 任务。