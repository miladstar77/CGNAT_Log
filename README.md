# CGNAT_Log

### NOTE: ********* this document has not completed yet ********
## overview

To collect CGNAT logs and store them in DB to search and find specific data faster.

In this project, Radius servers are used for AAA and Brases as ras. Also, we need some servers to collect logs and parse them into our appropriate data type, human-readable and store them in the clickhouse database.

Our users connect to Bras and Bras sends logs to its sensor called bras_sensor, each Bras has one sensor to collect Bras logs by Rsyslog and parse its logs.

Another sensor called Radius_sensor sniffs Radius traffic and stores logs by Tcpdump, then parse and extracts some specific radius attributes and stores them into radius DB called online user.

finally bras-report.sh collect data from radius.ou(online users) and bras sensor data and store them in the main DB.

For doing this project faster I use PyPy to run python code faster than usual.

# How to config:

### 1- Bras_sensor:
Please download all files (bras.py cgnet.conf extract.sh) and copy them into /home then follow the instruction:

required : python3 - rsyslog v8 and have access to radius_sensor & Main DB server
OS: Linux (resource should be considered according to the amount of traffic and logs generated by Bras)

Please configure Bras to send logs by Syslog format to bras_sensor (TCP or UDP port: 514). 

copy cgnat.conf file to /etc/rsyslog.d/.
Rsyslog parse logs(in this case Cisco Nat log format) and store them into /srv/log/cgnat/year.
```
cp cgnat.conf /etc/rsyslog.d/
```
restart rsyslog service:
```
systemctl restart rsyslog.service && systemctl enable rsyslog.service
```
create these directories and copy script file:
```
mkdir /srv/log/cgnat 
mkdir ~/bras directory 
cp brashostname.py ~/bras/brashostname.py 
cp bras.py ~/bras
mkdir ~/bras/bras-report 
mkdir ~/bras/radius-files
```
please change bras hostname in brashostname.py file. 

bras.py will open saved logs in  /srv/log/cgnat/year and connects to radius DB (online users table) then save the final file into /root/bras/bras-report and Main DB on the database server. (you can have more than one main DB, based on your data size)

to run scripts automatically do like this:
crontab runs bras.py and others clears old files:
```
*/2 * * * * root python3 /root/bras/bras.py &> /dev/null
2 3 * * * root echo "$(du -sh)" > /root/bras/rsyslog-backup.txt && find /srv/log/cgnat/sent/$(date \+\%Y)/ -ctime +1 -delete
3 2 * * * root echo "$(du -sh)" > /root/bras/report.txt && find /root/bras/bras-report/ -ctime +1 -delete
```
### 2- Radius_Sensor:

Please download all files (online-user.py pcap-screen.sh radiusDB.sh) and copy them into /home then follow the instruction:

required : clikhouse server and client – python3 ( pypy3 (pip– dpkt))- screen – tcpdump

to install clickhouse on your server please follow the clickhouse website: https://clickhouse.com/docs/en/install/
```
mkdir /etc/clickhouse-server/config.d/cgnat.xml
```
Configure clickhouse to listen on all NIC:
```
vi  /etc/clickhouse-server/config.d/cgnat.xml
```
copy this config to file:
```
<yandex>
        <listen_host>0.0.0.0</listen_host>
</yandex>
```
Install software:
```
apt-get -y update && apt-get -y install build-essential curl
cd /opt && curl -L https://downloads.python.org/pypy/pypy3.7-v7.3.5-linux64.tar.bz2 | tar xjf -
ln -s /opt/pypy3.7-v7.3.5-linux64/bin/pypy3 /usr/local/bin/pypy3
pypy3 -m ensurepip & pypy3 -m pip install --upgrade pip
pypy3 -m pip install dpkt
```
sniff received radius packets by Tcpdump and store them into ~/radius/pcap.

create:
```
mkdir ~/radius && mkdir ~/radius/pcap 
```
I use Screen to run Tcpdump in background session. use pcap-screen.sh file.
```
mkdir ~/scripts 
cp pcap-screen.sh ~/scripts
```
online-users.py will parse them and store data in Radius online user table and ~/radius/parsed-file.

create:
```
mkdir ~/radius/parsed-file
```
old pcap will move into /tmp/old-pcap.
```
mkdir /tmp/old-pcap 
````
everything is scheduled by crontab.
```
#radius sniffer
*/2 * * * * root pypy3 /root/radius/online-user.py &> /dev/null
1 3 * * * root echo "$(du -sh /tmp/old-pcap| cut -f1)" > /root/radius/old-pcap-removed-size.txt && find /tmp/old-pcap -ctime  +1 -delete
1 2 * * * root echo "$(du -sh /root/radius/parsed-file | cut -f1)" > /root/radius/parsed-file-removed-size.txt && find /root/radius/parsed-file -ctime +2 -delete
0 3 * * * root /root/scripts/radiusDB.sh
#@reboot root /root/scripts/pcap-screen.sh
```


### 3- Main database:

Please download all files (extract.sh) and copy them into /home then follow the instruction:

required :  clickhouse 
to install clickhouse on your server please follow the clickhouse website: https://clickhouse.com/docs/en/install/
```
vi  /etc/clickhouse-server/config.d/cgnat.xml
```
copy :
```
<yandex>
        <listen_host>0.0.0.0</listen_host>
</yandex>
```
create these directories:
```
mkdir /root/cgnat
mkdir /root/cgnat/final-report 
```
copy 
```
cp extract.sh to /root/cgnat
```
cron tabs:
```
#radius
*/5 * * * * root /root/cgnat/extract.sh &> /root/cgnat/log.txt
1   2 * * * root echo $(du -sh /root/cgnat/report) > /root/cgnat/report-size.txt  && find /root/cgnat/report/ -ctime +15 -delete
```
