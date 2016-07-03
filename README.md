# Customize Hadoop 2.6.0 on Ubuntu 14.04 within Ambari
If you wanna use Ambari to manage Hadoop 2.6.0 on Ubuntu 14.04, you may be disappointed at the unsupport by Ambari. Here we can follow Ambari HDP to extend Hadoop 2.6.0 ruuning on Ubuntu 14.04.  

##1. Build DEB Package
Normally you can get the package info and detail content of Hadoop 2.7.1 on Ubuntu 14.04 stack provided by HDP 2.2.0. For example, you can list all the content within the package and the package info shown as below.
<pre>
root@ambari1:/data/soft/hadoop/build# dpkg-deb -c hadoop_2.6.0.2.4.2.0-258_all.deb 
drwxr-xr-x root/root         0 2015-11-19 18:59 ./
drwxr-xr-x root/root         0 2015-11-19 18:59 ./usr/
drwxr-xr-x root/root         0 2015-11-19 18:59 ./usr/share/
drwxr-xr-x root/root         0 2015-11-19 18:59 ./usr/share/doc/
drwxr-xr-x root/root         0 2015-11-19 18:59 ./usr/share/doc/hadoop/
-rw-r--r-- root/root       142 2015-11-19 18:59 ./usr/share/doc/hadoop/changelog.Debian.gz
root@ambari1:/data/soft/hadoop/build# dpkg-deb -I hadoop_2.6.0.2.4.2.0-258_all.deb 
 new debian package, version 2.0.
 size 1008 bytes: control archive=412 bytes.
     277 bytes,     9 lines      control              
      75 bytes,     1 lines      md5sums              
 Package: hadoop
 Version: 2.6.0.2.4.2.0-258
 Architecture: all
 Maintainer: jenkins 
 Installed-Size: 25
 Depends: hadoop-2-6-0-2420258
 Section: misc
 Priority: extra
 Description: hadoop is a virtual package that brings hadoop-2-6-0-2420258 as a dependency.
 </pre>

 So that you can build new Hadoop 2.6.0 DEB package based on existing package. The following procedures are helpful for this.
 <pre>
 mkdir -p extract/DEBIAN 
 dpkg-deb -x package.deb extract/ 
 dpkg-deb -e package.deb extract/DEBIAN [...do something, e.g. edit the control file...] 
 mkdir build 
 dpkg-deb -b extract/ build/
 </pre>

 <pre>
 Usage: dpkg-deb [option ...] command

