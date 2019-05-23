AnarchyApe
==========

Fault injection tool for hadoop cluster from yahoo anarchyape

Pre-requisites
-----------

- Java JDK >= 1.7
- pdsh
- stress-ng for CPU and Memory hog

Compilation 
-----------
[Java with maven]
```
mvn package
mv target/anarchyape.jar ape.jar

```

[Java manual]
```
cd src/main/java
# download log4j-1.4.12.jar and apache commons-cli-1.2.jar

java -cp . ape/Main.java

rm ape.jar

# Use either (preferred): 
jar cfm ape.jar META-INF/MANIFEST.MF ape META-INF/services org
# either :
javac -cp .:log4j-1.4.12.jar:commons-cli-1.2.jar ape/*.java
```

[Perl]
```
   perl Makefile.PL
   cpan -i JSON
   make
   make test
   make install
```

Running 
-------
[Perl]
./ape.pl [remote_ip_list_file]

[Java]
```
java -jar ape.jar [commands]

log file: anarchyape.log

(old way)
java -cp .:log4j-1.4.12.jar ape/Main
```

(Local run)
```
java -jar ape.jar -L -S 100 5
```
injects slow network with delay 100 milliseconds for 5 seconds.

(Remote run)
Install pdsh:
```
yum install pdsh
apt-get install pdsh

cp ape /usr/local/bin/ape
chmod go-rwx /usr/local/bin/ape

java -jar ape.jar -R node1,node2,node3 -S 100 5 eth0
# creates a script to run on the remote hosts:
pdsh -Rssh -w node1,node2,node3 '/usr/local/bin/ape -L -S 100 5 eth0'
```

It seems not working as the remote hosts do not have /usr/local/bin/ape file.

Remote nodes can be specified in XML format: cluster-ip-list.xml

Currently, to create a scenario, the user constructs a shell
script specifying the types of errors to be injected or failures to be simulated, one after another. A sample line in a
scenario file could be as follows:

```
java -jar ape.jar -remote cluster-ip-list.xml -F lambda -k lambda
	where the -F is a “Fork Bomb” injection, the -k is a “Kill
	One Node” command, and the lambda specifies the failure rates.
```
Users can define lambda parameters by computing Mean
Time Between Failures (MTBF) of a system. MTBF is defined to be the average (or expected) lifetime of a system
and is one of the key decision-making criteria for data center infrastructure systems [1]. Equipment in data centers
is going to fail, and MTBF helps with predicting which systems are the likeliest to fail at any given moment. Based on
previous failure statistics, users can develop an estimate of
MTBF for various equipment failures; however, determining
MTBFs for many software failures is challenging.

[1] W. Torell and V. Avelar. Performing effective MTBF comparisons for data center infrastructure.
http://www.apcmedia.com/salestools/ASTE-5ZYQF2_R1_EN.pdf.

