---
title: "Postman-儲存JwtToken為全域變數"
date: 2022-08-06T14:09:08+08:00
categories: ["軟體工具"]
tags: ["Postman", "JWT", "API", "測試"]
draft: true
---

## 1. 新增script
```
const resp = pm.response.json();
pm.environment.set("jwtToken", resp.jwtToken)
```
![postman_test](/images/2/postman_test.jpg)

```
程式回傳jwt token
```
![postman_jwt](/images/2/postman_jwt.jpg)

## 2. 新增JWT變數
![postman_add_env](/images/2/postman_add_env.jpg)

![postman_save_env](/images/2/postman_save_env.jpg)

## 3. Request API上加入Token (Authorization)
![postman_setting_env](/images/2/postman_setting_env.jpg)

## 4. Request API上加入Token (Header)
![postman_setting_header](/images/2/postman_setting_header.jpg)

## 參考
[Postman 儲存 API 變數 JWT Token 驗證資訊](https://jscinin.medium.com/postman-jwt-%E5%88%A9%E7%94%A8%E8%AE%8A%E6%95%B8%E5%84%B2%E5%AD%98%E9%A9%97%E8%AD%89%E8%B3%87%E8%A8%8A-d5c988cf028a)


