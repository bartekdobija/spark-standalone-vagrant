# Spark dependencies
$spark_deps = <<SCRIPT

  SPARK_VER=spark-2.4.4
  SPARK_LINK=/opt/spark

  if [ ! -e ${SPARK_LINK} ]; then
    adduser spark
    echo "Installing Spark..."
    if [ "$(ls -la /vagrant/spark/ | grep ${SPARK_VER}.tgz | wc -l)" == "1" ]; then
      cp -f /vagrant/spark/${SPARK_VER}-bin-hadoop2.7.tgz /tmp/
    else
      wget http://us.mirrors.quenda.co/apache/spark/${SPARK_VER}/${SPARK_VER}-bin-hadoop2.7.tgz -q -P /tmp/
    fi
    tar zxf /tmp/${SPARK_VER}-bin-hadoop2.7.tgz -C /opt/ \
      && ln -s /opt/${SPARK_VER}-bin-hadoop2.7 ${SPARK_LINK} \
      && chown -R spark:spark /opt/${SPARK_VER}-bin-hadoop2.7 \
      && mkdir -p /var/log/spark/apps \
      && chown -R spark:spark /var/log/spark \
      && chmod -R 777 /var/log/spark/apps
  fi

  [ ! -e ${SPARK_LINK} ] && echo "Spark installation has failed!" && exit 1

  echo "Spark configuration..."
  echo "configuring /etc/profile.d/spark.sh"
  echo 'export PATH=$PATH'":${SPARK_LINK}/bin" > /etc/profile.d/spark.sh

  echo "configuring /opt/spark/conf/spark-env.sh"
  cat << SPCNF > /opt/spark/conf/spark-env.sh
#HADOOP_CONF_DIR=/etc/hadoop/conf/
#SPARK_DIST_CLASSPATH=\\$(hadoop classpath):${SPARK_LINK}/hive/lib/*
#LD_LIBRARY_PATH=\\${LD_LIBRARY_PATH}:/usr/lib/hadoop/lib/native
SPARK_LOG_DIR=/var/log/spark
SPCNF

  echo "configuring ${SPARK_LINK}/conf/spark-defaults.conf"
  cat << SPCNF > ${SPARK_LINK}/conf/spark-defaults.conf
