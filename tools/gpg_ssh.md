# 写此的目的
## mac下我在配置GPG,SSH时遇到的一些坑在此备注，旨在自己或他人遇到相同问题时更容易解决
* create GPG key 依照 https://help.github.com/en/articles/generating-a-new-gpg-key 
在github commit/push 时总是需要输入密码才行。
原因是在创建GPG key时输出了密码，commit或者push时需要使用GPG private key,但是private key被密码保护，所以需要解锁才能访问private key。
这种情况，通过git config --global credential.helper store 是没有用的，因为流程还没走到这一步

* 同样的SSH也有一样的问题，解决方法也是类似，在提示输入密码保护private key时，直接点‘enter’
* 如果你安装的是最新的GPG，安装完成后，执行 gpg --version 报这个错 -bash: gpg: command not found的话， 打此命令gpg2 --version，如果显示类似以下这样
  > gpg (GnuPG) 2.2.15
libgcrypt 1.8.4
Copyright (C) 2019 Free Software Foundation, Inc.

* 那就说明没有问题，接下来所有的命令 **gpg** 都需要替换成 **gpg2**
顺利生成了key，会被存储在一个 .rev文件中，**不要**直接打开这个文件copy内容。
假设你的sec为 '3AA5C34371567BD2'，使用以下命令导出，copy到GitHub add gpg key才能使用
  > gpg2 --armor --export 3AA5C34371567BD2

* 接下来需要执行以下命令,如果想要适用于任何地方--global不可省略，只在本工程中使用省略之
  >git config (--global) user.signingkey 3AA5C34371567BD2
git config (--global) commit.gpgsign true
* 这样还没有完全解决，因为在提交时git会自动调用**gpg**，所以需要执行之下的代码,，更改GitHub默认的gpg执行程序
  > git config --global gpg.program gpg2

###### 如果不确认之前生成了gpg private key 可使用 gpg2 --list-secret-keys --keyid-format LONG 查看
###### 如果重新生成了gpg key，配置了之前key的工程中需要执行
> git config (--global) user.signingkey newkey