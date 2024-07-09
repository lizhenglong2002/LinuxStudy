# Linux文件编程

## 一、原理简述

### 1.1 文件描述符

​		**对于内核而言，所有打开文件都由文件描述符引用。**文件描述符是一个非负整数。当打开一个现存文件获创建一个新文件时，**内核向文件返回一个文件描述符。**当读写一个文件时，用 open 和 creat 函数返回的文件描述符标识该文件，将其作为参数传递给 read 和 write 函数。

​		UNIX shell使用 文件描述符0 与 进程的标准输入 相结合，文件描述符1 与 标准输出 相结合，文件描述符2 与 标准错误输出 相结合。STDIN_FILENO丶STDOUT_FILENO丶STDERR_FILENO这几个宏代替了0丶1丶2这几个数。

​		文件描述符，这个数字在一个进程中表示一个特定含义，**当我们open一个文件时，操作系统在内存中构建了一些数据结构来表示这个动态文件，然后返回给应用程序一个数字作为文件描述符**，*这个数字就和我们内存中维护的这个动态文件的这些数据结构绑定上了，以后我们应用程序如果要操作这个动态文件，只需要用这个文件描述符区分。*

​		文件描述符的作用域就是当前进程，出了当前进程，文件描述符就没有意义了。



### 1.2 文件操作原理

1. 在linux中要操作一个文件，一般是先open打开一个文件，得到文件描述符，然后对文件进行读写操作（或其他操作），最后是close关闭文件即可。

2. 对文件进行操作时，一定要先打开文件，打开成功之后才能操作，如果打开失败，就不用进行后边的操作了，最后读写完成后，一定要关闭文件，否则会造成文件损坏。

3. 文件平时是存放在块设备中的文件系统文件中的，我们把这种文件叫静态文件。

   当我们去open打开一个文件时，linux内核做到操作包括：

   ```c
   //1:内核在进程中建立一个打开文件的数据结构，记录下我们打开的这个文件；
   
   //2:内核在内存中申请一段内存，并且将静态文件的内容从块设备中读取到内核中特定地址管理存放（叫动态文件）。
   ```

   **静态文件**：存放于磁盘,未被打开的文件

   **动态文件**：当使用open后,在linux内核会产生一个结构体来记录文件的信息，例如fd,buf,信息节点.此时的read，write都是对动态文件进行操作,当close时,才把缓存区所有的数据写回磁盘中。

4.  打开文件以后，以后对这个文件的读写操作，都是针对内存中的这一份动态文件的，而不是针对静态文件的。 

   **当我们对动态文件进行读写以后，此时内存中动态文件和块设备文件中的静态文件就不同步了，当我们close关闭动态文件时，close内部内核将内存中的动态文件的内容去更新（同步）块设备中的静态文件。**

5. 为什么这样设计，不直接对块设备操作。

   **块设备本身读写非常不灵活，是按块读写的， 而内存是按字节单位操作的，而且可以随机操作，很灵活。**

## 二、文件编程的相关函数

**==注意：==** **每次读写文件之后，要调用close函数来关闭文件，否则会导致文件的破坏、内容的丢失。**

### 2.1 open函数创建及打开文件

------

**头文件**

```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
```

**函数原型**

```
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

**参数**

1. pathname：**文件路径**

2. flags：打开文件的三种模式（三选一）

   **O_RDONLY（可读）,  O_WRONLY（可写）,  O_RDWR（可读可写）**

   另外，选择以上其中一个参数后还可以选择以下参数：

   **O_CREAT**

   ​	若文件不存在则创建它。使用此选项时，需同时说明参数3mode，其

   说明该新文件的存取许可权限；

   **O_EXCL**

   ​        如果同时指定了O_CREAT，而文件已经存在，则返回-1；
   **O_APPEND**

   ​        每次写时都加到文件尾端；
   **O_TRUNC**

   ​        属性去打开文件时，如果这个文件中本来是有内容的，而且为只读或者只写成功打开，则将其长度截短为0。

3. mode：**文件权限**

**返回值**

- **成功：**文件对应的文件描述符（大于0的整数）
- **失败：**返回  -1（Linux系统中默认 0 - 标准输入，1 - 标准输出，2 - 标准错误）

**示例代码**

```c
#include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
 
