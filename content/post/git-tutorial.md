---
title: "Git Tutorial"
date: 2022-08-06T16:20:33+08:00
categories: ["版本控制"]
tags: ["Git", "指令", "教學", "設定"]
draft: true
---
### Git設定
* 查看設定
> git config --list

* 設定帳號
> git config --global user.name "hezhengmin"

* 設定信箱
> git config --global user.email "zhengmin099@gmail.com"

### Git基本用法
* 建立Repository
> git init

* 複製他人專案
> git clone https://github.com/hezhengmin/Minesweeper.git

* 檢查狀態
> git status

* 查詢記錄
> git log

* 新增單一檔案
> git add <檔案名稱>

* 新增全部的檔案
> git add .  
> git add --all

* 全部修改提交
> git commit -am "modified"

* 提交版本
> git commit -m "填寫版本資訊"

* 推送到你的遠端
> git push origin master

* 遠端主機分支master更新到本地主機 
> git pull origin master

* 刪除檔案
> git rm "1. Two Sum.cpp"

### Git分支用法
* 建立新branch並切換過去 
> git checkout -b <branch名稱>

* 查看電腦上的branch(按q可以離開)
> git branch

* 查看所有的branch(包含remote)
> git branch -a 

### 其他  
* 建立資料夾
> mkdir Temp

* 列出所有檔名
> ls -al

* 不儲存直接離開
> :q

* 強制離開不存檔
> :q!

* 儲存並離開
> :wq