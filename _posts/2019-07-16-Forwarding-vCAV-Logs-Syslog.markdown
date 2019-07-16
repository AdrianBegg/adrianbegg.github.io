---
layout: post
title:  "Forwarding vCloud Availability logs to a remote syslog daemon"
date:   2019-07-16 16:35:24 +0200
author: Adrian Begg
categories: [vCloud Availability]
tags: [vcpp,vCAV,vCloud]
---
{% capture words %}
  {{ page.content | number_of_words | minus: 250 }}
{% endcapture %}
{% unless words contains "-" %}
  {{ words | plus: 250 | append: " words" }}
{% endunless %}

By default, in vCloud Availability 3.X there is no mechanism via the API/GUI to configure a remote syslog for forwarding the application logs which could be an issue if you require this for compliance reasons or to simplify troubleshooting. With some guidance from the VMWare Project Presto team I was able to successfully get the H4 service logs forwarded with relative ease using the following.

vCloud Availability uses [Logback](https://logback.qos.ch/) for service logging and the the logback.xml configuration file needs to be changed for each of the services for full service coverage. The logs can still be written to the local logging files however you can send copies to a remote syslog server for data warehousing.

| Appliance        | Service            | Configuration Path                           | Recommended Suffix Pattern                                            |
| :--------------- |:------------------:| :--------------------------------------------|:----------------------------------------------------------------------|
| vCAV Manager     | cloud.service      | /opt/vmware/h4/cloud/config/logback.xml      | [C4] [%X{operationID}] [%thread] %-40.40logger{39} : %msg             |
| vCAV Manager     | manager.service    | /opt/vmware/h4/manager/config/logback.xml    | [H4-Manager] [%X{operationID}] [%thread] %-40.40logger{39} : %msg     |
| vCAV Replicator  | replicator.service | /opt/vmware/h4/replicator/config/logback.xml | [H4-Replicator] [%X{operationID}] [%thread] %-40.40logger{39} : %msg  |
| vCAV Tunnel      | tunnel.service     | /opt/vmware/h4/tunnel/config/logback.xml     | [H4-Tunnel] [%X{operationID}] [%thread] %-40.40logger{39} : %msg      |

**Step 1.** Logon to the vCAV appliance (e.g. vCAV Manager) and edit the relevant logback.xml using vi (# vi /opt/vmware/h4/cloud/config/logback.xml) and add the following directly under the `<configation>` element editing the `<syslogHost>` to match the IP of the Remote Syslog Daemon and the `<suffixPattern>` to match the recommended suffix pattern for the service:
```
<appender name="SYSLOG" class="ch.qos.logback.classic.net.SyslogAppender">
     <syslogHost>192.168.88.10</syslogHost>
     <port>514</port>
     <facility>LOCAL7</facility>
     <suffixPattern>[C4] [%X{operationID}] [%thread] %-40.40logger{39} : %msg</suffixPattern>
</appender>
```
**Step 2.** Locate the element `<root level="INFO">` in the logback.xml file and add `<appender-ref ref="SYSLOG" />` element below the "FILE" appender.
```
<root level="INFO">
   <appender-ref ref="FILE" />
   <appender-ref ref="SYSLOG" />
</root>
```
**Step 3.** The file should appears similar to the below example, save the file and restart the service using the systemctl restart cmd : `#systemctl restart cloud.service`

![Example logback.xml for the cloud.service sending default logging to syslogd @ 192.168.88.10](/assets/vCAV-EnableSyslog-C4-logbackxml.png)

**Step 4.** Repeat the process for each of the services and verify on your syslog server that events are being received.