int main()
{
    //int open(const char *pathname, int flags);
 
    int fd = 0;
    fd = open("test.txt",O_RDWR);
 
    //打开文件失败
    if(fd == -1)
    {
        printf("fail to open file\r\n");
        //若打不开文件，就以读写的方式创建文件，并赋给它可读可写的权限
        fd = open("test.txt",O_RDWR|O_CREAT,0600);
        if(fd == -1)
        {
            //创建文件失败
            printf("fail to create file\r\n");
        }
        else
        {
            //创建文件成功
            printf("Successfully create file!\r\n");
        }
      close(fd);  
    }
    return 0;
}
```



### 2.2 write函数写入操作

**头文件**

```c
#include <unistd.h>
```

**函数原型**

```c
ssize_t write(int fd, const void *buf, size_t count); 
```

**参数**

1. ==fd==:**文件描述符，即open函数成功的返回值**
2. ==*buf==:**写入到文件的缓冲区**
3. ==count==:**要写入内容的字节数**

**返回值**

- **成功：**返回写入字节大小，没有内容写入则返回0
- **失败：**则返回 -1

**示例代码**

```c
#include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
 
int main()
{
	//int open(const char *pathname, int flags);
 
	int fd = 0;
	char *buf = "you are very good!";
	fd = open("test.txt",O_RDWR);	//打开该文件
 
	if(fd == -1)
	{
		//打开文件失败
		printf("fail to open file\r\n");
		//创建文件，并赋予权限
		fd = open("test.txt",O_RDWR|O_CREAT,0600);
		if(fd == -1)
		{
			//创建文件失败
			printf("fail to open file\r\n");
		}
		else
		{
			//创建文件成功
			printf("Successfully opened file\r\n");
		}
	}
	
	//ssize_t write(int fd, const void *buf, size_t count);
	//写操作，将要写的内容放入缓冲区中，然后写到对应的文件中（通过文件描述符）
	write(fd,buf,strlen(buf));
 
    //即，打开test.txt文件之后，可以看到文件中有我们要存入的内容
	close(fd);
	
	return 0;
}
```



### 2.3 read函数读操作

**头文件**

```c
 #include <unistd.h>
```

**函数原型**

```c
ssize_t read(int fd, void *buf, size_t count);
```

**参数**

1. ==fd==:**文件描述符，即open函数成功的返回值**
2. ==*buf==:**存放读出来内容的缓冲区**
3. ==count==:**要读取内容的字节数**

**返回值**

- **成功：**返回读取内容字节大小，没有内容则返回0
- **失败：**则返回 -1

**示例代码**

```c
#include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
 
int main()
{
    int fd = 0;
    char *readbuf = NULL;           //这是空指针
    int read_size = 0;
 
    //动态开辟内存
    readbuf = (char *)malloc(sizeof(char) * 128 + 1);
 
    fd = open("test.txt",O_RDONLY); //以只读的方式，打开该文件
    /*
     *  省略返回值的判断
     * */
 
    
    //ssize_t read(int fd, void *buf, size_t count);
    //读操作，将文件的内容，读出来并存到缓存区中
    read_size = read(fd,readbuf,128);
    
    printf("the size is %d,content is %s \r\n",read_size,readbuf);
    
    free(readbuf);  //释放内存
    close(fd);
    
    return 0;
}
```



### 2.4 Lseek函数光标移动

**头文件**

```c
#include <sys/types.h>
#include <unistd.h>
```

**函数原型**

```c
off_t lseek(int fd, off_t offset, int whence);
```

**参数**

1. ==fd：==        **文件描述符，即open函数成功的返回值***
2. ==buf：==        **存放读出来内容的缓冲区**
3. ==count：==        **文件的字节大小**

**返回值**

- **成功**：返回针对文件开头开始计算的偏移量
- **失败：**则返回 -1

**示例代码**

```c
include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
 
