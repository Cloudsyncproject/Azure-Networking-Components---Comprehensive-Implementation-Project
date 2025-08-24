# Azure-Networking-Components---Comprehensive-Implementation-Project
This project demonstrates a complete Azure networking architecture implementing a hybrid cloud solution for a fictional enterprise "TechCorp" with multiple business units requiring secure, scalable connectivity.



## Project Overview

This project demonstrates a complete Azure networking architecture implementing a hybrid cloud solution for a fictional enterprise "TechCorp" with multiple business units requiring secure, scalable connectivity.

### Architecture Components
- **Multi-tier VNet design** with hub-spoke topology
- **Hybrid connectivity** via ExpressRoute and VPN Gateway
- **Security layers** with Azure Firewall and Application Gateway
- **Global distribution** using Front Door
- **Load balancing** for high availability

## Project Structure

### Phase 1: Foundation - Virtual Networks and Subnets

#### 1.1 Hub VNet Design
```bash
# Hub VNet (Central US)
Resource Group: rg-networking-hub-centralus
VNet: vnet-hub-centralus
Address Space: 10.0.0.0/16

Subnets:
├── GatewaySubnet: 10.0.1.0/27 (Reserved for VPN/ExpressRoute Gateway)
├── AzureFirewallSubnet: 10.0.2.0/26 (Reserved for Azure Firewall)
├── AzureBastionSubnet: 10.0.3.0/27 (Reserved for Bastion Host)
├── snet-shared-services: 10.0.10.0/24 (Domain Controllers, DNS)
└── snet-management: 10.0.11.0/24 (Jump boxes, monitoring)
```

#### 1.2 Spoke VNets Design
```bash
# Production Spoke (East US)
VNet: vnet-prod-eastus
Address Space: 10.1.0.0/16

Subnets:
├── snet-web-tier: 10.1.1.0/24 (Application Gateway backend)
├── snet-app-tier: 10.1.2.0/24 (Application servers)
├── snet-data-tier: 10.1.3.0/24 (Database servers)
└── snet-integration: 10.1.4.0/24 (Logic Apps, Service Bus)

# Development Spoke (West US 2)
VNet: vnet-dev-westus2
Address Space: 10.2.0.0/16

Subnets:
├── snet-dev-web: 10.2.1.0/24
├── snet-dev-app: 10.2.2.0/24
└── snet-dev-data: 10.2.3.0/24
```

### Phase 2: VNet Peering Implementation

#### 2.1 Hub-Spoke Peering Configuration
```powershell
# Hub to Production Spoke Peering
$hubVnet = Get-AzVirtualNetwork -Name "vnet-hub-centralus" -ResourceGroupName "rg-networking-hub"
$prodVnet = Get-AzVirtualNetwork -Name "vnet-prod-eastus" -ResourceGroupName "rg-networking-prod"

# Create peering from Hub to Prod
Add-AzVirtualNetworkPeering -Name "hub-to-prod" `
  -VirtualNetwork $hubVnet `
  -RemoteVirtualNetworkId $prodVnet.Id `
  -UseRemoteGateways:$false `
  -AllowGatewayTransit:$true `
  -AllowForwardedTraffic:$true

# Create peering from Prod to Hub
Add-AzVirtualNetworkPeering -Name "prod-to-hub" `
  -VirtualNetwork $prodVnet `
  -RemoteVirtualNetworkId $hubVnet.Id `
  -UseRemoteGateways:$true `
  -AllowGatewayTransit:$false `
  -AllowForwardedTraffic:$true
```

### Phase 3: Hybrid Connectivity

#### 3.1 ExpressRoute Implementation
```bash
# ExpressRoute Circuit Configuration
Circuit Name: er-techcorp-primary
Location: Chicago
Service Provider: AT&T
Bandwidth: 1 Gbps
SKU: Standard
Peering Location: Chicago

# BGP Configuration
Primary Subnet: 192.168.1.0/30
Secondary Subnet: 192.168.1.4/30
VLAN ID: 100
ASN: 65001
```

#### 3.2 VPN Gateway for Backup Connectivity
```powershell
# Create VPN Gateway
$gateway = New-AzVirtualNetworkGateway `
  -Name "vgw-hub-centralus" `
  -ResourceGroupName "rg-networking-hub" `
  -Location "Central US" `
  -IpConfigurations $gwipconfig `
  -GatewayType "Vpn" `
  -VpnType "RouteBased" `
  -GatewaySku "VpnGw2" `
  -EnableActiveActiveFeature

# Site-to-Site VPN Configuration
$localGateway = New-AzLocalNetworkGateway `
  -Name "lgw-onprem-backup" `
  -ResourceGroupName "rg-networking-hub" `
  -Location "Central US" `
  -GatewayIpAddress "203.0.113.12" `
  -AddressPrefix "172.16.0.0/16"
