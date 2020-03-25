---
title: euler简单上手
date: 2020-03-25 10:13:22
tags: [图计算,TensorFlow]
categories: [TensorFlow,深度学习]
---

## 说明

本文主要介绍euler利用docker再单机上模拟分布式训练，分布式模型评估，分布式embedding的简单上手体验过程，一些基础的工作我已经准备好了。如果只是单机的训练等，直接参考官方文档就好了，可以在本地的环境或是利用现有的docker环境都可以。

本教程主要用`docker-compose在`单台物理机上来模拟多主机环境，后面如需要跑在k8s环境中可参考其中内容进行改造即可。

## 安装

### 源码下载

1. git clone --recursive https://github.com/alibaba/euler.git`

2. 如果过程中`clone`失败，那么进入主目录使用`submodule update --init --recursive --progress`检出各个子模块

3. 对于目录`D:\shadowsocks\euler\third_party`要检出第三方依赖的问题，

   1. 如果报的是`zookeeper`的`Unable to find current revision in submodule path`问题，那么可以在当前目录中直接将zookeeper的代码`clone`下来，然后再切换到源码中指定的`commit`版本：

      ```
      git clone https://github.com/apache/zookeeper.git`
      cd zookeeper
      git checkout 05b774a1b05374618300f657c9c91b0d5c6ddf71
      ```

   2. 如何因为网络原因无法`clone`，则可以先再网上找到相应的包，然后再修改`commit`版本：

      ```
      1. 使用此链接下载Fuzzer并解压：https://github-production-repository-file-5c1aeb.s3.amazonaws.com/165004157/2991980?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20200228%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200228T111759Z&X-Amz-Expires=300&X-Amz-Signature=0e9db781ee0e38e1b1c02cfbdeff669484631d9aa2c5bbfa64c9c243b6bab2f6&X-Amz-SignedHeaders=host&actor_id=8743639&response-content-disposition=attachment%3Bfilename%3DFuzzer.zip&response-content-type=application%2Fzip
      2. 将Fuzzer文件夹名称改成libFuzzer，并且进入此文件夹。
      3. 修改commit版本：git checkout 1b543d6e5073b56be214394890c9193979a3d7e1
      ```

   3. 如果别的依赖子模块出现问题可参考以上两种方法。

**在当前进度下，安装时建议使用master分支**

<!--more-->

### docker 安装 

1. 使用Dockerfile安装：

   ```
   cd euler
   docker build --net=host -f tools/docker/Dockerfile .
   对于新版本的docker build需要改成：
   docker build --network=host -f tools/docker/Dockerfile .
   ```

2. 如果报有错误`-- tensorflow_framework library found: in TF_FRAMEWORK-NOTFOUND`可在Dockerfile中`COPY . /tmp/Euler`语句后添加：

   ```
   RUN cd /usr/local/lib/python2.7/dist-packages/tensorflow/ && \
       ln -s libtensorflow_framework.so.1 libtensorflow_framework.so
   ```

3. 在docker中使用时，默认安装的TensorFlow是最新版本，但跑ppi的demo是会报`segmentation fault (core dumped)`的问题，所以要在Dockerfile中安装指定TensorFlow版本解决此问题：

   `pip --no-cache-dir install tensorflow==1.12`

**注意：在我的所有docker环境中安装的TensorFlow版本都是1.12，大家可以根据需要更改。**

## 集群搭建

制作好docker镜像之后，使用docker-compose来进行容器的编排。在整个集群环境中，需要hadoop的hdfs来进行数据的分布式分发和存储，需要zookeeper来进行euler图引擎各节点的协调工作。

### 集群的编排

下面是一个启动两个ps，4个worker训练集群的docker-compose文件。

```yaml
version: '3'
services:
  zookeeper:
    image: mirror.jd.com/9n/zookeeper:3.4.14
    restart: always
    user: root
    container_name: zookeeper

  hdfs:
    image: mirror.jd.com/9n/hadoop:2.7.0
    restart: always
    user: root
    container_name: hdfs

  ps1:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: ps1
    tty: true
    depends_on:
      - zookeeper
      - hdfs
  ps2:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: ps2
    tty: true
    depends_on:
      - zookeeper
      - hdfs

  worker1:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: worker1
    tty: true
    depends_on:
      - zookeeper
      - hdfs
      - ps1
      - ps2
  worker2:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: worker2
    tty: true
    depends_on:
      - zookeeper
      - hdfs
      - ps1
      - ps2

  worker3:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: worker3
    tty: true
    depends_on:
      - zookeeper
      - hdfs
      - ps1
      - ps2

  worker4:
    image: mirror.jd.com/9n/euler:latest_ping
    restart: always
    user: root
    container_name: worker4
    tty: true
    depends_on:
      - zookeeper
      - hdfs
      - ps1
      - ps2
