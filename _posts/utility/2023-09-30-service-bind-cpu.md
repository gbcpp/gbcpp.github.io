---
layout: post
title: '后台服务绑定 CPU 核'
date: 2023-09-30
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover: 'https://images.unsplash.com/photo-1653629154302-8687b83825e2'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- 后台
---

> 一个 Prod 用的服务启动升级脚本，检查了 CPU 核心数，并按照指定顺序绑定 CPU 核心，以提升性能。



```bash
#!/bin/bash
SERVICENAME=$1
VERSION=$2
BINPATH=/home/devops/$SERVICENAME/bin/
PACKAGE=/home/devops/$SERVICENAME/package/

# xxx 进程名|进程文件名
PROCESS=$SERVICENAME.exe
LOCK_FILE="/$BINPATH/.xxxxx.$PROCESS.lock"

# we think cpu binding means will enable cpu perception
USE_CPU_TOPOLOGY_SERVER_TYPE="server"
USE_CPU_TOPOLOGY_TAG="vm_use_cpu_topology:true vm_bind_cpu:true"
USE_CPU_TOPOLOGY_PROVIDER="aws"
USE_CPU_TOPOLOGY_IDC=""

USE_CPU_TOPOLOGY=

# control bind core and xdp in hybrid deployment machine
is_hybrid=0

function CheckDir() {
    if [ ! -d $BINPATH ]; then sudo mkdir -p $BINPATH; fi
}

function CheckHybridDeploy() {
    get_message=`curl -4 https://meta.xxxxx.co/latest/meta-data/tag --max-time 10`
    match=`echo ${get_message} | grep -a "servicevosmixed:1" | wc -l`
    if [ ${match} -gt 0 ]; then
        is_hybrid=1
    fi
    echo "Is hybrid : ${is_hybrid}"
}

# 排他锁，防止相同时间内进行其它操作，比如操作时，防止zabbix触发自动拉起
function Lock() {
    ls "$LOCK_FILE" >/dev/null 2>&1
    if [[ $? -ne 0 ]]; then
        sudo touch "$LOCK_FILE"
    else
        lockTime=$(stat -t "$LOCK_FILE" | awk '{print $12}')
        currentTime=$(date +%s)
        duration=$((currentTime - lockTime))
        echo $duration
        if [[ $duration -gt 20 ]]; then
            sudo touch "$LOCK_FILE"
        else
            echo "Less than 20 seconds since last operation"
            exit 1
        fi
    fi
}

function Unlock() {
    sudo rm -f "$LOCK_FILE"
}