```

### Phase 4: Security Implementation

#### 4.1 Azure Firewall Configuration
```bash
# Firewall Policy Structure
Policy Name: afwp-hub-centralus
Threat Intelligence: Alert and Deny
DNS Settings: Enable DNS Proxy

# Application Rules
Rule Collection Group: ApplicationRules
Priority: 300

Rules:
├── Allow-Web-Traffic
│   ├── Source: 10.1.1.0/24 (Web Tier)
│   ├── Destination: *.microsoft.com, *.azure.com
│   └── Protocol: HTTPS:443
├── Allow-API-Calls
│   ├── Source: 10.1.2.0/24 (App Tier)
│   ├── Destination: api.techcorp.com
│   └── Protocol: HTTPS:443
└── Block-Social-Media
    ├── Source: 10.0.0.0/8
    ├── Destination: *.facebook.com, *.twitter.com
    └── Action: Deny
```

#### 4.2 Network Security Groups
```powershell
# Web Tier NSG
$webNsg = New-AzNetworkSecurityGroup -Name "nsg-web-tier" `
  -ResourceGroupName "rg-networking-prod" `
  -Location "East US"

# Allow HTTPS from Application Gateway
$httpsRule = New-AzNetworkSecurityRuleConfig -Name "Allow-HTTPS-AppGW" `
  -Protocol "TCP" -Direction "Inbound" -Priority 100 `
  -SourceAddressPrefix "10.1.0.0/24" -SourcePortRange "*" `
  -DestinationAddressPrefix "10.1.1.0/24" -DestinationPortRange "443" `
  -Access "Allow"

$webNsg.SecurityRules.Add($httpsRule)
Set-AzNetworkSecurityGroup -NetworkSecurityGroup $webNsg
```

### Phase 5: Application Gateway Implementation

#### 5.1 Application Gateway Configuration
```json
{
  "name": "agw-prod-eastus",
  "location": "eastus",
  "sku": {
    "name": "WAF_v2",
    "tier": "WAF_v2",
    "capacity": 2
  },
  "gatewayIPConfigurations": [{
    "name": "appGatewayIpConfig",
    "subnet": {
      "id": "/subscriptions/{sub}/resourceGroups/rg-networking-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod-eastus/subnets/snet-appgw"
    }
  }],
  "frontendIPConfigurations": [{
    "name": "appGwPublicFrontendIp",
    "publicIPAddress": {
      "id": "/subscriptions/{sub}/resourceGroups/rg-networking-prod/providers/Microsoft.Network/publicIPAddresses/pip-agw-prod"
    }
  }],
  "backendAddressPools": [{
    "name": "web-tier-pool",
    "backendAddresses": [
      {"ipAddress": "10.1.1.10"},
      {"ipAddress": "10.1.1.11"}
    ]
  }]
}
```

#### 5.2 WAF Policy
```bash
# WAF Policy Configuration
Policy Name: wafp-prod-eastus
Mode: Prevention
Rule Set: OWASP 3.2

Custom Rules:
├── Rate Limiting: 100 requests/minute per IP
├── Geo Blocking: Block traffic from high-risk countries
└── Bot Protection: Enable bot management
```

### Phase 6: Load Balancing Solutions

#### 6.1 Internal Load Balancer (App Tier)
```powershell
# Create Internal Load Balancer for App Tier
$frontendIP = New-AzLoadBalancerFrontendIpConfig `
  -Name "lb-app-frontend" `
  -PrivateIpAddress "10.1.2.100" `
  -SubnetId $appSubnet.Id

$backendPool = New-AzLoadBalancerBackendAddressPoolConfig `
  -Name "app-tier-backend-pool"

$healthProbe = New-AzLoadBalancerProbeConfig `
  -Name "app-health-probe" `
  -Protocol "HTTP" `
  -Port 8080 `
  -RequestPath "/health" `
  -IntervalInSeconds 15

$lbRule = New-AzLoadBalancerRuleConfig `
  -Name "app-lb-rule" `
  -FrontendIpConfiguration $frontendIP `
  -BackendAddressPool $backendPool `
  -Protocol "TCP" `
  -FrontendPort 80 `
  -BackendPort 8080 `
  -Probe $healthProbe

