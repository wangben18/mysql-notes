> 拉取 MySQL 镜像
```shell
docker pull mysql
```
> 运行 MySQL server 容器
```shell
docker run --name $container_name -e MYSQL_ROOT_PASSWORD=$password -d mysql:5.7
```
> 创建 docker 容器网络
```shell
docker network create -d bridge $net_name
```
> 运行 MySQL client 连接 server 容器
```shell
docker run -it --network $net_name --rm mysql mysql -h$container_name -uroot -p$password
```

- 查看配置文件位置
```bash
docker exec -it $container_name mysql --help | grep my.cnf
```
> output:
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
> /etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf