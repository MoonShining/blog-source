---
title: 使用mina自动化部署Rails应用
date: 2018-02-05 09:41:57
tags: 自动化部署
---
![](http://upload-images.jianshu.io/upload_images/4073552-1881d52edbf495e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为什么需要自动化部署？

#### 因为我懒啊！！！

#### 简介

mina是rails社区中流行的部署方案，允许你用Ruby DSL描述部署过程，mina生成一份脚本在目标服务器上执行。除了Ruby以外，也支持shell脚本，所以可定制性非常高。相比于老牌的Capstrino，mina一次性上传所有脚本到目标服务器执行，效率上会高一些。

#### 核心步骤与原理

mina setup生成目标服务器上的文件夹结构

mina deploy进行部署

mina会自动从设置好的git仓库拉代码，运行bundle／migration等一系列流程，然后重启服务器

#### 简单对比

手动部署

1. git pull

2. bundle  install

3. rake db：migrate

4. rake assets：precompile

5. restart web server

自动部署

1. mina deploy

一条命令就可以跑完整个发布流程。当然前提是你的deploy task写的天衣无缝，这需要对rails与linux有一定了解才可以做到。

#### 举个栗子

```sh

require 'mina/bundler'

require 'mina/rails'

require 'mina/git'

require 'mina/rvm'

set :domain, '121.42.12.xx' ＃你的服务器地址或域名

set :deploy_to, '/var/www/api' ＃你打算把项目部署在服务器的哪个文件夹

set :repository, 'git@github.com:xxx.git' ＃git仓库

set :branch, 'master' ＃git分支

set :term_mode, nil  ＃mina的小bug，设为nil可以解决

set :shared_paths, ['config/sidekiq.yml', 'config/database.yml', 'config/secrets.yml', 'log', 'shared'] ＃很关键，这几个文件夹会在多次部署间，通过符号链接的形式共享

set :user, 'moon' ＃ssh 用户名

task :environment do

invoke :'rvm:use[ruby-2.1.0-p0@default]' ＃ruby 版本

end

task :setup => :environment do  ＃初始化task，创建文件夹结构

queue! %[mkdir -p "#{deploy_to}/#{shared_path}/log"]

queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/log"]

queue! %[mkdir -p "#{deploy_to}/#{shared_path}/config"]

queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/config"]

queue! %[mkdir -p "#{deploy_to}/#{shared_path}/shared"]

queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/shared"]

queue! %[touch "#{deploy_to}/#{shared_path}/config/database.yml"]

queue! %[touch "#{deploy_to}/#{shared_path}/config/secrets.yml"]

queue  %[echo "-----> Be sure to edit '#{deploy_to}/#{shared_path}/config/database.yml' and 'secrets.yml'."]

if repository

repo_host = repository.split(%r{@|://}).last.split(%r{:|\/}).first

repo_port = /:([0-9]+)/.match(repository) && /:([0-9]+)/.match(repository)[1] || '22'

queue %[

if ! ssh-keygen -H  -F #{repo_host} &>/dev/null; then

ssh-keyscan -t rsa -p #{repo_port} -H #{repo_host} >> ~/.ssh/known_hosts

fi

]

end

end

desc "Deploys the current version to the server."

task :deploy => :environment do

to :before_hook do

end

deploy do ＃部署流程

invoke :'git:clone'

invoke :'deploy:link_shared_paths'

invoke :'bundle:install'

invoke :'rails:db_migrate'

#invoke :'rails:assets_precompile'

invoke :'deploy:cleanup'

invoke :start

to :launch do

queue "mkdir -p #{deploy_to}/#{current_path}/tmp/"

queue "touch #{deploy_to}/#{current_path}/tmp/restart.txt"

end

end

end

desc "start puma & sidekiq"

task :start => :environment do

queue "cd #{deploy_to+"/current"}"

queue "sidekiq -e production -d"

queue "puma -e production"

end

```
