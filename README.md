# api62api7
# APISIX API Gateway Upgrade: Version 6 to 7

![APISIX](https://img.shields.io/badge/APISIX-7.x-blue)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.20+-green)
![License](https://img.shields.io/badge/License-Enterprise%20Ready-orange)

## 📋 Overview

This repository contains a comprehensive guide for upgrading Apache APISIX API Gateway from version 6 to version 7 in a Kubernetes environment, with full support for enterprise licensing.

### What's Included
- ✅ Step-by-step upgrade procedures
- ✅ Backup and rollback strategies
- ✅ Enterprise license configuration
- ✅ Kubernetes-native deployment
- ✅ Performance optimisation
- ✅ Troubleshooting guides

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Ingress       │    │   APISIX v7     │    │   Backend       │
│   Controller    │───▶│   Gateway       │───▶│   Services      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │
                              ▼
                       ┌─────────────────┐
                       │   etcd Cluster  │
                       │   (Config Store)│
                       └─────────────────┘
```

## 🚀 Quick Start

### Prerequisites

Before starting the upgrade, ensure you have:

- [ ] Kubernetes cluster (v1.20+)
- [ ] Helm 3.x installed
- [ ] kubectl configured
- [ ] APISIX v6 currently running
- [ ] Valid enterprise license (if applicable)
- [ ] Backup strategy in place

### Installation

1. **Clone this repository**
   ```bash
   git clone <repository-url>
   cd apisix-upgrade-guide
   ```

2. **Review current setup**
   ```bash
   kubectl get pods -n apisix-system
   helm list -n apisix-system
   ```

3. **Follow the upgrade guide below**

## 📊 Compatibility Matrix

| Component | v6 | v7 | Notes |
|-----------|----|----|-------|
| APISIX Core | 3.x | 3.8+ | Major version upgrade |
| Ingress Controller | 1.6.x | 1.8+ | API changes required |
| Dashboard | 2.x | 3.x | UI improvements |
| etcd | 3.4+ | 3.5+ | Backward compatible |
| Kubernetes | 1.18+ | 1.20+ | API version updates |

---

# APISIX API6 to API7 Upgrade Guide for Kubernetes

## Pre-Upgrade Preparation

### 1. Backup Current Configuration
```bash
# Backup etcd data
kubectl exec -n apisix-system etcd-0 -- etcdctl snapshot save /tmp/backup.db

# Export current APISIX configuration
kubectl get configmap apisix-config -n apisix-system -o yaml > apisix-config-backup.yaml

# Backup routes and services
kubectl get apisixroutes -n apisix-system -o yaml > routes-backup.yaml
kubectl get apisixservices -n apisix-system -o yaml > services-backup.yaml
```
##Recommment
Use adc
```
https://github.com/api7/adc
```

### 2. Review Breaking Changes
- Check APISIX 7.x release notes for breaking changes
- Verify plugin compatibility
- Update custom plugins if needed

## Step 1: Update Helm Repository

```bash
# Add/update APISIX Helm repository
helm repo add apisix https://charts.apiseven.com
helm repo update

# Check available versions
helm search repo apisix/apisix --versions
```

## Step 2: Prepare License Configuration

### For APISIX Enterprise License
```yaml
# license-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: apisix-license
  namespace: apisix-system
type: Opaque
data:
  license: <base64-encoded-license-content>
```

Apply the license secret:
```bash
kubectl apply -f license-secret.yaml
```

## Step 3: Update Helm Values

### values-v7.yaml
```yaml
# APISIX 7.x configuration
apisix:
  image:
    repository: apache/apisix
    tag: "3.8.0-debian"  # APISIX 7.x compatible version
  
  # License configuration (if using Enterprise)
  license:
    secretName: apisix-license
    secretKey: license
  
  # Updated configuration for v7
  config:
    apisix:
      node_listen: 9080
      enable_admin: true
      enable_admin_cors: true
      admin_key:
        - name: "admin"
          key: <your-admin-key>
          role: admin
      
      # New v7 features
      deployment:
        role: traditional
        role_traditional:
          config_provider: etcd
      
      # Updated plugin configuration
      plugins:
        - real-ip
        - ai
        - client-control
        - proxy-control
        - request-id
        - zipkin
        - ext-plugin-pre-req
        - fault-injection
        - mocking
        - serverless-pre-function
        - cors
        - ip-restriction
        - ua-restriction
        - referer-restriction
        - csrf
        - uri-blocker
        - request-validation
        - openid-connect
        - authz-casbin
        - authz-casdoor
        - wolf-rbac
        - ldap-auth
        - hmac-auth
        - basic-auth
        - jwt-auth
        - key-auth
        - consumer-restriction
        - authz-keycloak
        - opa
        - forward-auth
        - opa
        - proxy-cache
        - body-transformer
        - proxy-mirror
        - proxy-rewrite
        - workflow
        - api-breaker
        - limit-conn
        - limit-count
        - limit-req
        - gzip
        - server-info
        - traffic-split
        - redirect
        - response-rewrite
        - degraphql
        - kafka-logger
        - cors
        - echo
        - http-logger
        - splunk-hec-logging
        - skywalking-logger
        - google-cloud-logging
        - prometheus
        - datadog
        - loki-logger
        - elasticsearch-logger
        - tcp-logger
        - kafka-logger
        - rocketmq-logger
        - syslog
        - udp-logger
        - file-logger
        - loggly
        - example-plugin
        - aws-lambda
        - azure-functions
        - openwhisk
        - openfunction
        - tencent-cloud-cls
        - inspect
        - public-api

etcd:
  enabled: true
  replicaCount: 3
  
apisix-dashboard:
  enabled: true
  config:
    authentication:
      secret: <dashboard-secret>
      expire_time: 3600
      users:
        - username: admin
          password: <admin-password>

apisix-ingress-controller:
  enabled: true
  image:
    repository: apache/apisix-ingress-controller
    tag: "1.8.0"
  
  config:
    apisix:
      serviceNamespace: apisix-system
      serviceName: apisix-admin
      servicePort: 9180
      adminKey: <your-admin-key>
```

## Step 4: Rolling Upgrade Strategy

### Option A: Blue-Green Deployment (Recommended)
```bash
# Deploy new version alongside existing
helm install apisix-v7 apisix/apisix \
  --namespace apisix-system-v7 \
  --create-namespace \
  --values values-v7.yaml

# Test new deployment
kubectl port-forward -n apisix-system-v7 svc/apisix-admin 9180:9180

# Switch traffic gradually
# Update ingress/load balancer configuration
```

### Option B: In-Place Rolling Update
```bash
# Perform rolling upgrade
helm upgrade apisix apisix/apisix \
  --namespace apisix-system \
  --values values-v7.yaml \
  --timeout 10m0s

# Monitor the upgrade
kubectl rollout status deployment/apisix -n apisix-system
```

## Step 5: Post-Upgrade Verification

### 1. Check Pod Status
```bash
kubectl get pods -n apisix-system
kubectl logs -n apisix-system deployment/apisix
```

### 2. Verify API Functionality
```bash
# Test admin API
curl -H 'X-API-KEY: <your-admin-key>' http://localhost:9180/apisix/admin/routes

# Test data plane
curl -H 'Host: your-api.com' http://localhost:9080/your-endpoint
```

### 3. Validate License (if applicable)
```bash
# Check license status
curl -H 'X-API-KEY: <your-admin-key>' http://localhost:9180/apisix/admin/license
```

## Step 6: Configuration Migration

### Update Routes for v7 Compatibility
```yaml
# Example route update for v7
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: example-route
  namespace: default
spec:
  http:
    - name: rule1
      match:
        hosts:
          - api.example.com
        paths:
          - "/api/*"
      backends:
        - serviceName: backend-service
          servicePort: 80
      plugins:
        - name: cors
          enable: true
          config:
            allow_origins: "*"
            allow_methods: "GET,POST,PUT,DELETE"
```

## 🚨 Troubleshooting

### Common Issues and Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| License validation failed | Pods crash on startup | Check license format and secret |
| Plugin compatibility | Routes not working | Review v7 plugin changes |
| etcd connection | Gateway unreachable | Verify etcd cluster health |
| Resource limits | Performance degradation | Adjust CPU/memory limits |

### Debug Commands
```bash
# Check plugin status
curl -H 'X-API-KEY: <your-admin-key>' http://localhost:9180/apisix/admin/plugins/list

# Disable problematic plugins temporarily
kubectl patch apisixroute example-route --type='merge' -p='{"spec":{"http":[{"plugins":[]}]}}'

# Verify license secret
kubectl get secret apisix-license -n apisix-system -o yaml

# Check license format
kubectl exec -n apisix-system deployment/apisix -- cat /usr/local/apisix/conf/license

# Check etcd connectivity
kubectl exec -n apisix-system deployment/apisix -- \
  etcdctl --endpoints=http://etcd:2379 endpoint health

# Check pod logs
kubectl logs -n apisix-system deployment/apisix -f

# Test health endpoint
curl http://localhost:9080/apisix/prometheus/metrics
```

## Rollback Procedure

If issues occur, rollback using:
```bash
# Rollback Helm release
helm rollback apisix -n apisix-system

# Restore etcd backup if needed
kubectl exec -n apisix-system etcd-0 -- \
  etcdctl snapshot restore /tmp/backup.db
```

## 📈 Performance Optimization for v7

### Resource Requirements

| Environment | CPU | Memory | Replicas |
|-------------|-----|--------|----------|
| Development | 500m | 512Mi | 1 |
| Staging | 1000m | 1Gi | 2 |
| Production | 2000m | 2Gi | 3+ |

### Resource Allocation
```yaml
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi
```

### Horizontal Pod Autoscaling
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apisix-hpa
  namespace: apisix-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apisix
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## 🔍 Monitoring and Observability

### Key Metrics to Monitor
- Request rate and latency
- Error rates (4xx, 5xx)
- CPU and memory usage
- etcd performance
- License status

### Prometheus Integration
```yaml
# Enable Prometheus plugin
plugins:
  - name: prometheus
    enable: true
    config:
      prefer_name: true
```

### Grafana Dashboard
Import APISIX v7 compatible Grafana dashboard for monitoring.

## 🔒 Security Considerations

### Post-Upgrade Security Checklist
- [ ] Update all API keys and secrets
- [ ] Review RBAC configurations
- [ ] Validate SSL/TLS settings
- [ ] Check plugin security configurations
- [ ] Audit access logs
- [ ] Test authentication flows

1. Update API keys and secrets
2. Review new security features in v7
3. Update RBAC configurations
4. Validate SSL/TLS configurations

## 📚 Additional Resources

### File Structure
```
├── README.md                    # This file
├── upgrade-guide.md            # Detailed upgrade steps
├── values/
│   ├── values-v7-dev.yaml     # Development values
│   ├── values-v7-prod.yaml    # Production values
│   └── values-v7-staging.yaml # Staging values
├── scripts/
│   ├── backup.sh              # Backup script
│   ├── upgrade.sh             # Upgrade automation
│   └── rollback.sh            # Rollback script
└── manifests/
    ├── license-secret.yaml    # License configuration
    ├── hpa.yaml              # Auto-scaling config
    └── monitoring.yaml       # Monitoring setup
```

### Documentation Links
- [APISIX Official Documentation](https://apisix.apache.org/docs/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/)
- [Helm Charts Repository](https://charts.apiseven.com)

## 📝 Changelog

### v7.0.0 (Latest)
- ✅ Complete upgrade from v6 to v7
- ✅ Enterprise license support
- ✅ Kubernetes native deployment
- ✅ Performance optimizations
- ✅ Enhanced monitoring

### v6.x (Legacy)
- 🔄 Maintenance mode only
- ⚠️ Upgrade recommended

## 📞 Support

### Getting Help
- **Documentation**: Check the upgrade guide first
- **Community**: APISIX Slack/Discord channels
- **Enterprise**: Contact your license provider
- **Issues**: GitHub Issues for bugs/features

### Response Times
- **Critical**: 4 hours
- **High**: 24 hours
- **Normal**: 72 hours
- **Low**: 1 week

## Next Steps

1. Monitor performance metrics
2. Update documentation
3. Train team on new v7 features
4. Plan for future upgrades

---

## ⚡ Quick Commands Reference

```bash
# Backup current configuration
kubectl get apisixroutes -o yaml > routes-backup.yaml

# Upgrade to v7
helm upgrade apisix apisix/apisix -f values-v7.yaml

# Check upgrade status
kubectl rollout status deployment/apisix -n apisix-system

# Verify functionality
curl -H 'X-API-KEY: <key>' http://localhost:9180/apisix/admin/routes

# Rollback if needed
helm rollback apisix -n apisix-system
```

---

**⚠️ Important**: Always test the upgrade in a non-production environment first!

## 📄 License

This upgrade guide is provided under the Apache 2.0 License. APISIX is licensed under Apache 2.0. Enterprise features require a valid license from API7.
