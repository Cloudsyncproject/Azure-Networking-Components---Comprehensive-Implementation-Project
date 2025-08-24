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

I've created a comprehensive and interactive architecture diagram that visualizes the entire Azure networking project. The diagram features:
Visual Design:

Modern, interactive design with hover effects and animations
Color-coded components for easy identification
Regional boundaries clearly defined
Professional gradient backgrounds and shadows

 Architecture Components:

Hub-Spoke Topology: Central hub (Central US) with production (East US) and development (West US 2) spokes
Hybrid Connectivity: ExpressRoute primary connection with VPN backup
Global Services: Front Door and Azure DNS for worldwide distribution
Security Layers: Azure Firewall, WAF, and NSGs throughout
Load Balancing: Application Gateway and Internal Load Balancers

 Key Features:

IP Address Labeling: Clear CIDR notation for each VNet
Connection Lines: Animated lines showing ExpressRoute, VPN, and VNet peering
Component Legend: Color-coded legend for different service types
Traffic Flow Summary: Below the diagram showing common traffic patterns

Interactive Elements:

Hover effects on regions and components
Floating cloud animation for internet connectivity
Pulsing connection lines to show active data flow
Responsive design that works on different screen sizes

The diagram effectively shows how all the components from the project work together:

Traffic flows from internet through Front Door to Application Gateway
On-premises connectivity via ExpressRoute with VPN failover
Hub-spoke peering relationships
Security enforcement points
Load balancing at multiple tiers

This visual representation makes it easy to understand the complex networking architecture and can be used for presentations, documentation, or team discussions about the implementation.




