---
title: "一例Python发起HTTPS请求出现SSL报错"
date: 2025-08-13T23:00:25+08:00
draft: false
description: ""
tags: ["windows", "ops"]
---
# 起因
客户在winserver2019机器上执行```http.client.HTTPSConnection("xingchen-api.xf-yun.com", timeout=120) ```命令时报错  
错误为  
```
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1000) 
```
该报错表示SSL上下文无法验证服务器的SSL证书颁发机构,根本原因是找不到本地证书  

# 一线解决方案
一线执行```pip install --upgrade certifi```试图修复错误，失败  
禁用证书验证测试连通性,正常  
```python
import ssl
http.client.HTTPSConnection(
    "xingchen-api.xf-yun.com",
    timeout=120,
    context=ssl._create_unverified_context())
```

# 最终解决方案
通过```pip install pip_system_certs```安装解决
