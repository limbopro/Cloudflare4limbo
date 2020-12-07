# Cloudflare4limbo
利用 Cloudflare 强大的API及时封禁异常IP访问，自动开启防御、清理页面CDN缓存，过期解禁IP。进入 Cloudflare 控制面板后请自行更改界面语言为中文，本文将基于此条件下写成；

## 效果如下

![Cloudflare-dash][1]
![Cloudflare-dash][4]

[1]:https://raw.githubusercontent.com/limbopro/Cloudflare4limbo/main/photo_2020-03-06_16-28-57.jpg
[4]:https://raw.githubusercontent.com/limbopro/Cloudflare4limbo/main/photo_2020-01-04_16-31-16.jpg

## 原理

可参阅：[Cloudflare 的工作原理是什么？][3]

![Cloudflare 的工作原理][2]

[2]:https://support.cloudflare.com/hc/article_attachments/360029342112/What_is_Cloudflare_v7.png
[3]:https://support.cloudflare.com/hc/zh-cn/articles/205177068-Cloudflare-的工作原理是什么-

## 新建IP访问规则-API

- 对应手动操作页面：登入 Cloudflare  - 选择你已添加的网站 - 防火墙（Firewalls）；
- Create Access Rule API 参考：https://api.cloudflare.com/#user-level-firewall-access-rule-create-access-rule

```
curl -X POST "https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules" \
     -H "X-Auth-Email: user@example.com" \
     -H "X-Auth-Key: c2547eb745079dac9320b638f5e225cf483cc5cfdda41" \
     -H "Content-Type: application/json" \
     --data '{"mode":"challenge","configuration":{"target":"ip","value":"198.51.100.4"},"notes":"This rule is on because of an event that occured on date X"}'
```

- 运用示例 submit.blackip.sh

```
#!/bin/bash 

CFEMAIL="你的邮箱" # 你的 Cloudflare 注册邮箱
CFAPIKEY="你的APIKEY" # 用户中心右上角 - 我的个人资料 - API 令牌 - API 密钥 (用于访问 Cloudflare API 的密钥。) - Global API Key 点击查看；
ZONESID="你的ZONESID" # 在 Cloudflare 新增域名 - 进入你的域名 - 概述 - 页面右下角或页面底部 - 区域 ID（ZonesID） ；
IPADDR=$(</home/blackip.list) # /home/blackip.list 为存放异常IP的文本，一个IP占一行；

for IPADDR in ${IPADDR[@]}; do
echo $IPADDR

curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONESID/firewall/access_rules/rules" \
  -H "X-Auth-Email: $CFEMAIL" \
  -H "X-Auth-Key: $CFAPIKEY" \
  -H "Content-Type: application/json" \
  --data '{"mode":"block","configuration":{"target":"ip","value":"'$IPADDR'"},"notes":"limbo-auto-challenge ${1000ip}"}'
done

```

至此，大家可以新建一个文本，然后去运行一下 脚本，记得 ` chmod +x submit.blackip.sh`；

## 捕获异常IP并存入文本

```
#!/bin/bash 
maxrequest=1
> /home/challenge.log; #清除拉取的日志
maxtimes=1 #拉取最近N分钟的请求至临时日志

function define()
{
    #引入参数环节
    ori_log_path="/home/limbopro.xyz/access.log" #原始日志的存放位置
    tmp_log_path="/home/blackip.log" #生成的临时日志的存放位置
    date_stamp=`date -d "-"$maxtimes"min" +%Y:%H:%M:%S` #引入时间范围参数
    day_stamp=`date +%d` #日期
}

function gather()
{
    awk -F '[/ "[]' -vnstamp="$date_stamp" -vdstamp="$day_stamp" '$7>=nstamp && $5==dstamp' ${ori_log_path} > ${tmp_log_path}; #根据时间范围从原始日志处读取并写入临时日志
    log_num=`cat ${tmp_log_path} | wc -l`; #计算时间范围内的网络请求次数
    ipcounts=$(awk '{print $1}' $tmp_log_path | sort -n | uniq | wc -l); #计算IP数量
}

function output()
{
date=$(env LANG=en_US.UTF-8 date "+%e/%b/%Y/%R") #无所事事
}

function main()
{
    define
    gather
    output
}
## 拉取日志结束
main

## 生成IP黑名单
echocf=/home/blackip.list #Cloudflare 黑名单收集 会销毁

##记录每次操作

for ip in $(awk '{cnt[$1]++;}END{for(i in cnt){printf("%s\t%s\n", cnt[i], i);}}' ${tmp_log_path} | awk '{if($1>'$maxrequest') print $2}') 
do 
echo "${ip}" >> $echocf
done
```

此处生成的黑名单，将在 新建IP访问规则 时调用；

一些参考资料：[Cloudflare 下 Nginx 获取用户真实IP 地址](https://limbopro.xyz/archives/1481.html);

建站指南：[建站指南与优化/DDoS防御](https://limbopro.xyz/category/LNMP/)


