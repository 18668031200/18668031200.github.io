---
title: fastdfs部署及应用
author: ygdxd
type: 原创
date: 2021-07-29 23:07:10
tags: fastdfs
categories: fastdfs
---

简介
--------------
FastDFS是一个开源的轻量级分布式文件系统,为互联网应用量身定做,简单、灵活、高效,采用C语言开发,由阿里巴巴开发并开源.
FastDFS对文件进行管理,功能包括：文件存储、文件同步、文件访问(文件上传、文件下载、文件删除)等,解决了大容量文件存储的问题,特别适合以文件为载体的在线服务,如相册网站、文档网站、图片网站、视频网站等等.
FastDFS充分考虑了冗余备份、线性扩容等机制,并注重高可用、高性能等指标,使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务.

整体架构
-------------
FastDFS文件系统由两大部分构成,一个是客户端,一个是服务端.
客户端通常就是我们的程序,我们可以使用第三方的扩展包去连接fastdfs.

跟踪器(tracker)主要做调度工作,在内存中记录集群中存储节点storage的状态信息,是前端Client和后端存储节点storage的枢纽.
因为相关信息全部在内存中,Tracker server的性能非常高,一个较大的集群(比如上百个group)中有3台就足够了.
每个 storage 在启动后会连接 Tracker，告知自己所属 group 等信息，并保持周期性心跳。storage以group为单位,每个group里可以有多个storage单位.

存储节点(storage)用于存储文件,包括文件和文件属性(meta data)都保存到存储服务器磁盘上,完成文件管理的所有功能:文件存储、文件同步和提供文件访问等.
1.下载镜像
-------------
执行命令docker search fastdfs

```$shell
NAME                           DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
season/fastdfs                 FastDFS                                         77
ygqygq2/fastdfs-nginx          整合了nginx的fastdfs                                27                                      [OK]
luhuiguo/fastdfs               FastDFS is an open source high performance d…   25                                      [OK]
morunchang/fastdfs             A FastDFS image                                 20
delron/fastdfs                                                                 13
qbanxiaoli/fastdfs             FastDFS+FastDHT(单机+集群版)                         12                                      [OK]
```

这里根据star数选取最高的 season/fastdfs,打开docker hub搜索这个镜像https://hub.docker.com/r/season/fastdfs
里面有具体使用这个镜像的方法教程,这里点击tag页选取最新的版本1.2.
使用docker pull season/fastdfs:1.2等待docker 帮我们自动下载好镜像.

2. 启动tracker和storage服务
---------------

首先创建对应的文件夹 
```
mkdir -p /home/fastdfs
```

启动tracker 

```
docker run -id --name tracker \
-p 22122:22122 \
--restart=always --net host \
-v /home/ucmed/fastdfs/tracker/data:/fastdfs/tracker/data \
season/fastdfs:1.2 tracker
```

启动storage服务

```
docker run -id --name storage \
--restart=always --net host \
-v /home/fastdfs/data/storage:/fastdfs/store_path \
-e TRACKER_SERVER="192.168.3.134:22122" \
season/fastdfs:1.2 storage
```

这里笔者曾使用TRACKER_SERVER="127.0.0.1" 但是启动服务使用命令docker logs发现程序并为成功启动.
提示TRACKER_SERVER不能设置成127.0.0.1.

搭建完2个容器后进入tracker 进行测试
```
docker exec -it tracker bash
cd /etc/fdfs/
ls
cat client.conf
```
输出的 client.conf 都是默认配置，我们可以找到其中的 track_server 将默认的地址改成我们设置的storage地址
执行命令
```
docker cp trakcer:/etc/fdfs/client.conf /home/fastdfs/
vim client.conf
docker cp /home/fastdfs/client.conf tracker:/etc/fdfs 
```
修改其中的track_server地址

执行命令
```
docker exec -it tracker bash
fdfs_monitor /etc/fdfs/client.conf
```
控制台会输出DEBUG信息
```
DEBUG - base_path=/fastdfs/client, connect_timeout=30, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0
```
表示storage已经成功的连上了我们的tracker.
之后尝试上传文件
```
#fdfs_upload_file /etc/fdfs/client.conf **.txt
group1/M00/00/00/wKgDhmECa5GAGpHOAAAABncc3SA469.txt
```
这样fastdfs就搭建完成了


3.nginx 服务
-------------

如果想要访问我们上传的文件 就需要启动nginx modual.拷贝出nginx配置文件 修改其中的location节点

```
docker cp storage:/etc/nginx/conf/nginx.conf /home/fastdfs/nginx/

docker run -id --name fastdfs_nginx \
--restart=always \
-v /home/fastdfs/data/storage:/fastdfs/store_path \
-v /home/fastdfs/nginx.conf:/etc/nginx/conf/nginx.conf \
-p 80:80 \
-e TRACKER_SERVER=192.168.3.134:22122 \
season/fastdfs:1.2 nginx
```

这时候打开浏览器访问http://192.168.3.134/group1/M00/00/00/wKgDhmECa5GAGpHOAAAABncc3SA469.txt
出现刚刚上传的文件信息.


4.spring boo 集成
-------------

增加maven 依赖
{% codeblock pom.xml lang:java %}
<dependency>
            <groupId>com.github.tobato</groupId>
            <artifactId>fastdfs-client</artifactId>
            <version>1.27.2</version>
        </dependency>
{% endcodeblock %}

在Maven当中配置依赖以后，SpringBoot项目将会自动导入FastDFS依赖
添加config

{% codeblock ComponetImport.java lang:java %}
/**
 * 导入FastDFS-Client组件
 * 
 * @author tobato
 *
 */
@Configuration
@Import(FdfsClientConfig.class)
// 解决jmx重复注册bean的问题
@EnableMBeanExport(registration = RegistrationPolicy.IGNORE_EXISTING)
public class ComponetImport {
    // 导入依赖组件
}
{% endcodeblock %}

在application.yml当中配置Fdfs相关参数

{% codeblock application.yml lang:java %}
fdfs:
  connect-timeout: 500
  so-timeout: 5000
  thumb-image:
    width: 200
    height: 200
  tracker-list:
    - 192.168.3.134:22122
  pool:
    max-total: 3
    max-total-per-key: 5
    max-idle-per-key: 3
{% endcodeblock %}

编写上传接口

{% codeblock FastDfsController.java lang:java %}
@RequestMapping("/dfs")
@RestController
public class FastDfsController {
private final FastFileStorageClient fastFileStorageClient;

    public FastDfsController(FastFileStorageClient fastFileStorageClient) {
        this.fastFileStorageClient = fastFileStorageClient;
    }
    
    @PostMapping(value = "/upload",consumes = {MediaType.MULTIPART_FORM_DATA_VALUE, MediaType.IMAGE_PNG_VALUE})
        public ResponseEntity<?> upload(MultipartFile multipartFile) throws IOException {
    
            String extension = FilenameUtils.getExtension(multipartFile.getOriginalFilename());
            FastFile fastFile = new FastFile(multipartFile.getInputStream(), multipartFile.getSize(), extension, new HashSet<>());
            StorePath storePath = fastFileStorageClient.uploadFile(fastFile);
            return ResponseEntity.ok(storePath.getFullPath());
        }
}
{% endcodeblock %}

调用我们的接口后便会返回对应的路径.







