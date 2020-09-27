# Heartbeat

* heartbeat.yml

```yaml
output.elasticsearch:
  hosts: ["myEShost:9200"]
  username: "heartbeat_internal"
  password: "YOUR_PASSWORD" 
heartbeat.monitors:
- type: icmp
  schedule: '*/5 * * * * * *' 
  hosts: ["myhost"]
- type: tcp
  schedule: '@every 5s' 
  hosts: ["myhost:12345"]
  mode: any 
# 可选
setup.kibana:
  host: "mykibanahost:5601" 
  username: "my_kibana_user"  
  password: "{pwd}"
```



# Filebeat

> filebeat -e -c filebeat.yml -d "publish"

`PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1`.



```yml
# vi filebeat.yml
filebeat.prospectors:
- input_type: log
  paths:
  - /var/log/*.log

output.elasticsearch:
  hosts: ["localhost:9200"]
  
  
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/mysqld.log

output.elasticsearch:
  hosts: ["192.168.1.222:9200"]
  template.name: "filebeat"
  template.path: "filebeat.template.json"
  template.overwrite: false 
  

filebeat.inputs:
- type: log
  paths:
    - /usr/local/programs/logstash/logstash-tutorial.log

output.logstash:
  hosts: ["localhost:5044"]
```

