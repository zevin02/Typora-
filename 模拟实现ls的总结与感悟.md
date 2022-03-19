@[toc]
* 引言
我们每个人在学习Linux的时候使用的第一个命令都应该是ls，这个命令也开启了我们学习Linux的大门，但是应该很少有人知道ls是如何实现的，所以我便在自己的能力范围内对ls进行模拟实现
# 前期准备[^1]

[^1]: 对于模式函数的理解与认识
## argc与argv
我们在学习c语言的时候应该就应该对此有所了解了，其实main函数也是可以带参数的
> 
> argc，就是命令行所带的参数的个数，默认情况下argc是1
> argv，就是命令行后面的字符串，默认情况下argv[0]='.'
>~~~cpp
>ls . ..
>~~~
>ls后面带有两个参数，所以在1的基础上加上2
>argv[0]='.',argv[1]=".",argv[2]=“..”
>~~~cpp
>ls 
>~~~
>ls后面什么参数都没有，那么argc就是1
>argv[0]="."


## getopt

是一个用来解析命令参数的函数，就不用我们自己实现命令参数的解析
~~~cpp
#include<unist.h>
 int getopt(int argc, char * const argv[], const char *optstring);
~~~
>其中argc就是参数的个数，
argv里面就是命令行的所有参数，
而optstring就是我们要解析的参数如："ali",我们要解析的参数就是-a，-l，-i

失败的话就返回-1
>unistd里有个 optind　变量，每次getopt后，这个索引指向argv里当前分析的字符串的下一个索引[^2]，因此argv[optind]就能得到下个字符串，通过判断是否以 '-'开头就可



[^2]:即每次循环调用一次getopt，optind就会+1，
## stat
stat函数就是用来获取文件的信息
成功返回0，失败返回-1
~~~c
include <sys/types.h>
 #include <sys/stat.h> 
 #include <unistd.h>
 int stat(const char *path, struct stat *buf)
~~~
> path就是文件存储的路径，可以是相对路径也可以是绝对路径

### struct stat结构体
~~~c
struct stat
{
    dev_t     st_dev;     /* ID of device containing file */文件使用的设备号
    ino_t     st_ino;     /* inode number */    索引节点号 
    mode_t    st_mode;    /* protection */  文件对应的模式，文件，目录等
    nlink_t   st_nlink;   /* number of hard links */    文件的硬连接数  
    uid_t     st_uid;     /* user ID of owner */    所有者用户识别号
    gid_t     st_gid;     /* group ID of owner */   组识别号  
    dev_t     st_rdev;    /* device ID (if special file) */ 设备文件的设备号
    off_t     st_size;    /* total size, in bytes */ 以字节为单位的文件容量   
    blksize_t st_blksize; /* blocksize for file system I/O */ 包含该文件的磁盘块的大小   
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */ 该文件所占的磁盘块  
    time_t    st_atime;   /* time of last access */ 最后一次访问该文件的时间   
    time_t    st_mtime;   /* time of last modification */ /最后一次修改该文件的时间   
    time_t    st_ctime;   /* time of last status change */ 最后一次改变该文件状态的时间   
};
~~~


struct stat中st_mode还有很多情况
我列举几个常见的
~~~c
    S_IFMT   0170000    文件类型的位遮罩
    S_IFSOCK 0140000    套接字
    S_IFLNK 0120000     符号连接
    S_IFREG 0100000     一般文件
    S_IFBLK 0060000     区块装置
    S_IFDIR 0040000     目录
    S_IFCHR 0020000     字符装置
    S_IFIFO 0010000     先进先出
​
    S_ISUID 04000     文件的(set user-id on execution)位
    S_ISGID 02000     文件的(set group-id on execution)位
    S_ISVTX 01000     文件的sticky位
​
    S_IRUSR(S_IREAD) 00400     文件所有者具可读取权限
    S_IWUSR(S_IWRITE)00200     文件所有者具可写入权限
    S_IXUSR(S_IEXEC) 00100     文件所有者具可执行权限
