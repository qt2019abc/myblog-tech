
# 本地pypi私服搭建方法

## 背景
PyPI(Python Package Index，https://pypi.org/)) 是 Python 官方的第三方库的仓库，所有人都可以下载第三方库或上传自己开发的库到 PyPI。 通常我们使用 pip 安装 Python 包，默认就是从 https://pypi.python.org/pypi (https://pypi.org/) 上安装。当然，我们也可以通过配置 pip.conf 或者使用命令行从指定的 PyPI 源安装


有些是公司内部的项目，不方便放到外网上去，这个时候我们就要搭建自己的内网 PyPI 源服务器，需要安全并且拥有同样的舒适体验。

## 方案比较

Python 官方列出了几个成熟的实现方案，结合企业内网使用场景（安全、便捷、低运维成本），对主流方案进行详细对比，便于选择最适合的搭建方式。各方案核心特性、优缺点及适用场景如下，最终结合需求选定 pypiserver 方案。

1. pypiserver（最终选择）

项目地址：https://github.com/pypiserver/pypiserver，截止2020-03，Star数 900，最近更新 2个月前，最新版本 1.3.2。它是一款轻量级、高度可定制且易于部署的私有 PyPI 兼容服务器，基于 Bottle Web 框架构建，核心目标是为 Python 开发团队提供极简但功能完备的内部软件包分发与托管方案，设计理念是“简单即安全”。

核心特性：原生支持 pip 安装、上传操作，无需复杂配置；仅需指定目录即可作为包存储路径，自动解析标准 Python 分发格式并生成索引页；支持 HTTP Basic 认证（结合 .htpasswd 文件），可配合 Nginx 实现 HTTPS 加密；支持 twine、setup.py 等多种上传方式，也可通过 scp、rsync 直接写入包文件；部署简单，零外部重型框架依赖，运维成本极低。

优点：体积小、部署快，一行命令即可启动服务；学习成本低，无需专业运维知识；完全兼容 pip 工具，使用体验与官方 PyPI 一致；支持 Docker 快速部署，备份、迁移仅需操作包存储目录；轻量无冗余功能，资源占用少，适合内网小型团队使用。

缺点：不支持原生代理公共 PyPI 源（需额外配置反向代理实现）；无内置包上传验证和复杂权限管理；高可用能力较弱，以单节点部署为主；无可视化管理界面，包管理需通过文件系统或命令行操作。

适用场景：中小型企业内网、小型开发团队（10人以下）、对运维成本敏感、仅需实现内部包托管与分发的场景，尤其适合仅需“简单可用、安全可控”的需求场景。

2. DevPI

一款功能更全面的开源 PyPI 私服工具，主打“缓存+私有包托管”双重能力，是 pypiserver 的进阶替代方案，适合中等规模团队使用。

核心特性：内置公共 PyPI 源缓存功能，可减少外网依赖；支持自定义用户系统和 Token 认证，权限管理更精细；提供完整的用户/项目命名空间，支持多项目隔离；支持预发布测试和包版本回滚，适配 CI/CD 流水线；可配合 WSGI 容器启用 SSL/TLS 加密。

优点：功能完善，兼顾缓存与私有包管理；权限控制更灵活，适合多团队协作；日志记录完整，支持插件扩展，可满足中等规模团队的进阶需求。

缺点：部署和配置比 pypiserver 复杂，学习成本较高；运维成本高于轻量级方案，需要投入更多精力维护用户和权限；资源占用比 pypiserver 高，不适合极简部署场景。

适用场景：中型开发团队（10-50人）、需要公共包缓存、多项目隔离和精细化权限管理的场景。

3. Sonatype Nexus（企业级）

一款企业级多语言制品库，并非专为 Python 设计，可同时管理 PyPI、Maven、npm、Docker 等多种格式的制品，适合大型企业统一管理各类制品。

核心特性：支持本地索引和公共 PyPI 源代理，缓存能力强大；提供 LDAP、SAML、OAuth 等多种认证方式，权限管理精细到项目级别；支持集群部署，高可用能力强；内置监控指标，可集成 Prometheus 等监控工具；内置 HTTPS 支持，安全防护能力完善。

优点：功能强大、安全性高，适合企业级大规模部署；支持多语言制品统一管理，减少运维成本；具备强大的策略引擎，可结合 CI/CD 实现包上传验证和安全扫描；跨数据中心复制能力，适合大型分布式团队。

缺点：部署复杂，对服务器配置要求高；学习成本极高，需要专业运维人员维护；免费版功能有限，商业版成本较高；冗余功能多，对于仅需 PyPI 私服的小型团队来说过于笨重。

适用场景：大型企业（50人以上）、需要统一管理多语言制品、对安全性和高可用要求极高、有专业运维团队的场景。

4. JFrog Artifactory（商业主导）

商业主导的企业级制品管理平台，功能与 Nexus 类似，定位更高端，主打企业级安全治理和全生命周期管理。

核心特性：支持多语言制品管理，兼容 PyPI 规范；提供 LDAP、OAuth、SSO 等认证方式，细粒度权限控制；强大的策略引擎，可实现包黑白名单、安全扫描等功能；跨数据中心复制，高可用能力极强；集成 Let’s Encrypt 实现 HTTPS 自动配置。

优点：安全治理能力强，适合合规要求高的企业；自动化集成能力强，无缝适配 CI/CD 流水线；监控体系完善，可集成 Grafana 等工具实现可视化监控。

缺点：商业版收费较高，成本投入大；部署和维护复杂，对运维能力要求高；功能冗余，不适合小型团队和极简需求。

适用场景：大型企业、对制品安全和合规要求极高、有充足预算和专业运维团队的场景。

5. Bandersnatch（全量镜像）

官方推荐的 PyPI 全量镜像工具，核心功能是同步官方 PyPI 所有包，适合需要内网全量包访问的场景。

核心特性：可全量同步官方 PyPI 仓库，支持增量同步；部署后可作为内网全量 PyPI 源，无需手动上传公共包；支持配置同步规则，可筛选需要同步的包。

优点：全量同步官方包，满足内网无外网访问时的包安装需求；配置简单，同步规则灵活。

缺点：需要大量存储空间（需预留 TB 级空间）；同步耗时久，对网络带宽要求高；仅支持镜像功能，不适合托管内部私有包；无权限管理，安全性较弱。

适用场景：需要内网全量访问 PyPI 公共包、无内部私有包托管需求、有充足存储空间和带宽的场景。

方案选择结论

结合内网 PyPI 私服的核心需求（安全、便捷、低运维成本、适配内部私有包托管），对比上述方案后，最终选择 pypiserver，核心原因如下：

1.  契合核心需求：无需复杂功能，仅需实现内部私有包的上传、下载，pypiserver 可完美满足，且兼容 pip 工具，使用体验与官方一致；

2.  部署运维简单：一行命令即可启动，无需专业运维知识，备份、迁移仅需操作包存储目录，适合中小型团队快速落地；

3.  轻量高效：资源占用少，无需复杂依赖，可快速部署在普通内网服务器上，适配内网环境；

4.  安全可控：支持 HTTP Basic 认证，可配合 Nginx 实现 HTTPS 加密，满足企业内网的安全需求；

5.  成本可控：开源免费，无额外成本投入，无需为冗余功能付出运维和服务器资源成本。

综上，pypiserver 是中小型企业内网、小型开发团队搭建 PyPI 私服的最优选择，兼顾简单性、实用性和安全性。

## pypiserver 构建

项目地址： https://github.com/pypiserver/pypiserver 

使用文档 https://github.com/pypiserver/pypiserver#quickstart-installation-and-usage

Docker 方式安装
docker pull pypiserver/pypiserver:v1.3.2
docker run -p 8888:8080 --name mypypi -v /common/python/packages:/data/packages --privileged=true --restart=always pypiserver/pypiserver:v1.3.2
 
报错信息 [root@localhost ~]# docker run -p 8888:8080 -v /common/python/packages:/data/packages pypiserver/pypiserver:v1.3.2 Error: while trying to list root(/data/packages): [Errno 13] Permission denied: '/data/packages' [root@localhost ~]# docker run -p 8888:8080 pypiserver/pypiserver:v1.3.2

解决办法，添加 --privileged=true
 
### 验证pypiserver是否启动成功：

http://192.168.xxx.xxx:8888/
Welcome to pypiserver! This is a PyPI compatible package index serving 0 packages.
 
### 打包上传到Pypi测试站点

发布方法有好几种。
 
1.把whl包拷贝到服务器上对应的packages目录

2.代码构建时上传 pypi。 
1.  准备工作：确保本地环境安装setuptools、wheel、twine工具（可通过pip install setuptools wheel twine命令安装）；
2.  配置setup.py：在Python项目根目录创建setup.py文件，配置项目名称、版本、作者、依赖等核心信息，定义包的基础属性；

3.  构建包文件：执行命令python setup.py sdist bdist_wheel，生成源码包（.tar.gz）和wheel包（.whl），生成的包文件会存放在dist目录下；

4.  上传到私服：使用twine工具执行上传命令，指定私服地址、用户名和密码，命令格式为twine upload --repository-url 私服地址 dist/*，完成包的上传；

5.  验证上传：上传完成后，可通过浏览器访问pypiserver私服地址，查看包是否成功显示，也可通过pip install 包名 --index-url 私服地址命令测试安装。

参考 https://www.cnblogs.com/lazyboy/p/3830104.html
 
### 使用pypi私服库

pip3 install  mypackage -i http://192.168.xxx.xxx:8888/simple