<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Azure Networking Architecture</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
        }
        
        .container {
            max-width: 1800px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            padding: 30px;
            position: relative;
            overflow: hidden;
        }
        
        .container::before {
            content: '';
            position: absolute;
            top: -50%;
            left: -50%;
            width: 200%;
            height: 200%;
            background: linear-gradient(45deg, transparent 30%, rgba(102, 126, 234, 0.05) 50%, transparent 70%);
            animation: shimmer 20s infinite;
            pointer-events: none;
        }
        
        @keyframes shimmer {
            0%, 100% { transform: rotate(0deg); }
            50% { transform: rotate(180deg); }
        }
        
        h1 {
            text-align: center;
            color: #2c3e50;
            font-size: 2.5em;
            margin-bottom: 30px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
        }
        
        .diagram {
            position: relative;
            width: 100%;
            height: 1000px;
            background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
            border-radius: 15px;
            overflow: hidden;
            box-shadow: inset 0 4px 8px rgba(0,0,0,0.1);
        }
        
        .region {
            position: absolute;
            border: 3px solid;
            border-radius: 15px;
            padding: 20px;
            background: rgba(255, 255, 255, 0.9);
            backdrop-filter: blur(10px);
            box-shadow: 0 8px 16px rgba(0,0,0,0.1);
            transition: all 0.3s ease;
        }
        
        .region:hover {
            transform: translateY(-5px);
            box-shadow: 0 12px 24px rgba(0,0,0,0.15);
        }
        
        .hub-region {
            top: 400px;
            left: 700px;
            width: 400px;
            height: 300px;
            border-color: #e74c3c;
            background: linear-gradient(135deg, rgba(231, 76, 60, 0.1) 0%, rgba(231, 76, 60, 0.05) 100%);
        }
        
        .prod-region {
            top: 100px;
            left: 1200px;
            width: 350px;
            height: 280px;
            border-color: #27ae60;
            background: linear-gradient(135deg, rgba(39, 174, 96, 0.1) 0%, rgba(39, 174, 96, 0.05) 100%);
        }
        
        .dev-region {
            top: 750px;
            left: 1200px;
            width: 350px;
            height: 200px;
            border-color: #f39c12;
            background: linear-gradient(135deg, rgba(243, 156, 18, 0.1) 0%, rgba(243, 156, 18, 0.05) 100%);
        }
        
        .onprem {
            top: 50px;
            left: 50px;
            width: 300px;
            height: 200px;
            border-color: #8e44ad;
            background: linear-gradient(135deg, rgba(142, 68, 173, 0.1) 0%, rgba(142, 68, 173, 0.05) 100%);
        }
        
        .global-services {
            top: 50px;
            left: 400px;
            width: 250px;
            height: 150px;
            border-color: #3498db;
            background: linear-gradient(135deg, rgba(52, 152, 219, 0.1) 0%, rgba(52, 152, 219, 0.05) 100%);
        }
        
        .region-title {
            font-weight: bold;
            font-size: 1.2em;
            color: #2c3e50;
            margin-bottom: 15px;
            text-align: center;
            padding: 8px;
            border-radius: 8px;
            background: rgba(255,255,255,0.8);
        }
        
        .component {
            background: linear-gradient(145deg, #ffffff, #f0f0f0);
            border: 2px solid #ddd;
            border-radius: 8px;
            padding: 8px;
            margin: 5px 0;
            font-size: 0.85em;
            color: #2c3e50;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            transition: all 0.3s ease;
        }
        
        .component:hover {
            background: linear-gradient(145deg, #f8f9fa, #e9ecef);
            transform: scale(1.05);
        }
        
        .subnet {
            background: linear-gradient(145deg, #e8f4fd, #dbeafe);
            border: 1px solid #3498db;
            color: #2980b9;
        }
        
        .gateway {
            background: linear-gradient(145deg, #fff3cd, #ffeaa7);
            border: 1px solid #f39c12;
            color: #e67e22;
        }
        
        .security {
            background: linear-gradient(145deg, #f8d7da, #f5c6cb);
            border: 1px solid #dc3545;
            color: #721c24;
        }
        
        .load-balancer {
            background: linear-gradient(145deg, #d1ecf1, #bee5eb);
            border: 1px solid #17a2b8;
            color: #0c5460;
        }
        
        .connection {
            position: absolute;
            height: 3px;
            background: linear-gradient(90deg, #667eea, #764ba2);
            border-radius: 2px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
            animation: pulse 2s infinite;
        }
        
        @keyframes pulse {
            0%, 100% { opacity: 0.6; }
            50% { opacity: 1; }
        }
        
        .connection-label {
            position: absolute;
            background: rgba(255,255,255,0.95);
            padding: 4px 8px;
            border-radius: 4px;
            font-size: 0.75em;
            color: #2c3e50;
            font-weight: bold;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        
        /* Connection lines */
        .conn-er {
            top: 180px;
            left: 350px;
            width: 350px;
            transform: rotate(15deg);
        }
        
        .conn-vpn {
            top: 120px;
            left: 350px;
            width: 350px;
            transform: rotate(-15deg);
        }
        
        .conn-peering1 {
            top: 320px;
            left: 1100px;
            width: 100px;
            transform: rotate(-45deg);
        }
        
        .conn-peering2 {
            top: 680px;
            left: 1100px;
            width: 100px;
            transform: rotate(45deg);
        }
        
        .internet-cloud {
            position: absolute;
            top: 250px;
            left: 200px;
            width: 150px;
            height: 100px;
            background: linear-gradient(135deg, #74b9ff, #0984e3);
            border-radius: 50px;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-weight: bold;
            box-shadow: 0 8px 16px rgba(0,0,0,0.2);
            animation: float 6s ease-in-out infinite;
        }
        
        @keyframes float {
            0%, 100% { transform: translateY(0px); }
            50% { transform: translateY(-20px); }
        }
        
        .legend {
            position: absolute;
            bottom: 20px;
            right: 20px;
            background: rgba(255,255,255,0.95);
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
            font-size: 0.8em;
        }
        
        .legend-item {
            display: flex;
            align-items: center;
            margin: 5px 0;
        }
        
        .legend-color {
            width: 20px;
            height: 15px;
            border-radius: 3px;
            margin-right: 10px;
        }
        
        .ip-labels {
            position: absolute;
            font-size: 0.7em;
            color: #666;
            background: rgba(255,255,255,0.9);
            padding: 2px 6px;
            border-radius: 4px;
        }
        
        .hub-ip { top: 380px; left: 720px; }
        .prod-ip { top: 80px; left: 1220px; }
        .dev-ip { top: 730px; left: 1220px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🏗️ Azure Networking Architecture - TechCorp Enterprise</h1>
        
        <div class="diagram">
            <!-- IP Address Labels -->
            <div class="ip-labels hub-ip">Hub VNet: 10.0.0.0/16</div>
            <div class="ip-labels prod-ip">Prod VNet: 10.1.0.0/16</div>
            <div class="ip-labels dev-ip">Dev VNet: 10.2.0.0/16</div>
            
            <!-- Internet Cloud -->
            <div class="internet-cloud">
                ☁️ Internet
            </div>
            
            <!-- On-Premises -->
            <div class="region onprem">
                <div class="region-title">🏢 On-Premises</div>
                <div class="component">Corporate Network</div>
                <div class="component">172.16.0.0/16</div>
                <div class="component">BGP ASN: 65001</div>
                <div class="component">Domain Controllers</div>
                <div class="component">File Servers</div>
            </div>
            
            <!-- Global Services -->
            <div class="region global-services">
                <div class="region-title">🌍 Global Services</div>
                <div class="component security">Front Door + WAF</div>
                <div class="component">Azure DNS</div>
                <div class="component">app.techcorp.com</div>
            </div>
            
            <!-- Hub Region (Central US) -->
            <div class="region hub-region">
                <div class="region-title">🎯 Hub - Central US</div>
                <div class="subnet">GatewaySubnet: 10.0.1.0/27</div>
                <div class="gateway">ExpressRoute Gateway</div>
                <div class="gateway">VPN Gateway (Backup)</div>
                <div class="subnet">AzureFirewallSubnet: 10.0.2.0/26</div>
                <div class="security">Azure Firewall</div>
                <div class="subnet">Shared Services: 10.0.10.0/24</div>
                <div class="component">DNS Servers</div>
                <div class="component">Domain Controllers</div>
            </div>
            
            <!-- Production Region (East US) -->
            <div class="region prod-region">
                <div class="region-title">🚀 Production - East US</div>
                <div class="load-balancer">Application Gateway v2</div>
                <div class="security">WAF Protection</div>
                <div class="subnet">Web Tier: 10.1.1.0/24</div>
                <div class="subnet">App Tier: 10.1.2.0/24</div>
                <div class="load-balancer">Internal Load Balancer</div>
                <div class="subnet">Data Tier: 10.1.3.0/24</div>
                <div class="subnet">Integration: 10.1.4.0/24</div>
            </div>
            
            <!-- Development Region (West US 2) -->
            <div class="region dev-region">
                <div class="region-title">🔧 Development - West US 2</div>
                <div class="subnet">Dev Web: 10.2.1.0/24</div>
                <div class="subnet">Dev App: 10.2.2.0/24</div>
                <div class="subnet">Dev Data: 10.2.3.0/24</div>
                <div class="component">Scale Set (3 VMs)</div>
            </div>
            
            <!-- Connection Lines -->
            <div class="connection conn-er">
                <div class="connection-label" style="top: -25px; left: 120px;">ExpressRoute (1Gbps)</div>
            </div>
            
            <div class="connection conn-vpn">
                <div class="connection-label" style="top: 15px; left: 120px;">Site-to-Site VPN</div>
            </div>
            
            <div class="connection conn-peering1">
                <div class="connection-label" style="top: -25px; left: 20px;">VNet Peering</div>
            </div>
            
            <div class="connection conn-peering2">
                <div class="connection-label" style="top: 15px; left: 20px;">VNet Peering</div>
            </div>
            
            <!-- Additional connection lines for global services -->
            <div class="connection" style="top: 125px; left: 650px; width: 550px; transform: rotate(5deg);">
                <div class="connection-label" style="top: -25px; left: 200px;">HTTPS/443</div>
            </div>
            
            <!-- Legend -->
            <div class="legend">
                <h4 style="margin-top: 0;">🔍 Component Legend</h4>
                <div class="legend-item">
                    <div class="legend-color" style="background: linear-gradient(145deg, #e8f4fd, #dbeafe);"></div>
                    <span>Subnets & Networks</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: linear-gradient(145deg, #fff3cd, #ffeaa7);"></div>
                    <span>Gateways</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: linear-gradient(145deg, #f8d7da, #f5c6cb);"></div>
                    <span>Security Components</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: linear-gradient(145deg, #d1ecf1, #bee5eb);"></div>
                    <span>Load Balancers</span>
                </div>
            </div>
        </div>
        
        <div style="margin-top: 30px; padding: 20px; background: linear-gradient(135deg, #f8f9fa, #e9ecef); border-radius: 15px;">
            <h3>🔗 Traffic Flow Summary</h3>
            <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; margin-top: 15px;">
                <div style="background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
                    <h4 style="color: #e74c3c; margin-top: 0;">🌐 Internet → Application</h4>
                    <p>Internet → Front Door → Application Gateway → Web Tier VMs</p>
                </div>
                <div style="background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
                    <h4 style="color: #27ae60; margin-top: 0;">🏢 On-Premises → Azure</h4>
                    <p>Corporate Network → ExpressRoute/VPN → Hub VNet → Spoke VNets</p>
                </div>
                <div style="background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
                    <h4 style="color: #3498db; margin-top: 0;">🔄 Inter-VNet Communication</h4>
                    <p>Spoke VNets ↔ Hub VNet ↔ Other Spoke VNets via VNet Peering</p>
                </div>
                <div style="background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
                    <h4 style="color: #f39c12; margin-top: 0;">🛡️ Security Enforcement</h4>
                    <p>All traffic flows through Azure Firewall, WAF, and NSG controls</p>
                </div>
            </div>
        </div>
    </div>
</body>
</html>
