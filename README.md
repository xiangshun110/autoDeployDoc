# 利用WEBHOOK自动部署（Linux） #

## 有什么用： ##
我们再本地写好代码后，要把文件部署到服务器，通常，我们有两个做法:<br/>

####1.最简单的，利用FTP工具,选择要上传的文件传到服务器。坏处就是如果只修改几个文件也都要上传所有的文件，除非你把修改过的文件都记下来，显然这样做不刷算。<br/>####

####2.在本地跟服务器端都用git管理，本地push后再去服务器pull。这样比用FTP要稍微高级一点，但是也麻烦。####

所以自动部署就来了，自动部署简单的说就是本地push后，会触发一个事件，然后去执行我们写好的一个脚本，完成自动pull的过程，也就是我们只管本地push了，**只要本地一push服务器也就自动完成代码的更新了**，是不是很方便。

自动部署的功能是基于WebHook,具体WebHook是个啥东西，可以Google一下，现在主流的代码托管平台都是支持WebHook的，比如gihub,coding,oschina,包括gitlab也是可以的。

看这个原理好像很简单，就是写一个脚本，放到服务器，然后在代码托管平台新建一个Webhook就可以了，但是其实并不是这样，我折腾了快两天才搞定，尝试了网上说的各种方法,可能是因为我对linux的熟悉。

网上搜自动部署也有很多文章，但是我基本每个都试，结果都没成功，因为写文章的人对Linux都比较熟悉，所以他们有很多细节就跳过了，我写这篇文章希望可以帮助到跟我一样linux基础很差的同学有帮助，本文以coding为例。

# 开始了: #
1. 首先你得有一个linux服务器，我的阿里云的,系统是centerOS.

2. 然后你得装Web服务器环境，我是用的阿里云的一键安装包，新手建议用这个，很方便，用这个后续基本不用再装别的，因为它都包含了。

3. 在服务器上创建SSH Key用于在服务器上能拉取代码，创建步骤跟本地一样,用root用户登录服务器：<br/>
`cd ~  //进入root的根目录`<br/>
`cd ./.ssh  //进入ssh目录,没有这个目录的话用mkdir .ssh创一个`<br/>
`git config --global user.name "xiangshun"  //配置git的用户名`
`git config --global user.email "truma.xiang@edoctor.com"  //配置git的邮箱`
`ssh-keygen -t rsa -C "truma.xiang@edoctor.com"  //生存密钥，按3个回车，密码为空。`<br/>
`然后用ls查看可以看到有两个文件：id_rsa和id_rsa.pub,用cat id_rsa.pub查看公钥内容，复制下来，添加到你coding账户的ssh 公钥里面`

4. 本地新建一个项目，随便弄点东西，初始化git,传到coding。

5. 用root用户进入服务器，进入/alidata/www，把这个项目clone到这里面,假设项目名字是testProject.
6. 关键的部分了，编写php代码，到时候push后其实就是触发执行这个php文件:

	    <?php
			//执行指令，我只是执行pull操作，你也可以执行别的
			$output=shell_exec("cd /alidata/www/testProject &&git pull origin master 2>&1");
			
			//下面这个代码可以不要，是我测试用的，可以查看是否执行了这个脚本
			$myfile = fopen("/alidata/www/temp/testfile.txt", "w");
			$txt = "-".rand()."\n".$output;
			fwrite($myfile, $txt);
			fclose($myfile);
		?>
	然后把这个php文件传到服务器上，可以放在testProject里面，也可以放在在的地方。
7. 到coding上打开testProject这个项目，点击左侧的设置，然后点击WebHook，点击新建hook，把服务器上的那个php文件地址填进来。

8. 到这一步照理说应该就OK了，那我们可以测试一下，在本地修改一个文件，然后push。好了，让我们去服务器上看这个文件的记录/alidata/www/temp/testfile.txt，里面有文字，说明这个触发是成功了，但是我们看到有这样的文字:Host key verification failed。key验证失败，反正我猜应该是权限的问题。但是用root用户又是可以pull的。这说明php代码执行的这个pull的时候不是root用户。

	网上一查，用这段代码查看php执行时候的用户：`shell_exec("id -a");`结果是：`uid=503(www) gid=502(www) groups=502(www)`
	
	然后就要解决www用户权限问题，网上一查，可以用sudo来解决，但是我们要去给www用户配置这个权限
	
	执行`visudo`

	    //找到这一行：
		## Allow root to run any commands anywhere
    	root	ALL=(ALL)   	ALL
		//把www用户加进去：
		www     ALL=(ALL)       NOPASSWD:ALL
	这样还不行，会出现：you must have a tty to run sudo这样的错误，所以还要改一个地方：

		注释掉 Default requiretty 一行
		#Default requiretty

9. 修改php文件:

		<?php
			//执行指令，我只是执行pull操作，你也可以执行别的
			$output=shell_exec("cd /alidata/www/testProject &&sudo git pull origin master 2>&1");
			
			//下面这个代码可以不要，是我测试用的，可以查看是否执行了这个脚本
			$myfile = fopen("/alidata/www/temp/testfile.txt", "w");
			$txt = "-".rand()."\n".$output;
			fwrite($myfile, $txt);
			fclose($myfile);
		?>

10. 上传php文件，测试吧，没想到这么简单。  



## 其他: ##


- 我们如果把服务器上的shh公钥添加到coding的ssh Key里面去，那么coding上的部署公钥可以不用填，我已经测试过。

- php文件里面的2>&1，这一句主要是用于显示详细信息的。如果不加，如果出错$output没有任何信息。





			
		