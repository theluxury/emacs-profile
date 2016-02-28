# Support Engineer Take Home Assignment

## Linux Pop Quiz

1: The rsa file that is in ~/.ssh/ on master is not in child aggregator yet. I found this out by trying

    ssh ip-10-0-1-110.ec2.internal -v
  
in the child aggregator and the log said it could not find identity file ~/.ssh/id_rsa. I copied ~/.ssh/id_rsa from master aggregator to child aggregator and ssh worked after.

2: I got the public ips using

    wget http://ipinfo.io/ip -qO -
    
The ips for the respective nodes are <br>
<b>Child Aggregator</b>: 52.90.63.53 <br>
<b>Leaf 1</b>: 52.90.121.31 <br>
<b>Leaf 2</b>: 54.175.54.74

3: I personally use iterms which allows one to broadcast commands across multiple terminal windows. If that is not an option, I can write a bash script that loops through the nodes and runs the command.

It would appear that the master aggregator is under the most load from top.
```
ubuntu@MasterAggregator:~$ top
top - 03:22:06 up 21:14,  2 users,  load average: 0.00, 0.02, 0.05
Tasks: 127 total,   1 running, 126 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.6 us,  0.2 sy,  0.0 ni, 98.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  15400880 total, 14938680 used,   462200 free,  6880092 buffers
KiB Swap: 10485756 total,       32 used, 10485724 free.  2370544 cached Mem
```

compared to other nodes which were similar to
```
ubuntu@ChildAggregator-1:~$ top
top - 03:36:29 up 1 day,  8:11,  1 user,  load average: 0.00, 0.02, 0.05
Tasks: 130 total,   1 running, 129 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.3 us,  0.1 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  15400880 total,  4065708 used, 11335172 free,   133840 buffers
KiB Swap: 10485756 total,        0 used, 10485756 free.  2948060 cached Mem
```

4: Processors: 4
```
ubuntu@MasterAggregator:~$ cat /proc/cpuinfo | grep processor | wc -l
4
```
Disk space: 126GB
```
ubuntu@MasterAggregator:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1      126G   45G   76G  38% /
```

RAM: 14GB
```
ubuntu@MasterAggregator:~$ free -h
             total       used       free     shared    buffers     cached
Mem:           14G        14G       450M       348K       6.6G       2.3G
```
Ubuntu Version: 14.04
```
ubuntu@MasterAggregator:~$ cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
```

5: Initially, even root could not write new files. I eventually tracked down the source of this to a full /tmp/ directory by

    ubuntu@MasterAggregator:~$ du -sch /tmp*

I deleted `/tmp/memsql-ops.tar.gz` since it seemed a safe file to delete. Then, I ran 
```
ubuntu@MasterAggregator:~$ df -hi
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/xvda1       8.0M  8.0M   885  100% /
```
to see that inodes were full. I restarted the node and afterwards root was able to write files. After that, the solution to this problem became tricky. There are few options to give ubuntu write access to root partition, none of them seemed great.
I could `sudo chmod a+w /` to give all users root write access, but that is potentially unsafe.
Changing the owner of the root partition of ubuntu would not work. The best option is most likely `sudo usermod -aG sudo ubuntu`. 
However, I didn't think giving ubuntu full sudo permission was wise. Here is a link describing why all the choices are bad,
as well as an alternative solution using ACL.  http://serverfault.com/questions/188537/giving-user-read-permissions-everywhere-linux
In real life, I would probably ask for help before proceeding, so therefore I left this problem 'blank'

6: The alias already existed on the node. But the command to do it would be

    alias memsql=’mysql -u root -h 127.0.0.1’
    
You can add a --prompt=”memsql> “ at the end to make it look nicer.

## Automating a MemSQL Task

8: The bash file code is
```
echo "Please input username:"
read user
echo "Password: (leave blank if no password):"
read password
echo "Host:"
read host
echo "Path to write sql file:"
read output_path
# first get the grantees
mysql -u "$user" -h "$host" --password="$password" -N -e 'select distinct grantee from information_schema.user_privileges' |
# for everything in line, write the grants to sql file.
while IFS='' read -r line || [[ -n "$line" ]]; do
    output=`mysql -u "$user" -h "$host" --password="$password" -N -e "show grants for $line"`
    echo "${output};" >>"$output_path"
done
```
It will prompt the user to input a username, password, host, and path to write the .sql file. The location of the file is `/home/ubuntu/grant_script.sh`

## Query Optimization

10: I ran `explain select ...` and found
```
| id   | select_type | table      | type              | possible_keys         | key           | key_len | ref   | rows   | Extra                                     | Query | RealTable | Cost   |
|   12 | SIMPLE      | partsupp   | ref               | suppkey_index         | suppkey_index | NULL    | NULL  | 801168 |                                           | NULL  | partsupp  |   NULL |
```
Note: The original line did not have suppkey_index and it was doing the 800,000 row query without an index. This was confirmed by running

    show create table partsupp;
    
To fix this, I created an index on the suppkey column in the partsupp table with

    create index suppkey_index on partsupp(suppkey);
    
and afterwards the query speed dropped to ~0.1s. I also tried adding other indexes, such as on `part.size`, however the other indexes
seemed to have no effect on performance.

## Answering a Support Question

11:

Hello Pat,

I’m sorry that you’ve come across this error. The problem is not on your end: MemSQL unfortunately currently does not support the Left() function. It does, however, support the Substring() function, and an equivalent query that would work is

    select substring(address, 1, 10) from customer where acctbal>10 limit 100;

Though we at MemSQL do try to make the experience as smooth as possible, there will on occasion be MySQL queries that do not work for MemSQL. A full list of supported MemSQL queries can be found at http://docs.memsql.com/latest/ref/functions/. We at MemSQL are always here to help out, and please feel free to contact us again if you have any questions!

-Mark Wang <br>
MemSQL Support Engineer <br>
mark@memsql.com<br>