```

现有的镜像说明：

1. 镜像`mirror.jd.com/9n/euler:latest_ping`安装了ping命令，便于调试。
2. 镜像`mirror.jd.com/9n/euler:latest_network_host`在Dockerfile 的build是使用了`--network=host`方式，是得容器与主机共享一个网段。
3. 镜像`mirror.jd.com/9n/euler:latest`，比较纯净的镜像，不加`--network=host`方式，不加别的命令工具。
4. 镜像`mirror.jd.com/9n/euler_tmp:latest`，不加`--network=host`方式，加了`vim`和网络相关的命令工具，并且保留了编译后的源码，源码位置为`/tmp/Euler`。如果想进行源码调试，比如修改某些python代码，使用命令`docker exec -it containerid /bin/bash`后，那么必须先`source /etc/profile`让其中的`export PYTHONPATH=PYTHONPATH:/tmp/Euler`环境生效，否则生效的会是安装到全局目录下的Euler，此处可参考[安装编译中的安装步骤](<https://github.com/alibaba/euler/wiki/%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85>)。

大家可以根据需要定制镜像，定义集群。

定义好`docker-compose`文件之后，使用命令`docker-compose up -d`启动集群。通过`docker-compose stop`或是`docker-compose down`停止或是撤销docker集群。

## 数据准备

训练集群准备好之后，下一步就是要准备训练数据和测试数据。

本教程参考使用ppi4的数据进行分布式训练，对于训练数据，其中涉及到：一，数据如何获取；二，如何将原始数据转换成图的数据格式；三，再将这种符合图数据格式规范的数据转换成二进制图数据；四，最后进行数据分片。当然可以先进行分片再进行二进制转换也都可以，怎么方便怎么来。分布式训练必须是要按照一定规范将数据进行分片的，具体如何分，参考官方教程：[数据准备](<https://github.com/alibaba/euler/wiki/%E6%95%B0%E6%8D%AE%E5%87%86%E5%A4%87>)。由于使用的ppi4的数据已经是符合图数据格式规范的明文数据，所以无需上述的第二步，要做的只是将数据进行分片并且转换成二进制。

对于测试数据`ppi_test.id`无需进行分片。

### 数据获取

参考[数据获取](<https://github.com/alibaba/euler/wiki/%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B#1-%E6%95%B0%E6%8D%AE%E5%87%86%E5%A4%87>)。可使用本地或是现有的docker环境进行获取，使用现有的docker环境的话就不需要安装euler，并且可以使用euler自带的一些工具类进行数据处理，前提是docker容器能够与外界通信。

### 数据分片与转换

上一步骤获取的数据是位于单个文件中的，只适用于单机的训练环境，如果要想在分布式环境中使用必须按照规范对训练数据进行分片，规范参考[数据分片](<https://github.com/alibaba/euler/wiki/%E6%95%B0%E6%8D%AE%E5%87%86%E5%A4%87#%E6%95%B0%E6%8D%AE%E5%88%86%E7%89%87>)。

对于小规模数据，可使用awk命令进行内容的切分，对于大的数据量可以使用spark进行切分，对于小或是大的数据量的数据二进制转换，可使用已经安装好的euler环境的[工具进行转换](<https://github.com/alibaba/euler/wiki/%E6%95%B0%E6%8D%AE%E5%87%86%E5%A4%87#%E5%9B%BE%E6%95%B0%E6%8D%AE%E8%BD%AC%E6%8D%A2%E5%B7%A5%E5%85%B7>)。

### 数据上传

对于分布式训练和验证的数据都必须放在hdfs中。

在ppi4的demo中，数据是放在本地，下面是提供的数据上传脚本。

`prepare_data.sh`:

```shell
#!/usr/bin/env bash

