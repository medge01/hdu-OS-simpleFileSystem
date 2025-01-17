
## 简介

​本简单文件系统是hdu操作系统课程设计终极实验，文档或者代码中不可避免存在一些小问题，甚至是大问题。有问题请提issues 或者邮件**zzkzs01@163.com**。

其他同学实现的简单文件系统一般是运行时在内存上跑一次。本简单文件系统会将程序中的文件系统的数据持久化到本地文件 **store**，下一次运行程序时会从这里载入以实现永久性存储。

## 文件系统可以改进的地方

- 创建文件时的重名判断（项目结束才想起来）
- 多级索引文本文件（本文件系统一二三级索引未实现）
- cd 切换时使用绝对或者相对路径切换当前工作目录到任意目录
- 实现追加写（需要在`file_`结构体中设置文件末尾位置标识，改变`file_`的结构意味着FILESIZE和FILECOUT跟着要修改）
- 文件共享的实现（共享不只是能看得到其他用户创建的文件，还要求能用不同的路径名去访问该文件）
- 要将 store 文件当做磁盘，命令行每对文件修改之后，修改结果要持久化到 store 文件
- 真正实现多用户操作（多个命令行接口进入程序模拟多用户，考虑通过并发控制保证各个用户看到的文件系统具有一致性）
- 或者你自己创新的地方

## 文件系统根用户（用于登录）
```
username：root
password：123456
```

## 文件系统命令

### 用户操作

**`ccatuser`						创建用户（之后会要求输入用户名和密码）**

**`dropuser`						删除用户（之后会要求验证用户名和密码）**

### 文件操作

**`cat  [filename]  [filetype]`				创建文件 （filename 随便取）（filetype 0为目录文件 1为文本文件）**

**`ead [textfilename]`					读文本文件**

**`wrt [textfilename]`					写文本文件（覆盖式写）**

**`drop [filename]`					删除文件**

### 功能操作

**`cd [dirname]`					改变当前工作目录（只能到当前工作目录的上或下一级目录）（``cd ..``   表示回到上一级目录）**

**`ls` 							打印当前工作目录目录项**

**`exit`						退出问系统并保存系统数据到 store 文件中**



## 组和用户实现

### 用户描述

```c
user_{	
	char user_name[16];             		//用户名
	char password[16];               		//密码
	int group;                          		//所属组
}
group_{
	char name[128];					//组名
	user users[16];					//用户列表
	int count;					//组中用户数
}
workspace_{
	int userpos;					//工作用户
	file_ dirfile;               	  		//工作目录
}

group_ groups[3];					//存储各组及其用户
```



## 用户操作

### 用户登录：

根据键入的账号和密码，在`groups`数组遍历寻找`user_`，并创建为用户创建新的`workspace_`工作空间。找不到则提示并清除账号密码缓冲返回登录界面。

### 创建用户：
判断新创建的用户名是已经否存在，存在则提示操作者。不存在则判断用户名和用户密码是否合法。合法则往相应组的用户列表后填加，组用户数`count++`，如果用户表满则给提示。

### 删除用户：
在`groups`的用户表`users`中找要删除用户，找不到则给提示。找到则将其创建的文件全部链给`root`用户且修改文件的创建用户为`root`。其所在组的用户表中，要删除用户之后的用户全往前移一个单位，组用户数`count--`。

## 文件系统实现

### 磁盘块描述
```c
node_{
	int sign; 				   	//node节点类型
	char content[4096];		   		//block物理块
}

node_ nodes[1024];					//相当于整个文件系统的物理存储空间
int allocation[32][32];					//对应nodes[]的分配情况（0空闲、1已分配）
```
## 文件描述
```c
file_{
	    char  filename        			//文件名    
	    int type;               			//文件类型(目录0 普通1)
	    char ctl_auth[5];     			//控制权限（所属用户 所属组 普通组 其他组）
	    int userpos;             			//所属用户
	    int grouppos;             			//所属组
	    int time;              			//创建日期
	    int size;               			//文件大小（Byte）
	    char fadir[128];      			//父目录路径
	    int wflag;          			//写标志(多用户互斥写)
	    int alloc[n];         			//其block分配位视表
	    int nodepos;      				//文件具体块索引
}

```



## 特殊字段解释

### **sign**

| 0      | 1          | 2          | 3          |
| ------ | ---------- | ---------- | ---------- |
| 直接块 | 一级索引块 | 二级索引块 | 三级索引块 |

#### **content**

