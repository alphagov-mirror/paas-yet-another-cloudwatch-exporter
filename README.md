# YACE - yet another cloudwatch exporter [![Docker Image](https://quay.io/repository/invisionag/yet-another-cloudwatch-exporter/status?token=58e4108f-9e6f-44a4-a5fd-0beed543a271 "Docker Repository on Quay")](https://quay.io/repository/invisionag/yet-another-cloudwatch-exporter)

## Project Status

YACE is currently in quick iteration mode. Things will probably break in upcoming versions. However, it has been in production use at InVision AG for a couple of months already.

## Features
* Stop worrying about your AWS IDs - Auto discovery of resources via tags
* Filter monitored resources via regex
* Automatic adding of tag labels to metrics
* Allows to export 0 even if CloudWatch returns nil
* Static metrics support for all cloudwatch metrics without auto discovery
* Pull data from mulitple AWS accounts using cross-acount roles
* Supported services with auto discovery through tags:
  - alb - Application Load Balancer
  - ebs - Elastic Block Storage
  - ec - ElastiCache
  - ec2 - Elastic Compute Cloud
  - efs - Elastic File System
  - elb - Elastic Load Balancer
  - es - ElasticSearch
  - lambda - Lambda Functions
  - rds - Relational Database Service
  - s3 - Object Storage
  - vpn - VPN connection

## Image
* `quay.io/invisionag/yet-another-cloudwatch-exporter:x.x.x` e.g. 0.5.0
* See [Releases](https://github.com/ivx/yet-another-cloudwatch-exporter/releases) for binaries

## Configuration

Example of config File
```
discovery:
  exportedTagsOnMetrics:
    ec2:
      - Name
  jobs:
  - region: eu-west-1
    type: "es"
    searchTags:
      - Key: type
        Value: ^(easteregg|k8s)$
    metrics:
      - name: FreeStorageSpace
        statistics:
        - 'Sum'
        period: 600
        length: 60
      - name: ClusterStatus.green
        statistics:
        - 'Minimum'
        period: 600
        length: 60
      - name: ClusterStatus.yellow
        statistics:
        - 'Maximum'
        period: 600
        length: 60
      - name: ClusterStatus.red
        statistics:
        - 'Maximum'
        period: 600
        length: 60
  - type: "elb"
    region: eu-west-1
    searchTags:
      - Key: KubernetesCluster
        Value: production-19
    metrics:
      - name: HealthyHostCount
        statistics:
        - 'Minimum'
        period: 60
        length: 300
      - name: HTTPCode_Backend_4XX
        statistics:
        - 'Sum'
        period: 60
        length: 900
        delay: 300
        nilToZero: true
  - type: "alb"
    region: eu-west-1
    searchTags:
      - Key: kubernetes.io/service-name
        Value: .*
    metrics:
      - name: UnHealthyHostCount
        statistics: [Maximum]
        period: 60
        length: 600
  - type: "vpn"
    region: eu-west-1
    searchTags:
      - Key: kubernetes.io/service-name
        Value: .*
    metrics:
      - name: TunnelState
        statistics:
        - 'p90'
        period: 60
        length: 300
static:
  - namespace: AWS/AutoScaling
    region: eu-west-1
    dimensions:
     - name: AutoScalingGroupName
       value: Test
    customTags:
      - Key: CustomTag
        Value: CustomValue
    metrics:
      - name: GroupInServiceInstances
        statistics:
        - 'Minimum'
        period: 60
        length: 300
```

## Metrics Examples
```
### Metrics with exportedTagsOnMetrics
aws_ec2_cpuutilization_maximum{name="arn:aws:ec2:eu-west-1:472724724:instance/i-someid", tag_Name="jenkins"} 57.2916666666667

### Info helper with tags
aws_elb_info{name="arn:aws:elasticloadbalancing:eu-west-1:472724724:loadbalancer/a815b16g3417211e7738a02fcc13bbf9",tag_KubernetesCluster="production-19",tag_Name="",tag_kubernetes_io_cluster_production_19="owned",tag_kubernetes_io_service_name="nginx-ingress/private-ext"} 0
aws_ec2_info{name="arn:aws:ec2:eu-west-1:472724724:instance/i-someid",tag_Name="jenkins"} 0

### Track cloudwatch requests to calculate costs
yace_cloudwatch_requests_total 168
```

## Query Examples without exportedTagsOnMetrics

```
# CPUUtilization + Name tag of the instance id - No more instance id needed for monitoring
aws_ec2_cpuutilization_average + on (name) group_left(tag_Name) aws_ec2_info

# Free Storage in Megabytes + tag Type of the elasticsearch cluster
(aws_es_free_storage_space_sum + on (name) group_left(tag_Type) aws_es_info) / 1024

# Add kubernetes / kops tags on 4xx elb metrics
(aws_elb_httpcode_backend_4_xx_sum + on (name) group_left(tag_KubernetesCluster,tag_kubernetes_io_service_name) aws_elb_info)

# Availability Metric for ELBs (Sucessfull requests / Total Requests) + k8s service name
# Use nilToZero on all metrics else it won't work
((aws_elb_request_count_sum - on (name) group_left() aws_elb_httpcode_backend_4_xx_sum) - on (name) group_left() aws_elb_httpcode_backend_5_xx_sum) + on (name) group_left(tag_kubernetes_io_service_name) aws_elb_info

# Forecast your elasticsearch disk size in 7 days and report metrics with tags type and version
predict_linear(aws_es_free_storage_space_minimum[2d], 86400 * 7) + on (name) group_left(tag_type, tag_version) aws_es_info

# Forecast your cloudwatch costs for next 32 days based on last 10 minutes
# 1.000.000 Requests free
# 0.01 Dollar for 1.000 GetMetricStatistics Api Requests (https://aws.amazon.com/cloudwatch/pricing/)
((increase(yace_cloudwatch_requests_total[10m]) * 6 * 24 * 32) - 100000) / 1000 * 0.01
```

## IAM
The following IAM permissions are required for YACE to work.

```
"tag:getResources",
"cloudwatch:GetMetricStatistics",
```

## Kubernetes Installation
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: yace
data:
  config.yml: |-
    ---
    # Start of config file
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: yace
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: yace
    spec:
      containers:
      - name: yace
        image: quay.io/invisionag/yet-another-cloudwatch-exporter:x.x.x # release version as tag
        imagePullPolicy: IfNotPresent
        command:
          - "yace"
          - "--config.file=/tmp/config.yml"
        ports:
        - name: app
          containerPort: 5000
        volumeMounts:
        - name: config-volume
          mountPath: /tmp
      volumes:
      - name: config-volume
        configMap:
          name: yace
```

## Contribute
[Development Setup / Guide](/CONTRIBUTE.md)

# Thank you
* [Justin Santa Barbara](https://github.com/justinsb) - For telling me about AWS tags api which simplified a lot - Thanks!
* [Brian Brazil](https://github.com/brian-brazil) - Who gave a lot of feedback regarding UX and prometheus lib - Thanks!
