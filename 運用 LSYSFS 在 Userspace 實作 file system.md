# 運用 LSYSFS 在 Userspace 實作 file system
###### tags: `Linux`
### Basic Steps
#### LSYFS Installation on Linux Environment
1. Linux & Virtualbox
    ![](https://i.imgur.com/sd9nt5M.png)

2. 將 LSYSFS 從 github 上下載
    ```clike!
    git clone https://github.com/MaaSTaaR/LSYSFS.git
    ```
3. 進入 LSYSFS 資料夾更改 code 並編譯後 mount 在已建立的空資料夾 (資料夾路徑為/home/test/r)
    ```clike!
    cd LSYSFS
    ./lsysfs -f /home/test/r
    ```
4. 開一個新的 terminal 進入 r 資料夾進行驗證
![](https://i.imgur.com/6p2RSYh.png)

#### Add instruction
在最後新增 rmdir,unlink(rm),utime 指令
```clike!
    static struct fuse_operations operations = {
        .getattr	= do_getattr,
        .readdir	= do_readdir,
        .read		= do_read,
        .mkdir		= do_mkdir,
        .mknod		= do_mknod,
        .write		= do_write,
        .rmdir		= do_rmdir,
        .unlink		= do_unlink,
        .utime		= do_utime,
    };
```
#### Support basic commands
1. ls : 已修復 ls 只能在 / 目錄成功使用的問題 (i.e 可在其他層目錄中正常使用)，可以用 ls -al 顯示隱藏檔案
2. cd : 已修復 cd 回上層目錄後 ls 會回報錯誤的問題
+ 新增 int is_prefix() 函數，判斷 path 是否為 directory 的 prefix，是的話回傳 path length
+ 新增 int get_addr() 函數，根據 is_prefix() 切割 directory 字串 
+ 更改 int do_readdir() 函數，將 get_addr()得到的 file/directory 丟進 filler 後印出
3. mkdir : 未修改，功能正常
    ![](https://i.imgur.com/GmvTWDt.png)

4. touch : 新增 do_utime()函數，可以創建檔案並獲取檔案的時間資訊
![](https://i.imgur.com/yf82mWD.png)
![](https://i.imgur.com/43L9Ekz.png)

#### Support file and directory deletion

1. rmdir : 新增 do_rmdir(),remove_dir() 函數
+ directory 中有其他 directory 會無法刪除
+ directory 中有其他 file 會無法刪除
    ![](https://i.imgur.com/AXmCS86.png)

2. rm : 新增 do_rm(),remove_file() 函數 (截圖參考後面 echo)
+ 已確認刪除 file 後，每個 file 中的 content 不會錯亂

#### Writing a string to a file
1. echo
    ```clike!
    echo "test" >> file
    ```
    ![](https://i.imgur.com/5Jl9TDN.png)
#### Reading file in the in-memory filesystem
1. less (接上面 echo 截圖)
指令如下 : 
    ```clike!
    less file
    ```
    ![](https://i.imgur.com/G36HKlG.png)
    ![](https://i.imgur.com/tHf6UeZ.png)

### Advanced Modifications on In-memory Filesystem
#### Support big directory in the in-memory filesystem
#### Support big file in the in-memory filesystem

### References
* LSYSFS github
https://github.com/MaaSTaaR/LSYSFS
* FUSE version-2.9.4
https://github.com/libfuse/libfuse/releases?page=5


```clike!
hw2
/**
 * Less Simple, Yet Stupid Filesystem.
 * 
 * Mohammed Q. Hussain - http://www.maastaar.net
 *
 * This is an example of using FUSE to build a simple filesystem. It is a part of a tutorial in MQH Blog with the title "Writing Less Simple, Yet Stupid Filesystem Using FUSE in C": http://maastaar.net/fuse/linux/filesystem/c/2019/09/28/writing-less-simple-yet-stupid-filesystem-using-FUSE-in-C/
 *
 * License: GNU GPL
 */
 
#define FUSE_USE_VERSION 30

#include <fuse.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <time.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>
// ... //

char dir_list[ 256 ][ 256 ];
int curr_dir_idx = -1;

char files_list[ 256 ][ 256 ];
int curr_file_idx = -1;

char files_content[ 256 ][ 256 ];
int curr_file_content_idx = -1;



unsigned int dir_time[256][3] = {{0}};//[][0]: access time, [][1]: modified time, [][2]: change time
unsigned int files_time[256][3] = {{0}};

void add_dir( const char *dir_name )
{
	curr_dir_idx++;
	strcpy( dir_list[ curr_dir_idx ], dir_name );
}

int is_dir( const char *path )
{
	path++; // Eliminating "/" in the path
	
	for ( int curr_idx = 0; curr_idx <= curr_dir_idx; curr_idx++ )
		if ( strcmp( path, dir_list[ curr_idx ] ) == 0 )
			return 1;
	
	return 0;
}

void add_file( const char *filename )
{
	curr_file_idx++;
	strcpy( files_list[ curr_file_idx ], filename );
	
	curr_file_content_idx++;
	strcpy( files_content[ curr_file_content_idx ], "" );
}

int is_file( const char *path )
{
	path++; // Eliminating "/" in the path
	
	for ( int curr_idx = 0; curr_idx <= curr_file_idx; curr_idx++ )
		if ( strcmp( path, files_list[ curr_idx ] ) == 0 )
			return 1;
	
	return 0;
}

int get_file_index( const char *path )
{
	path++; // Eliminating "/" in the path
	
	for ( int curr_idx = 0; curr_idx <= curr_file_idx; curr_idx++ )
		if ( strcmp( path, files_list[ curr_idx ] ) == 0 )
			return curr_idx;
	
	return -1;
}

void write_to_file( const char *path, const char *new_content )
{
	int file_idx = get_file_index( path );
	
	if ( file_idx == -1 ) // No such file
		return;
		
	strcpy( files_content[ file_idx ], new_content ); 
}
int get_dir_index( const char *path )
{
	path++; // Eliminating "/" in the path
	
	for ( int curr_idx = 0; curr_idx <= curr_dir_idx; curr_idx++ )
		if ( strcmp( path, dir_list[ curr_idx ] ) == 0 )
			return curr_idx;
	return -1;
}
// ... //

static int do_getattr( const char *path, struct stat *st )
{
	st->st_uid = getuid(); // The owner of the file/directory is the user who mounted the filesystem
	st->st_gid = getgid(); // The group of the file/directory is the same as the group of the user who mounted the filesystem
	
	int cur_index = 0;
	
	if ( strcmp( path, "/" ) == 0)
	{
		st->st_mode = S_IFDIR | 0755;
		st->st_nlink = 2; // Why "two" hardlinks instead of "one"? The answer is here: http://unix.stackexchange.com/a/101536
		st->st_atime = time(NULL);
		st->st_mtime = time(NULL); 
		st->st_ctime = time(NULL);
	}
	else if ( is_file( path ) == 1 )
	{
		st->st_mode = S_IFREG | 0644;
		st->st_nlink = 1;
		st->st_size = 1024;
		cur_index = get_file_index(path);
		if(files_time[cur_index][0] == 0 && files_time[cur_index][1] == 0 && files_time[cur_index][2] == 0)
		{
			st->st_atime = time(NULL);
			st->st_mtime = time(NULL);
			st->st_ctime = time(NULL); 
			files_time[cur_index][0] = st->st_atime;//maintain the old time
			files_time[cur_index][1] = st->st_mtime;
			files_time[cur_index][2] = st->st_ctime;
		}
		else
		{
			st->st_atime = files_time[cur_index][0];//get the old time
			st->st_mtime = files_time[cur_index][1];
			st->st_ctime = files_time[cur_index][2];
		}
	}
	else if( is_dir( path ) == 1) 
	{
		st->st_mode = S_IFDIR | 0755;
		st->st_nlink = 2;
		cur_index = get_dir_index(path);
		if(dir_time[cur_index][0] == 0 && dir_time[cur_index][1] == 0 && dir_time[cur_index][2] == 0)
		{
			st->st_atime = time(NULL);
			st->st_mtime = time(NULL); 
			dir_time[cur_index][0] = st->st_atime;
			dir_time[cur_index][1] = st->st_mtime;
			dir_time[cur_index][2] = st->st_ctime;
		}
		else
		{
			st->st_atime = dir_time[cur_index][0];
			st->st_mtime = dir_time[cur_index][1];
			st->st_ctime = dir_time[cur_index][2];
		}
	}
	else
	{
		//printf("-ENOENT\n");
		return -ENOENT;
	}
	
	return 0;
}
int is_prefix(const char *path, const char *str)
{	
	if ( strcmp( path, "/" ) == 0 ) return 0;
	if(strlen(path)>strlen(str)) return -1;
	path++;
	printf("path++ is %s\n",path);
	
	if(strncmp(path,str,strlen(path))==0) return strlen(path);
	else return -2;
	
}

int get_addr(const char *path, const char *dir,int prefix,char *ans)
{

	if(prefix!=0){
		while(prefix>=1){
			dir++;
			printf("dirr is %s\n",dir);
			printf("prefix is %d\n",prefix);
			prefix--;
		}
		dir++;
	}
	printf("token is %s\n",dir);
	if(strchr(dir,'/')!=NULL)return 1;
	strcpy(ans,dir);
	return 0;
}
static int do_readdir( const char *path, void *buffer, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi )
{	
	int prefix = 0;
	char ans[256]="";
		
	filler( buffer, ".", NULL, 0 ); // Current Directory
	filler( buffer, "..", NULL, 0 ); // Parent Directory
	
	for(int curr_idx=0;curr_idx<=curr_dir_idx;curr_idx++){
		prefix = is_prefix(path,dir_list[curr_idx]);
		if(prefix>=0){
			if(get_addr(path,dir_list[curr_idx],prefix,ans)==0){
			filler(buffer,ans,NULL,0);}
		}
	}
	for(int curr_idx=0;curr_idx<=curr_file_idx;curr_idx++){
		prefix = is_prefix(path,files_list[curr_idx]);
		if(prefix>=0){
			if(get_addr(path,files_list[curr_idx],prefix,ans)==0){
			printf("ans is %s\n",ans);
			filler(buffer,ans,NULL,0);}
		}
	}
	return 0;
}

static int do_read( const char *path, char *buffer, size_t size, off_t offset, struct fuse_file_info *fi )
{
	int file_idx = get_file_index( path );
	
	if ( file_idx == -1 )
		return -1;
	
	char *content = files_content[ file_idx ];
	
	memcpy( buffer, content + offset, size );
		
	return strlen( content ) - offset;
}

static int do_rmdir( const char *path, mode_t mode )
{	
	printf("remove path is %s\n",path);
	int t=0;
	int prefix1;
	int prefix2;
	for(int i=0;i<curr_dir_idx+1;i++){
		prefix1 = is_prefix(path,dir_list[i]);
		printf("rprefix is %d\n",prefix1);
		if(prefix1>0){
			
			path++;
			if(strcmp(path,dir_list[i])!=0){
				printf("directory cant remove \n");
				t++;
				return -1;}
			}
		prefix2 = is_prefix(path,files_list[i]);
		printf("rprefix is %d\n",prefix2);
		if(prefix2>0){
			
			path++;
			if(strcmp(path,files_list[i])!=0){
				printf("directory cant remove \n");
				t++;
				return -1;}
			}
		
		
	}
	if(t==0) //directory is empty
	{ 
		path++;
		remove_dir(path);
	}
	return 0;

}
static int do_unlink( const char *path,mode_t mode)
{
	printf("remove path is %s\n",path);
	int t=0;
	int prefix;
	for(int i=0;i<curr_file_idx+1;i++){
		prefix = is_prefix(path,files_list[i]);
		printf("rprefix is %d\n",prefix);
		if(prefix>0){
			
			path++;
			if(strcmp(path,files_list[i])!=0){
				printf("file cant remove \n");
				t++;
				return -1;}
			}
	}
	if(t==0) //file is empty
	{ 
		path++;
		remove_file(path);
	}
	return 0;
}
void remove_file( const char *file_name )
{	
	int index=0;
	int t = 0;
	printf("dir name is %s\n",file_name);
	for(int i=0;i<curr_file_idx+1;i++){
		printf("dirlist name is %s\n",files_list[i]);
		if(strcmp(files_list[i],file_name)==0){
			printf("i is %d\n",i);
			index = i;
		}
		
	}
	if(curr_file_idx==0){
		strcpy(files_list[index],"");
		strcpy(files_content[index],"");
	}
	for(int j=index;j<curr_file_idx;j++){
		strcpy(files_list[j],files_list[j+1]);
		strcpy(files_content[j],files_content[j+1]);
		files_time[j][0] = files_time[j+1][0];
		files_time[j][1] = files_time[j+1][1];
		files_time[j][2] = files_time[j+1][2];}
	
	printf("c is %d\n",curr_file_idx);
	curr_file_idx--;
}
void remove_dir( const char *dir_name )
{	
	int index=0;
	int t = 0;
	printf("dir name is %s\n",dir_name);
	for(int i=0;i<curr_dir_idx+1;i++){
		printf("dirlist name is %s\n",dir_list[i]);
		if(strcmp(dir_list[i],dir_name)==0){
			printf("i is %d\n",i);
			index = i;
		}
		
	}
	if(curr_dir_idx==0){
		strcpy(dir_list[index],"");
	}
	for(int j=index;j<curr_dir_idx;j++){
		strcpy(dir_list[j],dir_list[j+1]);
		dir_time[j][0] = dir_time[j+1][0];
		dir_time[j][1] = dir_time[j+1][1];
		dir_time[j][2] = dir_time[j+1][2];}
	
	printf("c is %d\n",curr_dir_idx);
	curr_dir_idx--;
}

static int do_mkdir( const char *path, mode_t mode )
{
	path++;
	add_dir( path );
	
	return 0;
}

static int do_mknod( const char *path, mode_t mode, dev_t rdev )
{
	path++;
	add_file( path );
	
	return 0;
}

static int do_write( const char *path, const char *buffer, size_t size, off_t offset, struct fuse_file_info *info )
{
	write_to_file( path, buffer );
	
	return size;
}

int do_utime(const char *path, struct utimbuf *buf)
{
	int cur_index;
	buf->actime = time(NULL);
	buf->modtime = time(NULL);
	if(is_dir(path))
	{
		cur_index = get_dir_index(path);
		dir_time[cur_index][0] = buf->actime;
		dir_time[cur_index][1] = buf->modtime;
		return 0;
	}
	else if (is_file(path))
	{
		cur_index = get_file_index(path);
		files_time[cur_index][0] = buf->actime;
		files_time[cur_index][1] = buf->modtime;
		return 0;
	}
	else return -1;
}	
static struct fuse_operations operations = {
    .getattr	= do_getattr,
    .readdir	= do_readdir,
    .read		= do_read,
    .mkdir		= do_mkdir,
    .mknod		= do_mknod,
    .write		= do_write,
    .rmdir		= do_rmdir,
    .unlink		= do_unlink,
    .utime		= do_utime,
};

int main( int argc, char *argv[] )
{
	return fuse_main( argc, argv, &operations, NULL );
}
```