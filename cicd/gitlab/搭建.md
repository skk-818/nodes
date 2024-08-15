### docker-compose 安装 gitlab

首先访问 GitHub 地址 https://github.com/sameersbn/docker-gitlab/releases 下载最新版本的代码

目前我所遇到的最新版本是 16.8.2，下载 zip 包 docker-gitlab-16.8.2.zip 并进行解压缩，里面有 docker-compose.yml 文件

我们首先在自己的虚拟机上创建 /app/gitlab 目录，并创建相关的子目录，结构如下所示：

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214131951192-1523568016.png)

Gitlab 需要使用到 Redis 和 Postgresql 数据库，因此需要创建相应的子目录，存储其数据文件。

然后根据自己的需要，参考压缩包中的 docker-compose.yml 内容，复制过来修改一下，我经过测试，使用了最精简的内容：

```sh
version: '3.5'
 
services:
  redis:
    container_name: redis
    restart: always
    image: redis:6.2.6
    ports:
      - 6379:6379
    volumes:
      - /app/gitlab/redis-data:/data
    networks:
      - gitlab_net
 
  postgresql:
    container_name: postgresql
    restart: always
    image: sameersbn/postgresql:14-20230628
    volumes:
      - /app/gitlab/postgresql-data:/var/lib/postgresql
    ports:
      - 5432:5432
    environment:
      - DB_USER=gitlab
      - DB_PASS=gitlab123
      - DB_NAME=gitlabdb
      - DB_EXTENSION=pg_trgm,btree_gist
    networks:
      - gitlab_net
 
  gitlab:
    container_name: gitlab
    restart: always
    image: sameersbn/gitlab:16.8.2
    depends_on:
      - redis
      - postgresql
    ports:
      - "8080:80"
      - "8022:22"
    volumes:
      - /app/gitlab/gitlab-data:/home/git/data
    networks:
      - gitlab_net
    environment:
      - DEBUG=false
      # 配置连接 postgresql 的信息
      - DB_ADAPTER=postgresql
      - DB_HOST=postgresql
      - DB_PORT=5432
      - DB_USER=gitlab
      - DB_PASS=gitlab123
      - DB_NAME=gitlabdb
      # 配置连接 redis 的信息
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      # 配置时区
      - TZ=Asia/Shanghai
      - GITLAB_TIMEZONE=Asia/Shanghai
      # 禁用 https 以及自签名功能
      - GITLAB_HTTPS=false
      - SSL_SELF_SIGNED=false
      # 配置服务地址（由于在容器内，配置localhost即可），端口等信息
      - GITLAB_HOST=localhost
      - GITLAB_PORT=8080
      - GITLAB_SSH_PORT=8022
      # 以下 3 个配置项必须要有，否则无法启动 gitlab 服务
      - GITLAB_SECRETS_DB_KEY_BASE=long-and-random-alphanumeric-string
      - GITLAB_SECRETS_SECRET_KEY_BASE=long-and-random-alphanumeric-string
      - GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string
      # 禁用通知功能
      - GITLAB_NOTIFY_ON_BROKEN_BUILDS=false
      - GITLAB_NOTIFY_PUSHER=false
      # 禁用自动备份功能
      - GITLAB_BACKUP_SCHEDULE=disable
      # 禁用 smtp 功能
      - SMTP_ENABLED=false
      # 禁用 imap 功能
      - IMAP_ENABLED=false
      # 禁用 oAuth 认证
      - OAUTH_ENABLED=false
 
networks:
  gitlab_net:
    driver: bridge
```

有关 Gitlab 镜像中 environment 下面可以配置的相关参数及其含义，可以参考上面给出的 GitHub 访问地址的首页。

我在实际测试过程中发现：有些配置项使用的话，可能会报错；有的使用后不起作用。可能是我用法不当，也可能是新版本存在 bug，有待完善。

不过可以肯定的是：上面贴出来的 docker-compose.yml 内容是我踩坑摸索出来的精简内容，绝对可以放心使用。

然后运行 `docker-compose up -d` 命令启动服务，需要注意的时，Gitlab 服务启动速度比较慢，可能需要 2 分钟。

可以使用 `docker-compose logs -f gitlab` 实时查看 gitlab 服务的启动日志，过了几分钟等待日志不再滚动后就启动成功了。

> 访问 地址:8080

由于我采用的是 8080 端口，因此访问 `http://192.168.136.128:8080` 即可

默认的超级管理员账号是 `root`，邮箱是 `admin@example.com` ，密码是 `5iveL!fe`，不过首次访问会提示修改默认登录密码：

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132010512-934424095.png)

修改完密码后，就可以登录了，使用默认用户名是 `root` 或邮箱 `admin@example.com` ，结合修改后的密码都可以登录。

如果觉的用户名和邮箱名称不合适的话，登录进去之后，也是可以修改的，稍后会介绍如何修改。

登录进去之后，默认界面是英文界面，我们可以修改成中文界面，具体做法如下图：

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132017676-880581775.png)

在右侧页面往下拉，找到 Localization ，在 Language 下拉列表中选择简体中文，然后点击底部的 Save changes 保存。

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132027548-1282597782.png)

然后刷新一下页面，整个界面就变成中文界面了，这样就比较方便学习和研究 Gitlab 的相关功能了。

我们肯定会在 Gitlab 上创建其它用户账号，为了能够让新创建的账号，首次登录系统也展示中文界面，可以进行如下操作：

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132036991-704481102.png)

由于当前是超级管理员用户，因此在左侧菜单下面有【管理中心】按钮，点击进入管理中心后，找到本地化，下拉列也选择【简体中文】即可。

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132045105-112589939.png)

如果想要修改用户名和邮箱，使用新的用户名和邮箱进行登录，可以在左侧菜单【概览】下面选择【用户】，编辑用户的信息即可。

![image](https://img2024.cnblogs.com/blog/2502715/202402/2502715-20240214132052823-1018294045.png)



以上就是 docker-compose 部署 Gitlab 的介绍，不熟悉的小伙伴们，赶紧自己部署一下，仔细研究和体验一下 Gitlab 吧。

具体如何使用，这里就不介绍了，可以自己在 Gitlab 上创建项目，使用 IDEA 或 Visual Studio 进行代码上传和开发交互。

都已经变成中文界面了，学习和研究没有什么难度，尤其是开发人员，必须得熟练掌握 Gitlab ，不然没法在公司里面混的。











