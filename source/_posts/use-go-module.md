---
title: Go Module 迁移
date: 2020-07-14 12:20:55
category: golang
description: 项目仓库使用 go module 记录
tag:
    - golang
    - https
toc: true

---
### 迁移步骤
1. 升级 golang 到 1.13

2. 设置 go module 代理
    ```bash
    go env -w GOPROXY=https://goproxy.io,direct
    ```
     direct 为特殊指示符，用于指示 Go 回源到模块版本的源地址去抓取(比如 GitHub 等)，当值列表中上一个 Go module proxy 返回 404 或 410 错误时，Go 自动尝试列表中的下一个，遇见 “direct” 时回源
     
3. 设置私有仓库不走代理
    ```bash
    go env -w GOPRIVATE="*.100tal.com"
    ```
    
4. gitalb 添加机器密钥

5. 修改git配置，使支持 go get 拉取公司私有仓库
    打开 ~/.gitconfig 文件
    添加 
    ```text
    [url "ssh://git@git.100tal.com/"]
        insteadOf = https://git.100tal.com/
    ```
6. 初始化项目模块,生成 go.mod 文件
    ```bash
    go mod init git.100tal.com/suzhiguojijiaoyu_moli_wisroom-backend/xxxxxx
    ``` 
    
    go.mod 提供了module, require、replace和exclude 四个命令
    - module：用于定义当前项目的模块路径。
    - go：用于设置预期的 Go 版本。
    - require：用于设置一个特定的模块版本。
    - exclude：用于从使用中排除一个特定的模块版本。
    - replace：用于将一个模块版本替换为另外一个模块版本。   
7. 全局替换包名
    wisroom/ => git.100tal.com/suzhiguojijiaoyu_moli_wisroom-backend/
9. go.mod 添加 common 依赖
    ```bash
    go mod edit -require=git.100tal.com/suzhiguojijiaoyu_moli_wisroom-backend/common@v1.0.6
    ```
10. 下载依赖并整理
    ```bash
    go mod tidy
    ```

8. 构建

### 参考文章

1. [干货满满的 Go Modules 和 goproxy.cn](https://juejin.im/post/5d8ee2db6fb9a04e0b0d9c8b)
2. [Go Modules 不完全教程](https://mp.weixin.qq.com/s/v-NdYEJBgKbiKsdoQaRsQg)