>要学习zookeeper，不可避免的一项就是zookeeper源码的导入工作。本次使用的idea。

步骤：
安装java就省略啦~
# 手把手教你如何导入源码，zookeeper为例
#  软件
## 一，安装idea
官网[下载](https://www.jetbrains.com/idea/)，我下载的是2020.1的专业版本，嗯。至于如何那个，[参考](https://blog.csdn.net/moqianmoqian/article/details/112890989)这个第10条
![在这里插入图片描述](https://img-blog.csdnimg.cn/7b090a05a1e041fba87297c2a51b6718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 二，安装git
本次打算的是从远程拉取zookeeper源码，所以下载一下git（我用的Git-2.24.0-64-bit.exe）
如果你网速快，可以从官网下载，不然推荐这个：[链接](https://blog.csdn.net/moqianmoqian/article/details/119147632)
安装后，记得设置一下git地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/17637bd45cdb4835a0f8020f38309fa6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

## 三，安装maven
idea自带的maven感觉有点小慢，可以自己动手安装一个,博主[写的](https://www.cnblogs.com/arxive/p/8847030.html)很棒，我就不班门弄斧了，毕竟我也是照做的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/47b5e8eaa0a3463780e5e9d6117fefbe.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)


## 四，将zookeeper项目fork到自己的github
1 先将zookeeper的项目fork到自己的[github](https://github.com/apache/zookeeper)
![在这里插入图片描述](https://img-blog.csdnimg.cn/a02ce9ac450848208b909a21ca4b5daa.png)

2 然后将链接再添加到码云（当然，大佬网快当我没有说~）
（1）右上角，新建仓库
![在这里插入图片描述](https://img-blog.csdnimg.cn/eab82f252fae48d5a7ac7c79511d08da.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
（2）右上角  点击导入
![在这里插入图片描述](https://img-blog.csdnimg.cn/badf25fa994545229e480098e3396698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

（3）输入地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/3473e92397804ec09ae89980e9f2efcc.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

（4）导入完成后，可以点击克隆获取要拉取的地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/8fd9f37c5e41402886e6af42a5e5c218.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
（5）修改文件
有人会疑惑，那我从这里导入的，下次代码不就直接上传到这里了，我想上传到github怎么办，别急。
将目录.git，里面有一个config文件，我们只要将该文件的远成仓库地址更换成之间的gitHub项目地址即可：[参考链接](https://blog.csdn.net/calm_encode/article/details/105336486)
# 代码
## 五，导入
写一下你的url，或者你也可以登陆github去直接获取，如果登陆 github报gitHub Invalid authentication data.404 Not Found-Not Found，可以[参考](https://blog.csdn.net/moqianmoqian/article/details/112890989)这个
![在这里插入图片描述](https://img-blog.csdnimg.cn/ff35fa7c3043458986ac360254f9ecd0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
##  六，org.apache.zookeeper.data不存在
你以为这样就完了？不不不
![在这里插入图片描述](https://img-blog.csdnimg.cn/2bc1a684d01848a89631376ae6ffd14c.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
当你build后，首当其冲的问题就是org.apache.zookeeper.data不存在，或者maven plugins报错。
解决方法：如果是前者，compile一下
![在这里插入图片描述](https://img-blog.csdnimg.cn/cfe8a9886ef44f148e1979f3bc0f8d29.png)
后者的话，compile、install、clean、Reimport（左上角那个）轮番试用，总有一个适合你
![在这里插入图片描述](https://img-blog.csdnimg.cn/107cf8b9be8f4453bc010ff3b10b03e5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)


> 更多zookeeper可以看我的专栏，~~菜鸡~~ [大佬带你学zookeeper](https://blog.csdn.net/moqianmoqian/category_11015863.html)。
![在这里插入图片描述](https://img-blog.csdnimg.cn/18d087d16ff4467eba193a08d29eaf59.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
