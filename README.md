
### 阿里云IP遭受DDOS攻击 快速切换IP实践


#### \#1 介绍



> 运行平台: 阿里云
> 访问链路: 域名 \-\> 负载均衡EIP \-\> 容器


* 网站无法访问，查询服务运行正常，查询公网流量异常高后断流了
* 咨询工程师是公网IP遭受DDOS攻击后触发风控安全DDOS黑洞断流
* 阿里云EIP默认提供不超过5Gbps的基础DDoS防护能力
* 创建shell脚本检查网站断流后快速切换公网IP恢复




---


#### \#2、创建shell脚本实践


##### \#2\.1 检测域名是否可达



```


|  | # 域名 |
| --- | --- |
|  | domain_name="elvin.vip" |
|  | domain_sub="k8s-lb" |
|  | if ping -c 1 $domain_sub.$domain_name &> /dev/null; then |
|  | echo "$(date +'%F %T') $domain_sub.$domain_name is online" |
|  | else |
|  | echo "$(date +'%F %T') $domain_sub.$domain_name is not online." |
|  | fi |


```

##### \#2\.2 查询负载均衡器的公网IP



```


|  | #安装aliyun cli |
| --- | --- |
|  | wget https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz |
|  | tar -zxf aliyun-cli-linux-latest-amd64.tgz -C /usr/local/bin/ |
|  | rm -f aliyun-cli-linux-latest-amd64.tgz |
|  |  |
|  | # 设置阿里云访问密钥和区域ID |
|  | export ALICLOUD_ACCESS_KEY_ID='key_id_xxx' |
|  | export ALICLOUD_ACCESS_KEY_SECRET='key_secret_xxx' |
|  | export ALICLOUD_REGION_ID='cn-shanghai' |


```


```


|  | #负载均衡实例id |
| --- | --- |
|  | LOAD_BALANCER_ID="lb-xxxxxxxx" |
|  |  |
|  | # 查询绑定LB的EIP |
|  | EIP_INFO=$(aliyun vpc DescribeEipAddresses --RegionId $ALICLOUD_REGION_ID | jq --arg lb_id "$LOAD_BALANCER_ID" '.EipAddresses.EipAddress[] | select(.InstanceId == $lb_id and .InstanceType == "SlbInstance")') |
|  | # 提取EIP的ID和IP地址 |
|  | OLD_EIP_IP=$(echo $EIP_INFO | jq -r '.IpAddress') |
|  | OLD_EIP_ID=$(echo $EIP_INFO | jq -r '.AllocationId') |


```

##### \#2\.3 创建新的EIP



```


|  | # 创建EIP 按量付费模式 峰值带宽100M |
| --- | --- |
|  | EIP_OUTPUT=$(aliyun vpc AllocateEipAddress --RegionId $ALICLOUD_REGION_ID --InternetChargeType PayByTraffic --Bandwidth 100 --Name $domain_sub) |
|  | # 获取EIP的ID和IP地址 |
|  | EIP_ID=$(echo $EIP_OUTPUT | jq -r '.AllocationId') |
|  | EIP_IP=$(echo $EIP_OUTPUT | jq -r '.EipAddress') |


```

##### \#2\.4 负载均衡器绑定新的EIP



```


|  | # 解绑现有的EIP |
| --- | --- |
|  | aliyun vpc UnassociateEipAddress --RegionId $ALICLOUD_REGION_ID --AllocationId $OLD_EIP_ID --InstanceId $LOAD_BALANCER_ID --InstanceType SlbInstance |
|  |  |
|  | # 绑定EIP到负载均衡器 |
|  | ASSOCIATE_OUTPUT=$(aliyun vpc AssociateEipAddress --RegionId $ALICLOUD_REGION_ID --AllocationId $EIP_ID --InstanceId $LOAD_BALANCER_ID --InstanceType SlbInstance) |
|  |  |
|  | # 释放旧的EIP |
|  | aliyun vpc ReleaseEipAddress --AllocationId $OLD_EIP_ID |


```

##### \#2\.5 更新域名的A记录



