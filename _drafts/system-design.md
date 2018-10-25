## 系统架构设计基础

### 1. 系统功能简介

设计一个短链接网站,网站可以允许用户发送一些文本数据并生成一个共享的数据链接给其他用户访问,除了业务功能外(生成,编辑和删除),还需要具有用户管理功能(匿名访问,登入用户可修改编辑提交的信息).系统可支持 1000 万用户规模, 每月约生成 1000 万条记录, 1 亿左右的读取操作(10:1 读写比例).

### 2. 相关资源需求

每条记录 1KB 大小,数据库不会直接存储用户的内容,而是将其保存在服务器本地位置(text 格式文件). 因此记录中最长的应该是保存路径长度,设置 255-512 字节左右. 根据需求每秒约 40 个读操作,4 个写操作, 网络带宽要求较低,但是每个月磁盘空间新增占用: 1KB\*1000 万=10GB

![服务架构图](https://camo.githubusercontent.com/2e0371e591b8311e36f5f5fa6ae18711f252b1f8/687474703a2f2f692e696d6775722e636f6d2f424b73426e6d472e706e67)

### 3.设计细节

数据库使用 MySQL 等关系数据库,仅仅用与保存元数据,所有用户提交的内容信息单独使用文件服务器进行存储.数据库仅保存存储文件的 url 链接.

文件存储可以使用 Amazon S3 或者开源的对象存储服务比如 Swift,Minio 等服务,这些服务都提供比较简单的接口用户管理文件的上传和使用.

当用户提交信息到服务器后, 调用 WriteAPI 将执行下列操作:

1. 生成一个唯一的短链接 ID(注意检查是否数据重复)
2. 使用 ID 保存用户的内容到对象存储服务.
3. 用户信息,插入文件的链接信息,时间等作为一条记录插入到数据库中
4. 返回用户新的访问 url

```sql
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

生成短链接的方式比较多,可以基于用户的 IP 和时间戳进行 md5 计算,然后对于 md5 编码进行 Base62 编码(区别与 Base64 中增加了+和/符号对于 url 不太友好, 最后根据需求进行截取操作,比如上面的 7 个字符长度.

```python
url = base62_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

当用户访问的时候,通过查询数据库既可以看到是否该文件存在,如果存在则返回用户文件存储的位置或者直接是文本文件.

业务数据的删除, 可以设置一个定期的任务,扫描数据库表标记为删除状态或者直接删除.并将对应的文件删除.

### 4.扩展系统规模

系统设计最终如下图所示, 增加负载均衡,CDN 及 Cache 层,数据库也采用读写分离的方式进行设计扩展.整个过程逐步完成,根据需求进行添加.并且过程中需要对于性能进行不断的测试,找到其中的性能瓶颈. 分析数据库独立出来,作为数据仓库使用,采集日志进行实时或者离线的数据分析.

对于缓存数据的使用,需要将查询比较多的增加到缓存中,用户访问的时候先去查询缓存,如果已经存在的话则直接返回,否则查询后端数据库(读数据库)

![](https://camo.githubusercontent.com/4aee2d26ebedc20e7fa07a2c30780e332fa29f2c/687474703a2f2f692e696d6775722e636f6d2f346564584730542e706e67)

#### 4.1 DNS 服务

DNS 可以提供域名到 IP 的解析,利用 DNS 层可以实现流量的切换,负载均衡以及 A/B 测试.可以实现基于延迟或者地理位置的流量调度.同时访问 DNS 服务会导致一定的网络延迟,以及管理上的复杂度. Cloudflare 或者 Amazon Route53 都可提供相应的域名调度管理服务.

#### 4.2 CDN 服务

使用 CDN 服务可以加速用户对于静态文件的访问速度,通过将这些静态文件分发到 CDN 服务商自己的数据中心,然后根据 DNS 将用户调度到地理位置上距离最佳的数据中心,现在一些 CDN 还可以提供[动态文件的处理](https://figshare.com/articles/Globally_distributed_content_delivery/6605972). 同时 CDN 还提供[两种工作模式](http://www.travelblogadvice.com/technical/the-differences-between-push-and-pull-cdns/):

- PushCDN: 用户将自身内容推送到 CDN,并重写自己系统资源访问 url 到 CDN. 同时需要用户自己管理存放在 CDN 上的资源创建,过期,更新等操作, 比较适合与小型网站(内容少,更新频率低).
- PullCDN: 按需拉取, 当有新的资源访问时,CDN 负责从用户服务器拉取新的内容, 初始访问较慢,但是一旦缓存生效后,服务后期的查询相对较快.

当流量较大的时候,CDN 的使用花销较高,内容管理上也需要考虑更新数据与过期数据方面.同时服务需要修改 url 指向才可以正常提供 CDN 的功能.