​
    使用st_mode和S_IRGRP此类进行按位与&，如果为真就可以

    S_IRGRP 00040             用户组具可读取权限
    S_IWGRP 00020             用户组具可写入权限
    S_IXGRP 00010             用户组具可执行权限
​
    S_IROTH 00004             其他用户具可读取权限
    S_IWOTH 00002             其他用户具可写入权限
    S_IXOTH 00001             其他用户具可执行权限
​
    上述的文件类型在POSIX中定义了检查这些类型的宏定义：
    
    S_ISLNK (st_mode)    判断是否为符号连接
    S_ISREG (st_mode)    是否为一般文件
    S_ISDIR (st_mode)    是否为目录
    S_ISCHR (st_mode)    是否为字符装置文件
    S_ISBLK (s3e)        是否为先进先出
    S_ISSOCK (st_mode)   是否为socket
    若一目录具有sticky位(S_ISVTX)，则表示在此目录下的文件只能被该文件所有者、此目录所有者或root来删除或改名，在linux中，最典型的就是这个/tmp目录啦。
​
~~~
我们可以使用st_mode和 S_IFMT进行&，得到的来判断其的文件类型
同时还可以使用S_ISDIR（st_mode）此类函数判断其文件类型，

## sprintf与fprintf
sprintf可以将格式化的数据存放到一个字符串里面
~~~c
char fname[20];
sprintf(fname,"%s \%s","hello","world")；
fname=“hello \world”;
~~~
fprintf是将格式化的数据写入到一个文件流中
与printf的区别是printf是将内容写到stdout的标准输出流里面（即打印到显示屏上）

## opendir && closedir
opendir就是打开一个目录，可以类比打开一个文件
~~~c
#include<sys/types.h>
#include<dirent.h>
　　DIR* opendir (const char * path ); 
~~~
path就是存放文件的路径，
而DIR *相当于FILE*的指针
失败就返回一个NULL
>DIR 结构体的原型为：struct_dirstream
~~~c
 struct __dirstream
   {
     void *__fd; /* `struct hurd_fd' pointer for descriptor.   */
     char *__data; /* Directory block.   */
     int __entry_data; /* Entry number `__data' corresponds to.   */
     char *__ptr; /* Current pointer into the block.   */
     int __entry_ptr; /* Entry number `__ptr' corresponds to.   */
     size_t __allocation; /* Space allocated for the block.   */
     size_t __size; /* Total valid data in the block.   */
     __libc_lock_define (, __lock) /* Mutex lock for this structure.   */
    };
  ~~~  
closedir就是在每次调用完opendir后把目录关闭，避免内存泄漏

## readdir
读取opendir目录返回值的列表，
~~~c
#include<dirent.h>
struct dirent* readdir(DIR* dir_handle); 
~~~
返回dirent结构体的指针
~~~c
struct dirent

{undefined

long d_ino; /* inode number 索引节点号 */

off_t d_off; /* offset to this dirent 在目录文件中的偏移 */

unsigned short d_reclen; /* length of this d_name 文件名长 */

unsigned char d_type; /* the type of d_name 文件类型 */

char d_name [NAME_MAX+1]; /* file name (null-terminated) 文件名，最长255字符 */

}
~~~

# 实现过程中遇到的麻烦
## 颜色控制
printf("<u>\033[**31**m</u>  hello <u>\033[0m</u>")
划线处就是控制颜色输出的格式，黑体字就是我们需要他是什么颜色

> 背景色 字体色
> 
> 40: 黑 30: 黑
> 
> 41: 红 31: 红
> 
> 42: 绿 32: 绿
> 
> 43: 黄 33: 黄
> 
> 44: 蓝 34: 蓝
> 
> 45: 紫 35: 紫
> 
> 46: 深绿 36: 深绿
> 
> 47: 白色 37: 白色
记得在打印完之后，把颜色恢复成NONE[^ 4]，不然再后面的打印都会跟着变色。

