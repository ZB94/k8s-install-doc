参考: [鸟哥的 Linux 私房菜 第十三章 NFS服务器](http://cn.linux.vbird.org/linux_server/0330nfs.php)

1. 安装

    ```bash
    yum install nfs-utils rpcbind
    ```

2. 开放端口

    | 端口  | 开放类型 | 说明    |
    | ----- | -------- | ------- |
    | 111   | tcp,udp  | rpcbind |
    | 2049  | tcp      | nfs     |
    | 20048 | tcp      |         |

    ```bash
    firewall-cmd --zone=public --permanent --add-port=111/tcp --add-port=2049/tcp --add-port=20048/tcp
    ```

3. 配置

    编辑`/etc/exports`文件，按以下格式配置

    ```text
    共享目录 地址(权限)
    ```

    - 地址支持以下方式：

        - IP地址，如: `192.168.1.1`
        - 网域，如: `192.168.0.0/24`
        - 完整域名/主机名（支持通配符），如: `localhost`, `*.example.com`

    - 权限参数

        | 参数                                        | 说明                                                                                                                                           |
        | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
        | `rw`/`ro`                                   | 读写、只读                                                                                                                                     |
        | `sync`/`async`                              | `sync`代表数据会同步写入到内存与硬盘中<br />`async` 则代表数据会先暂存于内存当中，而非直接写入硬盘                                             |
        | `no_root_squash`/`root_squash`/`all_squash` | `no_root_squash`: 压缩成匿名用户<br />`root_squash`: 以`root`身份操作服务器的文件系统<br />`all_squash`: 不论登入身份为何， 都压缩成为匿名用户 |
        | `anonuid`                                   | 匿名用户ID，通常为 nobody(nfsnobody)。手动指定格式为`anonui=uid`                                                                               |
        | `anongid`                                   | 和`anonuid`类似，不过是匿名用户组ID                                                                                                            |

4. 启动

    ```bash
    systemctl enable --now rpcbind nfs
    ```

5. 查看已开放的NFS

    ```bash
    showmount -e localhost
    ```

6. 修改`/etc/exports`后在不重启服务的情况下生效

    ```bash
    exportfs -arv
    ```
