---
layout: post
title: "Что нам стоит Hadoop настроить?"
date: 2013-12-18 23:11:47.000000000 +04:00
---
Этот пост будет дублировать мою заметку на Habrahabr с небольшими исправлениями о том, как же всетаки настроить "слоника" версии 2.2.0 для опытов.

Несмотря на то, что в интернете на иностранных ресурсах есть полно материала про настройку/развертывание Hadoop, большинство из них либо описывают настройку ранних версий (0.X.X и 1.X.X), либо описывают только настройку в режиме single mode/pseudo distributed mode и лишь частично fully distributed mode. На русском языке материала практически нет вовсе.

Когда мне самому понадобился Hadoop, то я далеко не с первого раза смог все настроить. Материал был неактуален, часто попадались конфиги, которые используют deprecated параметры, поэтому использовать их нежелательно. А даже когда все настроил, то задавался многими вопросами, на которые искал ответы.

![](http://habr.habrastorage.org/post_images/b1b/ba9/391/b1bba93918fcd31dcd41dd83b9860542.png)


##Предварительные настройки

В качестве операционной системы для нашего кластера я предлагаю использовать `Ubuntu Server 12.04.3 LTS`, но при минимальных изменениях можно будет проделать все шаги и на другой ОС.

Все узлы будут работать на `VirtualBox`. Системные настройки для виртуальной машины я выставлял небольшие. Всего 8 GB пространства для жёсткого диска, одно ядро и 512 Мб памяти. Виртуальная машина также оснащена двумя сетевыми адаптерами: один NAT, а другой для внутренней сети.

После того, как была скачена и установлена операционная система, необходимо обновиться и установить `ssh` и `rsync`:

<pre>
</code>sudo apt-get update && sudo apt-get upgrade
sudo apt-get install ssh
sudo apt-get install rsync</code>
</pre>

###Java

Для работы Hadoop можно использовать либо 6 или 7 версию.
В данной статье будем работать с `OpenJDK 7` версии:

<code>sudo apt-get install openjdk-7-jdk</code>


Хотя можно использовать версию от `Oracle`.

И как?

Очищаем ОС от всех зависимостей OpenJDK 

<code>sudo apt-get purge openjdk*</code>

Устанавливаем `python-software-properties` который позволит добавлять новые PPA:


<code>sudo apt-get install python-software-properties</code>

Добавляем PPA с launchpad.net/~webupd8team/+archive/java
<pre>
<code>sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java7-installer</code>
</pre>

Подробнее: [INSTALL ORACLE JAVA 7 IN UBUNTU VIA PPA REPOSITORY](http://www.webupd8.org/2012/01/install-oracle-java-jdk-7-in-ubuntu-via.html)

###Создание отдельной учетной записи для запуска Hadoop

Мы будем использовать выделенную учетную запись для запуска `Hadoop`. Это не обязательно, но рекомендуется. Также предоставим новому пользователю права `sudo`, чтобы облегчить себе жизнь в будущем.

<pre>
<code>sudo addgroup hadoop
sudo adduser --ingroup hadoop hduser
sudo usermod -aG sudo hduser</code>
</pre>

Во время создания нового пользователя, необходимо будет ввести ему пароль.

###/etc/hosts

Нам необходимо, чтобы все узлы могли легко обращаться друг к другу. В большом кластере желательно использовать dns сервер, но для нашей маленькой конфигурации подойдет файл `hosts`. В нем мы будем описывать соответствие ip-адреса узла к его имени в сети. Для одного узла ваш файл должен выглядеть примерно так:

<pre>
<code>127.0.0.1       localhost
 
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
 
192.168.0.1 master</code>
</pre>

###SSH

Для управления узлами кластера `Hadoop` необходим доступ по `ssh`. Для созданного пользователя `hduser` предоставить доступ к `master`.
Для начала необходимо сгенерировать новый `ssh` ключ:

<code>ssh-keygen -t rsa -P ""</code>

Во время создания ключа будет запрошен пароль. Сейчас можно его не вводить.

Следующим шагом необходимо добавить созданный ключ в список авторизованных:

`cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys`

Проверяем работоспособность, подключившись к себе:

<code>ssh master</code>

###Отключение IPv6

Если не отключить IPv6, то в последствии можно получить много проблем.
Для отключения IPv6 в Ubuntu 12.04 / 12.10 / 13.04 нужно отредактировать файл `sysctl.conf`:

<code>sudo vim /etc/sysctl.conf</code>

Добавляем следующие параметры:

<pre>
<code># IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1</code>
</pre>

Сохраняем и перезагружаем операционную систему.

Но мне нужен IPv6!

Для того, чтобы отключить ipv6 только в hadoop можно добавить в файл `etc/hadoop/hadoop-env.sh`:

<code>export HADOOP_OPTS=-Djava.net.preferIPv4Stack=true</code>

###Установка Apache Hadoop

Скачаем необходимые файлы.
Актуальные версии фреймворка располагаются по адресу: http://www.apache.org/dyn/closer.cgi/hadoop/common/

На момент декабря 2013 года стабильной версией является 2.2.0.

Создадим папку `downloads` в корневом каталоге и скачаем последнюю версию:

<pre>
<code>sudo mkdir /downloads
cd downloads/
sudo wget http://apache-mirror.rbc.ru/pub/apache/hadoop/common/stable/hadoop-2.2.0.tar.gz</code>
</pre>

Распакуем содержимое пакета в `/opt`, переименуем папку и выдадим пользователю `hduser` права создателя:

<pre>
<code>sudo mv /downloads/hadoop-2.2.0.tar.gz /opt/
cd /opt/
sudo tar xzf hadoop-2.2.0.tar.gz
sudo mv hadoop-2.2.0 hadoop
chown -R hduser:hadoop hadoop</code>
</pre>

###Обновление $HOME/.bashrc

Для удобства, добавим в `.bashrc` список переменных:

<pre>
<code># Hadoop variables
export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-i386
export HADOOP_INSTALL=/opt/hadoop
export PATH=$PATH:$HADOOP_INSTALL/bin
export PATH=$PATH:$HADOOP_INSTALL/sbin
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_HOME=$HADOOP_INSTALL
export HADOOP_HDFS_HOME=$HADOOP_INSTALL
export YARN_HOME=$HADOOP_INSTALL</code>
</pre>

На этом шаге заканчиваются предварительные подготовки.

##Настройка Apache Hadoop

Все последующая работа будет вестись из папки /opt/hadoop.
Откроем `etc/hadoop/hadoop-env.sh` и зададим `JAVA_HOME`.

<pre>
<code>vim /opt/hadoop/etc/hadoop/hadoop-env.sh</code>
</pre>

Опишем, какие у нас будут узлы в кластере в файле etc/hadoop/slaves

<pre>
<code>master</code>
</pre>

Этот файл может располагаться только на главном узле. Все новые узлы необходимо описывать здесь.

Основные настройки системы располагаются в `etc/hadoop/core-site.xml`:

<pre>
<code>&lt;configuration>
    &lt;property>
       &lt;name>fs.defaultFS&lt;/name>
       &lt;value>hdfs://master:9000&lt;/value>
    &lt;/property>
&lt;/configuration></code>
</pre>


Настройки `HDFS` лежат в `etc/hadoop/hdfs-site.xml`:

<pre>
<code>&lt;configuration>
    &lt;property>
       &lt;name>dfs.replication&lt;/name>
       &lt;value>1&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>dfs.namenode.name.dir&lt;/name>
       &lt;value>file:/opt/hadoop/tmp/hdfs/namenode&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>dfs.datanode.data.dir&lt;/name>
       &lt;value>file:/opt/hadoop/tmp/hdfs/datanode&lt;/value>
    &lt;/property>
&lt;/configuration></code>
</pre>


Здесь параметр `dfs.replication` задает количество реплик, которые будут хранится на файловой системе. По умолчанию его значение равно 3. Оно не может быть больше, чем количество узлов в кластере.
Параметры `dfs.namenode.name.dir` и `dfs.datanode.data.dir` задают пути, где будут физически располагаться данные и информация в `HDFS`. Необходимо заранее создать папку `tmp`.

Сообщим нашему кластеру, что мы желаем использовать YARN. Для этого изменим `etc/hadoop/mapred-site.xml`:

<pre>
<code>&lt;configuration>
    &lt;property>
       &lt;name>mapreduce.framework.name&lt;/name>
       &lt;value>yarn&lt;/value>
    &lt;/property>
&lt;/configuration></code>
</pre>


Все настройки по работе `YARN` описываются в файле `etc/hadoop/yarn-site.xml`:

<pre>
<code>&lt;configuration>
    &lt;property>
       &lt;name>yarn.nodemanager.aux-services&lt;/name>
       &lt;value>mapreduce_shuffle&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.nodemanager.aux-services.mapreduce.shuffle.class&lt;/name>
       &lt;value>org.apache.hadoop.mapred.ShuffleHandler&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.resourcemanager.scheduler.address&lt;/name>
       &lt;value>master:8030&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.resourcemanager.address&lt;/name>
       &lt;value>master:8032&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.resourcemanager.webapp.address&lt;/name>
       &lt;value>master:8088&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.resourcemanager.resource-tracker.address&lt;/name>
       &lt;value>master:8031&lt;/value>
    &lt;/property>
    &lt;property>
       &lt;name>yarn.resourcemanager.admin.address&lt;/name>
       &lt;value>master:8033&lt;/value>
    &lt;/property>
&lt;/configuration></code>
</pre>


Настройки `resourcemanager` нужны для того, чтобы все узлы кластера можно было видеть в панели управления.

Отформатируем `HDFS`:

<pre>
<code>bin/hdfs namenode –format</code>
</pre>

Запустим `Hadoop` службы:

<pre>
<code>sbin/start-dfs.sh
sbin/start-yarn.sh</code>
</pre>

*В предыдущей версии Hadoop использовался скрипт sbin/start-all.sh, но с версии 2 он объявлен устаревшим.

Необходимо убедиться, что запущены следующие java-процессы:

<pre>
<code>hduser@master:/opt/hadoop$ jps
4868 SecondaryNameNode
5243 NodeManager
5035 ResourceManager
4409 NameNode
4622 DataNode
5517 Jps</code>
</pre>


Протестировать работу кластера можно при помощи стандартных примеров:

<pre>
<code>bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.2.0.jar</code>
</pre>

Теперь у нас есть готовый образ, который послужит основой для создания кластера.

Далее можно создать требуемое количество копий нашего образа.

На копиях необходимо настроить сеть. Необходимо сгенерировать новые MAC-адреса для сетевых интерфейсов и выдать и на них необходимые ip-адреса. В моем примере я работаю с адресами вида `192.168.0.X`.

Поправить файл `/etc/hosts` на всех узлах кластера так, чтобы в нем были прописаны все соответствия.

Пример

<pre>
<code>127.0.0.1       localhost
 
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
 
192.168.0.1 master
192.168.0.2 slave1
192.168.0.3 slave2</code>
</pre>

Для удобства, изменить имена новых узлов на `slave1` и `slave2`.

Как?

Необходимо изменить два файла: `/etc/hostname` и `/etc/hosts`.

Сгенерируйте на узлах новые SSH-ключи и добавьте их все в список авторизованных на узле `master`.

На каждом узле кластера, на котором будет запущен процесс `NameNode`, изменим значения параметра `dfs.replication` в `etc/hadoop/hdfs-site.xml`. Например, выставим везде значение 3.
etc/hadoop/hdfs-site.xml

<pre>
<code>&lt;configuration>
    &lt;property>
       &lt;name>dfs.replication&lt;/name>
       &lt;value>3&lt;/value>
    &lt;/property>
&lt;/configuration></code>
</pre>

Добавим на узле master новые узлы в файл `etc/hadoop/slaves`:

<pre>
<code>master
slave1
slave2</code>
</pre>


Когда все настройки прописаны, то на главном узле можно запустить наш кластер.

<pre>
<code>bin/hdfs namenode –format
sbin/start-dfs.sh
sbin/start-yarn.sh</code>
</pre>

На slave-узлах должны запуститься следующие процессы:

<pre>
<code>hduser@slave1:/opt/hadoop$ jps
1748 Jps
1664 NodeManager
1448 DataNode</code>
</pre>

Теперь у нас есть свой мини-кластер.

Давайте запустим задачу `Word Count`.
Для этого нам потребуется загрузить в `HDFS` несколько текстовых файлов.
Для примера, я взял книги в формате txt с сайта [Free ebooks — Project Gutenberg](http://www.gutenberg.org/).

###Тестовые файлы
<pre>
<code>cd /home/hduser
mkdir books
cd books
wget http://www.gutenberg.org/cache/epub/20417/pg20417.txt
wget http://www.gutenberg.org/cache/epub/5000/pg5000.txt
wget http://www.gutenberg.org/cache/epub/4300/pg4300.txt
wget http://www.gutenberg.org/cache/epub/972/pg972.txt
wget http://www.gutenberg.org/cache/epub/132/pg132.txt
wget http://www.gutenberg.org/cache/epub/1661/pg1661.txt
wget http://www.gutenberg.org/cache/epub/19699/pg19699.txt</code>
</pre>

Перенесем наши файлы в HDFS:

<pre>
<code>cd /opt/hadoop
bin/hdfs dfs -mkdir /in
bin/hdfs dfs -copyFromLocal /home/hduser/books/* /in
bin/hdfs dfs -ls /in</code>
</pre>


Запустим `Word Count`:

<pre>
<code>bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.2.0.jar wordcount /in /out</code>
</pre>

Отслеживать работу можно через консоль, а можно через веб-интерфейс ResourceManager'а по адресу http://master:8088/cluster/apps/

По завершению работы, результат будет располагаться в папке /out в HDFS. 
Для того, чтобы скачать его на локальную файловую систему выполним:

<pre>
<code>bin/hdfs dfs -copyToLocal /out /home/hduser/</code>
</pre>
