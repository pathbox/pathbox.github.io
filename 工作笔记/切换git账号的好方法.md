
一、场景
有个典型的场景如下：

我用公司的电脑，既要提交公司的 gitlab 仓库，也要维护一些开源的 github 仓库，这时候频繁交替交替两边仓库的代码简直就是一场灾难，为此我在网上找到一个自认为最便捷的方案。

二、生成不同的 ssh key

cd ~/.ssh
# 取名叫 id_rsa_personal
ssh-keygen -t rsa -C "chenneal@126.com"
# 取名叫 id_rsa_company
ssh-keygen -t rsa -C "staff@tencent.com"
这时候我们在 ~/.ssh 目录下面得到四个文件，分别对应公司和个人的公钥与私钥

1
id_rsa_personal  id_rsa_personal.pub  id_rsa_work  id_rsa_work.pub
接下来你就可以去公司的 gitlab 或者 gitHub 的设置页面里添加公钥了，比较简单，不再赘述。

三、本地配置私钥
我们在 ~/.ssh 目录下新建一个 config 文件，并输入以下内容：

# chenneal
Host personal
    HostName github.com
    User chenneal
    IdentityFile ~/.ssh/id_rsa_personal

# staff
Host company
    HostName gitlab.tencent.com
    User staff
    IdentityFile ~/.ssh/id_rsa_work
每个字段的含义如下：

Host : 配置别名
HostName : git服务器地址
User : 账户id
IdentityFile : 私钥文件位置
四、验证是否配置正确
可以用 ssh -T git@别名 验证是否成功
# 会输出 Hi chenneal! You've successfully authenticated, but GitHub does not provide shell access.
ssh -T git@personal
# 会输出 Welcome to GIT, staff! 这个因公司的 gitlab 实现, 不一定是这个内容.
ssh -T git@company
五、快速切换账户命令
以上都是阐述如何管理不同的 git 账户公私钥，还没有说怎么快速切换 git 账户，这里有个好的方案就是设置命令的 alias ，我们编辑 ~/.zshrc 文件，并设置 alias :
 
alias cgc='git config --global user.name staff && git config --global user.email staff@tencent.com && git config user.name && git config user.email'
alias cgg='git config --global user.name chenneal && git config --global user.email chenneal@126.com && git config user.name && git config user.email'
执行 source ~/.zshrc后改配置立即生效。

后续，我们只要输入cgc就快速切换到公司模式，输入cgg切换到个人模式，而且不需要输入密码，是不是感觉非常方便呢。