int main()
{
    int fd = 0;
    fd = open("test.txt",O_RDWR);
 
    //计算文件的大小用lseek
    int file_size;
    file_size = lseek(fd,0,SEEK_END);
 
 
    printf("the file size is %d\r\n",file_size);
    close(fd);
 
    return 0;
}
```



## 三、用C标准库中的相关函数

### 1.fopen与open的区别

**（1）来源：**

open是UNIX系统调用函数（包含LINUX等），返回的是文件描述符（File Description），它是文件在文件描述符表里的索引。

fopen是ANSIC标准中的C语言库函数，在不同的系统中应调用不同的内核API，返回一个指向文件结构的指针。

**（2）移植性：**

‘fopen’是C标准函数，因此拥有良好的移植性；而‘open’是UNIX系统调用的，移植性有限。如：windows下相似的功能使用API函数‘CreateFile’。

**（3）适用范围：**

open返回文件描述符，而文件描述符是UNIX系统下的一个重要概念，UNIX下的一切设备都是以文件的形式操作。如网络套接字、硬件设备等，也包括操作普通正规文件（Regular File）

fopen是用来操作普通正规文件（Regular File）

**（4）缓冲/非缓冲文件系统：**

1. **缓冲文件系统：**

   缓冲文件系统是借助于文件结构体指针 FILE* 来对文件进行管理，通过文件指针对文件进行访问，即可以读写字符、字符串、格式化数据，也可以读写二进制数据。

   **缓冲文件系统特点：**

   在内存中开辟一个“缓冲区”，为程序里每一个文件使用，当执行读文件操作时，从磁盘文件将数据先读入内存“缓冲区”，装满后再从内存“缓冲区”依次读入接收的变量。执行写文件操作时，也是先将数据写入内存“缓冲区”，待内存“缓冲区”装满后再写入文件。由此可以看出，内存“缓冲区”的大小，影响着实际操作外在的次数，内存“缓冲区”越大，则操作外存的次数就越少，执行速度就越快，效率就越高。一般来说，文件“缓冲区”的大小跟机器是相关的。

   **缓冲文件系统的IO函数主要包括：**

   ```c
   //fopen, fclose, fread, fwrite, fgetc, fgets, fputc, fputs, freopen, fseek, ftell, 
   //rewind等
   ```

2. **非缓冲文件系统：**

   - 非缓冲文件系统依赖于操作系统，通过操作系统的功能对文件进行读写，是系统级的输入输出

   - 它不设文件结构体指针，只能读写二进制文件（对于UNIX系统内核而言，文本文件和二进制代码文件并无区别），但效率高、速度快，由于ANSI标准不再包括非缓冲文件系统，因此，在读取正规的文件时，建议大家最好不要选择它。

   **非缓冲文件系统的IO函数主要包括：**

   ```c
   //open, close, read, write, getc, getchar, putc, putchar等
   ```

**（5）文件IO层次：**

```
open：低级IO函数

fopen：高级IO函数

高低级根据谁离系统内核更近

低级：运行在内核态

