# Hadoop Install

[Doc](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-common/ClusterSetup.html#Slaves_File)

| Hosts             |                                                          |
| ----------------- | -------------------------------------------------------- |
| centos-namenode-1 | Datanode, NodeManager, ResourceManager, JobHistoryServer |
| centos-datanode-1 | Datanode, NodeManager, SecondaryNameNode                 |
| centos-datanode-2 | Datanode, NodeManager                                    |
| centos-datanode-3 | Datanode, NodeManager                                    |


```bash
################################# System config
yum update -y

yum install -y epel-release

yum install -y pssh vim ntp net-tools

yum install -y java-1.8.0-openjdk-devel

vim /etc/fstab
###
... defaults,discard,nodiratime,noatime  ...
###

cat /sys/kernel/mm/transparent_hugepage/enabled
echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" >> /etc/rc.d/rc.local
echo "echo never > /sys/kernel/mm/transparent_hugepage/defrag" >> /etc/rc.d/rc.local


chmod +x /etc/rc.d/rc.local

vim /etc/default/grub
###
...
GRUB_CMDLINE_LINUX = "... transparent_hugepage=never"
...
###

grub2-mkconfig -o /boot/grub2/grub.cfg
#grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg

reboot

############################시간설정
vim /etc/ntp.conf
### 메인 노드
# 아래는 메인 서버에만
restrict 211.183.7.70 mask 255.255.255.0 nomodify notrp
# 기존의 server들을 주석처리
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
server [서버] iburst
###
### 슬레이브는 아래처럼
# 기존의 server들을 주석처리
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
# server 0.centos.pool.ntp.org.iburst
server centos-1
###

chkconfig ntpd on

timedatectl set-timezone Asia/Seoul
timedatectl set-local-rtc 0
timedatectl set-ntp 1

systemctl restart ntpd
############################## Account config

groupadd hadoop

useradd -mg hadoop hdfs
useradd -mg hadoop yarn
useradd -mg hadoop mapred

passwd hdfs
passwd yarn
passwd mapred

mkdir -p /data/hadoop/hdfs/nn
mkdir -p /data/hadoop/hdfs/snn
mkdir -p /data/hadoop/hdfs/dn
mkdir -p /data/hadoop/hdfs/yarn

chown -R hdfs:hadoop /data/hadoop/hdfs
chmod -R 775 /data/hadoop/hdfs

chown -R yarn:hadoop /data/hadoop/hdfs/yarn

mkdir -p /var/log/hadoop/logs

chown -R yarn:hadoop /var/log/hadoop
chmod -R 775 /var/log/hadoop

```

## 환경변수 설정 /etc/profile.d/hadoop.sh

```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64
export HADOOP_HOME=/opt/hadoop-2.9.2
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export HADOOP_PREFIX=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_LOG_DIR=/var/log/hadoop/logs
export YARN_LOG_DIR=/var/log/hadoop/logs
export HADOOP_MAPRED_PID_DIR=/var/log/hadoop/mapred
export HADOOP_MAPRED_LOG_DIR=/var/log/hadoop/mapred

export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```



```bash
# hadoop 받아오기
##HADOOP
chmod -R 775 /opt/hadoop-2.9.2
chown -R hdfs:hadoop /opt/hadoop-2.9.2
```



## core-site.xml

```xml
<configuration>
    <!-- name of the default file system for cluster -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://centos-1:9000</value>
    </property>

    <!-- default user name : hdfs -->
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>hdfs</value>
    </property>

    <!-- how long keep deleted files in HDFS -->
    <property>
        <name>fs.trash.interval</name>
        <value>1440</value> <!-- 1day -->
    </property>
</configuration>
```



## mapred-site.xml

```bash
cp $HADOOP_HOME/etc/hadoop/mapred-site.xml.template $HADOOP_HOME/etc/hadoop/mapred-site.xml
```

```xml
<configuration>
    <!-- runtime framework for executing mapreduce jobs -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <!-- where YARN store all the application-related information under HDFS -->
    <property>
        <name>yarn.app.mapreduce.am.staging-dir</name>
        <value>/user</value>
    </property>

    <!-- 위 경로 설정시 자동으로 아래도 넣어줘야함 -->
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>${yarn.app.mapreduce.am.staging-dir}/history/dir</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>${yarn.app.mapreduce.am.staging-dir}/history/done_intermediate</value>
    </property>

    <!-- RPC port and address for clients to query job history-->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>centos-namenode-1:10020</value>
    </property>

    <!-- Jobhistory server web UI -->
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>centos-namenode-1:19888</value>
    </property>

    <!-- MapReduce task Memory -->
    <property>
        <name>mapreduce.map.memory.mb</name>
        <value>2048</value>
    </property>
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>2048</value>
    </property>

    <!-- MapReduce JVM Heap Size -->
    <property>
        <name>mapreduce.map.java.opts</name>
        <value>-Xmx1600m</value>
    </property>
    <property>
        <name>mapreduce.reduce.java.opts</name>
        <value>-Xmx1600m</value>
    </property>
</configuration>
```



## yarn-site.xml

```xml
<configuration>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.Shufflehandler</value>
    </property>

    <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>centos-namenode-1</value>
    </property>

    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>4</value>
    </property>

    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>8192</value>
    </property>

    <!-- aggregate application logs -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>

    <!-- how long will keep log files -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>

    <!-- directory in HDFS where to store log files -->
    <!-- log 저장할 HDFS 디렉토리 -->
    <property>
        <name>yarn.nodemanager.remote-app-log-dir</name>
        <value>/tmp/logs</value>
    </property>

    <!-- log 저장할 로컬 디렉토리 -->
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>/var/log/hadoop/logs/yarn</value>
    </property>

    <!-- local directory where to store output of MapReduce -->
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>/data/hadoop/hdfs/yarn/nm</value>
    </property>

    <property>
         <name>yarn.log.server.url</name>
         <value>http://centos-namenode-1:19888/jobhistory/logs</value>
    </property>

    <!-- classpath, JAR files -->
    <property>
        <name>yarn.application.classpath</name>
            <value>
                $HADOOP_HOME/etc/hadoop/conf,
                $HADOOP_HOME/share/hadoop/common/*,
                $HADOOP_HOME/share/hadoop/common/lib/*,
                $HADOOP_HOME/share/hadoop/hdfs/*,
                $HADOOP_HOME/share/hadoop/hdfs/lib/*,
                $HADOOP_HOME/share/hadoop/yarn/*,
                $HADOOP_HOME/share/hadoop/yarn/lib/,
                $HADOOP_HOME/share/hadoop/mapreduce/*,
                $HADOOP_HOME/share/hadoop/mapreduce/lib/*,
            </value>
    </property>
</configuration>
```



## hdfs-site.xml

```xml
<configuration>
    <!-- URI for cluster's NameNode, DataNode will use this URI to register with NameNode -->
    <property>
    	<name>fs.default.name</name>
        <value>hdfs://centos-1:9000</value>
    </property>
    
	<!-- number of data block -->
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    
    <!-- where local file system DataNode stores its Datablock -->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/data/hadoop/hdfs/dn</value>
    </property>

    <!-- where to store NameNode's key metadata -->
    <property>
    	<name>dfs.namenode.name.dir</name>
        <value>/data/hadoop/hdfs/nn</value>
    </property>
    
    <!-- where to store Standy(secondary) NameNode stores metadata -->
    <property>
    	<name>dfs.namenode.checkpoint.dir</name>
        <value>/data/hadoop/hdfs/snn</value>
    </property>
    
    <!-- size of block -->
    <!-- default block size is(128MB) -->
    <!-- 128m, 1g, 128k 등으로 표기가능 -->
    <property>
    	<name>dfs.block.size</name>
        <value>32m</value>
    </property>
    
    <!-- none-HDFS storage percent -->
    <property>
    	<name>dfs.datanode.du.reserved</name>
        <value>10</value>
    </property>

</configuration>
```

## Hadoop Startup

centos-1

```bash
[hdfs] hdfs namenode -format
[hdfs] hadoop-daemon.sh start namenode
[hdfs] hadoop-daemon.sh start datanode
[yarn] yarn-daemon.sh start resourcemanager
[yarn] yarn-daemon.sh start nodemanager
```

```bash
[hdfs] hdfs dfs -mkdir /user/history
[hdfs] hdfs dfs -chmod -R 1777 /user/history
[hdfs] hdfs dfs -chown mapred:hadoop /user/history
[hdfs] hdfs dfs -mkdir -p /user/hdfs
[hdfs] hdfs dfs -mkdir -p /user/yarn
[hdfs] hdfs dfs -chmod -R 1755 /user/hdfs
[hdfs] hdfs dfs -chmod -R 1755 /user/yarn
[hdfs] hdfs dfs -chown hdfs:hadoop /user/hdfs
[hdfs] hdfs dfs -chown yarn:hadoop /user/yarn

[hdfs] hdfs dfs -mkdir /tmp/logs
[hdfs] hdfs dfs -chown -R hdfs:hadoop /tmp/logs
[hdfs] hdfs dfs -chmod -R 1775 /tmp/logs
```

centos-2

```bash
[hdfs] hadoop-daemon.sh start secondarynamenode
[hdfs] hadoop-daemon.sh start datanode
[yarn] yarn-daemon.sh start nodemanager
[mapred] mr-jobhistory-daemon.sh start historyserver
```

centos-3

```bash
[hdfs] hadoop-daemon.sh start datanode
[yarn] yarn-daemon.sh start nodemanager
```



### Script

```bash
ssh hdfs@vm-centos-1 'hadoop-daemon.sh start namenode'
ssh hdfs@vm-centos-1 'hadoop-daemon.sh start datanode'
ssh yarn@vm-centos-1 'yarn-daemon.sh start resourcemanager'
ssh yarn@vm-centos-1 'yarn-daemon.sh start nodemanager'

ssh hdfs@vm-centos-2 'hadoop-daemon.sh start secondarynamenode'
ssh hdfs@vm-centos-2 'hadoop-daemon.sh start datanode'
ssh yarn@vm-centos-2 'yarn-daemon.sh start nodemanager'
ssh mapred@vm-centos-2 'mr-jobhistory-daemon.sh start historyserver'

ssh hdfs@vm-centos-3 'hadoop-daemon.sh start datanode'
ssh yarn@vm-centos-3 'yarn-daemon.sh start nodemanager'
```


