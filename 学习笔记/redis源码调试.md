## Redis源码学习-6.0.15

-- 请在linux系统下操作



### 1、获取源码



#### 1.1、源码地址

> https://github.com/redis/redis.git



#### 1.2、fork出一个仓库

fork 这个上面的源码仓库，并在6.0 这个分支上创建一个自己的学习分支

>?为什么需要fork和建立自己的分支呢？
>因为我们在源码学习过程中会debug源码，会写一些自己的理解、注释，甚至修改代码、提pr等等，因此fork一个自己的仓库也是很有必要的



#### 1.3、clone和添加upstream

clone fork后的仓库到本地，给该仓库添加upstream(源apache项目的地址)，这会方便我们后续更新最新代码和提交pr



#### 1.4、打开项目

在linux系统下，使用clion打开项目，执行makefile，如下图所示：

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830152141.png)

>?为什么使用clion而不是vscode打开项目?
>
>因为俺是java程序员

执行完将会生成很多.o、cc等文件，以及我们熟悉的redis-server、redis-cli



#### 1.5、设置Custom Build Targets

打开设置，找到Custom Build Targets，如图：

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830152455.png)

点击上图右边圈住的3个点，添加External Tool，需要添加两个，一个是build一个是clean

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830152937.png)

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830153019.png)

Note: 上面两张图是添加两个External Tool

添加完就这样：

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830153158.png)

最后选择上这两个exteral tool

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830153322.png)



#### 1.6、创建Custom Build Application

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830153658.png)

创建完就可以debug运行了(我把上面的Custom Build Application起名为了redis-server)

![](https://sign-pic-1.oss-cn-shenzhen.aliyuncs.com/img/20210830153801.png)

也可以在server.c main方法打上断点debug试试