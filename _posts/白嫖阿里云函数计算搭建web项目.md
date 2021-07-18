# 白嫖阿里云函数计算搭建web项目

有时间写完一个玩具，想部署一下让大家也试试看一看。通常就需要一个服务器来跑，这成本太高了。

最近看到阿里云函数计算每个月都有40GB/S的免费额度，支持HTTP触发就能nice，可以白嫖部署玩具了。

现在要讲的就是，一个简单的go web项目使用 github + ci ，自动部署项目 到阿里云函数计算。  

## 

这里准备了一个很简单的项目 [isworkday](https://github.com/tangtj/isworkday)，今天是工作日吗？从政府网站获取的节假日数据写到json文件中，结合是否为周一到周五做的判断。



在阿里云函数计算我们新建一个函数，选择HTTP函数->选择运行环境目前选项中可选的有 NodeJs，python,java，php，.Net Core 好像还不能直接支持到go，但是可以选 Custom container ，看来我们需要把go项目使用docker打包成容器，选了之后在镜像容器中只能选择阿里云镜像容器。

就又用到了阿里云的另一个服务“阿里云镜像容器”，新建一个镜像仓库，绑定我们的github，选择我们需要的仓库。勾上代码变更自动构建镜像。添加构建规则，类型 branch Branch/Tag 选master，填写dockerfile的文件和目录， 那么每次master 有变更都会使用指定的dockerfile文件进行自动构建。