高级：运行在用户态
```

**（6）补充：**

> 如果文件的大小是8k。
>
> 你如果用read/write，且只分配了2K的缓存，则要将此文件读出需要做4次系统调用来实际从磁盘上读出。如果你用fread/fwrite，则系统自动分配缓存，则读出此文件只要一次系统调用从磁盘上读出。也就是用read/write要读4次磁盘，而用fread/fwrite则只要读1次磁盘。效率比read/write要高4倍。
>
> 如果程序对内存有限制，则用read/write比较好。都用fread 和fwrite，它自动分配缓存，速度会很快，比自己来做要简单。
>
> 如果要处理一些特殊的文件，用read 和write，如 套接口，管道之类的设备文件。
>
> 系统调用write的效率取决于你buffer的大小和你要写入的总数量，如果buffer太小，你进入内核空间的次数大增，效率就低下。而fwrite会替你做缓存，减少了实际出现的系统调用，所以效率比较高。
>

### 2.fread、fwrite和read、write的区别

##### fread、fwrite是带缓冲的,read、write不带缓冲



## 四、小练习

### 4.1 实现cp指令

**思路：**

1. 打开文件A
2. 读取文件A内容
3. 打开文件B
4. 将读取内容写入文件B
5. 关闭所有文件

**示例代码：**

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc,char *argv[])
{
        char *readBuf = NULL;
        int readfd;
        int filesize;
        int writefd;
		//确认执行文件指令是三个
        if(argc != 3){
                printf("program error\n");
                exit(-1);
        }
		
        readfd = open(argv[1],O_RDWR);
		
    	//读取文件大小
        filesize = lseek(readfd,0,SEEK_END);
        lseek(readfd,0,SEEK_SET);

    	//为读取缓冲区分配空间
        readBuf = (char *)malloc(sizeof(char)*filesize + 8);
        read(readfd,readBuf,filesize);

    	//打开写入文件，若存在——使用O_TRUNC清除原有内容，若不存在——使用O_CREAT创建新文件，并给i与权限0600
        writefd = open(argv[2],O_RDWR|O_CREAT|O_TRUNC,0600);
 		write(writefd,readBuf,filesize);

        free(readBuf);
        close(readfd);
        close(writefd);
        return 0;
}
```

### 4.2 修改配置文件

**思路：**

1. 打开文件A。
2. 读取文件A内容。
3. 找到要修改的内容位置。
4. 修改。
5. 写入文件A。
6. 关闭文件A。

**示例代码：**

```c
#include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
 
int main(int argc,char **argv)
{
 
	/*修改文件的内容*/
	if(argc != 2)
	{
		printf("the param is error\r\n");
		exit(-1);
	}
 
	//先打开源文件，进行读操作，读到缓冲区
	int fdSrc = 0;
	char *readbuf = NULL;
 
	fdSrc = open(argv[1],O_RDWR);
	
	/*动态开辟内存，不会造成浪费*/
	int size = lseek(fdSrc,0,SEEK_END);
	readbuf = (char *)malloc(sizeof(char) * size + 8);
 
	lseek(fdSrc,0,SEEK_SET);
	read(fdSrc,readbuf,size);
 
	char *p = strstr(readbuf,"LENG=");
	p = p + strlen("LENG=");
 
	*p = '9';	
 
	lseek(fdSrc,0,SEEK_SET);
	write(fdSrc,readbuf,size);
	
    free(readbuf);
	close(fdSrc);
	
	return 0;
}
```



### 4.3 将各类数据写入文件/从文件中读取（此处演示结构体）

**函数原型：**

```c
ssize_t write(int fd, const void *buf, size_t count); 
ssize_t read(int fd, void *buf, size_t count);
```

可以看到，buf是无类型的，所以任何类型的类型都能写入

**示例代码：**

```c
#include "stdio.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

struct Test{
  int a;
  char b;
};

int main()
{
    //
    if(argc != 2)
	{
		printf("the param is error\r\n");
		exit(-1);
	}
    
	//先打开源文件，进行读操作，读到缓冲区
	int fdSrc = 0;
    struct Test test1[3] = {{1,'a'},{2,'b'},{3,'c'}};
	struct Test test1[2] ;
 	
	fdSrc = open(argv[1],O_RDWR);
	
    //写入
    wirte(fdSrc,&test1,sizeof(struct Test));
    
    lseek(fdSrc,0,SEEK_SET);
	
    //读取
 	read(fdSrc,readbuf,size);
    
	close(fdSrc);
	return 0;
}
```

