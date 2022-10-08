#MySQL #Backup #Scripts

```
#!/bin/bash
######################################## metadata ###########################################################
# name:         xtrabackupScript_${version}.sh
# version:      v2.1
# function:     Backup MySQL 5.7/8.0 With percona-xtrabackup tools
# createBy:     zhenxing
# createDate:   2020-12-02
# updateDate:   2022-03-31
# percona-xtrabackup refLink
#   - privileges      https://www.percona.com/doc/percona-xtrabackup/8.0/using_xtrabackup/privileges.html
#   - download        https://www.percona.com/downloads/Percona-XtraBackup-LATEST/
#   - release_notes   https://www.percona.com/doc/percona-xtrabackup/8.0/release-notes.html
# releaseNote
#   - 2019-01-07 v1.0 Support MySQL 5.7 backup with innobackupex command
#   - 2020-10-27 v1.1 Fix restore logic for full backup only
#   - 2020-12-02 v2.0 Optimize log output format
#   - 2020-12-09 v2.1 Use xtrabackup replace innobackupex to maintain MySQL 5.7 and MySQL8.0 compatibility
#   - 2020-12-10 v2.2 Add check qpress and libev package
#   - 2020-12-11 v2.3 Add check xtrabackup and MySQL compatibility information
#   - 2020-12-15 v2.4 Add Print backup piece size and disable MD5 check(default)
#   - 2020-12-15 v2.5 Add support Xtrabackup 8.0.22-15 version check
#   - 2021-04-01 v2.6 repair checkBackupDirExists function logical
#   - 2021-04-09 v2.7 Identify which parameters need to be modified / Priority to run checkBackupDirExists function
#   - 2022-03-31 v2.8 repair verify BACKUP_PIECE_SIZE 5G->50G / remove --incremental parameter for increment backup(innobackupex was removed on xtrabackup 8.0)
# notice
#   - MySQL 5.7.x Use percona-xtrabackup 2.4.{LATEST}
#   - MySQL 8.0.x Use percona-xtrabackup 8.0.{LATEST}
######################################## metadata ###########################################################
######################################## Need to modify  ########
# Define MySQL Software Base
export MYSQL_BASE=/data/mysql/3306/base
export PATH=$MYSQL_BASE/bin:$PATH
# Define Database Connection Information
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=backup
MYSQL_PASS='backup'
MYSQL_CONFIG=/data/mysql/3306/my.cnf.3306
BACKUP_DIR=/data/mysql/3306/backup
######################################## The following parameters do not need to be modified normally ########
# Backup Type(full or incr)
BACKUP_TYPE=$1
# Define Backup Directory Information
BACKUP_PARAMS="--no-version-check --no-server-version-check --slave-info --parallel=4 --stream=xbstream --compress  --compress-chunk-size=256K --compress-threads=4 --ftwrl-wait-query-type=all --ftwrl-wait-timeout=120 --ftwrl-wait-threshold=120 --kill-long-queries-timeout=120 --kill-long-query-type=select"
BACKUP_PIECE_DIR=percona-backup-$(date +"%Y-%m-%d_%H%M%S")
BACKUP_PIECE_SUBFIX=qp.xbstream
BACKUP_LOG="xtrabackupScript.log"
#server_version=`mysql --no-defaults -h$MYSQL_HOST -P$MYSQL_PORT -u$MYSQL_USER -p$MYSQL_PASS -ss -e "select substring_index(@@version,'.',2);"`
# Define Script help/usage Information
usage(){
    echo ""
    echo "=====> Usage: sh $(basename $0) [full|incr]"
    echo "       全量备份:  sh $(basename $0) full"
    echo "       增量备份:  sh $(basename $0) incr"
}
# Logging Function
# 输出正常日志
logWrite()
{
    echo -e "`date +'%Y-%m-%d %H:%M:%S'` \033[32m[INFO] ${1} \033[0m" | tee --append ${BACKUP_DIR}/${BACKUP_LOG}
}
# 输出警告日志
logWriteWarning()
{
    echo -e "`date +'%Y-%m-%d %H:%M:%S'` \033[33m[Warning] ${1} \033[0m" | tee --append ${BACKUP_DIR}/${BACKUP_LOG}
}
# 输出错误日志并退出脚本
logWriteError()
{
    echo -e "`date +'%Y-%m-%d %H:%M:%S'` \033[31m[ERROR] ${1} \033[0m" | tee --append ${BACKUP_DIR}/${BACKUP_LOG}
    exit 1
}
## 检查MySQL是否可连接
checkMySQLAvailable(){
    server_version=`mysql --no-defaults -h$MYSQL_HOST -P$MYSQL_PORT -u$MYSQL_USER -p$MYSQL_PASS -ss -e "select concat(@@version,' ',@@version_comment);" 2>/dev/null`
    if [ `echo $?` -ne 0 ]; then
		logWriteError "连接MySQL数据库失败,请检查连接配置信息!!! \033[0m"
		logWriteError "HOST=$MYSQL_HOST PORT=$MYSQL_PORT USER=$MYSQL_USER PASS='***********'"
    else
		logWrite "连接MySQL数据库成功,MySQL版本为: ${server_version}"
    fi
}
## 检查xtrabackup是否安装
checkXtrabackupPackage(){    
    res=`which xtrabackup 2>&1 &>/dev/null`
    if [ `echo $?` -eq 0 ];then
        xtrabackup_version=`xtrabackup --version 2>&1 |awk '{match($0,/xtrabackup version +([0-9\.\-]+) based.*/,version);if (RSTART !=0){print(version[1])}}'`
        logWrite "Xtrabackup 已安装,当前版本为: ${xtrabackup_version}"
    else
        logWriteError "Xtrabackup 当前未安装,请下载安装"
    fi
}
## 检查备份工具与数据库的兼容性
compatibilityCheck(){
    xtrabackup_version=`xtrabackup --version 2>&1 |awk '{match($0,/xtrabackup version +([0-9]+\.[0-9]+)\.([0-9\-]+) based.*/,version);if (RSTART !=0){print(version[1])}}'`
    server_version=`mysql --no-defaults -h$MYSQL_HOST -P$MYSQL_PORT -u$MYSQL_USER -p$MYSQL_PASS -ss -e "select substring_index(@@version,'.',2);" 2>/dev/null` 
    ## 判断Xtrabackup与MySQL的兼容性
    if [ $xtrabackup_version == '8.0' ];then
        if [ $server_version == '8.0' ];then
            logWrite "Xtrabackup 大版本为[$xtrabackup_version],与MySQL大版本[$server_version]兼容"
        elif [ $server_version == '5.7' ];then
            logWriteError "Xtrabackup 大版本为[$xtrabackup_version],与MySQL大版本[$server_version]不兼容"
        else
            logWriteError "Xtrabackup 版本错误"
        fi
    elif [ $xtrabackup_version == '2.4' ];then
        if [ $server_version == '5.7' ];then
            logWrite "Xtrabackup 大版本为[$xtrabackup_version],与MySQL大版本[$server_version]兼容"
        elif [ $server_version == '8.0' ];then
            logWriteError "Xtrabackup 大版本为[$xtrabackup_version],与MySQL大版本[$server_version]不兼容"
        else
            logWriteError "MySQL 版本错误"
        fi
    else 
        logWriteError "Xtrabackup 版本错误"
    fi
}
## 检测qpress/libev依赖包是否安装
checkXtrabackupDependentsPackage(){
    res=`which qpress 2>&1 &>/dev/null`
    if [ `echo $?` -eq 0 ];then
        logWrite "Xtrabackup 依赖包 qpress 当前已安装"
    else
        logWriteError "qpress 当前未安装,需手动安装,下载地址为:http://www.quicklz.com/qpress-11-linux-x64.tar"
    fi
    res=`rpm -qa|grep -Ew libev 2>&1 &>/dev/null`
    if [ `echo $?` -eq 0 ];then
        logWrite "Xtrabackup 依赖包 libev  当前已安装"
    else
        logWriteError "libev 当前未安装,需手动安装"
    fi
}
## 检查备份目录是否存在
checkBackupDirExists(){
  if [ ! -d "$BACKUP_DIR/tmp" ];then
        mkdir -p $BACKUP_DIR/tmp
    logWrite "${BACKUP_DIR}/tmp 备份目录/临时目录已创建"
  else
    logWrite "${BACKUP_DIR} 备份目录已存在,无需创建"
  fi
}
## 触发全量备份逻辑
fullMySQLBackup(){
    
    ## 创建全量备份目录
    logWrite "创建全量备份目录: ${BACKUP_DIR}/$BACKUP_PIECE_DIR"
    mkdir -p ${BACKUP_DIR}/${BACKUP_PIECE_DIR}
    ## 触发全量备份
    logWrite "全量备份 操作开始,请等待..."
    xtrabackup --defaults-file=${MYSQL_CONFIG} --host=${MYSQL_HOST} --port=${MYSQL_PORT} --user=${MYSQL_USER} --password=${MYSQL_PASS} --tmpdir=${BACKUP_DIR}/tmp --extra-lsndir=${BACKUP_DIR}/tmp ${BACKUP_PARAMS} --backup  --target-dir=${BACKUP_DIR}/tmp >${BACKUP_DIR}/${BACKUP_PIECE_DIR}/${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} 2>>${BACKUP_DIR}/${BACKUP_PIECE_DIR}/${BACKUP_PIECE_DIR}'.log'
    if [ `echo $?` -ne 0 ]; then
	    logWriteError "全量备份失败,请查看全量备份日志 ${BACKUP_DIR}/${BACKUP_PIECE_DIR}/${BACKUP_PIECE_DIR}.log"
    else
	    logWrite "全量备份 操作完成"
	    logWrite "备份文件保存在: ${BACKUP_DIR}/${BACKUP_PIECE_DIR}"
        ## 输出备份大小
        cd ${BACKUP_DIR}/${BACKUP_PIECE_DIR} && BACKUP_PIECE_SIZE=`ls -al ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} |awk -F ' ' '{print $5}'`
        logWrite "本次全量备份大小为 ${BACKUP_PIECE_SIZE} bytes"
    fi 
}
## 触发增量备份逻辑
icrementalMySQLBackup(){
    if [ -f ${BACKUP_DIR}/tmp/index ];then	
        INCR_BASE_DIR=`head -n1 ${BACKUP_DIR}/tmp/index |awk -F '.' '{print $1}'`
        logWrite "获取到上一次全量备份 ${BACKUP_DIR}/${INCR_BASE_DIR}"
	else
        logWriteError "未获取到上一次全量备份文件元数据信息,请查看 ${BACKUP_DIR}/tmp 目录中是否存在index文件"
	fi
    logWrite "从上一次全量备份文件中获取增备备份起始点LSN" 
	INCREMENTAL_LSN=`tail -n1 ${BACKUP_DIR}/tmp/index |awk -F ':' '{print $2}'`
    logWrite "本次增量备份LSN为 ${INCREMENTAL_LSN}"
	logWrite "增量备份 操作开始,请等待..."
	xtrabackup --defaults-file=${MYSQL_CONFIG} --host=${MYSQL_HOST} --port=${MYSQL_PORT} --user=${MYSQL_USER} --password=${MYSQL_PASS} --tmpdir=${BACKUP_DIR}/tmp --extra-lsndir=${BACKUP_DIR}/tmp  ${BACKUP_PARAMS} --backup --incremental-lsn=$INCREMENTAL_LSN --target-dir=${BACKUP_DIR}/tmp   >$BACKUP_DIR/${INCR_BASE_DIR}/${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} 2>>$BACKUP_DIR/${INCR_BASE_DIR}/${BACKUP_PIECE_DIR}'.log'
	if [ `echo $?` -ne 0 ]; then
	    logWriteError "增量备份失败,请查看增量备份日志 ${BACKUP_DIR}/${INCR_BASE_DIR}/${BACKUP_PIECE_DIR}.log"
	else
		logWrite "增量备份 操作完成"
        logWrite "备份文件目录如下: ${BACKUP_DIR}/${INCR_BASE_DIR}"
        cd ${BACKUP_DIR}/${INCR_BASE_DIR} && BACKUP_PIECE_SIZE=`ls -al ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} |awk -F ' ' '{print $5}'`
        logWrite "本次增量备份大小为 ${BACKUP_PIECE_SIZE} bytes"
    
    fi
}
## 备份参数配置文件
backupMySQLConfigFile(){
    logWrite "开始备份MySQL参数配置文件 ${MYSQL_CONFIG}"
    cp ${MYSQL_CONFIG} ${BACKUP_DIR}/${BACKUP_PIECE_DIR}/my.cnf
    
    if [ `echo $?` -ne 0 ]; then
	    logWriteError "参数文件备份失败,请查看是否存在及权限是否正确"
    else
	    logWrite "参数文件备份成功,保存在 ${BACKUP_DIR}/${BACKUP_PIECE_DIR}/my.cnf"
	fi
}
## 生成全量备份索引文件
createFullBackupIndexFile(){
    logWrite "开始生成全量备份索引文件供增量备份使用"
    TO_LSN=`cat ${BACKUP_DIR}/tmp/xtrabackup_checkpoints |grep to_lsn |awk -F ' = ' '{print $2}'`
    echo "${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX}:${TO_LSN}" >${BACKUP_DIR}/tmp/index
    
    if [ `echo $?` -ne 0 ]; then
	    logWriteError "生成备份索引文件失败,请查看备份临时目录是否存在 ${BACKUP_DIR}/tmp"
    else
        logWrite "备份索引创建文件成功,保存在 ${BACKUP_DIR}/${BACKUP_PIECE_DIR}/index"
    fi
    logWrite "拷贝备份临时目录的元数据文件到全量备份目录"
    cp ${BACKUP_DIR}/tmp/* ${BACKUP_DIR}/${BACKUP_PIECE_DIR}
}
## 生成全量备份MD5校验文件
createFullMD5ChecksumFile(){
    logWrite "对备份集做MD5校验,生成校验码"
    cd ${BACKUP_DIR}/${BACKUP_PIECE_DIR}
    BACKUP_PIECE_SIZE=`ls -al ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} |awk -F ' ' '{print $5}'`
    ## 备份文件大于50G时不做MD5校验操作,防止备份时间过长
    logWrite "本次全量备份大小为 ${BACKUP_PIECE_SIZE} bytes"
    if [ ${BACKUP_PIECE_SIZE} -gt 5000000000 ];then
        logWriteWarning "备份文件大于50G,防止生成MD5操作耗时过长,不做MD5校验操作"
    else
        cd ${BACKUP_DIR}/${BACKUP_PIECE_DIR} && md5sum ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} >> md5sum.base
        if [ `echo $?` -ne 0 ]; then
	        logWriteError "生成校验码失败,请查看是否备份过大导致"
        else
            logWrite "生成校验码成功,保存在 ${BACKUP_DIR}/${BACKUP_PIECE_DIR}/md5sum.base"
        fi
    fi
  
}
## 创建增量备份索引文件
createIncrBackupIndexFile(){
    logWrite "开始生成增量备份索引文件供后续增量备份使用"
    TO_LSN=`cat ${BACKUP_DIR}/tmp/xtrabackup_checkpoints |grep to_lsn |awk -F ' = ' '{print $2}'`
    echo "${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX}:${TO_LSN}" >>${BACKUP_DIR}/tmp/index
    if [ `echo $?` -ne 0 ]; then
	    logWriteError "生成增量备份索引文件失败,请查看备份临时目录是否存在 ${BACKUP_DIR}/tmp"
    else
        logWrite "生成增量备份索引文件成功 ${BACKUP_DIR}/${INCR_BASE_DIR}/index"
    fi
    cp ${BACKUP_DIR}/tmp/* ${BACKUP_DIR}/${INCR_BASE_DIR}
    
}
## 生成增量备份MD5校验文件
createIncrMD5ChecksumFile(){
    logWrite "对备份集做MD5校验,生成校验码"
    cd ${BACKUP_DIR}/${INCR_BASE_DIR}
    
    ## 备份文件大于50G时不做MD5校验操作,防止备份时间过长
    logWrite "本次增量备份大小为 ${BACKUP_PIECE_SIZE} bytes"
    BACKUP_PIECE_SIZE=`ls -al ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} |awk -F ' ' '{print $5}'`
    if [ ${BACKUP_PIECE_SIZE} -gt 50000000000 ];then
        logWriteWarning "备份文件大于50G,防止生成MD5操作耗时过长,不做MD5校验操作,备份大小为 ${BACKUP_PIECE_SIZE} bytes"
    else
        cd ${BACKUP_DIR}/${INCR_BASE_DIR} && md5sum ${BACKUP_PIECE_DIR}.${BACKUP_PIECE_SUBFIX} >> md5sum.base
        if [ `echo $?` -ne 0 ]; then
	        logWriteError "生成校验码失败,请查看是否备份过大导致"
        else
            logWrite "生成校验码成功 ${BACKUP_DIR}/${INCR_BASE_DIR}/md5sum.base"
        fi
    fi
}
## 识别过期的备份集
markBackupAsExpired(){
    logWrite "以下备份文件即将过期,可手工删除[默认保留30天]"
    for i in `find ${BACKUP_DIR} -maxdepth 1 -mtime +30 -name "percona-backup*"`
    do
	    logWriteWarning "${i} 已被标记为过期,建议手工确认后删除!!!"
    done
}
## 主函数入口
Main(){
    case ${BACKUP_TYPE} in
    full)
        logWrite "===============> FULL BACKUP FLAG"
        checkBackupDirExists
        checkMySQLAvailable
        checkXtrabackupPackage
        compatibilityCheck
        checkXtrabackupDependentsPackage
        checkBackupDirExists
        fullMySQLBackup
        backupMySQLConfigFile
        createFullBackupIndexFile
        createFullMD5ChecksumFile
        markBackupAsExpired;;
    incr)
        logWrite "===============> INCR BACKUP FLAG"
        checkBackupDirExists
        checkMySQLAvailable
        checkXtrabackupPackage
        compatibilityCheck
        checkXtrabackupDependentsPackage
        icrementalMySQLBackup
        createIncrBackupIndexFile
        createIncrMD5ChecksumFile
        markBackupAsExpired;;
    *)
        usage;;
    esac
}
Main
```