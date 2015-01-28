How to configure hdfs and mapreduce on Mesos with Ansible 


We last talked about setting up Mesos and Zookeeper with Ansible.  This time we are going to talk about installing and getting hdfs and mapreduce to run on top of Mesos.


Prerequisites and assumptions

All the ansible playbooks and config files can be downloaded with this command

      git clone git://github.com/sc-sf/mesos-hdfs-ansible.git

This git repo continues on where we left off from configuring Mesos and Zookeeper.  At the bottom of this guide, you can see the directory layout of this repo. While most of the prerequisite items from the last guide still apply, you will have to download hadoop-2.5.0-cdh5.2.0.tar.gz from here http://archive.cloudera.com/cdh5/cdh/5/ for this repo to work.


Setting up hdfs user and unpack Hadoop package.

Before we start creating the hdfs user on the remote machines using the playbook, we need to setup the same hdfs user account on the local ansible server as a boilerplate. This can be done easily with the included useradd_hdfs_local.yml playbook. It creates 2 groups hdfs (primary group) and hadoop. It also generates the ssh user key pair which is by default passwordless using rsa with 2048-bits in size. In case you need to login locally or su as the hdfs user, the password hashed using grub-crypt is also included as well. The only thing is that the ansible's authorized_key module doesn't seem to run at all when the id_rsa.pub key is specified using full path. You'll need to create it manually should you find it useful to ssh locally as hdfs onto the ansible server. Permission of the authorized_key has to be 600 or else it won't get used at all and falls back to password authentication. The last 2 user related tasks - one is the copy of private key, which is optional. It is needed only when you think you might need to ssh to other machines from that remote machine. The second is setting up some environment variables making it convenient for hdfs user to run Hadoop commands without having to specifying the full path.


      # ------------------configure the hdfs user and unpack hadoop package--------------------------

      - name: create hdfs and hadoop user groups
        group: name={{item}} state=present
        with_items:
          - hdfs
          - hadoop

      # putting in wheel group makes hdfs user a sudoer 
      - name: create hdfs user - password is "password1234" hashed using "grub-crypt --sha-512"
        user: name=hdfs group=hdfs groups=hdfs,hadoop,wheel append=yes state=present password=$6$AGLYCh0AM8JGAN46$g3nU4C0wUbBxnovk1wp6cZdVv0kOKwdDZRDx0LZLRYtmZLMAxID6i8u1etlg0OTD7EpCfMYWAI7Urme1WqRQP/

      # create authorized key based on the pub key of boilerplate hdfs user on ansible server - chmod is 600 by default
      - name:  create authorized_keys file for hdfs user
        authorized_key: user=hdfs key="{{ lookup('file', 'id_rsa.pub') }}"     # " # the last double quote  

      - name: copy private key
        copy: src=id_rsa owner=hdfs group=hdfs mode=600 dest=/home/hdfs/.ssh

      - name: setup hadoop env variables in hdfs user's bash_profile
        template: src=bash_profile.j2 dest=/home/hdfs/.bash_profile owner=hdfs group=hdfs mode=644

      - name: extract hadoop_cdh_gz package - can take a while...
        unarchive: src={{hadoop_cdh_gz}} copy=yes dest=/opt owner=hdfs group=hdfs # mode=755  

      - name: fix owner permission thing because unarchive above ignores it
        file: path=/opt/{{hadoop_cdh}} state=directory group=hdfs owner=hdfs recurse=yes


Once hadoop is unpacked, change some default settings a bit to use mapreduce1 instead of the default mapreduce2


      # -----------------------------change the default mapreduce2 to mapreduce1---------------------------------------

      - name: change default mapreduce2 to mapreduce1 - 1 of 4
        shell: cd /opt/hadoop-2.5.0-cdh5.2.0 ; sudo -u hdfs mv bin bin-mapreduce2 ; \
          sudo -u hdfs mv examples examples-mapreduce2

      - name: change default mapreduce2 to mapreduce1 - 2 of 4
        shell: cd /opt/hadoop-2.5.0-cdh5.2.0 ; sudo -u hdfs ln -s bin-mapreduce1 bin ; \
          sudo -u hdfs ln -s examples-mapreduce1 examples

      - name: change default mapreduce2 to mapreduce1 - 3 of 4
        shell: cd /opt/hadoop-2.5.0-cdh5.2.0/etc ; sudo -u hdfs mv hadoop hadoop-mapreduce2 ; \
          sudo -u hdfs ln -s hadoop-mapreduce1 hadoop

      - name: change default mapreduce2 to mapreduce1 - 4 of 4
        shell: cd /opt/hadoop-2.5.0-cdh5.2.0/share/hadoop ; sudo -u hdfs rm mapreduce ; \
          sudo -u hdfs ln -s mapreduce1 mapreduce

      # the cdh5 has hdfs binary file in default mapreduce2 directory... make copy to mapreduce1
      - name: copy hdfs binary from mapreduce2 to mapreduce1 dir
        shell: sudo -u hdfs cp {{hadoop_bin_dir}}/../bin-mapreduce2/hdfs {{hadoop_bin_dir}}


Configure hdfs and mapreduce

There are three configuration files that we need to modify: core-site.xml, hdfs-site.xml and mapred-site.xml. You can find the values of the ansible variables defined in these ansible templates in the "all" variable file. Their names should give you a good picture of their purpose. Below are just some of them. Notice the leading_master (the namenode) variable is referenced several times. The mapred.mesos.executor.uri references the same url and the same port 8020 as the fs.defaultFS, pointing to where the pre-packaged mapreduce binary is located. As far as running Hadoop on top of Mesos, the two components (Hadoop distributed file system (hdfs) and Hadoop mapreduce) from the same Hadoop package are handled a bit differently . The Hadoop dfs runs on each of the nodes in the cluster as either a namenode or datanode. The mapreduce tasks are sent and executed by Mesos' executor to the slaves. When slaves run these tasks for the first time, they need to download the mapreduce binaries from the hdfs. Because of this, the mapreduce binaries have to be uploaded before hand and the url for the download is specified as the mapred.mesos.executor.uri property of the mapred-site.xml file.

core-site.xml: 
fs.defaultFS=> hdfs://{{leading_master}}:8020

mapred-site.xml: 
mapred.job.tracker=> {{leading_master}}:9001 
mapred.mesos.master=> zk://{{zk_ensemble_url}}/mesos
mapred.mesos.executor.uri=> hdfs://{{leading_master}}:8020/{{hadoop_cdh_gz}}

The last ansible tasks of this section takes care of configuring the directory volumes and the appropriate namenode and datanode init script to masters and slaves respectively.


      # -----------------------------configure the hadoop file system and mapreduce-------------------------------------

      - name: core-site.xml template
        template: src=core-site.xml.j2 dest={{hadoop_config_dir}}/core-site.xml owner=hdfs group=hdfs mode=644

      - name: hdfs-site.xml template
        template: src=hdfs-site.xml.j2 dest={{hadoop_config_dir}}/hdfs-site.xml owner=hdfs group=hdfs mode=644

      - name: mapred-site.xml template
        template: src=mapred-site.xml.j2 dest={{hadoop_config_dir}}/mapred-site.xml owner=hdfs group=hdfs mode=644

      - name: hadoop-env.sh template
        template: src=hadoop-env.sh.j2 dest={{hadoop_config_dir}}/hadoop-env.sh owner=hdfs group=hdfs mode=644

      - name: create namenode data directories
        file: path={{item}} state=directory group=hdfs owner=hdfs mode=755
        with_items: 
          nn_data_dir
        when: inventory_hostname in groups['masters']

      # make sure namenode daemon is NOT running when formating 
      - name: hadoop-namenode.conf init script template
        template: src=hadoop-namenode.conf.j2 dest=/etc/init/hadoop-namenode.conf owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']

      - name: create datanode data directories
        file: path={{item}} state=directory group=hdfs owner=hdfs mode=755
        with_items: 
          dn_data_dir
        when: inventory_hostname in groups['slaves']

      - name: hadoop-datanode.conf init script template
        template: src=hadoop-datanode.conf.j2 dest=/etc/init/hadoop-datanode.conf owner=0 group=0 mode=644
        when: inventory_hostname in groups['slaves']
        notify:
          - start hadoop-datanode


Take care of some of minor bugs from the version of Hadoop we are running

The first has to do with setting the NullAppender. It complains about the NullAppender value not being found. Simply add the following line to log4j.properties file.

      log4j.appender.NullAppender=org.apache.log4j.varia.NullAppender