$internalLB = New-AzLoadBalancer `
  -Name "ilb-app-tier" `
  -ResourceGroupName "rg-networking-prod" `
  -Location "East US" `
  -Sku "Standard" `
  -FrontendIpConfiguration $frontendIP `
  -BackendAddressPool $backendPool `
  -Probe $healthProbe `
  -LoadBalancingRule $lbRule
```

#### 6.2 Public Load Balancer (External Access)
```bash
# Standard Public Load Balancer
Name: plb-prod-eastus
SKU: Standard
Frontend IP: pip-plb-prod (Static Public IP)
Backend Pool: VM Scale Set with 3 instances
Health Probe: HTTP:80/health
Load Balancing Rules: HTTP:80 → HTTP:80
```

### Phase 7: Global Distribution with Front Door

#### 7.1 Front Door Configuration
```json
{
  "name": "fd-techcorp-global",
  "resourceGroup": "rg-networking-global",
  "frontendEndpoints": [{
    "name": "techcorp-frontend",
    "hostName": "app.techcorp.com",
    "sessionAffinityEnabledState": "Disabled"
  }],
  "backendPools": [{
    "name": "eastus-backend",
    "backends": [{
      "address": "agw-prod-eastus.eastus.cloudapp.azure.com",
      "httpPort": 80,
      "httpsPort": 443,
      "weight": 100,
      "priority": 1
    }],
    "healthProbeSettings": {
      "path": "/health",
      "protocol": "HTTPS",
      "intervalInSeconds": 30
    }
  }],
  "routingRules": [{
    "name": "default-routing",
    "frontendEndpoints": ["techcorp-frontend"],
    "acceptedProtocols": ["HTTPS"],
    "patternsToMatch": ["/*"],
    "backendPool": "eastus-backend"
  }]
}
```

#### 7.2 WAF Policy for Front Door
```bash
# Front Door WAF Configuration
Policy Name: waf-frontdoor-global
Mode: Prevention
Managed Rules: Microsoft Default Rule Set 2.1

Custom Rules:
├── IP Allow List: Corporate office IPs for admin paths
├── Rate Limiting: 1000 requests/5min per IP
└── Bot Protection: Challenge suspicious traffic
```

### Phase 8: DNS and Traffic Management

#### 8.1 Azure DNS Configuration
```powershell
# Create DNS Zone
$dnsZone = New-AzDnsZone -Name "techcorp.com" `
  -ResourceGroupName "rg-networking-global"

# A Records for services
New-AzDnsRecordSet -Name "app" -RecordType "A" `
  -ZoneName "techcorp.com" `
  -ResourceGroupName "rg-networking-global" `
  -Ttl 300 `
  -DnsRecords (New-AzDnsRecordConfig -IPv4Address "20.62.146.124")

# CNAME for Front Door
New-AzDnsRecordSet -Name "www" -RecordType "CNAME" `
  -ZoneName "techcorp.com" `
  -ResourceGroupName "rg-networking-global" `
  -Ttl 300 `
  -DnsRecords (New-AzDnsRecordConfig -Cname "fd-techcorp-global.azurefd.net")
```

### Phase 9: Monitoring and Diagnostics

#### 9.1 Network Watcher Implementation
```bash
# Enable Network Watcher
Location: All regions with deployed resources
Features:
├── Connection Monitor: End-to-end connectivity testing
├── IP Flow Verify: NSG rule verification
├── Next Hop: Route table analysis
├── Security Group View: Effective security rules
└── Packet Capture: Network troubleshooting
```

#### 9.2 Diagnostic Settings
```json
{
  "name": "network-diagnostics",
  "logs": [
    {
      "category": "ApplicationGatewayAccessLog",
      "enabled": true,
      "retentionPolicy": {"days": 30, "enabled": true}
    },
    {
      "category": "ApplicationGatewayPerformanceLog",
      "enabled": true,
      "retentionPolicy": {"days": 30, "enabled": true}
    },
    {
      "category": "ApplicationGatewayFirewallLog",
      "enabled": true,
      "retentionPolicy": {"days": 90, "enabled": true}
    }
  ],
  "metrics": [
    {
      "category": "AllMetrics",
      "enabled": true,
      "retentionPolicy": {"days": 30, "enabled": true}
    }
  ],
  "workspaceId": "/subscriptions/{sub}/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-techcorp"
}
```

### Phase 10: Security and Compliance

