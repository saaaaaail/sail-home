---
title: Tomcat定时检查服务状态的启动脚本
date: 2020-11-04 11:29:25
tags:
- Tomcat
---

# Tomcat启动脚本 
检查tomcat服务有没有挂掉
```shell
#! /bin/bash
FILE="/usr/local/tomcat/app-8080"
TOMCAT="/bin/startup.sh"
URL="127.0.0.1:8080/sys/getConfigContent?flag=DEFAULT_USER_HEAD_ICON"

RESP="$( curl -I -m 5 -o /dev/null -s -w %{http_code} $URL)"
echo $RESP

if [[ $RESP == *"200"* ]]; then
    echo '服务正常'
else
    echo '服务异常'
	PID=$(ps -ef|grep "$FILE"|grep -v "grep"|awk '{print $2}')
	echo $PID
	if [[ $PID == "" ]]; then
	   echo '为空了'
	else
	   echo '杀死进程'
	   kill -9 $PID
	fi
	sh $FILE$TOMCAT
	echo '重新启动'
fi
```

# 定时任务
```
* */1 * * * * sh xxx.sh 1>>1.txt 2>/dev/null
* */1 * * * * sleep 30;sh xxx.sh 1>>1.txt 2>/dev/null
```