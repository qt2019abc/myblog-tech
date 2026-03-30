
## 基础知识

只有RUN，COPY，ADD会创建layers。其他命令会创建临时的中间镜像，但是不会增加构建的镜像大小。

构建映像时，Docker按顺序逐步执行Dockerfile中的每个指令。在检查每条指令时，Docker会在其缓存中查找可重用的现有镜像，而不是创建新的镜像。遵循的基本规则概述如下：

- 从已在缓存中的父映像开始，将下一条指令与从该基础镜像派生的所有子镜像进行比较，以查看是否其中一个是使用完全相同的指令构建的。如果不是，则缓存无效。

- 对于ADD和COPY指令，将检查镜像中文件的内容，计算每个文件的checksum。

- 除了ADD和COPY命令之外，缓存检查不会查看容器中的文件来确定缓存是否匹配，仅使用命令字符串本身来查找匹配项。

- 缓存无效后，所有后续Dockerfile命令都会生成新映像，并且不使用缓存。


## 注意事项示例

### 相同文件修改集中在同一层中

**错误的例子**

```
RUN curl -O http://site/bigfile
RUN chown root:root bigfile
```

**正确的例子**

```
RUN curl -O http://site/bigfile && \\
chown root:root bigfile
```

根据docker构建镜像的原理，例1会比例2多一层来保存bigfile，最终结果就是例1得到的镜像和例2相比多出一个bigfile的大小


### 不要用ADD添加安装包

**错误的例子**

```
ADD install /
RUN /install && \\
rm -rf /install
```

**正确的例子**

```
RUN curl -O http://site/install && \\
./install && \\
rm -rf install
```

docker构建时会扫描工作目录下所有文件(除非使用.dockerignore排除)并计算sha256，影响构建时间

另外在安装文件后可以删除不再需要的文件，而不必在镜像中添加另一层


### 排序多行参数

**错误的例子**


```
RUN yum install unzip git nginx supervisor
```

**正确的例子**

```
RUN yum install \\
git \\
nginx \\
supervisor \\
unzip
```

尽可能通过字母数字排序多行参数来简化以后的更改，这样做的好处是更易读，避免参数重复，同时在git中快速比较差异

### 不要安装不必要的软件包

避免RUN apt-get upgrade

在多服务的项目中使用相同的基础镜像
