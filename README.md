# telemetryMDU
 Telemetry TIG stack for students of the Mälardalens Universitet (MDU) Västerås

![](https://github.com/dankrauscisco/telemetryMDU/blob/master/png/telemetry_tig_lab.drawio.png)

# Introduction

This repository contains a Docker Compose file to spin up Telegraf, InfluxDB, and Grafana.
Grafana will be pre-provisioned using the Docker Compose file and the datasource.yml file within the grafana/grafana-provisioning/datasources folder.
Telegraf will be pre-provisioned using the Docker Compose file and the telegraf.conf file within the conf/telegraf folder.
Influxdb will be pre-provisioned using the Docker Compose file.

# How to use the repository

1. Clone the repository
```
git clone https://github.com/dankrauscisco/telemetryMDU.git
```

2. Change into the cloned directory

```
cd telemetryMDU
```

3. Edit the "urls" section in the telegeraf.conf file within the conf/telegraf folder to reflect the IP address of the host OS running the docker containers

```
# Outputs to influxdb

[[outputs.influxdb_v2]]
  urls = [ "http://<IP address>:8086" ]
```  

4. Dial-out telemetry

    The telegraf.conf file within the conf/telegraf folder has a section that specifies on which ports telegraf will listen to dial-out telemtry and which transport protocol it will use.
    
    Apply any changes as needed.

```
# gRPC dial-out (routers sending MDT to telegraf)

[[inputs.cisco_telemetry_mdt]]
  transport = "grpc"
  service_address = ":57000"

# TCP dial-out (routers sending MDT to telegraf)

[[inputs.cisco_telemetry_mdt]]

  transport = "tcp"
  service_address = ":57001"
```

5. Dial-in telemetry

    The telegraf.conf file within the conf/telegraf folder has a section that specifies the following parameters for dia-in telemetry.
    These telemetry collections towards the network devices will start/stop with the telegraf docker container.

    [[inputs.gnmi]]

    - addresses; list of IP:port for dial-in target routers
    - username/password; authentication for dial-in telemetry 

    [[inputs.gnmi.subscription]]

    - To find additional models use gnmic (https://gnmic.openconfig.net/) or ANX (https://github.com/cisco-ie/anx/)

    Apply any changes as needed.

```
# gRPC dial-in (telegraf subscribing with gNMI to routers)

[[inputs.gnmi]]
  addresses = ["192.168.101.141:57400", "192.168.101.142:57400", "192.168.101.143:57400", "192.168.101.144:57400"]
  username = "cisco"
  password = "cisco123"

# OC interface counters                                 
                                                        
[[inputs.gnmi.subscription]]                            
  name = "openconfig-interfaces:interfaces"             
  origin = "openconfig-interfaces"                      
  path = "/interfaces/interface/state/counters"         
  subscription_mode = "sample"                          
  sample_interval = "10s"                               
                                                        
# OC CPU counters                                       
                                                        
[[inputs.gnmi.subscription]]                            
  name = "openconfig-platform:components"               
  origin = "openconfig-platform"                        
  path = "/components/component/cpu"                    
  subscription_mode = "sample"                          
  sample_interval = "10s"                               
                                                        
# IOS XR memory counters                                
                                                        
[[inputs.gnmi.subscription]]                            
  name = "Cisco-IOS-XR-nto-misc-oper:memory-summary"    
  origin = "Cisco-IOS-XR-nto-misc-oper"                 
  path = "/memory-summary/nodes/node/detail"            
  subscription_mode = "sample"                          
  sample_interval = "10s"                               
```

6. Bring up the TIG stack

```
docker-compose up -d
```

7. Verify the containers are up and running

```
docker-compose ps
```

8. Configuration example for dial-out telemetry IOS XR:
```
telemetry model-driven
 destination-group telegraf
  address-family ipv4 10.48.188.61 port 57001
   encoding self-describing-gpb
   protocol tcp
!
 sensor-group telemtry
  sensor-path Cisco-IOS-XR-qos-ma-oper:qos/interface-table/interface/input/statistics
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
!
 subscription MDT
  sensor-group-id telemetry sample-interval 1000
  destination-id telegraf
!
```
9. Configuration example for dial-out telemetry IOS XE:
```
telemetry receiver protocol telegraf
 host ip-address 10.48.188.61 57000
 protocol grpc-tcp
 !
telemetry ietf subscription 100
 encoding encode-kvgpb
 filter xpath /memory-ios-xe-oper:memory-statistics/memory-statistic
 receiver-type protocol
 stream yang-push
 update-policy periodic 1000
 receiver name telegraf
 !
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
 receiver-type protocol
 stream yang-push
 update-policy periodic 1000
 receiver name telegraf
 !
telemetry ietf subscription 102
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/process-cpu-ios-xe-oper:cpu-utilization
 receiver-type protocol
 stream yang-push
 update-policy periodic 1000
 receiver name telegraf

```

10. Configuration example for dial-in telemetry IOS XR:
```
grpc
 port 57400
 no-tls
 address-family ipv4
!

!!! Below is needed on some XR platforms!!!

tpa
 vrf default
  address-family ipv4
   default-route mgmt
  !
 !
!
```
10. Configuration example for dial-in telemetry IOS XE:
```
gnmi-yang
gnmi-yang server
gnmi-yang port 50000
```