docker exec hdfs /usr/local/hadoop/bin/hdfs dfsadmin -safemode wait
sleep 5s
echo "copy the ppi4 data to docker..."
docker cp ppi4 hdfs:/home
echo "create folders in  hdfs..."
docker exec -it hdfs  /bin/bash -c 'source /etc/profile && hadoop fs -mkdir ppi4'
docker exec -it hdfs  /bin/bash -c 'source /etc/profile && hadoop fs -mkdir ppi4_test'
echo "put splited files used for training to the hdfs...."
docker exec -it hdfs  /bin/bash -c 'source /etc/profile && hadoop fs -put /home/ppi4/ppi_data_0.dat  ppi4\
                                    && hadoop fs -put /home/ppi4/ppi_data_1.dat  ppi4\
                                    && hadoop fs -put /home/ppi4/ppi_data_2.dat  ppi4\
                                    && hadoop fs -put /home/ppi4/ppi_data_3.dat  ppi4'
echo "put the id flies used for evaluating to the hdfs..."
docker exec -it hdfs  /bin/bash -c 'source /etc/profile && hadoop fs -put /home/ppi4/ppi.id  ppi4_test'
sleep 3s

```

## 分布式训练

集群和数据都准备好之后，就需要提交任务进行分布式训练了。首先通过`docker exec -it containerid /bin/bash`登录各个docker容器。

### 启动ps server

由于不管是训练，评估，还是在embedding输出，任务的提交都是再worker节点上，所以我们可以先启动ps节点。

```shell
ps1 容器：
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=ps --task_index=0

ps2 容器：
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=ps --task_index=1
```

### 训练

然后在各个worker节点上启动训练任务：

```shell
worker1容器：
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=0 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode train

worker2容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=1 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode train

worker3容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=2 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode train

worker4:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=3 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode train

```



### 模型评估

模型评估脚本，多了`--id_file`这个参数。`hdfs://hdfs:9000/user/root/ppi4_test/ppi.id `这个就是测试文件。

```shell
worker1容器：
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=0 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --id_file hdfs://hdfs:9000/user/root/ppi4_test/ppi.id --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode evaluate

worker2容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=1 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --id_file hdfs://hdfs:9000/user/root/ppi4_test/ppi.id --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode evaluate

worker3容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=2 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --id_file hdfs://hdfs:9000/user/root/ppi4_test/ppi.id --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode evaluate

worker4:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=3 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --id_file hdfs://hdfs:9000/user/root/ppi4_test/ppi.id --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode evaluate

```

观察点：分布式训练中的模型效果要比单机的差。

### embedding输出

与模型训练的区别就只是最后一个`--mode`参数。

```shell
worker1容器：
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=0 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode save_embedding

worker2容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=1 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode save_embedding

worker3容器:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=2 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode save_embedding

worker4:
python -m tf_euler --ps_hosts=ps1:1998,ps2:1999 --worker_hosts=worker1:2000,worker2:2001,worker3:2002,worker4:2003 --job_name=worker  --task_index=3 --data_dir hdfs://hdfs:9000/user/root/ppi4 --model_dir hdfs://hdfs:9000/user/root/ckpt --euler_zk_addr zookeeper:2181 --euler_zk_path /tf_euler_train --max_id 56944 --feature_idx 1 --feature_dim 50 --label_idx 0 --label_dim 121 --model graphsage_supervised --mode save_embedding
```

### 注意

1. `docker-compose`集群中通过容器名称就可以访问目标容器的服务，所以会有类似`worker1:2000,worker2:2001`这样的表示方法。
2. 配置和启动任务时，注意各个容器（主机）的对应顺序。比如`task_index=3`的命令对应的是`worker4`容器（主机），也只能在`worker4`上启动此条命令。

