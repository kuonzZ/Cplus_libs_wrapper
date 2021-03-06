## 1.在Windows7下创建一个FTP服务器
在Windows7下创建FTP服务器，例如FTP服务器地址为：ftp://145.100.244.111/uploadDir/，远程访问的用户名和密码分别为：username和123456，
可参考[win7下如何建立ftp服务器](https://jingyan.baidu.com/article/574c5219d466c36c8d9dc138.html)
## 2.使用libcurl库的一个FTP上传示例程序[ftpupload.c](https://curl.haxx.se/libcurl/c/ftpupload.html)，在此基础上修改FTP地址和远程服务器路径，用户名和密码，
修改后的代码如下：
```cpp
/***************************************************************************
 *                                  _   _ ____  _
 *  Project                     ___| | | |  _ \| |
 *                             / __| | | | |_) | |
 *                            | (__| |_| |  _ <| |___
 *                             \___|\___/|_| \_\_____|
 *
 * Copyright (C) 1998 - 2017, Daniel Stenberg, <daniel@haxx.se>, et al.
 *
 * This software is licensed as described in the file COPYING, which
 * you should have received as part of this distribution. The terms
 * are also available at https://curl.haxx.se/docs/copyright.html.
 *
 * You may opt to use, copy, modify, merge, publish, distribute and/or sell
 * copies of the Software, and permit persons to whom the Software is
 * furnished to do so, under the terms of the COPYING file.
 *
 * This software is distributed on an "AS IS" basis, WITHOUT WARRANTY OF ANY
 * KIND, either express or implied.
 *
 ***************************************************************************/
#include <stdio.h>
#include <string.h>

#include <curl/curl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#ifdef WIN32
#include <io.h>
#else
#include <unistd.h>
#endif

/* <DESC>
 * Performs an FTP upload and renames the file just after a successful
 * transfer.
 * </DESC>
 */

#define LOCAL_FILE      "/tmp/uploadthis.txt"
#define UPLOAD_FILE_AS  "while-uploading.txt"
#define REMOTE_URL      "ftp://145.100.244.111/uploadDir/"  UPLOAD_FILE_AS  // 修改自己在Windows7下创建的FTP服务器地址
#define RENAME_FILE_TO  "renamed-and-fine.txt"

/* NOTE: if you want this example to work on Windows with libcurl as a
   DLL, you MUST also provide a read callback with CURLOPT_READFUNCTION.
   Failing to do so will give you a crash since a DLL may not use the
   variable's memory when passed in to it from an app like this. */
static size_t read_callback(void *ptr, size_t size, size_t nmemb, void *stream)
{
  curl_off_t nread;
  /* in real-world cases, this would probably get this data differently
     as this fread() stuff is exactly what the library already would do
     by default internally */
  size_t retcode = fread((FILE*)ptr, size, nmemb, stream);

  nread = (curl_off_t)retcode;

  fprintf(stderr, "*** We read %" CURL_FORMAT_CURL_OFF_T
          " bytes from file\n", nread);
  return retcode;
}

int main(void)
{
  CURL *curl;
  CURLcode res;
  FILE *hd_src;
  struct stat file_info;
  curl_off_t fsize;

  struct curl_slist *headerlist = NULL;
  static const char buf_1 [] = "RNFR " UPLOAD_FILE_AS;
  static const char buf_2 [] = "RNTO " RENAME_FILE_TO;

  /* get the file size of the local file */
  if(stat(LOCAL_FILE, &file_info)) {
    printf("Couldn't open '%s': %s\n", LOCAL_FILE, strerror(errno));
    return 1;
  }
  fsize = (curl_off_t)file_info.st_size;

  printf("Local file size: %" CURL_FORMAT_CURL_OFF_T " bytes.\n", fsize);

  /* get a FILE * of the same file */
  hd_src = fopen(LOCAL_FILE, "rb");

  /* In windows, this will init the winsock stuff */
  curl_global_init(CURL_GLOBAL_ALL);

  /* get a curl handle */
  curl = curl_easy_init();
  if(curl) {
    /* build a list of commands to pass to libcurl */
    headerlist = curl_slist_append(headerlist, buf_1);
    headerlist = curl_slist_append(headerlist, buf_2);

    /* we want to use our own read function */
    curl_easy_setopt(curl, CURLOPT_READFUNCTION, read_callback);

    /* enable uploading */
    curl_easy_setopt(curl, CURLOPT_UPLOAD, 1L);

    /* specify target */
    curl_easy_setopt(curl, CURLOPT_URL, REMOTE_URL);

    /* pass in that last of FTP commands to run after the transfer */
    curl_easy_setopt(curl, CURLOPT_POSTQUOTE, headerlist);

    /* now specify which file to upload */
    curl_easy_setopt(curl, CURLOPT_READDATA, hd_src);

    /* Set the size of the file to upload (optional).  If you give a *_LARGE
       option you MUST make sure that the type of the passed-in argument is a
       curl_off_t. If you use CURLOPT_INFILESIZE (without _LARGE) you must
       make sure that to pass in a type 'long' argument. */
    curl_easy_setopt(curl, CURLOPT_INFILESIZE_LARGE,
                     (curl_off_t)fsize);

	curl_easy_setopt(curl, CURLOPT_USERPWD, "username:123456");   // 设置远程访问的服务器用户名和密码

    /* Now run off and do what you've been told! */
    res = curl_easy_perform(curl);
    /* Check for errors */
    if(res != CURLE_OK)
      fprintf(stderr, "curl_easy_perform() failed: %s\n",
              curl_easy_strerror(res));

    /* clean up the FTP commands list */
    curl_slist_free_all(headerlist);

    /* always cleanup */
    curl_easy_cleanup(curl);
  }
  fclose(hd_src); /* close the local file */

  curl_global_cleanup();
  return 0;
}
```
## 3.在Ubuntu18.10下编译安装libcurl，安装到/usr/local目录
```shell
sudo git clone https://github.com/curl/curl.git
cd curl
./configure --prefix=/usr/local/
make
make install
```
关于如何在Linux下编译安装libcurl,可参考[how to install curl and libcurl](https://curl.haxx.se/docs/install.html)
## 4.打开一个终端，进入源文件所在目录，编译ftpupload.c源代码，生成可执行文件ftpupload
```shell
root@ubuntu:/home/username/Programming/GithubProjects/curl/docs/examples# gcc ftpupload.c -o ftpupload -I/usr/local/curl -L/usr/local/lib -lcurl
```
## 5.运行ftpupload，将/tmp/uploadthis.txt文件上传到FTP服务器所在目录ftp://145.100.244.111/uploadDir/，并把上传的文件命名为：renamed-and-fine.txt
```shell
root@ubuntu:/home/havealex/Programming/GithubProjects/curl/docs/examples# ./ftpupload
Local file size: 307476 bytes.
*** We read 65536 bytes from file
*** We read 65536 bytes from file
*** We read 65536 bytes from file
*** We read 65536 bytes from file
*** We read 45332 bytes from file
