# 1. 运行&使用
本项目基于青岛大学OJ，原项目见
https://github.com/QingdaoU/OnlineJudge
https://github.com/QingdaoU/JudgeServer
文档见
https://docs.onlinejudge.me/#/judgeserver/api
## 1.1 运行
以下均在linux或类linux环境下运行
需要：
* docker
* docker-compose

添加docker镜像：
http://docker.mirrors.ustc.edu.cn
http://hub-mirror.c.163.com
![image](https://i.loli.net/2019/05/25/5ce906555fd5830722.png)


```
git clone git@github.com:nkucs/JudgeServer.git
cd JudgeServer
docker-compose up
```

## 1.2 测试&使用
本项目对外接口有二：
1. 上传测试用例
2. 请求判题
这里将会以A+B题目为例(即读入两个数，输出它们的和)

### 测试工具: postman(其他也可)
https://www.getpostman.com/downloads/

### 1.2.1 上传测试用例
编写一个文件1.in，内容如下：
```
1 1
```
编写一个文件1.out，内容如下：
```
2
```
若有多个测例，编号依次为2, 3, 4, 5...
输入数据后缀名.in，期望输出后缀名.out
将所有.in文件和.out文件压缩为压缩包，名字不限

使用postman上传测试用例，(点击"send")
![image](https://i.loli.net/2019/05/25/5ce909ea5a0c386320.png)

期望响应如下：
![image](https://i.loli.net/2019/05/25/5ce90a3e8aa2b25631.png)

记住返回的"id"

### 1.2.2 请求判题
在header中添加`X-Judge-Server-Token`，内容为`b824cecedb22b06c3883b1f1dd9dd3150608fc24f8d0c16b0f85af8c8c761667`("CHANGE_THIS"的sha256摘要，注意摘要中是小写字母，有的网站生成会生成大写字母)
![image](https://i.loli.net/2019/05/25/5ce90d678a0f261977.png)

在body格式选择raw， JSON(application/json)
![image](https://i.loli.net/2019/05/25/5ce90dcc0dbf280286.png)
内容如下:（此处的"test_case_id"即为在1.2.1上传测试用例中记录的"id"）
```
{
	"src": "#include <cstdio> \n int main() { printf(\"2\\n\"); return 0; } ",
	"language_config": {
	    "compile": {
	        "src_name": "main.cpp",
	        "exe_name": "main",
	        "max_cpu_time": 3000,
	        "max_real_time": 5000,
	        "max_memory": 134217728,
	        "compile_command": "/usr/bin/g++ -DONLINE_JUDGE -O2 -w -fmax-errors=3 -std=c++11 {src_path} -lm -o {exe_path}"
	    },
	    "run": {
	        "command": "{exe_path}",
	        "seccomp_rule": "c_cpp",
	        "env": ["LANG=en_US.UTF-8", "LANGUAGE=en_US:en", "LC_ALL=en_US.UTF-8"]
	    }
	},
	"max_cpu_time": 1000,
	"max_memory": 1048576,
	"test_case_id": "14caa51efd3828c3b6f372d503498186",
	"output": true
}
```
期待运行结果如下：
![image](https://i.loli.net/2019/05/25/5ce90e300d97e14957.png)

# 2. 部分原理
判题的模块是不加改动的[JudgeServer](https://github.com/QingdaoU/JudgeServer)(flask框架)，上传测试用例的模块是精简版的[OnlineJudge](https://github.com/QingdaoU/OnlineJudge)(Django框架)，两个模块通过Docker结合到一起。
## 2.1 Docker
docker是一种类似虚拟机的技术，使程序运行在定制的环境（操作系统、编译器版本、应用程序...）中。
本节对docker的描述可能有误，请自行学习、勘误。
自行安装docker及docker-compose，docker-compose推荐按照https://docs.docker.com/compose/install/ 安装，使用apt安装可能会出问题。
## image&container
image: 镜像
container: 容器
一个定制的环境对应一个image，image是静态的。
有程序要在这个环境下运行时，就会根据这个[镜像]创造一个[容器]，此后对这个[容器]做出的改动局限于本[容器]，不会影响到[镜像]。
## docker-compose
docker-compose使用一个yml/yaml文件，把多个[镜像]合成一个service，协作提供需要的服务。
本项目docker-compose.yaml文件如下:
（基于[OnlineJudgeDeploy](https://github.com/QingdaoU/OnlineJudgeDeploy)修改）
```
# attemp1
version: "3"
services:

  t2server:
    image: registry.cn-hangzhou.aliyuncs.com/onlinejudge/judge_server
    container_name: t2s
    restart: always
    read_only: true
    cap_drop:
      - SETPCAP
      - MKNOD
      - NET_BIND_SERVICE
      - SYS_CHROOT
      - SETFCAP
      - FSETID
    tmpfs:
      - /tmp
    volumes:
      - ./data/backend/test_case:/test_case:ro
      - ./data/judge_server/log:/log
      - ./data/judge_server/run:/judger
    environment:
      - SERVICE_URL=http://judge-server:8080
      - BACKEND_URL=http://oj-backend:8000/api/judge_server_heartbeat/
      - TOKEN=CHANGE_THIS
      # - judger_debug=1
    ports:
      - "8080:8080"
  
  t2backend:
    build: ./oj
    container_name: t2b
    restart: always
    depends_on:
      - t2server
    volumes:
      - ./oj:/app
      - ./data/backend:/data
    environment:
      - JUDGE_SERVER_TOKEN=CHANGE_THIS
    ports:
      - "8100:8100"
```


其中`image`指定已上传的镜像，`build`会让docker-compose去指定的文件夹下寻找`Dockerfile`进行镜像构建，两者都用于指定镜像。第一个镜像是JudgeServer，完成判题工作；第二个镜像则负责存储测例。
`ports`指定端口映射
`volumes`指定容器和宿主机之间的文件关联，格式是`宿主机目录:容器内目录`或`宿主机目录:容器内目录:ro`，`ro`指read only。在本项目的docker-compose.yaml中，第一个镜像和第二个镜像都使用了./data/backend/test_case，这使得第二个镜像存储的测例文件可以被第一个镜像读取到。

## 2.2 Django
Django是一个python服务器框架，请学习相关内容，掌握url和view。

## 2.3 API相关细节
JudgeServer[文档](https://docs.onlinejudge.me/#/judgeserver/api)
指定6个参数，`src`, `max_cpu_time`, `max_memory`, `output`比较容易理解。
`test_case_id`是上传测例返回的id，依赖于第二个镜像。
`language_config`，按照文档所说，应当参考https://github.com/QingdaoU/JudgeServer/blob/master/client/Python/languages.py