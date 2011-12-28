---
layout: post
title: "推荐一些 Ruby on Rails 学习资料"
date: 2011-01-12 02:04
comments: true
categories: [Ruby, Rails]
---

## Ruby ##

开始之前应该看看 [Ruby 官方网站](http://www.ruby-lang.org/) 上的 [About Ruby](http://www.ruby-lang.org/en/about/)、[Ruby in Twenty Minutes](http://www.ruby-lang.org/en/documentation/quickstart/) 和 [Ruby From Other Languages](http://www.ruby-lang.org/en/documentation/ruby-from-other-languages/) 得到初步的印象和感性认识。在页面底部可以选择语言查看中文版。

经验比较丰富的开发者可以通过 [Ruby User's Guide](http://www.rubyist.net/~slagell/ruby/) [注1] 快速入门 Ruby，之后应该准备一本 [The Ruby Programming Language](http://books.google.com/books?id=jcUbTcr5XWwC) 作为日常参考。因为作为 Ruby 语言创始人松本行弘参与编写的书籍，它对 Ruby 语言的介绍最完整。而世界上第一本介绍 Ruby 语言的英文书籍 [Programming Ruby](http://ruby-doc.org/docs/ProgrammingRuby/) 大概是最多人用于入门 Ruby 的书籍，虽然对于有经验的开发者来说它稍显啰嗦。Programming Ruby 第一版有提供免费的在线版本。如果你还没有任何程序设计经验，[Ruby Programming: 向Ruby之父学程序设计](http://www.china-pub.com/195252) 应该是不错的选择，作者高桥征义是日本 Ruby 协会会长。

## Rails ##
同样，有经验的开发者可以直接通过 [Ruby on Rails Guides](http://guides.rubyonrails.org/) 入门 Rails。而 [Agile Web Development with Rails](http://pragprog.com/titles/rails4/agile-web-development-with-rails) 则大概是最多人用于入门 Rails 的书籍，它的第四版已经使用目前最新的 Rails 3。

## 中文资料 ##

[@ihower](http://twitter.com/ihower) 组织的 [Ruby Taiwan](http://ruby.tw) 社区有提供 [Ruby User's Guide](http://guides.ruby.tw/ruby/) 的繁体中文翻译以及 [Ruby on Rails Guides](http://guides.ruby.tw/rails3/) 前两章的繁体中文翻译。[@ihower](http://twitter.com/ihower) 自己编写的 [Ruby on Rails 實戰手冊](http://ihower.tw/rails3/) 也是一部很不错的面向有一定经验开发者的在线书籍。

## 其它 ##

Ruby on Rails 社区非常注重代码的美观及可读性。使用相同的 coding style 是保证代码美观可读的有效措施之一，所以在自己尝试写代码时应该看看 [Ruby Style Guide](https://github.com/bbatsov/ruby-style-guide)。

真正开始使用 Ruby on Rails 之后，[Rails Searchable API Doc](http://www.railsapi.com/) 和 [RubyDoc.info](http://rdoc.info/) 一定会是最常用的两个在线文档服务。

注1: Ruby User's Guide 写于 Ruby 1.8.3 时代，现在建议使用的 Ruby 版本是 1.9.3，[1.8 系列已经进入其生命的最后阶段](http://www.ruby-lang.org/en/news/2011/10/06/plans-for-1-8-7/)。文中提到的 eval.rb 应该使用 irb 取代，另外可以使用 [irbtools](https://github.com/janlelis/irbtools) 大幅度增强 irb。Ruby Taiwan 的繁体中文翻译版本对类似问题有提供译注，建议参考。
