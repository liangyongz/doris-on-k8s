# doris-on-k8s

# 前言

当前使用**hostNetWork**方式部署。包括：1个FE，1个BE。

说明：

- StatefulSet+SVC方式（已舍弃）

  优点：常见的部署方式，一个节点可部署多个BE。

  缺点：Doris服务连续性缺失。测试过程中发现，因为FE记录的是podIP，所以每次重启BE-pod后，podIP变化，需手动重新ADD BACKEND。

- hostNetwork方式（目前部署方式）

  优点：PodIP都是节点IP，不用每次ADD BACKEND。保证了Doris服务的连续性。

  缺点：每个节点只能启动一个BE，因为对于同Deployment下的hostNetwork: true启动的Pod，每个node上只能启动一个。



## 文件说明

* Doris-FE

  FE-pv.yaml：创建FE需使用的存储，包括pv、sc、pvc。

  FE-deployment.yaml：FE部署脚本

* Doris-BE

  FE-pv.yaml：创建BE需使用的存储，包括pv、sc、pvc。

  FE-deployment.yaml：BE部署脚本
 
## 镜像说明

* docker pull 359261016/apache-doris-be:v0.15
* docker pull 359261016/apache-doris-fe:v0.15

也可自己参照官方文档构建镜像
http://doris.incubator.apache.org/zh-CN/installing/compilation.html#%E4%BD%BF%E7%94%A8-docker-%E5%BC%80%E5%8F%91%E9%95%9C%E5%83%8F%E7%BC%96%E8%AF%91-%E6%8E%A8%E8%8D%90

# 部署步骤

## 1.修改pv.yaml

下载四个yaml，**将FE-pv.yaml，BE-pv.yaml中的节点亲和性，修改成自己集群的node**。或删除节点亲和性相关配置。

```
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-010000000233
```

修改后执行命令即可完成FE、BE的部署

```shell
kubectl apply -f FE-pv.yaml
kubectl apply -f FE-deployment.yaml
kubectl apply -f BE-pv.yaml 
kubectl apply -f BE-deployment.yaml
```

<img src="/Users/liangcongcong/Code/doris-on-k8s/image-20211224154401551.png" alt="image-20211224154401551"  />

## 2. 添加BE到FE

执行完查看doris服务，启动正常

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1640153315318-55e7f8aa-062d-4cf0-a99a-a1fd03645094.png)

添加BE到FE

```yaml
#连接FE，172.16.0.161换成自己的节点IP
mysql -h172.16.0.161 -P9030 -uroot
SHOW PROC '/backends';
#添加BE，如果Alive=true，添加成功
ALTER SYSTEM ADD BACKEND "172.16.0.161:9050";
```

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1640153257035-4725cc76-c171-4f83-9c51-ca563b66fae4.png)

BE已添加，可正常使用Doris

**至此doris on k8s部署结束**。后续使用请参考官方文档：https://doris.apache.org/master/zh-CN/installing/compilation.html



## 3. 简单使用

```sql
#简单样例
CREATE DATABASE example_db;
USE example_db;
CREATE TABLE table1
(
    siteid INT DEFAULT '10',
    citycode SMALLINT,
    username VARCHAR(32) DEFAULT '',
    pv BIGINT SUM DEFAULT '0'
)
AGGREGATE KEY(siteid, citycode, username)
DISTRIBUTED BY HASH(siteid) BUCKETS 10
PROPERTIES("replication_num" = "1");

insert into table1 values(1,1,"test",2);
select * from table1;
```

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1640153489339-deed85b3-0341-4e75-9be4-0fdfbc28fec6.png)

# 附问题记录

## 1.每次BE重启后都需手动ADD BACKEND

现象：原本使用StatefulSet+SVC方式部署BE，执行ALTER SYSTEM ADD BACKEND "doris-be-0.doris-be-svc:9050"; BE添加到集群，当前podIP是10.108.2.107。

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639988922169-2a92a32d-2b4f-4e26-9e0d-4c5d34356c9f.png)

手动delete BE pod后SHOW PROC '/backends';后，新起的BE pod无法使用

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639988941548-3635bdf0-4fb4-4656-9689-48359707aac1.png)
	原因：ADD BACKEND命令记录的是IP，而不是host。所以当拉起新的BE pod后，即使host不变，但是PodIP已经变化。所以检查失败。

解决：不使用StatefulSet+SVC。改用Deployment+hostNetwork方式，将PodIP固定为NodeIP。

演示：手动删除并启动新BE，Doris服务如下，无需手动ADD BACKEND。

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639990207541-0fdb9a0e-4db8-48b4-8955-cebb2295fa8f.png)



## 2.重启FE，每次都新clusterId，导致原BE无法加入集群

场景：StatefulSet+SVC方式部署场景下，BE与FE运行正常，此时BE中记录当前的clusterId。此时手动delete FE，新起的FE的clusterId会变化，导致执行ADD BACKEND不成功，报错invalid cluster id。

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639991310239-f64ca104-c8f4-4053-93d6-eafce4b8c6a6.png)

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639990632935-21c2810f-7833-4f76-a95b-a26f1ecf4f95.png)

原因：

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639991198029-5be7a4bb-d6b3-475e-9864-a036bd220e3a.png)

解决：改用Deployment+hostNetwork方式，将FE的PodIP固定为NodeIP。

演示：手动删除并启动新FE，Doris服务如下，无需手动ADD BACKEND。

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1639990825586-1673bf43-40ef-466f-9a67-21bdbb0b5a77.png)



## 3.hostNetWork方式下，Doris服务连续性验证

手动重启BE，重启过程中提示backend does not exist or not alive，启动后恢复正常

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1640153928156-97b8b4f4-09c2-4509-a711-ee66b27d2af8.png)



手动重启FE，重启过程中提示Can't connect to MySQL server，启动后恢复正常

![img](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/13456375/1640154218000-8d2d396a-1f4a-4964-a05c-542ef7a92e64.png)