Available Commands 
------------------
Here are some common failures in Hadoop environments:
```
# Data node is killed
# Application Master (AM) is killed
# Application Master is suspended
# Node Manager (NM) is killed
# Node Manager is suspended
# Data node is suspended
# Tasktracker is suspended
# Node panics and restarts
# Node hangs and does not restart
# Random thread within data node is killed
# Random thread within data node is suspended
# Random thread within tasktracker is killed
# Random thread within tasktracker is suspended
# Network becomes slow
tc qdisc add dev eth0 root netem delay 100.0ms && sleep 30.0 && tc qdisc del dev eth0 root netem
# Network is dropping significant numbers of packets
# Network disconnect (simulate cable pull)
iptables -A INPUT -p tcp --dport 8080 -j DROP # block port
iptables -D INPUT -p tcp --dport 8080 -j DROP # unblock port
# One disk gets VERY slow
# CPU hog consumes x% of CPU cycles, for example by running stress-ng command remotely with :
stress-ng -c 1 --verify -t 1m -v
# Mem hog consumes x% of memory, for example by running stress-ng command remotely with :
stress-ng --vm 4 --vm-bytes 90% --vm-method all --verify -t 1m -v
# Corrupt blocks of a given file on rw-mounted disk
```
Command line options:
```
usage: ape [options] ... <failure command>
           options:
 -u,--udp-flood <hostname> <port> <duration>       Flood the target
                                                   hostname with a DoS attack.  For proper effect, use the -R flag and
                                                   designate more than one host.
 -C,--corrupt-block <meta/ord> <size> <offset>     Corrupt a random HDFS
                                                   block file with a size in bytes as the 2nd arg and offset in bytes as the
                                                   3rd argument
 -d,--network-disconnect <time> <nic>              Disconnect the network
                                                   for a certain period of time specified in the argument on a given network
                                                   interface, and then resumes
 -p,--network-drop <percentage> <duration> <nic>   Drops a specified
                                                   percentage of all inbound network packets for a duration specified in
                                                   seconds.
 -c,--corrupt-file <file> <size> <offset>          Corrupt the file given
                                                   the address as the first argument, size as the 2nd arg, and offset as the
                                                   3rd argument
 -e,--continue-node <NodeType>                     Continues a tasktracker
                                                   or a datanode at the given hostname that has already been suspended
 -k,--kill-node <nodetype>                         Kills a datanode,
                                                   tasktracker, jobtracker, or namenode.
 -s,--suspend-node <NodeType>                      Suspends a tasktracker
                                                   or a datanode at the given hostname
 -r,--remount                                      Remounts all
                                                   filesystems as read-only
 -S,--network-slow <delay> <duration> <nic>        Delay all network
                                                   packet delivery by a specified amount of time (in milliseconds) for a
                                                   period specified in seconds on a given network interface
 -F,--forkbomb                                     Hangs a host by
                                                   executing a fork bomb
 -L,--local                                        Run commands locally
 -P,--panic                                        Forces a kernel panic
                                                   and does not restart the system.
 -R,--remote <HostnameList>                        Run commands remotely
 -V,--version                                      Displays the version
                                                   number
 -h,--help                                         Displays this help menu
 -t,--touch                                        Touches a file called
                                                   /tmp/foo.tst
 -v,--verbose                                      Turn on verbose mode
command:
 -fb <lambda> -k <lambda>	fork bomb
 -kp <lambda> -k <lambda>	kernel panic
 -r <lambda> -k <lambda>	remount root as read only
 -kn <lambda> -k <lambda>	kill a node process
 -dos <lambda> -k <lambda>	denial of service by launching 4 bombarding threads
 -cb <lambda> -k <lambda>	corrupt a random HDFS block
 -cf <lambda> -k <lambda>	corrupt a file at the given address
 -nic <lambda> -k <lambda>	interface
 -p <lambda> -k <lambda>	packet drop
```

Example of usage local mode
------------------
```
#java -jar ape.jar -h
#java -jar ape.jar -L -p,--network-drop <percentage> <duration> <nic>
java -jar ape.jar -L -p 80 30 eth0
#java -jar ape.jar -R -S <delay in millisec> <duration in sec> <nic>
#java -jar ape.jar -L -S 100 30 eth0
#java -jar ape.jar -L -d,--network-disconnect <time in seconds> <nic>
#java -jar ape.jar -L -d 30 eth0
#java -jar ape.jar -L --remount 
#java -jar ape.jar -L --panic 
#java -jar ape.jar -L --forkbomb

#java -jar ape.jar -L -c,--corrupt-file <file> <size> <offset>
#java -jar ape.jar -L --corrupt-file <file> <size> <offset>
#java -jar ape.jar -L -C,--corrupt-block <meta/ord> <size> <offset>

#java -jar ape.jar -R precise386 -S 100 30 eth0
#java -jar ape.jar -R cluser-ip-list.xml -S 100 5 eth0

#java -jar ape.jar -remote cluster-ip-list.xml -fb lambda -k lambda
#java -jar ape.jar -remote cluster-ip-list.xml -F lambda

#java -cp .:log4j.jar ape/Main -h
#java -cp .:log4j.jar ape/Main -R slaves
#java -cp .:log4j.jar ape/Main -V
```

Example of usage remote mode
------------------
```
#java -jar ape.jar -h
#java -jar ape.jar -R -S <delay in millisec> <duration in sec> <nic>
java -jar ape.jar -R precise386 -S 100 30 eth0
#java -jar ape.jar -R cluser-ip-list.xml -S 100 5 eth0

#java -jar ape.jar -remote cluster-ip-list.xml -fb lambda -k lambda
#java -jar ape.jar -remote cluster-ip-list.xml -F lambda
#java -jar ape.jar -L -v
#java -jar ape.jar -R -F
#java -jar ape.jar -R -S 1 1 eth0

#java -cp .:log4j.jar ape/Main -h
#java -cp .:log4j.jar ape/Main -R slaves
#java -cp .:log4j.jar ape/Main -remote slaves -v
#java -cp .:log4j.jar ape/Main -V
```