# enable logging of spark application events
spark.eventLog.enabled true
spark.eventLog.dir /var/log/spark/apps
spark.history.fs.logDirectory /var/log/spark/apps
spark.history.fs.cleaner.enabled true
spark.shuffle.service.enabled true
# Execution Behavior
spark.broadcast.blockSize 4096
spark.dynamicAllocation.enabled true
spark.speculation true
spark.scheduler.mode FAIR
spark.kryoserializer.buffer.max 1000m
spark.driver.maxResultSize 0
spark.serializer org.apache.spark.serializer.KryoSerializer
spark.master spark://spark.instance.com:7077
spark.rdd.compress true
# Local execution of selected Spark functions
spark.sql.parquet.binaryAsString true
spark.sql.parquet.compression.codec snappy
# use lz4 compression for broadcast variables as Snappy is not supported on MacOSX
spark.broadcast.compress true
spark.checkpoint.compress true
spark.io.compression.codec lz4
spark.executor.extraJavaOptions -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedOops
spark.driver.extraJavaOptions -XX:+UseCompressedOops -XX:MaxPermSize=1g
spark.executor.extraClassPath /usr/local/lib/jdbc/mysql/*.jar
spark.driver.extraClassPath /usr/local/lib/jdbc/mysql/*.jar
SPCNF


SCRIPT

$hive_deps = <<SCRIPT


  HIVE_VER=2.3.6
  HIVE_LINK=/opt/hive

  if [ ! -e ${HIVE_LINK} ]; then
    adduser hive
    echo "Installing Apache Hive ${HIVE_VER}"
    wget http://mirrors.whoishostingthis.com/apache/hive/hive-${HIVE_VER}/apache-hive-${HIVE_VER}-bin.tar.gz -q -P /tmp/ \
      && tar zxf /tmp/apache-hive-${HIVE_VER}-bin.tar.gz -C /opt/ \
      && ln -s /opt/apache-hive-${HIVE_VER}-bin ${HIVE_LINK} \
      && chown hive:hive /opt/apache-hive-${HIVE_VER}-bin
  fi

  echo "configuring ${HIVE_LINK}/conf/hive-site.xml"
  cat << HIVECNF > ${HIVE_LINK}/conf/hive-site.xml

<configuration>
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>hive</value>
</property>
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
</property>
<property>
  <name>hive.metastore.uris</name>
  <value>thrift://spark.instance.com:9083</value>
</property>
<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>file:///user/hive/warehouse</value>
</property>
</configuration>

HIVECNF


SCRIPT


# MySQL dependencies
$mysql_deps = <<SCRIPT

  MYSQL_REPO=https://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
  MY_CNF=/etc/my.cnf
  DEV_PASSWORD=hadoop

  [ ! -e /etc/yum.repos.d/mysql-community.repo ] && rpm -ivh ${MYSQL_REPO}

  yum install --nogpgcheck -y mysql-community-server

  if [ -e /etc/systemd/system/mysql.service ] && [ -z "$(grep -R vagrant ${MY_CNF})" ]; then
    echo "# InnoDB settings" >> ${MY_CNF}
    echo "default_storage_engine = innodb" >> ${MY_CNF}
    echo "innodb_file_per_table = 1" >> ${MY_CNF}
    echo "innodb_flush_log_at_trx_commit = 2" >> ${MY_CNF}
    echo "innodb_log_buffer_size = 64M" >> ${MY_CNF}
    echo "innodb_buffer_pool_size = 1G" >> ${MY_CNF}
    echo "innodb_thread_concurrency = 8" >> ${MY_CNF}
    echo "innodb_flush_method = O_DIRECT" >> ${MY_CNF}
    echo "innodb_log_file_size = 512M" >> ${MY_CNF}
    echo "explicit_defaults_for_timestamp = 1" >> ${MY_CNF}
    systemctl enable mysqld.service \
      && systemctl start mysqld.service \
      && /usr/bin/mysqladmin -u root password "${DEV_PASSWORD}" &> /dev/null \
      && echo "# vagrant provisioned" >> ${MY_CNF}

    mysql -u root -p${DEV_PASSWORD} \
      -e "create schema if not exists hive; grant all on hive.* to 'hive'@'localhost' identified by 'hive'"
  fi

SCRIPT

# OS configuration
$system_config = <<SCRIPT

  # disable IPv6
  if [ "$(grep disable_ipv6 /etc/sysctl.conf | wc -l)" == "0" ]; then
    echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.conf \
      && echo "net.ipv6.conf.default.disable_ipv6=1" >> /etc/sysctl.conf \
      && sysctl -f /etc/sysctl.conf \
      && sysctl -p

    sysctl -w net.ipv6.conf.all.disable_ipv6=1
    sysctl -w net.ipv6.conf.default.disable_ipv6=1

  fi


  # this should be a persistent config
  ulimit -n 65536
  ulimit -s 10240
  ulimit -c unlimited


  DEV_USER=hadoop_user
  DEV_PASSWORD=hadoop
  PROXY_CONFIG=/etc/profile.d/proxy.sh

  systemctl disable firewalld && systemctl stop firewalld

  # Add entries to /etc/hosts
  ip=$(ip a s eth1 | awk '/inet/ {split($2, a,"/"); print a[1] }')
  host=$(hostname)
  echo "127.0.0.1 localhost" > /etc/hosts
  echo "$ip $host" >> /etc/hosts

  # disable selinux
  sudo setenforce 0
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

  # disable swap
  swapoff -a


  # Add a dev user - don't worry about the password
  if ! grep ${DEV_USER} /etc/passwd; then
    echo "Creating user ${DEV_USER}" && useradd -p $(openssl passwd -1 ${DEV_PASSWORD}) ${DEV_USER} \
      && echo "${DEV_USER}  ALL=(ALL)  NOPASSWD:  ALL" > /etc/sudoers.d/${DEV_USER}
  fi

#  if [ "$(grep vm.swappiness /etc/sysctl.conf | wc -l)" == "0" ]; then
#    echo "vm.swappiness=10" >> /etc/sysctl.conf && sysctl vm.swappiness=10
#  fi

SCRIPT

# DNF configuration
$yum_config = <<SCRIPT
  rpm -i https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm 2> /dev/null
  yum update
  yum -y install wget curl openssl java-1.8.0-openjdk
SCRIPT

$information = <<SCRIPT
  ip=$(ip a s eth1 | awk '/inet/ {split($2, a,"/"); print a[1] }')
  echo "Guest IP address: $ip"
  echo "Namenode UI available at: http://$ip:9870"
  echo "Resource Manager UI available at: http://$ip:8088"
  echo "Spark historyserver available at: http://$ip:18080"
  echo "Spark available under /opt/spark"
  echo "MySQL root password: hadoop"
  echo "You may want to add the below line to /etc/hosts:"
  echo "$ip spark.instance.com"
SCRIPT

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  config.vm.hostname = "spark.instance.com"
  config.vm.network :public_network

  config.vm.provider "virtualbox" do |vb|
    vb.name = "dev-spark-standalone-env"
    vb.cpus = 4
    vb.memory = 8192
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "100"]
  end

  config.vm.provision :shell, :name => "system_config", :inline => $system_config
  config.vm.provision :shell, :name => "yum_config", :inline => $yum_config
  config.vm.provision :shell, :name => "mysql_deps", :inline => $mysql_deps
  config.vm.provision :shell, :name => "spark_deps", :inline => $spark_deps
  config.vm.provision :shell, :name => "hive_deps", :inline => $hive_deps
  config.vm.provision :shell, :name => "information", :inline => $information

end