Commands:
  -b|--build [[directory> [[[deb>]   Build an archive.
  -c|--contents [[deb>              List contents.
  -I|--info [[deb> [[[cfile> ...]    Show info to stdout.
  -W|--show [[deb>                  Show information on package(s)
  -f|--field [[deb> [[[cfield> ...]  Show field(s) to stdout.
  -e|--control [[deb> [[[directory>] Extract control info.
  -x|--extract [[deb> [[directory>   Extract files.
  -X|--vextract [[deb> [[directory>  Extract & list files.
  -R|--raw-extract [[deb> [[directory>
                                   Extract control info and files.
  --fsys-tarfile [[deb>             Output filesystem tarfile.

  -?, --help                       Show this help message.
      --version                    Show the version.

[[deb> is the filename of a Debian format archive.
[[cfile> is the name of an administrative file component.
[[cfield> is the name of a field in the main control file.
 </pre>

##2.  mapreduce.jobhistory.recovery.store.class
mapdrecueV2中， 2.6.0版本没有提供org.apache.hadoop.mapreduce.v2.hs.HistoryServerLeveldbStateStoreService，需要替换为org.apache.hadoop.mapreduce.v2.hs.HistoryServerFileSystemStateStoreService.

此配置在Advanced mapred-site中可以手工配置默认值。


###3. 定制service stack中的metainfo.xml
例如，HDFS的metainfo.xml定制如下，注意其中的version必须跟#2中build的DEB package版本保持一致。

`<metainfo>`
`  <schemaVersion>2.0</schemaVersion>`
`  <services>`
`    <service> `
`      <name>HDFS</name> `
`      <version>2.6.0.2.4.2.0-258</version> `

      <osSpecifics>
        <osSpecific>
          <osFamily>any</osFamily>
          <packages>
            <package>
              <name>rpcbind</name>
            </package>
          </packages>
        </osSpecific>

        <osSpecific>
          <osFamily>redhat7,amazon2015,redhat6,suse11</osFamily>
          <packages>
            <package>
              <name>hadoop_2_4_*</name>
            </package>
            <package>
              <name>snappy</name>
            </package>
            <package>
              <name>snappy-devel</name>
            </package>
            <package>
              <name>lzo</name>
              <skipUpgrade>true</skipUpgrade>
            </package>
            <package>
              <name>hadooplzo_2_4_*</name>
            </package>
            <package>
              <name>hadoop_2_4_*-libhdfs</name>
            </package>
          </packages>
        </osSpecific>

        <osSpecific>
          <osFamily>debian7,ubuntu12,ubuntu14</osFamily>
          <packages>
            <package>
              <name>hadoop-2-6-0-2420258-client</name>
            </package>
            <package>
              <name>hadoop-2-6-0-2420258-hdfs-datanode</name>
            </package>
            <package>
              <name>hadoop-2-6-0-2420258-hdfs-journalnode</name>
            </package>
            <package>
              <name>hadoop-2-6-0-2420258-hdfs-namenode</name>
            </package>
            <package>
              <name>hadoop-2-6-0-2420258-hdfs-secondarynamenode</name>
            </package>
            <package>
              <name>hadoop-2-6-0-2420258-hdfs-zkfc</name>
            </package>
            <package>
              <name>libsnappy1</name>
            </package>
            <package>
              <name>libsnappy-dev</name>
            </package>
            <package>
              <name>hadooplzo-2-6-0-2420258</name>
            </package>
            <package>
              <name>libhdfs0-2-6-0-2420258</name>
            </package>
          </packages>
        </osSpecific>
      </osSpecifics>

    </service>
  </services>
`</metainfo>`

##4. YARN Log4J log4j.appender.EWMA
在YARN的log4j中，Ambari脚本使用了org.apache.hadoop.yarn.util.Log4jWarningErrorMetricsAppender作为EWMA。
<pre>
log4j.appender.EWMA=org.apache.hadoop.yarn.util.Log4jWarningErrorMetricsAppender
log4j.appender.EWMA.cleanupInterval=${yarn.ewma.cleanupInterval}
log4j.appender.EWMA.messageAgeLimitSeconds=${yarn.ewma.messageAgeLimitSeconds}
log4j.appender.EWMA.maxUniqueMessages=${yarn.ewma.maxUniqueMessages}
</pre>

Hadoop 2.6.0没有相应的log类，仅仅是在2.7.1新加的EWMA类。所以需要定制此类。
<a href="Log4j/org/apache/hadoop/yarn/util/Log4jWarningErrorMetricsAppender.java">org.apache.hadoop.yarn.util.Log4jWarningErrorMetricsAppender</a>


##5.其他配置
####1.HDFS的权限
默认是打开的，如果你的应用需要disable permission,需要在配置中设置：
dfs.permissions.enabled = false

####2. YARN Node Manager不能识别HDFS
处理方式：把HDFS Lib里的包放到YARN里或者部署一个HDFS Client服务

####3. Resource Manager
Resource Manager的默认端口不是8032，需要跟实际的应用匹配上。yarn.resourcemanager.address=ambari4.cn.XXX.com:8032

##6. Trouble Shooting
####1. 常用命令
 从根目录开始查找所有扩展名为.log的文本文件，并找出包含”ERROR”的行

` find / -type f -name "*.log" | xargs grep "ERROR" `

例子：从当前目录开始查找所有扩展名为.in的文本文件，并找出包含”thermcontact”的行

` find . -name "*.in" | xargs grep "thermcontact" `

####2. linux查看文件夹下的文件个数
` ls -l | grep '^-'| wc -l `

` ls -l | grep -c '^-'  `

ls -l 长列表输出该目录下文件信息(注意这里的文件，不同于一般的文件，可能是目录、链接、设备文件等)

grep ^- 这里将长列表输出信息过滤一部分，只保留一般文件，如果只保留目录就是 ^d。-c命令可以直接计算过滤部分的个数。

wc -l 统计输出信息的行数，因为已经过滤得只剩一般文件了，所以统计结果就是一般文件信息的行数，又由于一行信息对应一个文件，所以也就是文件的个数