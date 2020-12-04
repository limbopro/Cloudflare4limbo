# Cloudflare4limbo
利用 Cloudflare 强大的API及时封禁异常IP访问，自动开启防御、清理页面CDN缓存，过期解禁IP。

## 效果如下

![Cloudflare][1]

[1]:https://raw.githubusercontent.com/limbopro/Cloudflare4limbo/main/photo_2020-01-04_16-31-16.jpg

## 原理

可参阅：[Cloudflare 的工作原理是什么？][3]

![Cloudflare 的工作原理][2]

[2]:https://support.cloudflare.com/hc/article_attachments/360029342112/What_is_Cloudflare_v7.png
[3]:https://support.cloudflare.com/hc/zh-cn/articles/205177068-Cloudflare-的工作原理是什么-

## 新建IP访问规则API

- 对应页面：登入 Cloudflare  - 选择你已添加的网站 - 防火墙（Firewalls）；
- 官方 API 参考：https://api.cloudflare.com/#user-level-firewall-access-rule-create-access-rule

```
curl -X POST "https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules" \
     -H "X-Auth-Email: user@example.com" \
     -H "X-Auth-Key: c2547eb745079dac9320b638f5e225cf483cc5cfdda41" \
     -H "Content-Type: application/json" \
     --data '{"mode":"challenge","configuration":{"target":"ip","value":"198.51.100.4"},"notes":"This rule is on because of an event that occured on date X"}'
```

- 运用示例

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