function NeedUseCPUTopology() { # not zero mean need bind cpu

    if [[ -n ${USE_CPU_TOPOLOGY} ]]; then
        return ${USE_CPU_TOPOLOGY}
    fi
    
    # check is vm
    if [[ "${USE_CPU_TOPOLOGY_SERVER_TYPE}" == *"$(hostnamectl | grep Chassis | cut -d ':' -f2 | xargs)"* ]]; then
        USE_CPU_TOPOLOGY=1
        return ${USE_CPU_TOPOLOGY}
    fi
    
    # get machine tag
    for machine_tag in $(timeout 10 curl https://meta.xxxxx.co/2022-09-13/meta-data/tag 2>/dev/null); do
        if [[ "${USE_CPU_TOPOLOGY_TAG}" == *"${machine_tag}"* ]]; then
            USE_CPU_TOPOLOGY=1
            return ${USE_CPU_TOPOLOGY}
        fi
    done
    
    # # get cluster name
    # if [[ "${USE_CPU_TOPOLOGY_IDC}" == *"$(curl https://meta.xxxxx.co/2022-09-13/meta-data/cluster 2>/dev/null)"* ]]; then
    #     USE_CPU_TOPOLOGY=1;
    #     return ${USE_CPU_TOPOLOGY}
    # fi
    
    # get provider
    # if [[ "${USE_CPU_TOPOLOGY_PROVIDER}" == *"$(curl https://meta.xxxxx.co/2022-09-13/meta-data/provider 2>/dev/null)"* ]]; then
    #     USE_CPU_TOPOLOGY=1;
    #     return ${USE_CPU_TOPOLOGY}
    # fi
    
    USE_CPU_TOPOLOGY=0
    return ${USE_CPU_TOPOLOGY}

}

# 获取当前机器物理核心数
function GetCpuNumber() {
    # if not use topology, we will presume machine have open HT, and just return all process count / 2
    NeedUseCPUTopology && { echo $(($(grep -c processor /proc/cpuinfo) / 2)) && return 0; }
    
    CORE=$(cat /proc/cpuinfo | grep "core id" | sort | uniq | wc -l)
    PHYSICAL=$(cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l)
    LOGIC=$((CORE * PHYSICAL))
    echo $LOGIC
}

function GetMemTotalSize() {
    memTotalSize=$(cat /proc/meminfo | grep MemTotal | awk '{print $2}')
    echo $memTotalSize
}

# 设置进程数
function GetProcessNumber() {
    local processNumber=1
    cpuNumber=$(GetCpuNumber)
    memTotalSize=$(GetMemTotalSize)
    if [ $cpuNumber -ge 10 ]; then
        processNumber_tmp=$((cpuNumber - 2))
        # 内存总容量足够的机器按满配起
        if [[ $memTotalSize -ge $((processNumber_tmp * 2359296)) ]]; then
            processNumber_pre=$processNumber_tmp
            if [[ $approuter_running -eq 1 ]]; then
                processNumber=$((processNumber_pre - 4))
            else
                processNumber=$processNumber_pre
            fi
        # 内存总容量不够的机器计算实际要起的进程数
        else
            processNumber_pre=$((memTotalSize / 2359296))
            if [[ $approuter_running -eq 1 ]]; then
                processNumber=$((processNumber_pre - 4))
            else
                processNumber=$processNumber_pre
            fi
        fi
    elif [[ $cpuNumber -gt 1 ]]; then
        processNumber_pre=$((cpuNumber - 1))
        if [[ $approuter_running -eq 1 ]]; then
            processNumber=$((processNumber_pre - 4))
        else
            processNumber=$processNumber_pre
        fi
    elif [[ $cpuNumber -eq 1 ]]; then
        processNumber=1
    fi
    if [ $processNumber -gt 30 ]; then
        processNumber=30
    fi
    echo $processNumber
}

function CheckApprouterProcessRunningStaus() {
    approuterNumber=$(ps aux | grep -w "reuseport.exe" | grep -v grep | wc -l)
    if [ $approuterNumber -gt 0 ]; then
        approuter_running=1
    else
        approuter_running=0
    fi
}

# 检查当前进程数
function CheckProcessNumber() {
    processNumber=$(GetProcessNumber)
    currentNumber=$(ps aux | grep -w "./$PROCESS" | grep -v 'gzip' | grep -v grep | grep -v $0 | grep -v "appRestart\|appStart\|appStop\|appUpgrade" | wc -l)
    
    if [[ $currentNumber -ne $processNumber ]]; then
        echo "current process number: $currentNumber, need process number: $processNumber"
        Unlock
        exit 1
    fi
}

# 检查当前进程数
function CheckNoProcess() {
    currentNumber=$(ps aux | grep -w "./$PROCESS" | grep -v 'gzip' | grep -v grep | grep -v $0 | grep -v "appRestart\|appStart\|appStop\|appUpgrade" | wc -l)

    if [[ $currentNumber -ne 0 ]]; then
        echo "current process number: $currentNumber, should be 0"
        Unlock
        exit 1
    fi
}

# 停止进程
function Stop() {
    sudo killall -q "$PROCESS"
    sleep 30
    echo "Stop Process finished"
    CheckNoProcess
}

# 升级程序
function Upgrade() {
    #检查文件是否存在
    if test ! -e "$PACKAGE/$SERVICENAME.exe"; then
        echo "upgrade package not exist"
        exit 1
    fi
    
    #检查程序及版本
    local exeVersion=$("$PACKAGE/$SERVICENAME.exe" -v | awk '{sub(/^[\t ]*/, "");print}')
    local upVersion="$PROCESS"_"$exeVersion"
    if [[ "$upVersion" != "$VERSION" ]]; then
        echo "version mismatch, package version: $upVersion, need version: $VERSION"
        Unlock
        exit 1
    fi
    
    if test ! -d "$BINPATH"; then
        sudo mkdir -p "$BINPATH"
    fi
    
    # copy程序到bin下
    sudo cp -f "$PACKAGE/$SERVICENAME.exe" "$BINPATH/$PROCESS"
}

# 启动进程
function Start() {
    cd "$BINPATH"
    sudo bash -c "echo 3 > /proc/sys/vm/drop_caches"
    sleep 10
    
    processNumber=$(GetProcessNumber)
    currentNumber=$(ps aux | grep -w "./$PROCESS" | grep -v 'gzip' | grep -v grep | grep -v $0 | grep -v "appRestart\|appStart\|appStop\|appUpgrade" | wc -l)
    
    echo "Need: $processNumber, Current: $currentNumber"
    run_process_command=${PROCESS}
    if [ ${is_hybrid} -eq 1 ]; then
            run_process_command="${run_process_command} --disable_xdp=true"
    fi
    for ((i = currentNumber; i < "$processNumber"; i++)); do
        sudo ./$run_process_command
        sleep 3
    done
}

# 绑定核心
function BindingProcesstoPhysicalCores() {

    NeedUseCPUTopology && return 0

    if [ ${is_hybrid} -eq 1 ]; then
        return 0
    fi
    
    TOTAL_CPU=$(grep -c processor /proc/cpuinfo)
    CORE=$(cat /proc/cpuinfo | grep "core id" | sort | uniq | wc -l)
    PHYSICAL=$(cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l)
    LOGIC=$((CORE * PHYSICAL))
    PIDS=$(ps aux | grep -w "./$PROCESS" | grep -v grep | grep -v $0 | grep -v "start\|ansible" | awk '{print $2}')
    
    if [[ $(($TOTAL_CPU / 2)) -eq $LOGIC ]]; then
        if [[ $(cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list | grep - | wc -l) -eq 1 ]]; then
            cpu_thread_seq="continuous" #CPU-No. vs Core ID Like CPU0-Core0_Thread0,CPU1-Core0_Thread1,CPU2-Core1_Thread0,CPU3-Core1_Thread1...
        else
            cpu_thread_seq="discontinuous" #CPU-No. vs Core ID Like CPU0-Core0_Thread0,CPU1-Core1_Thread0,..,CPU10-Core0_Thread1,CPU1-Core0_Thread1...
        fi
    elif [[ $TOTAL_CPU -eq $LOGIC ]]; then
        cpu_thread_seq="na" # CPU do not enable or not support Hyper-Threading
    fi
    
    if [[ $cpu_thread_seq = "discontinuous" ]]; then
        if [[ $approuter_running -eq 1 ]]; then #有app_router保留4个核心，从倒数第5个物理核开始使用
            cpu=$(($LOGIC - 5))
        else
            cpu=$(($LOGIC - 1)) #没有app_router正常从最后1个物理核开始使用
        fi
    
        for pid in ${PIDS[@]}; do
            sudo taskset -p --cpu-list $cpu $pid
            cpu=$(($cpu - 1))
        done
    elif [[ $cpu_thread_seq = "continuous" ]]; then
        if [[ $approuter_running -eq 1 ]]; then #有app_router保留4个核心即8个逻辑核，从倒数第10个逻辑核开始使用
            cpu=$(($TOTAL_CPU - 10))
        else
            cpu=$(($TOTAL_CPU - 2)) #没有app_router正常从倒数第2个逻辑核开始用
        fi
    
        for pid in ${PIDS[@]}; do
            sudo taskset -p --cpu-list $cpu $pid
            cpu=$(($cpu - 2))
        done
    elif [[ $cpu_thread_seq = "na" ]]; then
        for pid in ${PIDS[@]}; do
            sudo taskset -p 0xFFFF $pid
        done
    fi

}

CheckApprouterProcessRunningStaus &&
    CheckDir &&
    CheckHybridDeploy &&
    Lock &&
    Stop &&
    Upgrade &&
    Start &&
    BindingProcesstoPhysicalCores &&
    CheckProcessNumber
Unlock
```