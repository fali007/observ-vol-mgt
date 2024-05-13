# PoC usecase
![demofigure](../../../docs/images/demo.png)

The PoC demonstrates two edges connected to a central cloud. Each edge comprises of metric generator whose metrics are scraped by prometheus. The prometheus does a remote write to thanos (running in the central cloud) for long term storage and analysis. The remote write is intercepted by our processor proxy running at each edge location. The processor applies various transformation to the collected metrics.

In this PoC we showcase two transformations. 
- On cloud 1 we showcase the need to control the bandwidth utilization. We identify the increase in bandwidth utilization using alerts and our manager triggers addition of the filter transformation on processor running on cloud 1. 
- On cloud 2, we showcase the need to change the frequency of metric collection for a particular app/cluster. This scenario arises when for example an issue is identified at a node in a cluster and you need metrics for that node at a higher frequency to fix this issue. The identification of the issue in the PoC is also done with help of alerts on the metrics. 

Both the PoC scenarios which can also happen simulatenously, we also showcase how we revert the applied transformation once the issue is resolved. We instrument the issue with increase in metric values and reduce the value to revert the issue. 

## Environment Setup

The PoC requires docker installed on the machine to test the scenario. The following containers get installed when the PoC environment is bought up.\
Central containers:
- `thanos-receive`
- `thanos-query`
- `thanos-ruler`
- `ruler-config`
- `alertmanager`
- `manager`\
Edge containers:
- `metricgen1,2`
- `prometheus1,2`
- `pmf_processsor1,2`

## Running the PoC story

- Bring up the PoC environment
``` 
docker-compose -f docker-compose-quay.yml up -d
```
- Add the rules and actions (in the form of transformation) corresponding to the rule
```
curl -X POST --data-binary @demo_2rules.yaml -H "Content-type: text/x-yaml" http://0.0.0.0:5010/api/v1/rules
```
- Confirm the two rules are added correctly in the `thanos ruler` UI:
`http://9.1.43.241:10903/rules`
- Confirm the metrics are flowing correctly in the `thanos query` UI:
`http://9.1.43.241:19192/`\
Search for `cluster_hardware_metric_0` and `app_A_network_metric_0`. You should see 6 metrics (3 for each edge) for each of these. The edge can be identified by the `processor` label in the metric.
- Next, we instrument the issue scenario with metric value change.
  - For issue at edge 1: `curl -X POST --data-binary @change_appmetric.yml -H "Content-type: text/x-yaml" http://0.0.0.0:5002`
  - For issue at edge 2: `curl -X POST --data-binary @change_hwmetric.yml -H "Content-type: text/x-yaml" http://0.0.0.0:5003`
- You can visualize the change in the metrics value in the `thanos query` UI (in the graph mode)
- The alert should also be firing and can be seen in the `thanos ruler` UI (`http://9.1.43.241:10903/alerts`)
- The manager triggers adding of the corresponding transforms. To confirm the same exec into the manager container
  ```
  docker exec -it manager /bin/bash
  cat app/logs/manager.log
  ``` 
  You should see `INFO - POST request successful for processor with id 1` `INFO - POST request successful for processor with id 2`\
  P.S. This step is just for extra confirmation and is optional
- To visualize the transformation thanos query UI (in the graph mode). 
  - If you search `app_A_network_metric_0{processor="1"}` you will see the metric with label `IP:192.168.1.3` has changing value but the other metrics with label `IP:192.168.1.1` and `IP:192.168.1.2` with no change (straight line; showcasing no value change or should stop completely).\ This is because we have filtered and allowed `app_A_network_metric_0` only for app with `IP:192.168.1.3`.
  - If you search `cluster_hardware_metric_0{processor="2"}` you will see the metric with label `node:'0'` having a nice sinusoidal wave. This is because its value is changing every 5 sec. The other metrics with label `node:'1'` and `node:'2'` will be blocky showing their frequency is still 30 sec. 
This showcases the transformation happening in an automated fashion.
- We will just revert issue in edge cloud 1 and showcase that we work on specific cloud as well.
  ```
  curl -X POST --data-binary @change_appmetricRESET.yml -H "Content-type: text/x-yaml" http://0.0.0.0:5002
  ```
  In sometime, you should see the alert rule 1 resolved (in thanos ruler UI (`http://9.1.43.241:10903/alerts`)) and should see the metric `app_A_network_metric_0{processor="1"}` again coming in for labels `IP:192.168.1.1` and `IP:192.168.1.2` as well demontrating the removal of `filter` transform from edge cloud 1. 


