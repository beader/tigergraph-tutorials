# 背景介绍

这个案例来自于一个2017年的一个数据挖掘比赛中的一个子赛题，利用 DNS 解析日志以及 WHOIS 信息，针对给定的一个域名，进行溯源分析。

## 题面辅助

某公司发现内⽹一台重要服务器频繁主动连接可疑域名 `updgoog1e.com`，并发送⼤量加密数据。现在要求你根据提供的 WHOIS 历史数据、DNS 解析日志，挖掘出和可疑域名背后组织高度相关的信息，包括以下内容: 邮箱、域名、IP、管理员姓名。

其中 DNS 解析日志包含 70 余万条记录，WHOIS 记录 10 余万条，最终需要选手提交不超过 20 条有价值的线索。

## 数据预览

原始数据包含 2 个文件:

{% code title="domainIpHistory.txt" %}
```text
ouo.press    20160402    ["104.20.3.244","104.20.2.244"]
ouo.press    20160610    ["104.20.3.244","104.20.89.25","104.20.90.25","104.20.2.244"]
ouo.press    20160611    ["104.20.89.25","104.20.90.25"]
rightmove.co.uk    20100624    ["80.64.55.62"]
rightmove.co.uk    20111202    ["146.101.201.252","213.161.94.26","80.64.55.7"]
rightmove.co.uk    20120323    ["80.64.55.62"]
...
```
{% endcode %}

{% code title="domainWhoisHistory.txt" %}
```text
ouo.press    20170124    {"admin_city":"Panama","admin_company":"WhoisGuard, Inc.","admin_country":"PA","admin_email":"700cd184942c4521bcf533def60784ed.protect@whoisguard.com","admin_fax":"+51.17057182","admin_id":"ozebibbf4wp4gxcc","admin_name":"WhoisGuard Protected","admin_phone":"+507.8365503","admin_state":"Panama","admin_street":"P.O. Box 0823-03411","admin_zip":"00000","cdate":"20160331000000","create_date":"2016-03-31","dnssec":"unsigned","domain_id":"D19293883-CNIC","domain_name":"ouo.press","edate":"20170331000000","expiry_date":"2017-03-31","name_server":"coby.ns.cloudflare.com|lola.ns.cloudflare.com","registrant_city":"Panama","registrant_company":"WhoisGuard, Inc.","registrant_country":"PA","registrant_email":"700cd184942c4521bcf533def60784ed.protect@whoisguard.com","registrant_fax":"+51.17057182","registrant_id":"5dim4e42uxw2eb4m","registrant_name":"WhoisGuard Protected","registrant_phone":"+507.8365503","registrant_state":"Panama","registrant_street":"P.O. Box 0823-03411","registrant_zip":"00000","registrar_abuse_email":"abuse@namecheap.com","registrar_abuse_phone":"+1.6613102107","registrar_iana_id":"1068","registrar_name":"NAMECHEAP INC","registrar_url":"http://www.namecheap.com","registrar_whois":"whois.namecheap.com","reseller":"NAMECHEAP INC","status":"addperiod|clienttransferprohibited|servertransferprohibited","tech_city":"Panama","tech_company":"WhoisGuard, Inc.","tech_country":"PA","tech_email":"700cd184942c4521bcf533def60784ed.protect@whoisguard.com","tech_fax":"+51.17057182","tech_id":"v33fmfa2krknsfdt","tech_name":"WhoisGuard Protected","tech_phone":"+507.8365503","tech_state":"Panama","tech_street":"P.O. Box 0823-03411","tech_zip":"00000","udate":"20160331000000","update_date":"2016-03-31"}
rightmove.co.uk    20151203    {"alexa":"726","edate":"20170907000000","expiry_date":"2017-09-07"}
rightmove.co.uk    20151001    {"audit_update_date":"2015-07-18 00:00:00 UTC","cdate":"19990907000000","create_date":"1999-09-07","domain_name":"rightmove.co.uk","edate":"20150907000000","expiry_date":"2015-09-07","name_server":"213.161.77.66|80.64.55.4|ns1.rightmove.co.uk|ns3.rightmove.co.uk|ns2
...
```
{% endcode %}

`domainIpHistory.txt` 为 DNS 解析日志，有 3 个字段，分别为 `域名`、`日期`、`IP`，其中 `IP` 字段为一个列表，表示当前日期该域名解析到多个 `IP`。

`domainWhoisHistory.txt` 为 WHOIS 记录，有 3 个字段，分别为 `域名`、`日期`、`whois`，其中 `whois` 字段为一个字典，里面包含了当前日期该域名的 whois 登记信息。

## 大致思路

需要提交的线索包括 邮箱、域名、IP、姓名 等，原始数据中每一行包含了线索之间等关联。我们需要从起始线索 `updgoog1e.com` 开始溯源，找到与其相关的其他线索，然后从这些线索出发继续做更深层次的关联。因此可以把溯源问题当作一个图遍历问题来看。

## 数据前处理

本案例为了便于新手理解，需要先将原始数据预先处理成规范的表格型 \(Long Format 长表\) 数据。后续随着技能的熟练，将会学到更多更复杂的数据导入方法。

每份文件当中，存在着一些冗余字段，我们只保留与线索相关的字段。其中管理员姓名来自于 whois 信息中的 `admin_name` ，管理员邮箱来自于 whois 信息中的 `admin_email` 与 `registrant_email`。在该项目中，我们忽略日期字段。

处理后的数据如下:

{% code title="domain\_ips.txt" %}
```text
domain,ip
ouo.press,104.20.3.244
ouo.press,104.20.2.244
ouo.press,104.20.3.244
ouo.press,104.20.89.25
ouo.press,104.20.90.25
ouo.press,104.20.2.244
ouo.press,104.20.89.25
ouo.press,104.20.90.25
rightmove.co.uk,80.64.55.62
rightmove.co.uk,146.101.201.252
rightmove.co.uk,213.161.94.26
rightmove.co.uk,80.64.55.7
rightmove.co.uk,80.64.55.62
rightmove.co.uk,146.101.201.252
...
```
{% endcode %}

{% code title="domain\_whois.txt" %}
```text
domain,admin_name,admin_email,registrant_email
ouo.press,WhoisGuard Protected,700cd184942c4521bcf533def60784ed.protect@whoisguard.com,700cd184942c4521bcf533def60784ed.protect@whoisguard.com
rightmove.co.uk,,,
rightmove.co.uk,,,
...
```
{% endcode %}

其中 `domain_ips.txt` 中每一行包含了一个域名与IP的关系，`domain_whois.txt` 则包含多个关系。

