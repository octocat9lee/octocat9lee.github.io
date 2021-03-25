---
title: shell 实现 TCP 重传率监测
tags:
  - 技术点滴
toc: true
date: 2021-03-25 10:16:52
---
  本文介绍一种使用 shell 统计 TCP 重传率脚本。
<!--more-->
``` bash

#!/bin/bash
SHELLDIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

tcp_re_ratio() {
    netstat -s -t >/tmp/netstat_s 2>/dev/null

    s_r=$(cat /tmp/netstat_s | grep 'segments send out' | awk '{print $1}')
    s_re=$(cat /tmp/netstat_s | grep 'segments retransmited' | awk '{print $1}')

    [ -e ${SHELLDIR}/s_r ] || touch ${SHELLDIR}/s_r
    [ -e ${SHELLDIR}/s_re ] || touch ${SHELLDIR}/s_re

    l_s_r=$(cat ${SHELLDIR}/s_r)
    l_s_re=$(cat ${SHELLDIR}/s_re)

    echo $s_r >${SHELLDIR}/s_r
    echo $s_re >${SHELLDIR}/s_re

    tcp_re_rate=$(echo "$s_r $s_re $l_s_r $l_s_re" | awk '{printf("%.2f",($2-$4)/($1-$3)*100)}')
    echo "$s_r | $s_re | $tcp_re_rate"
}

main() {
    while true
    do
        tcp_re_ratio
        sleep 5
    done
}

main

```