```


|  | # 获取域名RecordId |
| --- | --- |
|  | RECORD_ID=$(aliyun alidns DescribeDomainRecords --DomainName $domain_name | jq -r --arg rr "$domain_sub" '.DomainRecords.Record[] | select(.RR == $rr) | .RecordId') |
|  |  |
|  | # 更新域名的A记录 |
|  | aliyun alidns UpdateDomainRecord --RecordId $RECORD_ID --RR $domain_sub --Type A --Value $EIP_IP |


```



---


#### \#3 完整的shell实例



```


|  | #!/bin/bash |
| --- | --- |
|  | # aliyun.lb.eip.update.sh |
|  | # */12 * * * *  bash /opt/aliyun.lb.eip.update.sh |
|  |  |
|  | # 域名和负载均衡相关信息 |
|  | domain_name="elvin.vip" |
|  | domain_sub="k8s-lb" |
|  | LOAD_BALANCER_ID="lb-xxxxxxxx" |
|  |  |
|  | #file |
|  | [ -d /data/txt ] || mkdir -p /data/txt |
|  | ckFile=/data/txt/$domain_sub.$domain_name.ck |
|  | runLog=/data/txt/$domain_sub.$domain_name.log |
|  |  |
|  | #跳过一次执行 |
|  | if [ -f $ckFile ]; then |
|  | now_time=$(date +%s) |
|  | file_time=$(stat -c %Y $ckFile) |
|  | time_diff=$((now_time - file_time)) |
|  | # 判断是否超过10分钟 |
|  | if [ $time_diff -ge 600 ]; then |
|  | rm -f $ckFile |
|  | echo "$(date +'%F %T') skip run once" >>$runLog |
|  | exit 0 |
|  | fi |
|  | fi |
|  |  |
|  | # 检测域名是否可达,错误时连续检查3次 |
|  | for((i=1; i<4; i++));do |
|  | if ping -c 1 $domain_sub.$domain_name &> /dev/null; then |
|  | echo "$(date +'%F %T') $domain_sub.$domain_name is online" >>$runLog |
|  | nk=99 |
|  | i=99 |
|  | exit 0 |
|  | else |
|  | echo "$(date +'%F %T') $domain_sub.$domain_name is not online. Retrying..." >>$runLog |
|  | sleep 5 |
|  | fi |
|  | done |
|  | if [ "$nk" = "99" ];then |
|  | exit 0 |
|  | else |
|  | echo "$(date +'%F %T') Domain is not reachable after 3 attempts." >>$runLog |
|  | fi |
|  |  |
|  | # 设置阿里云访问密钥和区域ID |
|  | export ALICLOUD_ACCESS_KEY_ID='key_id_xxx' |
|  | export ALICLOUD_ACCESS_KEY_SECRET='key_secret_xxx' |
|  | export ALICLOUD_REGION_ID='cn-shanghai' |
|  |  |
|  | # 查询绑定LB的EIP |
|  | EIP_INFO=$(aliyun vpc DescribeEipAddresses --RegionId $ALICLOUD_REGION_ID | jq --arg lb_id "$LOAD_BALANCER_ID" '.EipAddresses.EipAddress[] | select(.InstanceId == $lb_id and .InstanceType == "SlbInstance")') |
|  | # 提取EIP的ID和IP地址 |
|  | OLD_EIP_IP=$(echo $EIP_INFO | jq -r '.IpAddress') |
|  | OLD_EIP_ID=$(echo $EIP_INFO | jq -r '.AllocationId') |
|  | # 验证结果 |
|  | if [ -z "$OLD_EIP_IP" ]; then |
|  | echo "$(date +'%F %T') Failed to find OLD_EIP_IP" >>$runLog |
|  | exit 1 |
|  | fi |
|  | echo "$(date +'%F %T') Old EIP: $OLD_EIP_IP" >>$runLog |
|  |  |
|  |  |
|  | # 创建EIP 按量付费模式 峰值带宽100M |
|  | EIP_OUTPUT=$(aliyun vpc AllocateEipAddress --RegionId $ALICLOUD_REGION_ID --InternetChargeType PayByTraffic --Bandwidth 100 --Name $domain_sub) |
|  | # 获取EIP的ID和IP地址 |
|  | EIP_ID=$(echo $EIP_OUTPUT | jq -r '.AllocationId') |
|  | EIP_IP=$(echo $EIP_OUTPUT | jq -r '.EipAddress') |
|  | # 验证EIP创建 |
|  | if [ -z "$EIP_ID" ] || [ -z "$EIP_IP" ]; then |
|  | echo "$(date +'%F %T') Test Failed: Failed to create EIP." >>$runLog |
|  | echo "eip_create: $EIP_OUTPUT" >>$runLog |
|  | exit 1 |
|  | fi |
|  | echo "$(date +'%F %T') New EIP: $EIP_IP" >>$runLog |
|  |  |
|  |  |
|  | # 解绑现有的EIP |
|  | echo "$(date +'%F %T') Remove LB-EIP" >>$runLog >>$runLog |
|  | aliyun vpc UnassociateEipAddress --RegionId $ALICLOUD_REGION_ID --AllocationId $OLD_EIP_ID --InstanceId $LOAD_BALANCER_ID --InstanceType SlbInstance >>$runLog |
|  |  |
|  | sleep 2 |
|  | # 绑定EIP到负载均衡器 |
|  | ASSOCIATE_OUTPUT=$(aliyun vpc AssociateEipAddress --RegionId $ALICLOUD_REGION_ID --AllocationId $EIP_ID --InstanceId $LOAD_BALANCER_ID --InstanceType SlbInstance) |
|  | # 验证绑定 |
|  | if [ $? -ne 0 ]; then |
|  | echo "$(date +'%F %T') EIP add to LB Failed." >>$runLog |
|  | echo "eip_update: $ASSOCIATE_OUTPUT" >>$runLog |
|  | exit 1 |
|  | else |
|  | # echo "EIP add to LB successfully." |
|  | echo "$(date +'%F %T') eip_update: $ASSOCIATE_OUTPUT" >>$runLog |
|  | fi |
|  |  |
|  | sleep 2 |
|  | # 释放旧的EIP |
|  | echo "$(date +'%F %T') Release old EIP $OLD_EIP_ID" >>$runLog |
|  | aliyun vpc ReleaseEipAddress --AllocationId $OLD_EIP_ID  >>$runLog |
|  |  |
|  |  |
|  |  |
|  | # 获取域名RecordId |
|  | RECORD_ID=$(aliyun alidns DescribeDomainRecords --DomainName $domain_name | jq -r --arg rr "$domain_sub" '.DomainRecords.Record[] | select(.RR == $rr) | .RecordId') |
|  |  |
|  | # 更新域名的A记录 |
|  | echo "$(date +'%F %T') Update IP: $domain_sub.$domain_name  $EIP_IP" >>$runLog |
|  | aliyun alidns UpdateDomainRecord --RecordId $RECORD_ID --RR $domain_sub --Type A --Value $EIP_IP >>$runLog |
|  |  |
|  |  |
|  | #notie msg |
|  | #dingtalk |
|  | export ddtxt="notice from ip-update \n$domain_sub.$domain_name \n$EIP_IP" |
|  | export ddtoken="10b70b4fcb8a5ddad86b7a4396183639a6a99c2660xxxxxx" |
|  | curl -ks -m 5 http://files.elvin.vip/shell/ddmsg.url.txt.sh |bash |
|  | #lark |
|  | export txtmsg="notice from ip-update \n$domain_sub.$domain_name \n$EIP_IP" |
|  | export larktoken="f6bfc69d-2617-46d7-a42b-123xxxxxx" |
|  | curl -ks -m 5 http://files.elvin.vip/shell/lkmsg.txt.sh |bash |
|  |  |
|  | # 记录完成时间 |
|  | date +"%F %T" >$ckFile |
|  | exit 0 |


```

source: [https://gitee.com/alivv/elvin\-demo/blob/master/shell/aliyun.lb.eip.update.sh](https://github.com)


 本博客参考[Flowercloud 机场订阅加速](https://flowercloud6.com)。转载请注明出处！
