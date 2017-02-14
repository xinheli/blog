---
layout: post
title: "Pygments unknown language"
date: 2014-09-21 15:39:14 +0800
comments: true
categories: octopress 
---
##背景
最近一直用octopress写blog，原生的语法高亮插件`Pygments`很强大，支持语言也很多。不过最近碰到一个Bug，每次`rake generate` 或者`rake preview`的时候，总是碰到如下的错误`Pygments unknown language </p>`
##分析
表面上看是`Pygments`不能识别该语言，很奇怪语言的名字竟然是`</p>`，可是都是用默认语言，都没有指定。

网上查找了，也没有找到解决办法，自己对ruby也不很熟悉。报错信息也太少，只能用二分法，逐步删除代码，来测试具体是那一块有问题，但是效率太低。而且即使解决了，也没有找到根本原因，以后写blog，总不能每次都二分去找。

想彻底找到原因。首先`grep "unknow language"`，存在`plugins/pygments_code.rb`文件中。报错代码如下:
<!--more-->
``` ruby
def self.pygments(code, lang)
    if defined?(PYGMENTS_CACHE_DIR)
      path = File.join(PYGMENTS_CACHE_DIR, "#{lang}-#{Digest::MD5.hexdigest(code)}.html")
      if File.exist?(path)
        highlighted_code = File.read(path)
      else
        begin
          highlighted_code = Pygments.highlight(code, :lexer => lang, :formatter => 'html', :options => {:encoding => 'utf-8', :startinline => true})
        rescue MentosError
          raise "Pygments can't parse unknown language: #{lang}."
        end
        File.open(path, 'w') {|f| f.print(highlighted_code) }
      end
```
可以看到异常处理代码，添加对code的输出，`raise "Pygments can't parse unknown language: #{lang}#{code}."`就很容易定位到具体哪行代码出了问题。
##解决
再次执行`rake generate`，很容易找到出错位置和原因了。原来反引号代码标示后面不能有空行!坑！