The second complains about the class path containing multiple SLF4J bindings. It's a good idea to first take a look and see the full path of these duplicates as in some cases they might be symlinks point to the same file. But in our case, they are actual dups in different location. So we just need to remove one of them and that'll take care of this error.

There's actually a third warning complaining about not being able to load the native-hadoop library. However, as the warning suggests, the builtin-java classes will be used where applicable, so it doesn't really effect running Hadoop.


      # ---------------take care of some bugs of running the current version of hadoop -------------------

      - name: set NullAppender in log4j.properties
        template: src=log4j.properties.j2 dest={{hadoop_config_dir}}/log4j.properties owner=hdfs group=hdfs mode=644

      - name: remove duplicate slf4j-log4j12-1.7.5.jar
        file: path={{hadoop_slf4j_jar}} state=absent


Copy the Hadoop-Mesos bridge library into the Hadoop package before repackaging

The version of this library is hadoop-mesos-0.0.8.jar which is also included in the download and it needs to be copied into the path of hadoop_home/share/hadoop/common/lib of the package.
After all the configurations and the library jar file are in place, the tgz file can then be generated and ready to get uploaded to the hdfs.

As you can see, the whole ansible playbook does a lot for us. But before the famous mapreduce wordcount example can be carried out, we need to format the hdfs and upload the modified Hadoop package.

Before formatting hdfs, the namenode daemon can't be running. You need to execute the hadoop or hdfs command as the hdfs user. So either you login as hdfs or use sudo. After the file system is formatted, we can now start it. As soon as the file system is up and running, we can now cd to directory where the modified hadoop package is located and uploading it to hadoop dfs.


      sudo stop hadoop-namenode
      sudo -u hdfs hadoop namenode -format
      sudo start hadoop-namenode
      cd /home/hdfs
      sudo -u hdfs hadoop fs -put hadoop-2.5.0-cdh5.2.0.tar.gz /hadoop-2.5.0-cdh5.2.0.tar.gz


If everything works as expected, you can run the hadoop version of the ls command to see the uploaded package. You can also run the dfsadmin command to get some info of the file system. It will also show you the slaves that we are connected as well. This assumes that the datanode daemon on each slave is started automatically, which is done by the ansible playbook.

      hadoop fs -ls /
      hadoop dfsadmin -report



      # ---------------in order to run mapreduce on top of mesos,----------------------------------------------------
      # ---------------the archived of the fully configured hadoop package-------------------------------------------
      # ----------------need to be uploaded onto hadoop file system--------------------------------------------------

      - name: copy hadoop-mesos-0.0.8.jar needed for communication between hadoop and mesos
        copy: src=hadoop-mesos-0.0.8.jar owner=hdfs group=hdfs mode=644 \
          dest=/opt/hadoop-2.5.0-cdh5.2.0/share/hadoop/common/lib

      - name: repackage hadoop as tgz file for uploading onto hadoop file system
        shell: cd /opt ; sudo tar -czf /home/hdfs/{{hadoop_cdh_gz}} {{hadoop_cdh}}
        when: inventory_hostname in groups['namenode']

      - name: make hdfs owner of the repackaged hadoop tar before uploading to hadoop fs
        file: path=/home/hdfs/{{hadoop_cdh_gz}} state=file group=hdfs owner=hdfs mode=644
        when: inventory_hostname in groups['namenode']


Now we are ready to run our first wordcount example


      echo "Hello World Bye World" > /tmp/file0
      echo "Hello Hadoop Goodbye Hadoop" > /tmp/file1
      sudo -u hdfs hdfs dfs -mkdir -p /user/joe/data
      sudo -u hdfs hdfs dfs -copyFromLocal /tmp/file? /user/joe/data


Start jobtracker manually will run in the foreground and output the logs right on the terminal. This is useful when testing a mapreduce job.

      sudo -u hdfs hadoop jobtracker

Then on another terminal, run the wordcount example. The actual example jar file used below is in /opt/hadoop-2.5.0-cdh5.2.0/share/hadoop/mapreduce1 directory. You might have to specify the full path. 

      sudo -u hdfs hadoop jar hadoop-examples-2.5.0-mr1-cdh5.2.0.jar wordcount /user/foo/data /user/foo/out

      sudo -u hdfs hdfs dfs -ls /user/foo/out
      sudo -u hdfs hdfs dfs -cat /user/foo/out/part*

   