[^4]:在第二个\033处，\033[m

<u>其他的printf格式就由你们去探索吧</u>

## Linux中多文件操作
>我们在早期写写代码的时候，可以就在一个文件上写就可以了，但是如果一旦代码量大的话，一个文件就显得太少了，且每次查找也十分麻烦，那么我们就应该在多个文件中，进行编写代码

   但是在Linux中，多个文件可能有一点麻烦
   
--- 
1. 全局变量的定义
	<u>在.h文件中</u>
	最顶部加上这两行，ifndef后面的是任意都可以
不过下面define后要添加一样的内容，在最后加上#endif
之后声名全局变量，前加一个extern，不需要赋初值，在我们需要的.c文件中赋初值即可
~~~c
#ifndef _LS_H_
#define _LS_H_
extern int a;
……

#endif

~~~
		
在.c文件中赋初值
~~~c
int a=0;//前面也要加变脸类型
~~~


# 代码实现
我们会实现ls 以及 -r -l -t -R -s -i -a 

* ls.h
~~~c

#ifndef _LS_H_
#define _LS_H_
#include <stdio.h>
#include <time.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <dirent.h>
#include <pwd.h>
#include <grp.h>
#include<string.h>
#include<stdlib.h>

extern int aflag;
extern int lflag;//作为标识符，如果aflag lflag为1则有-a和-l这个参数，执行选项
extern int iflag;
extern int sflag;
extern int rflag;
extern int tflag;
extern int Rflag;
extern char filename[256][260];
extern long filetime[256];

void read_dir(char *dir);

void isFile(char *name,char*filename);
void print(struct stat buf,char* filename);
void display_rights(struct stat buf);
void display_mode(struct stat buf);
void display_sflag(char*fname,char*nname);
void display_file(char *fname , char *nname);//fname里面存放的是目录的路径,显示文件的信息
void display_iflag(char*fname,char*nname);
void getfilename(char* dir,int *cnt);
void sortbyletter(int cnt);
void gettime(int cnt,char* dir);//获得时间的数字
void display_dir(char *dir);//显示目录下的所有文件，同时判断是否有-a选项，里面的是目录的路径名，有相对路径，也有绝对路径
void judge_mode(int argc,char*argv[],int ch,char *s);
void IsFile(char* dir);
void dp_R(char*dir);
#endif
~~~
* ls.c
~~~c
#include"ls.h"
//打印颜色

int aflag=0;
int lflag=0;//作为标识符，如果aflag lflag为1则有-a和-l这个参数，执行选项
int iflag=0;
int sflag=0;
int rflag=0;
int tflag=0;
int Rflag=0;
char filename[256][260];
long filetime[256];
void print(struct stat buf,char* filename)
{
                if(S_ISREG(buf.st_mode))//一般文件
                {
                  if(buf.st_mode&S_IXOTH)//可执行文件,打印成绿色
                  {
                  printf("\033[32m %s  \033[0m",filename);
                  }
                  else 
                  {

                printf("%s  ",filename);//直接接打印文件名,不显示文件
                  }
                }
                else if(S_ISDIR(buf.st_mode))//是一个目录打印蓝色
                {
                  printf("\033[34m %s  \033[0m",filename);
                }
}
void display_rights(struct stat buf)
{
  
      if(buf.st_mode & S_IRUSR)
				printf("r");
			else
				printf("-");
			if(buf.st_mode & S_IWUSR)
				printf("w");
			else
				printf("-");
			if(buf.st_mode & S_IXUSR)
				printf("x");
			else
				printf("-");
      //组权限
			if(buf.st_mode & S_IRGRP)
				printf("r");
			else
				printf("-");
			if(buf.st_mode & S_IWGRP)
				printf("w");
			else
				printf("-");
			if(buf.st_mode & S_IXGRP)
				printf("x");
			else
				printf("-");
     //其他人权限
			if(buf.st_mode & S_IROTH)
				printf("r");
			else
				printf("-");
			if(buf.st_mode & S_IWOTH)
				printf("w");
			else			
				printf("-");
			if(buf.st_mode & S_IXOTH)
				printf("x");
			else
				printf("-");
}


//打印文件类型
void display_mode(struct stat buf)
{
  
            switch(buf.st_mode & S_IFMT)//s_ifmt是一个子域掩码，按位与的结果来判断是什么权限
            {
                case S_IFSOCK:  printf("s");  break;
                case S_IFLNK: printf("l");  break;
                case S_IFREG: printf("-");  break;
                case S_IFBLK: printf("b");  break;
                case S_IFDIR: printf("d");  break;
                case S_IFCHR: printf("c");  break;
                case S_IFIFO: printf("p");  break;
                                                                                                                                  
            }
}


void display_sflag(char*fname,char*nname)
{

    struct stat buf;//buf用来接收文件的各种信息
    stat(fname,&buf);
    if(stat(fname,&buf) ==-1)//路径存放失败
    {
       perror("stat error\n");
       return ;                 
    }
   long long size= buf.st_size/1024;
   if(size<=4)
   {
     printf("4 ");
   }
   else 
   {
     printf("%4lld ",size);
   }
   print(buf,nname);
}


void display_file(char *fname , char *nname)//fname里面存放的是目录的路径,显示文件的信息
{
            struct stat buf;//buf用来接收文件的各种信息
            //   struct tm *t;//用来接收时间的参数，

           if(stat(fname,&buf) ==-1)//路径存放失败
          {
           perror("stat error\n");
           return ;                 
          }
          if(iflag)//有-i这个选项，就在首行打印
          {
            printf("%ld ",buf.st_ino);
          }

          if(sflag)//有-s这个选项
          {
   long long size= buf.st_size/1024;
   if(size<=4)
   {
     printf("%3d ",4);
   }
   else 
   {
     printf("%3lld ",size);
   }
          }

    //打印文件类型
           display_mode(buf);
            //打印权限
           // user other group
            display_rights(buf);
            printf(" %2ld ",buf.st_nlink);//硬链接数
          //打印用户名和所属组
            printf("%10s ",getpwuid(buf.st_uid)->pw_name);
            printf("%10s ",getgrgid(buf.st_gid)->gr_name);

            printf(" %8ld  ",buf.st_size);//所占的字节大小
            char *ctime();
            printf("%.12s  ",4+ctime(&(&buf) -> st_mtime));
            print(buf,nname);//打印的同时显示颜色
            printf("\n");
      return ;

}






//实现- i选项
void display_iflag(char*fname,char*nname)
{

    struct stat buf;//buf用来接收文件的各种信息
    stat(fname,&buf);
    if(stat(fname,&buf) ==-1)//路径存放失败
    {
       perror("stat error\n");
       return ;                 
    }
      printf("%ld  ",buf.st_ino);
     // printf("%5s  ",nname);
     print(buf,nname);
}
//获取文件的名字
void getfilename(char* dir,int *cnt)
{

    DIR * cntdir;
    struct dirent* cntitem;
    cntdir = opendir(dir);
    int len=0;
    while((cntitem=readdir(cntdir))!=NULL)//记录文件名，之后再进行文件名的字典序排序
    
    {
      strcpy(filename[*cnt],cntitem->d_name);
      len=strlen(filename[*cnt]);
      len=0;
     (*cnt)++;
    }
}
//按照字典序排列,默认按升序排列，如果有-r的话就逆序排列
void sortbyletter(int cnt)
{
  char temp[260];
  int j=0;
  int i=0;
  for(i=0;i<(cnt)-1;i++)
  {
    for(j=0;j<(cnt)-1-i;j++)
    {
      if(tflag)
      {
      if(filetime[j]<filetime[j+1])
      {
         strcpy(temp,filename[j]);
         strcpy(filename[j],filename[j+1]);
         strcpy(filename[j+1],temp);
      }
      }
      if(rflag)
      {

      if(strcmp(filename[j],filename[j+1])<0)
      {
         strcpy(temp,filename[j]);
         strcpy(filename[j],filename[j+1]);
         strcpy(filename[j+1],temp);
      }
      }
      else 
      {

      if(strcmp(filename[j],filename[j+1])>0)
      {
         strcpy(temp,filename[j]);
         strcpy(filename[j],filename[j+1]);
         strcpy(filename[j+1],temp);
      }
      }
    }
  }

}

void gettime(int cnt,char* dir)//获得时间的数字
{
  struct stat buf;
  char fname[256];
  int j=0;
    sprintf(fname,"%s/%s",dir,filename[j]);
    stat(fname,&buf);
  for(j=0;j<(cnt);j++)
  {

    filetime[j]=buf.st_mtime;
  }
}




void display_dir(char *dir)//显示目录下的所有文件，同时判断是否有-a选项，里面的是目录的路径名，有相对路径，也有绝对路径
{
    int i=0;
    DIR *mydir;
    struct dirent *myitem;
    char fname[256];
    struct stat buf;//获取文件属性打印文件颜色
    if((mydir = opendir(dir)) == NULL)
    {
        perror("fail to opendir!\n");
        return ;                     
    }
    int cnt=0;
    getfilename(dir,&cnt);//得到目录下文件的名字以及时间
    gettime(cnt,dir);
    sortbyletter(cnt);
    int j=0;
       for(j=0;j<cnt;j++)
       {
        //fname里面是目录的名字和文件的名字,全都弄到fname里面
           sprintf(fname,"%s/%s",dir,filename[j]);//dir这个目录的路径名字，文件名，这些名字全都答应到fname这个字符串里面来接收
           stat(fname,&buf);
           if(filename[j][0] == '.' && aflag==0)//没有-a参数，如果if条件成立的就继续下一次循环，否则往下执行
             
         {
           continue;//在目录往下继续搜索，遇到隐藏文件就跳过
         }


          if(lflag)//走到这里就说明有-a参数，ls -l -a dir，同时也有-l参数
        {
           //srvprintf(fname,"%s/%s",dir,myitem->d_name);//dir这个目录的路径名字，文件名，这些名字全都答应到fname这个字符串里面来接收
       //    printf("%s",fname);
      
            display_file(fname,filename[j]);//把文件的秘密打印出来,第一个是目录的路径名，第二个是里面的文件名                                      
        }

          else if(Rflag)//出现了-R选项
          {
            //display_Rflag(fname,filename[j],buf);//fname里面是路径名，filename[j]里面都是文件
          }
          else if(iflag)//出现了-i选项就执行
          {

           //sprintf(fname,"%s/%s",dir,myitem->d_name);//dir这个目录的路径名字，文件名，这些名字全都答应到fname这个字符串里面来接收
            display_iflag(fname,filename[j]);
            i++;
            if(i%5==0)
            {
              printf("\n");
            }
          }
          else if(sflag)
          {
            display_sflag(fname,filename[j]);
          }
          else // ls -a dir,没有参数，只显示一个目录，
          { 
            if(iflag)
            {
            display_iflag(fname,filename[j]);
            }
            else 
            {
            //如果是目录的话就用蓝色
            //如果是可执行文件的话就用绿色
            print(buf,filename[j]);
                }
         //   printf("%5s  ",filename[j]);// 显示文件名，
            }
          }
         printf("\n");
         closedir(mydir);

}
//判断是否有带什么参数
void judge_mode(int argc,char*argv[],int ch,char *s)
{

       while((ch = getopt(argc,argv,s)) != -1)//getopt用来解析命令行的参数，返回int，错误就返回-1,解析-a和-l两个命令,getopt处理-开头的参数
              //每次getopt后，这个索引指向argv里当前分析的字符串的下一个索引，因此,optind也就往后移动
              //argv[optind]就能得到下个字符串
              //
        {
                 switch(ch)//用ch来接收返回值
                      {
                   case 'a': 
                        aflag = 1;
                        break;                                  
                   case 'l': 
                        lflag = 1;  
                        break;
                   case 'i': 
                        iflag = 1;  
                        break;
                   case 's':
                        sflag=  1;
                        break;

                   case 'r'://递归显示文件，从根目录开始
                        rflag=  1;
                        break;
                   case 't':
                        tflag=1;
                        break;
                   case 'R':
                        Rflag=1;
                        break;
                   default:  
                        printf("wrong option:%c\n",optopt);
                        return ;
                      }
          }
}



void isFile(char *name,char*filename)
{
  char fname[256];
    int ret = 0;
    struct stat sb;
    ret = stat(name, &sb);
    if(S_ISDIR(sb.st_mode))
    {
      printf("%s: \n",name);
     
    if(ret == -1)
          {
                perror("stat error\n");
                    return;
          }
    DIR*dp;
    struct dirent* sdp;
    dp=opendir(name);
    while(sdp=readdir(dp))
    {
      sprintf(fname,"%s/%s",name,sdp->d_name);
            stat(fname,&sb);
                    if(strcmp(sdp->d_name,".") == 0 ||strcmp(sdp->d_name,"..") == 0||sdp->d_name[0]=='.')
                    {
                            continue;
                                
                    }
            print(sb,sdp->d_name);                        
          //  printf("%s ",sdp->d_name);
    }
    printf("\n");
                  //是真就是目录//要考虑目录下还有目录
                      read_dir(name);
                  //        
         closedir(dp);
    }
     else 
     {
       return;
     }//          
         //   printf("%s ",filename);//路径

}

void read_dir(char *dir)
{
    char path[PATH_MAX];
      DIR *dp;
        struct dirent *sdp;
          dp = opendir(dir);
            if(dp == NULL)
            {
                  perror("opendir error");
                      return;
                        
            }
              while(sdp = readdir(dp))
              {
                
                    if(strcmp(sdp->d_name,".") == 0 ||strcmp(sdp->d_name,"..") == 0||sdp->d_name[0]=='.')
                    {
                            continue;
                                
                    }

                        //读这个目录项，继续判断是不是目录，如果是目录iu还得再进入，如果是文件就打印
                        //    //ifFile(sdp->d_name);//不能直接这样，要绝对路径才可以
                                sprintf(path, "%s/%s", dir, sdp->d_name);
                                isFile(path,sdp->d_name);

              }
printf("\n");
                closedir(dp);
                  return;

}
~~~
* main.c
~~~c
#include"ls.h"
int main(int argc,char *argv[])
{
    int ch,i;
    struct stat buf;//用来记录文件的信息

      //opterr = 0;
      //解析命令
      //用来解析命令行参数命令,控制是否向STDERR打印错误。为0，则关闭打印
      //optind默认是1，调用一次getopt就会+1
      // 判断是否带有参数 
        judge_mode(argc,argv,ch,"liasrtR");
          
      // 没有带参直接ls当前目录,后面没有参数，默认就是访问当前目录
       if(argc==1||*argv[argc-1]=='-')             
          display_dir(".");
          
            for(i = optind; i < argc ; i++) //ls name1 name2....，从optind开始执行,执行后面的文件名字
           {
               if(stat(argv[i],&buf)==-1)//stat,判断的结果不成立，我们就直接返回-1，结束循环
              {
                 // perror("cannot access argv[i]!\n");
                  printf("cannot access %s!\n",argv[i]);
                  printf(": No such file or directory\n ");
                  return -1;
              }
               if(S_ISDIR(buf.st_mode))//是目录，判断此路径下有目录文件,里面的参数是st_mode
              {
                if(Rflag)
                {
                  IsFile(argv[i]);
                }
                  printf("%s:\n",argv[i]);
                  display_dir(argv[i]);//就对其进行打印, 对后面的参数,答应argv[i]代表的目录
              }
              else//file，为文件
              {
              if(lflag)//ls -l file,只打印一个
              {
                display_file(argv[i],argv[i]);//显示文件信息
              }
              else// ls file
              {
                print(buf,argv[i]);
              } 
              }
           }
        printf("\n"); 
      return 0;
 }
 ~~~


具体详细的代码可以查看我的github仓库
[模拟实现ls](https://github.com/zevin02/Linuxstudy/tree/master/myls).



