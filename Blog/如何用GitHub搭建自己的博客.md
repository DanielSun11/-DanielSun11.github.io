# 如何用github搭建自己的博客
## 创建仓库
创建名为**用户名.github.io**的仓库，此处强制要求用户名为github用户名一致。
## 仓库中添加名为_config.yml的文件
**_config.yml**文件内容如何编写可参考[Adding a theme to your GitHub Pages site using Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/adding-a-theme-to-your-github-pages-site-using-jekyll)  

比如设置为：  
```  
title: DanielSun11  
description: Welcome to DanielSun's Blog.  
remote_theme: pages-themes/architect@v0.2.0 #主题名称,有多个主题可选择 
plugins:  
- jekyll-remote-theme # add this line to the plugins list if you already have one  
gem "github-pages", group: :jekyll_plugins   
```
## 仓库中添加 index.md文件
index.md 中的内容将会展示到你的博客主页中。
## 访问博客主页
例如 <https://danielsun11.github.io/> 将danielsun11换为你的用户名即可
