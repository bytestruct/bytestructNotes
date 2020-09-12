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

在 `~/.ssh` 目录下新建一个 `config` 文件, 添加如下内容.

```
# gitee
Host gitee.com #别名, 随便定
HostName gitee.com # 网站的域名或 IP
PreferredAuthentications publickey # 配置登录时用什么权限认证--可设为publickey,password publickey,keyboard-interactive等
IdentityFile ~/.ssh/gitee_id_rsa #密钥文件的地址, 注意是私钥

# github
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/github_id_rsa
```

如果相同网站有多个用户, 会通过别名确定key.

## Bad owner or permissions on ~/.ssh/config

如果出现该问题, 可以使用以下方式解决:

```shell
chown $USER ~/.ssh/config
chmod 644 ~/.ssh/config
```

## 注意

 git 的配置分为三级别, System —> Global —> Local. System 即系统级别, Global 为配置的全局, Local 为仓库级别, 优先级是 Local > Global > System.
 
 既然已经配置多帐号了, 所以建议清除全局配置.
 
 ```
git config --global --unset user.name
git config --global --unset user.email
 ```
 
而是对仓库进行单独配置.

```
git config --local user.name "xxxxx"
git config --local user.email "xxxxx@xxxxx.xxxxx"
```

执行完毕后, 通过以下命令查看本仓库的所有配置信息.

```
git config --local --list
```

## 参考资料

[Git配置多个SSH-Key](https://gitee.com/help/articles/4229#article-header0)

[ssh returns “Bad owner or permissions on ~/.ssh/config”](https://serverfault.com/questions/253313/ssh-returns-bad-owner-or-permissions-on-ssh-config)