#### 10.1 Network Security Best Practices
```bash
# Implementation Checklist
Security Measures:
├── Zero Trust Network Architecture
│   ├── Micro-segmentation with NSGs
│   ├── Just-in-time VM access
│   └── Conditional access policies
├── Encryption in Transit
│   ├── TLS 1.2+ for all web traffic
│   ├── IPSec for site-to-site VPN
│   └── ExpressRoute encryption
├── Network Access Control
│   ├── Azure Firewall centralized filtering
│   ├── Application Gateway WAF protection
│   └── Front Door global WAF
└── Monitoring and Alerting
    ├── Security Center recommendations
    ├── Sentinel SIEM integration
    └── Custom alert rules
```

### Phase 11: Disaster Recovery and Business Continuity

#### 11.1 Multi-Region Setup
```bash
# Secondary Region Implementation
Primary: East US
Secondary: West US 2
Recovery Objectives:
├── RTO: 4 hours
├── RPO: 1 hour
└── Availability: 99.95%

DR Components:
├── Paired VNet in West US 2
├── ExpressRoute circuit redundancy
├── Application Gateway in secondary region
├── Front Door automatic failover
└── Database geo-replication
```

### Phase 12: Cost Optimization

#### 12.1 Cost Management Strategies
```bash
Optimization Areas:
├── Reserved Instances for VPN Gateway and Application Gateway
├── Azure Firewall vs NSG cost analysis
├── ExpressRoute vs VPN cost comparison
├── Front Door vs Traffic Manager evaluation
└── Resource rightsizing based on metrics

Monthly Cost Estimate:
├── ExpressRoute Circuit: $500
├── VPN Gateway (VpnGw2): $350
├── Azure Firewall: $800
├── Application Gateway (WAF_v2): $300
├── Front Door: $150
├── Load Balancers: $100
└── Total: ~$2,200/month
```

## Implementation Timeline

### Week 1-2: Foundation
- Deploy Hub VNet and core subnets
- Configure basic NSGs and route tables
- Set up Azure Bastion for secure access

### Week 3-4: Connectivity
- Implement VNet peering
- Deploy and configure VPN Gateway
- Establish ExpressRoute connectivity

### Week 5-6: Security
- Deploy Azure Firewall with policies
- Configure Application Gateway with WAF
- Implement comprehensive NSG rules

### Week 7-8: Global Services
- Configure Front Door with global distribution
- Set up DNS zones and records
- Implement SSL/TLS certificates

### Week 9-10: Monitoring & Testing
- Enable Network Watcher and diagnostics
- Perform end-to-end connectivity testing
- Validate security policies and rules

## Testing and Validation

### Connectivity Testing
```powershell
# Test VNet Peering
Test-NetConnection -ComputerName "10.1.1.10" -Port 443

# Test ExpressRoute
Get-AzExpressRouteCircuitRouteTable -ResourceGroupName "rg-networking-hub" `
  -ExpressRouteCircuitName "er-techcorp-primary" -PeeringType "AzurePrivatePeering"

# Test Application Gateway
Invoke-WebRequest -Uri "https://app.techcorp.com/health" -UseBasicParsing
```

### Security Validation
```bash
# Azure Firewall logs query
AzureDiagnostics
| where Category == "AzureFirewallApplicationRule"
| where TimeGenerated > ago(1h)
| summarize count() by Action

# WAF blocked requests
ApplicationGatewayFirewallLog
| where ruleSetType_s == "OWASP"
| where action_s == "Blocked"
| summarize count() by clientIP_s
```

## Key Learnings and Best Practices

### Design Principles
- **Hub-spoke topology** provides centralized connectivity and security
- **Defense in depth** with multiple security layers
- **Network segmentation** limits blast radius of security incidents
- **Global distribution** improves performance and availability
- **Monitoring first** approach enables proactive management

### Common Pitfalls to Avoid
- Overlapping IP address spaces between VNets
- Incorrect NSG rule priorities causing traffic blocks
- Missing health probes causing load balancer failures
- Inadequate ExpressRoute bandwidth planning
- Forgotten route table associations

### Performance Optimization
- Use proximity placement groups for low-latency requirements
- Configure Application Gateway v2 for auto-scaling
- Implement Front Door caching for static content
- Monitor and tune Azure Firewall rules for performance
- Regular review of ExpressRoute circuit utilization

This comprehensive project demonstrates enterprise-grade Azure networking implementation with hybrid connectivity, multi-layered security, global distribution, and operational excellence practices.
