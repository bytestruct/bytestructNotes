# Git配置多个SSH-Key

当有多个  git  账号时, 比如:

a. 一个 gitee, 用于公司内部的工作开发;
b. 一个 github, 用于自己进行一些开发活动;

## 解决方法

生成一个公司用的 SSH-Key

```shell
$ ssh-keygen -t rsa -C 'xxxxx@company.com' -f ~/.ssh/gitee_id_rsa
```

生成一个 github 用的 SSH-Key

```shell
$ ssh-keygen -t rsa -C 'xxxxx@qq.com' -f ~/.ssh/github_id_rsa
```

在 `~/.ssh` 目录下新建一个 `config` 文件, 添加如下内容(其中 Host 和 HostName 填写 git 服务器的域名, IdentityFile 指定私钥的路径)

```
# gitee
Host gitee.com
HostName gitee.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/gitee_id_rsa

# github
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/github_id_rsa
```

## Bad owner or permissions on ~/.ssh/config

如果出现该问题, 可以使用以下方式解决:

```shell
chown $USER ~/.ssh/config
chmod 644 ~/.ssh/config
```

## 参考资料

[Git配置多个SSH-Key](https://gitee.com/help/articles/4229#article-header0)

[ssh returns “Bad owner or permissions on ~/.ssh/config”](https://serverfault.com/questions/253313/ssh-returns-bad-owner-or-permissions-on-ssh-config)

