Observability / Monitoring:

1.) why do we need monitoring?
    -> Proactively detecting issue before the actual application crashes

-> if there is an issue reported, e.g.: site slowness, site not opening, some transactions failing

2.) What do we monitor?

Observability -
    -> Metrics - CPU stats, Memory / RAM stats, Storage stats, network byte in /out, IOPS

    Tools which were used to monitor:
    
    Open Source tools:
        - Nagios
        - Zabbix
        - Prometheus (monitoring) & Grafana (dashboard)

    Cloud:
        - CloudWatch in AWS
        - Azure Monitor in Azure

    Enterprise tools:
        - DataDog
        - Splunk (Cisco Company)
        - NewRelic
        - AppDynamics
        - Dynatrace
        - IBM Tivoli
        - victoriametrics

    -> Logs Monitoring - you have an app running on 10 EC2s.. This application is serving live traffic... suddenly you start getting user reported issues that their login is failing intermittently... 
    
    how would you troubleshoot this? 

    Without having a central logs monitoring solution:
        - 3 of us would sit together, we would distribute the servers and view the application logs and try to find the exact error

    With a logs monitoring solution:
        - e.g., splunk agent would run on all 10 ec2s, those agents will have have config file, wherein you specify that which logs file/s to sync to the central splunk server

        - central splunk server will have a dashboard where in you would be able to view the logs

    ELK (elastic search - search database + logstash + kibana - dashboard) / Elastic Stack (filebeat / fluentbit / fluentd + Elastic search + kibana) - Open Source & Enterprise - Elastic 

    -> Traces / Application Performance Monitoring (APM): 
        - They want to view detailed application stats like, number of api calls processed in a min, time taken between application to the database for a transaction to complete, 504 gateway timeout, how does the request from load balance to application to database goes, if there is a lag


------

If I have to monitor the stats of my application running on 5 EC2 and also monitor the RDS instance where in my database in running, I have a lambda function which runs a specific workflow in AWS, how to do this?

--> CloudWatch

timeseries - it captures the cpu / memory stats at 10:49:00, 10:54, 10:59


