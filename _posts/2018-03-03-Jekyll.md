---
layout:     post
title:      Windows 使用 Jekyll 创建 GitHub Pages 个人博客
subtitle:   函数式编程框架 ReactiveCocoa 进阶
date:       2018-03-03
author:     Allen Vork
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Jekyll
    
---

# Windows 使用 Jekyll 创建 GitHub Pages 个人博客

## what is Jekyll, exactly?
Jekyll是一个简单的静态网站生成器，它包含一个拥有各种格式的源文件的模板目录，并通过一个转换器（如 MarkDown) 和 Liquid 渲染器生成一个完整的可以发布的静态网站。由于 Jekyll 刚好是 GitHub Pages 的引擎，我们就可以用 Jekyll 来免费为我们的博客或网站等提供服务了。

## Installation on Windows x64
下面来具体讲解下如何在 Windows x64 上安装 Jekyll。
> + 安装 ruby    
下载 [ruby](https://rubyinstaller.org/downloads/) 安装 Ruby 2.3.3 版本（注意不能安装高版本，因为 DevKit 要求 Ruby 版本在 2.0-2.3)。 安装完成后，添加环境变量（如 D:\software\ruby\Ruby23-x64\install\bin）到系统中去。    
打开命令行工具输入`ruby -v`来验证是否正常安装了。     
> + 安装 DevKit     
在上面的 ruby 网站的最下面找到 `DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe ` （注意我这是 64 位版本的，32 位的注意下另一个），然后解压出来。打开命令行工具进入到解压出来的 DevKit 目录中，然后执行 `ruby dk.rb init` 来初始化 devkit 并将其绑定到 Ruby 的安装目录下（我们可以打开 devkit 目录中的 config.yml 里面将 ruby 的 bin 目录写进去了）。完成后执行 `ruby dk.rb install`就 ok 了。    
执行 ` gem install json --platform=ruby` 来检测 Ruby 是否已成功安装。
> + 安装 Jekyll    
执行 `gem install jekyll bundler`
> + 安装 Python 和 Pip   
可以参考[python安装教程](http://blog.csdn.net/lengqi0101/article/details/61921399)

到此所有的安装步骤都完成了。

## 使用 Jekyll 创建网页
在本地执行`jekyll new myblog`生成