## 训练流程总结

当前目录结构如下：

```shell
.
├── docker-compose.yml
├── evaluate_cmd.txt
├── ppi4
│   ├── ppi_data_0.dat
│   ├── ppi_data_1.dat
│   ├── ppi_data_2.dat
│   ├── ppi_data_3.dat
│   └── ppi.id
├── prepare_data.sh
├── save_embedding_cmd.txt
└── train_cmd.txt
```

只要将数据包拷贝到linux目录下。

1. 通过`docker-compose` up  -d启动集群。
2. 通过运行`sh prepare_date.sh`文件将训练和测试文件上传到指定的hdfs目录下。
3. 通过`docker exec -it containerid /bin/bash`登录各个docker容器，在对应的docker容器上先启动ps任务，然后再在对应的docker容器上启动worker任务，分别进行训练，模型评估，embedding输出。
4. 如果要销毁集群只需要`docker-compose down`即可。

## 问题

1. 如果是使用的ubuntu桌面版，那么在安装时在`/etc/profile`里面配置的`export`将可能导致系统登录不进去，建议装在docker内。

2. 各个`ps`和`worker`节点的配置要注意保持对应关系。

3. 如果容器中出现`tensorflow.python.framework.errors_impl.UnknownError: Could not start gRPC server`可先查看之前的跑的任务是以后台运行的方式存在，杀掉之后可重新提交任务。

4. 访问hdfs，或是镜像mirror.jd.com/9n/euler_tmp的容器想在`tmp/Euler/`中修改代码调试时，通过`docker exec -it worker1 /bin/bash`进入镜像时，首先`source /etc/profile`。

5. 原有的euler分布式训练中存在无法同步退出的问题，我针对退出逻辑做了修改，`集群的编排`模块中介绍的镜像都是我修改python代码后打的。修改的文件[euler/tf_euler/python/utils/hooks.py](https://github.com/alibaba/euler/blob/master/tf_euler/python/utils/hooks.py)的内容如下：

   ```python
   class SyncExitHook(tf.train.SessionRunHook):
     def __init__(self, num_workers, task_index, is_chief):
       self._task_index = task_index
       self._is_chief = is_chief
       self._num_workers = num_workers
       self._counter_vars = []
       self._counter_add_ops = []
       for i in range(self._num_workers):
         counter_var = tf.Variable(0, name="num_finished_workers-{}".format(i), collections=[tf.GraphKeys.LOCAL_VARIABLES])
         self._counter_vars.append(counter_var)
         counter_var_ops = tf.assign(counter_var, 1, use_locking=True)
         self._counter_add_ops.append(counter_var_ops)
   
     def end(self, session):
       session.run(self._counter_add_ops[self._task_index])
       while True:
         num_finished_workers = 0
         for i in range(self._num_workers):
           state = session.run(self._counter_vars[i])
           if i == self._task_index and state == 0:
             session.run(self._counter_add_ops[i])
             state = session.run(self._counter_vars[i])
   
           num_finished_workers = num_finished_workers + state
   
         tf.logging.info("%d workers have finished ...", num_finished_workers)
         if num_finished_workers >= self._num_workers:
           break
   
         time.sleep(1)
   ```

   然后再[euler/tf_euler/python/run_loop.py](https://github.com/alibaba/euler/blob/master/tf_euler/python/run_loop.py)中调用`SynExitHook`的地方增加传入的参数即可：`SyncExitHook(len(flags_obj.worker_hosts), flags_obj.task_index, is_chief)`

   仍然存在的问题，如果某个节点过早退出了，那么此节点再zookeeper中的数据也会相应地删除，euler监测到zookeeper中节点数量不满足那么剩下的节点就不会退出，不过出现的概率小，一般出现在`evaluate`模式中。

## 后话

如果想要支持大规模的集群训练和性能提升还需要做很多的优化。官方文档中也给了一些建议，也需要在实战中提升。