考虑到全局变量数组写进文件中，不采用malloc方式获取空间（返回指针）（原因[见下](# 3.)）。void大小是一个字节，由于void类型不能申请数组，采用char类型等价替换。

#### **ctl_auth[5]**

其中控制权限为一个数字，这个数字是以下权限的任意加和，表示拥有的权限。该思想模拟linux的文件控制权限，但是简单文件系统只实现文本文件，故取消了可执行权限4，将其改为共享权限：

| 读   | 写   | 共享 |
| ---- | ---- | ---- |
| 1    | 2    | 4    |


取出用户对应权限控制的字符位，对该字符进行运算直接判断是否有相应权限:

| return    c%2; | return    (c%4)/2; | return    c/4; |
| -------------- | ------------------ | -------------- |

#### **nodepos**

nodepos代表的是文件对应的块在全局变量nodes[nodepos]中的content。nodes[x].content相当于一个block物理块。（这里之所以不用指针的原因[见下](# 3.)）

#### **nodes[1024]**

其中**nodes[0]存根目录**，**nodes[1]存用户信息groups[]**，**nodes[2]存位示图**



## 对文件的解释

文件分为**目录文件**和**文本文件**两种，但都本质上都是文件，所以都用file_结构体表示。但也有区分，他们nodepos指向的块存放的内容是不一样的，虽然都起到目录的作用。

![文件解释](https://user-images.githubusercontent.com/76037078/177200687-34b183e1-ac45-4030-a2fb-34431c5e6635.jpg)


目录文件的nodepos指向的块划分成n小块，n取决于一个block能存多少个`file_`。默认第0个小块存目录文件自己的`file_`，即 . 项。第1个小块存其父目录的file，即 .. 项。剩下的n-2块就可以用来存放该目录的子文件或者子目录的file_。系统整体的目录就是一棵树。
普通文件的nodepos并不直接储存其内容，而是其存储具体内容的block的指针，就相当于普通文件的nodepos指向的块是索引块，通过索引块去查找具体内容块。其中一级存储块12个，二级存储块2个，三级存储块1个。其中只有以及存储块存储具体内容，而二级存储块存储的是一级块的指针，三级块存储的是二级块的指针，其实类似于一个树。

值得注意的是，为表示文件nodepos的块内分配，给文件定义了一个块内分配表alloc，置1表示已分配，置0表示小块空闲。目录文件的alloc主要用于打印文件、创建文件和删除文件。（注：由于创建新文件会更改父目录的分配表，这个更改是在程序运行时的内存上的，要把它写入相应的nodes[]物理空间，凡是存有父目录`file_`的目录都要更新：**祖父目录的子目录项，父目录的 . 项，所有子目录的 .. 项**）。普通文件的alloc用于打印文件内容和写文件内容判断。

文件主要由两部分组成，一部分是文件描述结构体`file_`，另一部分是文件具体内容存储的nodes，其整型下标指针在文件的`file_`_结构体中。文件的`file_`存储在父目录的`node_.content`中，而指向node.content的整型下标指针又存在`file_`中和块内小块中（文本文件），结构比较复杂。使用指针和内存复制memcpy()或者memmove()将`file_`和`node_.content`组织起来，构成完整的文件系统。

位示图用于nodes[]数组的分配情况，利用公示转换直接获取转换i行j列为nodes[index]对应块的索引下标：**i = index / 32	j = index %32**



## 文件操作实现

### 载入系统

​	从windows读**store**文件，将里面的内容赋值给文件系统物理空间`nodes[1024]`。根据`nodes[]`的分配[见上](#**nodes[1024]** )，将`nodes[]`系统块的数据读到`groups[]`, `rootdir`和`allocation`表中。

### 持久化系统

​	根据`nodes[]`的分配[见上](#**nodes[1024]** )，将`groups[]`, `rootdir`和`allocation[][]`表的数据写到`nodes[]`系统块中，将`nodes[]`里的内容写进本地文件**store**中。



## 文件操作实现

### 目录显示： ###

访问目录文件对应块，即`nodes[x].centent`，根据目录的allco表，挨个读取子文件的`file_`，再根据`file_`判断是否该用户共享此子文件，共享才能打印见到。

### 创建目录：

生成`file_`结构体并初始化，从`nodes[]`选取空闲node的下标给目录的nodepos，没有分配到node则创建失败。把自己的`file_`和该目录父目录的`file_`分别写到其块内的` . `和` .. `项，并修改新建目录的`alloc`。判断其父目录的`alloc`是否还有空位，有则把创建的目录的`file_`内存复制到父目录的块内小块中并修改其父目录的`alloc`，否则创建失败。父目录分配表改变，即父目录的`file_`改变，为保证文件系统的一致性，**所有存有父目录的`file_`的目录块都要修改相应小块**（祖父目录的子目录项，父目录的 . 项，所有子目录的 .. 项）。

### 删除目录：

删除子文件，如果其子文件是目录文件则递归删除，普通文件则只需要删除该文件。其占在父目录块的内存不需要释放，只需修改其父目录的`alloc`表，相应位置置零表示删除。最后回收删除文件`nodepos`指向的块，即修改`nodes[]`的分配表`allocation`。

### 创建文本文件：

首先判断该目录下是否已有重名文件，有则给提示。没有则生成`file_`并初始化，从`nodes[]`中分配一个node的下标给`file_`变量的`nodepos`，分配不到则创建失败。判断其父目录的`alloc`是否还有空位，有则把创建的文件的`file_`内存复制到父目录的块并修改其父目录的`alloc`分配表，否则创建失败。父目录分配表改变，即父目录的file改变，为保证文件系统的一致性，所有存有父目录的`file_`的目录块都要修改相应小块，[同上](# 创建目录：)。代码中文本文件和目录文件的创建是一起实现的。

### 删除文本文件：

回收删除文件的中的存储的直接块和一级二级三级块。在文本文件的父目录块删除与该文本文件`file_`对应的小块，修改父目录`alloc`表，父目录更变，所有存有父目录`file_`的块都要同步更新。随后回收文本文件`nodepos`指向的node。

### 读文本文件：

在工作目录中找到要读文件，判断当前工作空间用户是否有read权限，没有权限则给提示，有则打开文件，打印文本文件内容到屏幕，（设置`buf`，根据普通文件的`alloc`表，不断读从文件的直接块和一二三级块读到`buf`并输出`buf`到屏幕）（也可实现重定向打印到其他文件中）

### 写文本文件：

先用字符数组`buf`接收控制台输入的文本。在工作目录中找到要写文件然后打开，判断该文件是不是文本文件，目录文件不可写，给提示然后退出。再判断当前工作空间用户是否有wirte权限，没有则给提示并退出。有则再判断文件1`file_`中写标志`wfalg`是否置为1，置1则提示退出；置0则改为改写标志为1，然后写如`buf`中的内容，写完后更新文件`size`，最后该文件写标志再置0，以此达到多用户对同一文件的互斥写。

### 共享：

多用户公用一个文件系统资源，但是在显示目录文件时根据用户是否有共享权来显示。如果有共享权限则共享，没有则不共享。

### 保护：

不同用户在对文件进行操作时，根据文件的的控制权限和当前工作用户所属组进行比对。有相应的读写权限才能执行相应的读写操作，以此达到文件保护。

### 打开文件：

创建一个临时`file_`变量，从要打开文件的父目录的块的相应小块内存复制要读文件的`file_`到临时创建的`file_`变量中，即是打开文件。

### 关闭文件：

释放打开文件时临时创建的`file_`变量的内存即是关闭文件。

## 遇到问题及解决方法（杂七杂八）

#### 1．

​	一个功能用在多处，每用一次都需要实现一次。代码不复杂，但是容易少语句或者顺序不一致导致错误。所以即使是两行代码，为保证功能的完整性，一定要将其单独封装成一个函数。例如：查找空闲node和分配node应当单独写成一个函数。

#### 2．

​	文件系统除第一次需要初始化外各全局变量外，后面的使用都是直接从文件中导入数据给全局变量。要读就要先写，在写的时候，由于最开始设计的的结`node_`构体中存储的是`void`指针，指针指向的是一个内存地址。但是在`fwrite()`写入时，只能写入`void`指针的地址，而不能写入地址指向的内存空间的内容。**解决**：在`node_`结构体中不再采用`void`指针来链接内存空间，而是直接声明`char`数组来表示空间。`void`类型不能声明连续型空间（即数组外），`char`可以而且在其他方面基本可以等同于`void`。

#### 3．

​	在之前的设计中，`file_`具体块的指向用的是指针而不是数字索引。但是在退出保存文件系统再再入后发生一系列莫名其妙的错误。**解决**：不再采用指针去指向文件对应的具体node块，而是存储node块在全局变量`nodes[1024]`中的索引下标。**原因**：设计的文件系统在退出后存储全局信息的`nodes[]`会写到windows文件中，在下一次打开时之前对文件系统的操作结构仍然存在。因为两次程序运行的环境是不相同的，所以前一次的指针和当前所需的具体块的内存地址是不同的，会发生段错误。

#### 4．

​	一些读入问题，比如`scanf`的使用方式读取空格自动结束读取输入。但是在输入命令中需要含有空格，而在写入文本文件时也是需要接收换行符的。解决：前一种将平常使用的`%s`改为``%[^\n]``，后一种则改为``%[^EOF]``。这样就能重新设定结束输入的结束字符。

#### 5．

​	在解析命令时不仅需要`strcmp()`具体命令和由命令解析出来的`argv[]`，还需要`argc`来限定。比如`ls fasdfad`能够执行ls，在增加采用`argc`判断后就能正确判断为不能执行`ls`。

#### 6．

​	为保证文件系统运行时的临时结构体和`nodes[]`中的保持一致，当我们每一次创建文件或者删除文件时父目录的分配表改变，为保证系统的一致性，要马上把更改的``file_``写入文件系统的物理块`nodes[]`中，因为cd改变工作路径时获取当前工作目录文件``file_``是从`nodes[]`中取的。所有存有被更改`file_`的目录都要同步更新该`file_`对应的块内小块。即：**祖父目录的子目录项，父目录的` . `项，所有子目录的` .. `项**。

#### 7．

​	`memmove()`或者`memcpy()`设置的内存复制大小一定要等同于复制到的`file_`结构体的大小。之前由于设置要复制的大小大于`file_`大小，导致声明`file_`后再声明的整型计数值i异常，因为两者的空间的分配几乎连续。为此我更改采用全局变量i计数，还实现了简单的堆栈（调用多个函数可能都需要i），浪费了很多时间精力。最后修改全局变量相应值才解决。
