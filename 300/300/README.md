# 300 - Azure Virtual Network Reverse Engineering

## From Azure Portal to Network Packets - A Complete Breakdown

A comprehensive deconstruction of how Azure Virtual Network works, from what you click in the portal down to how packets are routed, isolated, and processed at the physical network layer.

-----

## Table of Contents

- [Layer 1: What You See (Azure Portal)](#layer-1-what-you-see-azure-portal)
- [Layer 2: Virtual Network Components & Structure](#layer-2-virtual-network-components--structure)
- [Layer 3: How Virtual Networks Work](#layer-3-how-virtual-networks-work)
- [Layer 4: Under the Hood - The Technology](#layer-4-under-the-hood---the-technology)
- [Layer 5: Mapping to TCP/IP Model](#layer-5-mapping-to-tcpip-model)
- [Layer 6: Traffic Flow Patterns](#layer-6-traffic-flow-patterns)
- [Layer 7: Physical Implementation](#layer-7-physical-implementation)
- [Layer 8: Practical Examples](#layer-8-practical-examples)

-----

## Layer 1: What You See (Azure Portal)

### The User Interface View

When you navigate to a Virtual Network in the Azure Portal, here’s what you see:

```
Azure Portal → Resource Groups → Your Virtual Network
├── Overview
│   ├── Essentials (Name, Resource Group, Location, Address Space)
│   ├── Connected devices
│   ├── Subnets
│   └── Peerings
├── Address space
├── Subnets
├── Connected devices
├── Peerings
├── Service endpoints
├── Private endpoints
├── DNS servers
├── DDoS protection
├── Firewall
├── Security
│   ├── Azure Firewall
│   └── Network security groups
├── Monitoring
│   ├── Insights
│   ├── Diagnostic settings
│   └── Logs
└── Settings
```

### Creating a Virtual Network: The Simple View

**Portal Steps**:

1. Click “Create a resource”
1. Search “Virtual Network”
1. **Basics Tab**:

- **Name**: `MyVNet`
- **Region**: `West Europe`
- **Resource Group**: Select or create

1. **IP Addresses Tab**:

- **IPv4 address space**: `10.0.0.0/16`
- **Add subnet**:
  - Name: `default`
  - Address range: `10.0.1.0/24`

1. **Security Tab**:

- **BastionHost**: Enable/Disable
- **DDoS Protection**: Basic/Standard
- **Firewall**: Enable/Disable

1. **Tags** and **Review + Create**

**What Just Happened?**

- Azure allocated IP address space (private RFC 1918)
- Created software-defined network (SDN) infrastructure
- Configured routing tables automatically
- Enabled VM-to-VM communication within VNet
- Isolated this VNet from other VNets
- Set up DNS resolution
- Configured default outbound internet access

### The Abstraction

**What Azure Hides From You**:

```
Simple Portal View:
┌──────────────────────────────────────┐
│  Virtual Network                      │
│  - Name: MyVNet                      │
│  - Address Space: 10.0.0.0/16       │
│  - Subnet: 10.0.1.0/24               │
│  - Region: West Europe               │
└──────────────────────────────────────┘

What's Actually Running:
┌────────────────────────────────────────────────────┐
│ Azure Virtual Network Infrastructure                │
│ - Software Defined Networking (SDN) controllers    │
│ - Virtual Filtering Platform (VFP) on every host   │
│ - Distributed routing tables across hosts          │
│ - VXLAN/NVGRE tunnel encapsulation                │
│ - Network isolation via VNI (Virtual Network ID)   │
│ - Automatic ARP proxy services                     │
│ - Distributed firewall (NSG) enforcement           │
│ - Load balancer integration (Azure LB fabric)      │
│ - NAT gateway services                             │
│ - Service endpoint policy enforcement              │
│ - Private Link connectivity                        │
│ - VNet peering mesh routing                        │
│ - Flow logging and packet capture capability       │
│ - DDoS protection at edge                          │
│ - Integration with Azure Backbone                  │
└────────────────────────────────────────────────────┘
```

**Key Concept**: Azure Virtual Network is NOT a physical network—it’s a software-defined overlay network that provides Layer 3 (IP) connectivity with complete isolation between different VNets, all running on shared physical infrastructure.

### Virtual Network vs Traditional Network

**Critical Difference**:

```
Traditional Physical Network:
┌────────────────────────────────────┐
│ Physical Switch                     │
│ - MAC address learning             │
│ - Physical VLANs                   │
│ - Spanning tree protocol           │
│ - Physical cables and ports        │
│ - Hardware-based forwarding        │
└────────────────────────────────────┘

Azure Virtual Network:
┌────────────────────────────────────┐
│ Software Defined Network            │
│ - SDN controller managed           │
│ - Virtual network isolation        │
│ - Overlay networking (VXLAN)       │
│ - Dynamic routing                  │
│ - Software-based forwarding        │
│ - Policy-based networking          │
│ - No spanning tree needed          │
│ - Instant provisioning             │
└────────────────────────────────────┘

Key Differences:

Physical Network:
- Fixed topology
- Manual configuration
- Physical cable limitations
- Hardware costs
- Configuration drift
- Slow provisioning (hours/days)

Azure VNet:
- Dynamic topology
- API-driven configuration
- Software-defined connectivity
- No hardware costs
- Consistent configuration
- Instant provisioning (seconds)
```

-----

## Layer 2: Virtual Network Components & Structure

### Anatomy of a Virtual Network

A Virtual Network consists of many interconnected components:

#### 1. Virtual Network Resource

```json
{
  "name": "MyVNet",
  "location": "westeurope",
  "type": "Microsoft.Network/virtualNetworks",
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "10.0.0.0/16"
      ]
    },
    "dhcpOptions": {
      "dnsServers": []
    },
    "subnets": [],
    "virtualNetworkPeerings": [],
    "enableDdosProtection": false,
    "enableVmProtection": false
  }
}
```

**Address Space Explained**:

```
RFC 1918 Private Address Ranges:
├── 10.0.0.0/8     (16,777,216 IPs) - Class A
├── 172.16.0.0/12  (1,048,576 IPs)  - Class B
└── 192.168.0.0/16 (65,536 IPs)     - Class C

Common VNet Sizes:

Large Organization:
10.0.0.0/8 → 16.7 million IPs
Use case: Large enterprise with many VNets

Medium Organization:
10.0.0.0/16 → 65,536 IPs
Use case: Standard production environment

Small Deployment:
10.0.0.0/24 → 256 IPs
Use case: Development/test environment

Best Practice:
- Plan for growth
- Don't use entire /8 for one VNet
- Reserve space for VNet peering
- Avoid overlapping with on-premises
- Use /16 or /20 for most VNets
```

**Address Space Calculation**:

```
CIDR Notation to IP Count:

/8  → 2^(32-8)  = 16,777,216 IPs
/16 → 2^(32-16) = 65,536 IPs
/20 → 2^(32-20) = 4,096 IPs
/24 → 2^(32-24) = 256 IPs
/28 → 2^(32-28) = 16 IPs

Example: 10.0.0.0/16
- Network: 10.0.0.0
- First usable: 10.0.0.4 (Azure reserves .0-.3)
- Last usable: 10.0.255.254 (Azure reserves .255)
- Broadcast: 10.0.255.255
- Total IPs: 65,536
- Azure reserved: 5 per subnet
- Usable: 65,531 (if single subnet)
```

#### 2. Subnets

**Subnets** divide the VNet address space into segments:

```json
{
  "name": "WebSubnet",
  "properties": {
    "addressPrefix": "10.0.1.0/24",
    "networkSecurityGroup": {
      "id": "/subscriptions/.../networkSecurityGroups/WebNSG"
    },
    "routeTable": {
      "id": "/subscriptions/.../routeTables/WebRouteTable"
    },
    "serviceEndpoints": [
      {
        "service": "Microsoft.Storage"
      },
      {
        "service": "Microsoft.Sql"
      }
    ],
    "delegations": [],
    "privateEndpointNetworkPolicies": "Disabled",
    "privateLinkServiceNetworkPolicies": "Enabled"
  }
}
```

**Subnet Design Patterns**:

```
Three-Tier Application:
VNet: 10.0.0.0/16

├─ WebSubnet: 10.0.1.0/24
│  └─ Web servers (public-facing)
│
├─ AppSubnet: 10.0.2.0/24
│  └─ Application servers (internal)
│
├─ DataSubnet: 10.0.3.0/24
│  └─ Database servers (internal)
│
├─ ManagementSubnet: 10.0.4.0/24
│  └─ Jump boxes, monitoring
│
└─ GatewaySubnet: 10.0.255.0/27
   └─ VPN/ExpressRoute gateway (special name required)

Security Model:
- WebSubnet: NSG allows 80/443 from Internet
- AppSubnet: NSG allows traffic only from WebSubnet
- DataSubnet: NSG allows traffic only from AppSubnet
- ManagementSubnet: NSG allows only admin IPs
- GatewaySubnet: Managed by Azure
```

**Azure Reserved IP Addresses**:

```
Every Subnet Reserves 5 IPs:

Example Subnet: 10.0.1.0/24 (256 IPs)

Reserved Addresses:
├─ 10.0.1.0   - Network address
├─ 10.0.1.1   - Azure default gateway
├─ 10.0.1.2   - Azure DNS mapping
├─ 10.0.1.3   - Azure DNS mapping
└─ 10.0.1.255 - Network broadcast

Available: 10.0.1.4 - 10.0.1.254 (251 IPs)

Why Azure Reserves These:
.1 → Default gateway for VMs
.2, .3 → DNS servers (168.63.129.16 maps to these)
.0, .255 → Network/broadcast (RFC standard)

Practical Impact:
/29 subnet (8 IPs) → Only 3 usable IPs
/28 subnet (16 IPs) → Only 11 usable IPs
/24 subnet (256 IPs) → 251 usable IPs

Minimum Subnet Sizes:
- Regular workload: /27 (32 IPs, 27 usable)
- GatewaySubnet: /27 minimum (recommended /26)
- AzureBastionSubnet: /26 or larger (required)
- AzureFirewallSubnet: /26 or larger (required)
```

#### 3. Network Security Groups (NSGs)

**NSGs** are stateful firewalls attached to subnets or NICs:

```json
{
  "name": "WebNSG",
  "properties": {
    "securityRules": [
      {
        "name": "AllowHTTPS",
        "properties": {
          "priority": 100,
          "direction": "Inbound",
          "access": "Allow",
          "protocol": "Tcp",
          "sourceAddressPrefix": "*",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "443"
        }
      },
      {
        "name": "AllowHTTP",
        "properties": {
          "priority": 110,
          "direction": "Inbound",
          "access": "Allow",
          "protocol": "Tcp",
          "sourceAddressPrefix": "*",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "80"
        }
      },
      {
        "name": "DenyAllInbound",
        "properties": {
          "priority": 4096,
          "direction": "Inbound",
          "access": "Deny",
          "protocol": "*",
          "sourceAddressPrefix": "*",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "*"
        }
      }
    ]
  }
}
```

**NSG Rule Processing**:

```
NSG Rule Evaluation:

Priority: 100-4096 (lower number = higher priority)
Processing: Sequential by priority
Type: Stateful (return traffic automatic)

Example Traffic Flow:
Incoming packet: TCP 1.2.3.4:52000 → 10.0.1.4:443

Rule Evaluation:
┌─────────────────────────────────────────┐
│ Priority 100: Allow 443                 │
│ - Source: * (matches 1.2.3.4)           │
│ - Destination Port: 443 (matches)       │
│ - Protocol: TCP (matches)               │
│ → MATCH: ALLOW ✓                        │
│ → Stop processing (first match wins)    │
└─────────────────────────────────────────┘

Return Traffic (Automatic):
10.0.1.4:443 → 1.2.3.4:52000
- NSG automatically allows (stateful)
- No explicit rule needed for return traffic

Different Packet: TCP 1.2.3.4:52000 → 10.0.1.4:22

Rule Evaluation:
┌─────────────────────────────────────────┐
│ Priority 100: Allow 443                 │
│ - Destination Port: 443 ≠ 22            │
│ → No match, continue                    │
│                                         │
│ Priority 110: Allow 80                  │
│ - Destination Port: 80 ≠ 22             │
│ → No match, continue                    │
│                                         │
│ Priority 4096: Deny All                 │
│ - Source: * (matches)                   │
│ - Destination Port: * (matches)         │
│ → MATCH: DENY ✗                         │
└─────────────────────────────────────────┘

Result: Packet dropped
```

**Default NSG Rules**:

```
Every NSG Includes Default Rules (Cannot Delete):

Inbound Default Rules:
Priority 65000: Allow VirtualNetwork → VirtualNetwork
Priority 65001: Allow AzureLoadBalancer → *
Priority 65500: Deny * → *

Outbound Default Rules:
Priority 65000: Allow VirtualNetwork → VirtualNetwork
Priority 65001: Allow * → Internet
Priority 65500: Deny * → *

Service Tags:
- VirtualNetwork: All IPs in this VNet and peered VNets
- Internet: All public IPs
- AzureLoadBalancer: Azure health probe IPs
- Storage: Azure Storage IPs
- Sql: Azure SQL IPs
- AzureActiveDirectory: Azure AD IPs

Example Rule Using Service Tags:
{
  "name": "AllowStorage",
  "sourceAddressPrefix": "VirtualNetwork",
  "destinationAddressPrefix": "Storage",
  "destinationPortRange": "443",
  "access": "Allow"
}

Benefits of Service Tags:
✓ No need to track IP ranges
✓ Automatically updated by Azure
✓ Regional scoping available
✓ Simplified rule management
```

#### 4. Route Tables (User-Defined Routes)

**Route Tables** control routing decisions:

```json
{
  "name": "WebRouteTable",
  "properties": {
    "routes": [
      {
        "name": "RouteToFirewall",
        "properties": {
          "addressPrefix": "0.0.0.0/0",
          "nextHopType": "VirtualAppliance",
          "nextHopIpAddress": "10.0.100.4"
        }
      },
      {
        "name": "RouteToOnPremises",
        "properties": {
          "addressPrefix": "192.168.0.0/16",
          "nextHopType": "VirtualNetworkGateway"
        }
      }
    ],
    "disableBgpRoutePropagation": false
  }
}
```

**Route Types**:

```
Azure Route Types:

1. System Routes (Automatic):
   - Within VNet: Direct
   - VNet Peering: Automatic routes
   - Internet: Via Azure edge
   - VPN Gateway: Via gateway

2. User-Defined Routes (UDR):
   - Override system routes
   - Force traffic through appliance
   - Control routing explicitly

Next Hop Types:
┌──────────────────────────────────────────┐
│ VirtualAppliance                         │
│ - Azure Firewall                         │
│ - Network Virtual Appliance (NVA)       │
│ - Custom routing device                  │
│ Example: Send all traffic to firewall   │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ VirtualNetworkGateway                    │
│ - VPN Gateway                            │
│ - ExpressRoute Gateway                   │
│ Example: Route to on-premises           │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ VnetLocal                                │
│ - Keep traffic in VNet                   │
│ - Override default routing               │
│ Example: Force subnet-to-subnet direct  │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ Internet                                 │
│ - Direct to Azure edge                   │
│ - Default for public IPs                 │
│ Example: Explicit internet route         │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ None                                     │
│ - Drop traffic (blackhole)               │
│ - No forwarding                          │
│ Example: Prevent internet access         │
└──────────────────────────────────────────┘
```

**Route Selection**:

```
Route Table Decision Process:

VM in 10.0.1.0/24 wants to reach 10.0.2.5

Available Routes:
1. System Route: 10.0.0.0/16 → VnetLocal
2. UDR: 0.0.0.0/0 → VirtualAppliance (10.0.100.4)

Routing Decision:
- Most specific prefix wins
- 10.0.0.0/16 more specific than 0.0.0.0/0
- Use System Route: VnetLocal
- Traffic goes direct to 10.0.2.5

VM wants to reach 8.8.8.8 (Google DNS):

Available Routes:
1. System Route: 0.0.0.0/0 → Internet
2. UDR: 0.0.0.0/0 → VirtualAppliance (10.0.100.4)

Routing Decision:
- Same specificity (/0)
- UDR takes precedence over System Route
- Use UDR: Next hop 10.0.100.4
- Traffic sent to firewall first

Route Priority:
1. Most specific prefix (/32 > /24 > /16 > /0)
2. User-Defined Routes > System Routes
3. BGP routes (if enabled)

Example Complex Routing:
Routes:
- 0.0.0.0/0 → VirtualAppliance (catch-all)
- 10.0.0.0/16 → VnetLocal (this VNet)
- 10.1.0.0/16 → VnetPeering (peered VNet)
- 192.168.0.0/16 → VPN Gateway (on-premises)
- 20.0.0.0/8 → Internet (Azure public services)
- 168.63.129.16/32 → Internet (Azure DNS)

Traffic to 10.0.2.5: VnetLocal (most specific)
Traffic to 10.1.2.5: VnetPeering (most specific)
Traffic to 192.168.1.1: VPN Gateway (most specific)
Traffic to 8.8.8.8: VirtualAppliance (default route)
Traffic to 168.63.129.16: Internet (most specific)
```

#### 5. VNet Peering

**VNet Peering** connects VNets directly:

```json
{
  "name": "VNet1-to-VNet2",
  "properties": {
    "remoteVirtualNetwork": {
      "id": "/subscriptions/.../virtualNetworks/VNet2"
    },
    "allowVirtualNetworkAccess": true,
    "allowForwardedTraffic": true,
    "allowGatewayTransit": false,
    "useRemoteGateways": false,
    "peeringState": "Connected"
  }
}
```

**Peering Types**:

```
Regional VNet Peering:
┌────────────────────────────────────────┐
│ VNet1 (West Europe)                     │
│ 10.0.0.0/16                            │
└────────────────────────────────────────┘
              ↕ (Peered)
┌────────────────────────────────────────┐
│ VNet2 (West Europe)                     │
│ 10.1.0.0/16                            │
└────────────────────────────────────────┘

Global VNet Peering:
┌────────────────────────────────────────┐
│ VNet1 (West Europe)                     │
│ 10.0.0.0/16                            │
└────────────────────────────────────────┘
              ↕ (Global Peering)
┌────────────────────────────────────────┐
│ VNet2 (East US)                         │
│ 10.1.0.0/16                            │
└────────────────────────────────────────┘

Key Differences:
Regional Peering:
- Same Azure region
- Lower latency
- No data transfer cost within region
- Faster setup

Global Peering:
- Different Azure regions
- Higher latency (geographic distance)
- Data transfer charges apply
- Cross-region DR scenarios
```

**Peering Properties**:

```
allowVirtualNetworkAccess: true
- Enables IP connectivity between VNets
- VMs can communicate via private IPs
- No internet gateway needed

allowForwardedTraffic: true/false
- Allow traffic from other VNets via this VNet
- Required for hub-spoke topology
- Enables transit routing

Example:
VNet1 ←→ Hub VNet ←→ VNet2
        (Firewall)

VNet1 → Hub VNet:
  allowForwardedTraffic: true (on Hub peering)

Hub VNet → VNet2:
  allowForwardedTraffic: true (on VNet2 peering)

Result: VNet1 can reach VNet2 via Hub
(Hub must have UDR to route traffic)

allowGatewayTransit: true/false
- Share VPN/ExpressRoute gateway
- Only one VNet has gateway
- Others use it via peering

Hub VNet (has VPN Gateway):
  allowGatewayTransit: true

Spoke VNet:
  useRemoteGateways: true

Result: Spoke uses Hub's gateway to reach on-premises

useRemoteGateways: true/false
- Use peer's gateway
- Peer must have allowGatewayTransit: true
- Cannot have own gateway if using remote

Peering State:
- Initiated: Peering created, waiting for peer
- Connected: Both sides configured, active
- Disconnected: Peering broken
- Updating: Configuration change in progress
```

**Peering Limitations**:

```
❌ Non-Transitive by Default:
VNet1 ←→ VNet2 ←→ VNet3

VNet1 cannot reach VNet3 directly!
Need explicit peering: VNet1 ←→ VNet3
Or use NVA in VNet2 with routing

❌ No Overlapping Address Spaces:
VNet1: 10.0.0.0/16
VNet2: 10.0.0.0/16 ← CANNOT PEER!

Must use non-overlapping:
VNet1: 10.0.0.0/16
VNet2: 10.1.0.0/16 ✓

✓ Limitations:
- Max 500 peerings per VNet
- Cross-subscription supported
- Cross-tenant supported (with permissions)
- No SLA impact (same as single VNet)
- Encryption: Always encrypted (Azure backbone)
```

#### 6. Service Endpoints

**Service Endpoints** provide direct access to Azure services:

```json
{
  "subnet": {
    "properties": {
      "serviceEndpoints": [
        {
          "service": "Microsoft.Storage",
          "locations": ["westeurope"]
        },
        {
          "service": "Microsoft.Sql",
          "locations": ["westeurope"]
        },
        {
          "service": "Microsoft.KeyVault",
          "locations": ["westeurope"]
        }
      ]
    }
  }
}
```

**How Service Endpoints Work**:

```
Without Service Endpoint:
VM (10.0.1.4) → Internet → Storage Account
- Traffic via NAT gateway or public IP
- Uses storage account public endpoint
- Travels over internet
- Can be blocked by NSG default deny internet

With Service Endpoint:
VM (10.0.1.4) → Azure Backbone → Storage Account
- Traffic stays on Azure network
- Optimized route (Microsoft backbone)
- Better performance
- Source IP preserved (VM private IP)
- Can restrict storage to specific VNets

Configuration Flow:
1. Enable service endpoint on subnet
   Subnet → Properties → Microsoft.Storage

2. Configure service firewall
   Storage Account → Networking
   - Allow access from: Selected networks
   - Add VNet: MyVNet/WebSubnet

3. Traffic automatically routed via backbone

Benefits:
✓ Improved security (no public IP needed)
✓ Better performance (Azure backbone)
✓ Source IP preservation
✓ Service-side firewall (allow only specific VNets)
✓ No additional cost

Available Services:
- Microsoft.Storage (Blob, File, Queue, Table)
- Microsoft.Sql (Azure SQL Database)
- Microsoft.KeyVault
- Microsoft.ServiceBus
- Microsoft.CosmosDB
- Microsoft.EventHub
- Microsoft.AzureActiveDirectory
- Microsoft.ContainerRegistry
```

**Service Endpoint Policies**:

```json
{
  "name": "StoragePolicy",
  "properties": {
    "serviceEndpointPolicyDefinitions": [
      {
        "name": "AllowSpecificStorage",
        "properties": {
          "service": "Microsoft.Storage",
          "serviceResources": [
            "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/mystore"
          ]
        }
      }
    ]
  }
}
```

```
Service Endpoint Policy:
- Restrict which specific services can be accessed
- Works with service endpoints
- Example: Allow only specific storage accounts
- Prevent data exfiltration

Without Policy:
VM can access any storage account via service endpoint

With Policy:
VM can only access approved storage accounts
- mystore.blob.core.windows.net ✓
- otherstorage.blob.core.windows.net ✗ (blocked)

Use Case:
Prevent developer from copying data to personal storage
```

#### 7. Private Endpoints

**Private Endpoints** bring Azure services into your VNet:

```json
{
  "name": "StoragePrivateEndpoint",
  "properties": {
    "subnet": {
      "id": "/subscriptions/.../subnets/PrivateEndpointSubnet"
    },
    "privateLinkServiceConnections": [
      {
        "name": "StorageConnection",
        "properties": {
          "privateLinkServiceId": "/subscriptions/.../storageAccounts/mystore",
          "groupIds": ["blob"]
        }
      }
    ]
  }
}
```

**Service Endpoint vs Private Endpoint**:

```
Service Endpoint:
┌─────────────────────────────────────────┐
│ VM: 10.0.1.4                            │
│ Subnet: ServiceEndpoint enabled         │
└─────────────────────────────────────────┘
         ↓ (Azure Backbone)
┌─────────────────────────────────────────┐
│ Storage Account                          │
│ Public Endpoint: mystore.blob...         │
│ Firewall: Allow VNet traffic            │
│ Still accessible from internet (if FW)  │
└─────────────────────────────────────────┘

Characteristics:
- Uses storage public endpoint
- VNet traffic routed via backbone
- Storage firewall restricts access
- Storage still has public IP
- Multiple VNets can use same endpoint

Private Endpoint:
┌─────────────────────────────────────────┐
│ VM: 10.0.1.4                            │
│ Subnet: Has private endpoint            │
└─────────────────────────────────────────┘
         ↓ (Private IP)
┌─────────────────────────────────────────┐
│ Private Endpoint NIC: 10.0.2.4          │
│ DNS: mystore.blob... → 10.0.2.4         │
└─────────────────────────────────────────┘
         ↓ (Private Link)
┌─────────────────────────────────────────┐
│ Storage Account                          │
│ Private IP: 10.0.2.4 (in your VNet)     │
│ Public endpoint can be disabled         │
│ Fully private connectivity              │
└─────────────────────────────────────────┘

Characteristics:
- Storage gets private IP in your VNet
- No internet connectivity required
- DNS automatically updated
- Can disable public access entirely
- Each VNet needs own private endpoint

Comparison:
┌──────────────────┬────────────────┬─────────────────┐
│ Feature          │ Service EP     │ Private EP      │
├──────────────────┼────────────────┼─────────────────┤
│ Service IP       │ Public         │ Private (VNet)  │
│ DNS              │ Public FQDN    │ Private IP      │
│ Network path     │ Azure backbone │ VNet direct     │
│ Public access    │ Possible       │ Blockable       │
│ Cost             │ Free           │ $7.30/month     │
│ VNet limitation  │ Same region    │ Any region      │
│ Data exfiltration│ Harder to stop │ Fully controlled│
└──────────────────┴────────────────┴─────────────────┘

When to Use:
Service Endpoint:
✓ Cost-sensitive
✓ Simple scenarios
✓ Public access still needed elsewhere

Private Endpoint:
✓ Maximum security
✓ Zero internet exposure required
✓ Cross-region access
✓ Compliance mandates
```

**Private Endpoint DNS**:

```
DNS Resolution with Private Endpoint:

Before Private Endpoint:
nslookup mystore.blob.core.windows.net
→ Returns: 20.50.60.70 (public IP)

After Private Endpoint:
1. Create private endpoint in VNet
2. Azure creates Private DNS Zone (optional but recommended)
   - privatelink.blob.core.windows.net
   - A record: mystore → 10.0.2.4

3. Link Private DNS Zone to VNet

4. DNS resolution (from VM in VNet):
   nslookup mystore.blob.core.windows.net
   
   Query chain:
   mystore.blob.core.windows.net
   → CNAME: mystore.privatelink.blob.core.windows.net
   → A record (Private DNS): 10.0.2.4 ✓

5. From internet/other VNets without zone:
   nslookup mystore.blob.core.windows.net
   → Returns: 20.50.60.70 (public IP)
   → But blocked by firewall

DNS Configuration:
┌────────────────────────────────────────┐
│ Private DNS Zone                        │
│ privatelink.blob.core.windows.net       │
│                                        │
│ Records:                               │
│ mystore  A  10.0.2.4                   │
└────────────────────────────────────────┘
         ↓ (Linked to)
┌────────────────────────────────────────┐
│ Virtual Network                         │
│ VMs use Azure-provided DNS             │
│ Automatically resolves private IPs     │
└────────────────────────────────────────┘

Benefits:
✓ No application changes needed
✓ Same FQDN works (mystore.blob...)
✓ Automatic private IP resolution
✓ Transparent to applications
```

#### 8. NAT Gateway

**NAT Gateway** provides scalable outbound internet:

```json
{
  "name": "MyNATGateway",
  "properties": {
    "sku": {
      "name": "Standard"
    },
    "idleTimeoutInMinutes": 4,
    "publicIpAddresses": [
      {
        "id": "/subscriptions/.../publicIPAddresses/NAT-IP"
      }
    ],
    "publicIpPrefixes": []
  }
}
```

**NAT Gateway vs VM Public IP**:

```
Without NAT Gateway:
┌────────────────────────────────────────┐
│ VM1: 10.0.1.4 (no public IP)            │
└────────────────────────────────────────┘
         ↓ (SNAT)
┌────────────────────────────────────────┐
│ Azure Load Balancer                     │
│ Dynamic public IP for outbound         │
│ Shared ports, potential exhaustion     │
└────────────────────────────────────────┘

Issues:
- Port exhaustion (64K ports shared)
- Non-deterministic IP
- Connection failures under load
- Cannot whitelist IP

With NAT Gateway:
┌────────────────────────────────────────┐
│ VM1: 10.0.1.4                           │
│ VM2: 10.0.1.5                           │
│ VM3: 10.0.1.6                           │
└────────────────────────────────────────┘
         ↓ (All outbound traffic)
┌────────────────────────────────────────┐
│ NAT Gateway                             │
│ Public IP: 40.50.60.80 (static)        │
│ 64K ports per IP                       │
│ Up to 16 public IPs (1M ports)         │
└────────────────────────────────────────┘
         ↓
    Internet

Benefits:
✓ Predictable outbound IP
✓ 64K ports per IP per destination
✓ Scales to 16 IPs automatically
✓ Higher throughput (up to 50 Gbps)
✓ Lower latency
✓ Can whitelist IP at external services

NAT Gateway Features:
- Automatic port allocation
- Idle timeout: 4-120 minutes
- Zone redundant (optional)
- Replaces load balancer for outbound
- Works with service endpoints

Use Cases:
✓ High-volume outbound connections
✓ Many concurrent flows
✓ API calls to external services
✓ Whitelisting at destination
✓ Predictable outbound IP
```

#### 9. Azure Bastion

**Azure Bastion** provides secure RDP/SSH:

```
Without Bastion:
┌────────────────────────────────────────┐
│ VM with Public IP                       │
│ RDP port 3389 exposed                  │
│ SSH port 22 exposed                    │
│ NSG rules to allow your IP             │
│ Attack surface: Internet-facing        │
└────────────────────────────────────────┘

Security Issues:
❌ RDP/SSH exposed to internet
❌ Brute force attacks
❌ Must maintain public IPs
❌ Complex NSG rules
❌ Credential attacks

With Azure Bastion:
┌────────────────────────────────────────┐
│ Azure Bastion (PaaS)                    │
│ Deployed in AzureBastionSubnet          │
│ Public IP (managed by Azure)           │
│ RDP/SSH over TLS (port 443)            │
└────────────────────────────────────────┘
         ↓ (Private connectivity)
┌────────────────────────────────────────┐
│ VM without Public IP                    │
│ Private IP only: 10.0.1.4               │
│ No RDP/SSH ports exposed                │
│ No inbound NSG rules needed             │
└────────────────────────────────────────┘

How It Works:
1. User → Azure Portal → Connect via Bastion
2. Portal → Bastion (HTTPS/443)
3. Bastion → VM (RDP 3389 or SSH 22 privately)
4. Session over TLS in browser
5. No client software needed

Benefits:
✓ No public IP on VMs
✓ No RDP/SSH exposed
✓ Portal-integrated
✓ TLS encryption
✓ Just-in-Time access (with JIT)
✓ MFA integration
✓ Centralized access logging

Bastion SKUs:
Basic:
- 2 concurrent sessions
- RDP/SSH only
- $140/month

Standard:
- 50 concurrent sessions
- Shareable link
- Native client support
- $370/month

Subnet Requirements:
Name: AzureBastionSubnet (exact name required)
Size: /26 or larger (64 IPs)
NSG: Specific rules required
```

-----

## Layer 3: How Virtual Networks Work

### IP Address Allocation

**DHCP in Azure VNets**:

```
Azure DHCP Service:
- Automatic IP allocation
- VMs get IPs from subnet range
- Excludes Azure-reserved IPs (.0-.3, .255)
- Allocation: Dynamic or Static

Dynamic Allocation (Default):
┌────────────────────────────────────────┐
│ VM Created in 10.0.1.0/24              │
│ Azure DHCP assigns: 10.0.1.4           │
│ (First available IP)                   │
└────────────────────────────────────────┘

IP Stays with VM as long as:
- VM is running
- VM is stopped (deallocated)
- VM lease not expired

IP May Change if:
- VM deleted and recreated
- VM moved to different subnet
- VM resized (some cases)

Static Allocation:
Set IP to: 10.0.1.10

┌────────────────────────────────────────┐
│ VM always gets: 10.0.1.10              │
│ Survives stop/start                    │
│ Survives resize                        │
│ Released only when NIC deleted         │
└────────────────────────────────────────┘

Best Practices:
- Use dynamic for most VMs
- Use static for:
  * Domain controllers
  * DNS servers
  * Application servers (referenced by IP)
  * VPN endpoint IPs

DHCP Lease Renewal:
- Azure DHCP lease: ~Infinite
- VM keeps IP while allocated
- No lease time concerns like traditional DHCP
```

**Network Interface (NIC) Configuration**:

```
NIC Properties:
┌────────────────────────────────────────┐
│ Primary NIC (VM)                        │
│ Name: vm1-nic                          │
│                                        │
│ IP Configurations:                     │
│ ├─ Primary (Private IP)                │
│ │  - Private IP: 10.0.1.4             │
│ │  - Allocation: Dynamic/Static       │
│ │  - Subnet: WebSubnet               │
│ │                                     │
│ ├─ Public IP (Optional)               │
│ │  - Public IP: 40.50.60.70          │
│ │  - SKU: Basic/Standard             │
│ │                                     │
│ └─ Secondary IPs (Optional)           │
│    - Additional private IPs          │
│    - For multi-IP scenarios          │
│                                        │
│ Network Security Group:                │
│ - Associated NSG rules                │
│                                        │
│ Effective Routes:                      │
│ - System + UDR routes                 │
│                                        │
│ DNS Servers:                           │
│ - Inherit from VNet or custom         │
│                                        │
│ IP Forwarding:                         │
│ - Disabled (default)                  │
│ - Enable for NVA/router               │
│                                        │
│ Accelerated Networking:                │
│ - SR-IOV for performance              │
│ - Up to 30 Gbps                       │
└────────────────────────────────────────┘

Multiple NICs:
Some VM sizes support multiple NICs

Example: 2-NIC VM
├─ NIC 1: Management (10.0.10.0/24)
└─ NIC 2: Data (10.0.20.0/24)

Use Cases:
- Network isolation
- Separate traffic flows
- Network Virtual Appliance (firewall)
- High availability configurations

Accelerated Networking:
Bypass host virtualization stack
VM ←─ SR-IOV ─→ Physical NIC

Benefits:
✓ Lower latency
✓ Higher throughput
✓ Lower CPU utilization
✓ Up to 30 Gbps

Requirements:
- Supported VM size (most Dsv3, Fsv2)
- Linux or Windows
- Enabled at creation (or stop VM to enable)
```

### Routing Mechanics

**Intra-VNet Routing**:

```
Same Subnet Communication:
VM1: 10.0.1.4 → VM2: 10.0.1.5

┌────────────────────────────────────────┐
│ VM1 checks: Is destination in my       │
│ subnet? (10.0.1.0/24)                  │
│ Yes → Send directly                    │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Azure VFP (Virtual Filtering Platform)  │
│ - Performs ARP resolution              │
│ - Gets VM2 MAC address                 │
│ - Forwards frame to VM2 host           │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VM2 receives packet                     │
└────────────────────────────────────────┘

Different Subnet Communication:
VM1: 10.0.1.4 → VM3: 10.0.2.5

┌────────────────────────────────────────┐
│ VM1 checks routing table:               │
│ 10.0.2.0/24 → Default gateway (10.0.1.1)│
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Packet sent to gateway: 10.0.1.1        │
│ (Azure SDN, not a real VM)             │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Azure SDN routing:                      │
│ - Checks NSG rules                     │
│ - Applies UDRs if present              │
│ - Forwards to destination subnet       │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VM3 receives packet                     │
│ Source: 10.0.1.4 (preserved)           │
└────────────────────────────────────────┘

Key Points:
- No actual router VM
- Azure fabric handles routing
- Source IP preserved
- Latency: <1ms typically
- No additional hops visible to VM
```

**Inter-VNet Routing (Peering)**:

```
VNet Peering Communication:
VNet1 VM: 10.0.1.4 → VNet2 VM: 10.1.2.5

┌────────────────────────────────────────┐
│ VNet1 (10.0.0.0/16)                     │
│ VM1: 10.0.1.4                          │
└────────────────────────────────────────┘
         ↓ (Route: 10.1.0.0/16 via peering)
┌────────────────────────────────────────┐
│ Azure SDN Fabric                        │
│ - Checks peering status                │
│ - Validates NSG rules                  │
│ - Forwards via peering link            │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VNet2 (10.1.0.0/16)                     │
│ VM2: 10.1.2.5                          │
└────────────────────────────────────────┘

Peering Routing Properties:
- Automatic route injection
- No gateway needed
- Lowest latency path
- Microsoft backbone (global peering)
- Encrypted by default

Route Table Entry (Automatic):
Destination: 10.1.0.0/16
Next Hop: VNetPeering
Status: Active
Source: Peering: VNet1-to-VNet2

Hub-Spoke Routing:
Spoke1 → Hub → Spoke2

Spoke1: 10.0.0.0/16
Hub: 10.100.0.0/16 (NVA: 10.100.1.4)
Spoke2: 10.1.0.0/16

Configuration:
1. Spoke1 ←→ Hub peering
   - allowForwardedTraffic: true
   
2. Hub ←→ Spoke2 peering
   - allowForwardedTraffic: true
   
3. UDR in Spoke1:
   Destination: 10.1.0.0/16
   Next Hop: VirtualAppliance (10.100.1.4)
   
4. UDR in Spoke2:
   Destination: 10.0.0.0/16
   Next Hop: VirtualAppliance (10.100.1.4)

Traffic Flow:
Spoke1 VM → UDR routes to Hub NVA
Hub NVA → Forwards to Spoke2 (IP forwarding enabled)
Spoke2 VM receives packet

Requirements:
✓ allowForwardedTraffic on both peerings
✓ NVA has IP forwarding enabled
✓ UDRs in both spokes
✓ NVA configured to route
```

### Network Isolation

**How VNets Are Isolated**:

```
VNet Isolation Mechanism:

┌────────────────────────────────────────┐
│ VNet1 (Customer A)                      │
│ VNI: 5001                              │
│ Address Space: 10.0.0.0/16             │
│ VM1: 10.0.1.4                          │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ VNet2 (Customer B)                      │
│ VNI: 5002                              │
│ Address Space: 10.0.0.0/16 (same!)     │
│ VM2: 10.0.1.4 (same IP!)               │
└────────────────────────────────────────┘

Physical Network:
┌────────────────────────────────────────┐
│ Shared Azure Physical Network          │
│                                        │
│ VM1 packet encapsulation:              │
│ Outer Header: VNI 5001                 │
│ Inner Header: Src 10.0.1.4             │
│                                        │
│ VM2 packet encapsulation:              │
│ Outer Header: VNI 5002                 │
│ Inner Header: Src 10.0.1.4             │
└────────────────────────────────────────┘

Key Isolation Techniques:

1. Virtual Network Identifier (VNI):
   - Unique ID per VNet
   - VXLAN/NVGRE encapsulation
   - Prevents cross-VNet traffic

2. Software-Defined Networking:
   - VFP on every host
   - Policy enforcement
   - Traffic segregation

3. Overlay Networking:
   - Tunneling protocols
   - Logical isolation
   - Physical network shared

VM1 cannot reach VM2 even with same IP:
- Different VNI
- SDN blocks cross-VNet traffic
- No routing between VNets (unless peered)
- Complete isolation

Security Boundaries:
VNet = Security Boundary
- Default deny between VNets
- Explicit peering required
- NSGs enforce rules
- No ARP across VNets
```

**Network Security Group Processing**:

```
NSG Evaluation Order:

Subnet NSG + NIC NSG = Combined Rules

Inbound Traffic:
Internet → Subnet NSG → NIC NSG → VM

Outbound Traffic:
VM → NIC NSG → Subnet NSG → Internet

Example Scenario:
VM1 (10.0.1.4) in WebSubnet
- Subnet NSG: WebSubnetNSG
- NIC NSG: VM1-NIC-NSG

Inbound Packet: 1.2.3.4:52000 → 10.0.1.4:80

Step 1: Subnet NSG Evaluation
┌────────────────────────────────────────┐
│ WebSubnetNSG Rules:                     │
│ Priority 100: Allow 80 from Internet   │
│ → MATCH: ALLOW                         │
└────────────────────────────────────────┘
         ↓ (Allowed, continue)
Step 2: NIC NSG Evaluation
┌────────────────────────────────────────┐
│ VM1-NIC-NSG Rules:                      │
│ Priority 100: Allow 80 from 10.0.0.0/16│
│ Source: 1.2.3.4 (Internet)             │
│ → NO MATCH                             │
│ Priority 200: Deny All                 │
│ → MATCH: DENY ✗                        │
└────────────────────────────────────────┘

Result: Packet DROPPED at NIC NSG

Key Points:
- Both NSGs must allow traffic
- Most restrictive wins
- Subnet NSG ≠ bypass for NIC NSG
- Order: Subnet → NIC (inbound)
- Order: NIC → Subnet (outbound)

Best Practices:
- Subnet NSG: Broad rules (role-based)
- NIC NSG: Specific rules (VM-specific)
- Or use one or the other (not both)

Example Strategy:
Subnet NSG:
- Allow web traffic (80/443)
- Allow from internal subnets
- Deny all else

NIC NSG:
- None (rely on subnet NSG)

Simpler management, consistent rules
```

-----

## Layer 4: Under the Hood - The Technology

### Azure Software-Defined Networking (SDN)

**SDN Architecture**:

```
Azure SDN Stack:

┌────────────────────────────────────────┐
│ Control Plane (SDN Controller)          │
│ - Manages network policies             │
│ - Distributes routing tables            │
│ - Coordinates VFP instances             │
│ - Handles VNet configuration            │
└────────────────────────────────────────┘
         ↓ (Policy distribution)
┌────────────────────────────────────────┐
│ Data Plane (VFP on Hyper-V Hosts)      │
│ - Packet forwarding                    │
│ - Policy enforcement                   │
│ - Encapsulation/decapsulation          │
│ - NSG rule processing                  │
└────────────────────────────────────────┘

Components:

1. Network Controller:
   - Centralized management
   - REST API for configuration
   - Distributes policies to hosts
   - Monitors network state

2. Virtual Filtering Platform (VFP):
   - Hyper-V extensible switch extension
   - Runs on every compute node
   - Processes packets at wire speed
   - Enforces NSG, routes, NAT, LB

3. Host Networking Service (HNS):
   - Creates virtual switches
   - Manages network namespaces
   - Configures container networking
   - Linux: eBPF/XDP

4. VTEP (VXLAN Tunnel Endpoint):
   - Encapsulates packets
   - VXLAN/NVGRE tunneling
   - Outer IP header
   - VNI tagging
```

**Virtual Filtering Platform (VFP)**:

```
VFP Pipeline:

Packet Arrives at Physical NIC
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 1: Parsing                    │
│ - Extract headers                      │
│ - Identify flow                        │
│ - Determine VNet (VNI)                 │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 2: Decapsulation              │
│ - Remove VXLAN header                  │
│ - Extract inner packet                 │
│ - Get original src/dst                 │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 3: NSG Processing             │
│ - Check NSG rules                      │
│ - Priority-based matching              │
│ - Allow/Deny decision                  │
└────────────────────────────────────────┘
         ↓ (If allowed)
┌────────────────────────────────────────┐
│ VFP Layer 4: Routing                    │
│ - Lookup routing table                 │
│ - Apply UDRs                           │
│ - Determine next hop                   │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 5: NAT/Load Balancer          │
│ - DNAT for inbound (if LB/PIP)         │
│ - SNAT for outbound                    │
│ - Connection tracking                  │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 6: Encapsulation              │
│ - Add VXLAN header                     │
│ - Set VNI                              │
│ - Destination host IP                  │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Layer 7: Forwarding                 │
│ - Send to physical NIC                 │
│ - Or deliver to local VM               │
└────────────────────────────────────────┘

VFP Performance:
- Hardware offload (SR-IOV)
- Flow caching
- Stateful connection tracking
- Line-rate processing (40/100 Gbps)
```

### Encapsulation Protocols

**VXLAN (Virtual Extensible LAN)**:

```
VXLAN Packet Structure:

┌────────────────────────────────────────┐
│ Outer Ethernet Header                   │
│ - Src MAC: Host1 physical NIC          │
│ - Dst MAC: Host2 physical NIC          │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ Outer IP Header                         │
│ - Src IP: Host1 physical IP            │
│ - Dst IP: Host2 physical IP            │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ Outer UDP Header                        │
│ - Src Port: Random ephemeral           │
│ - Dst Port: 4789 (VXLAN)               │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ VXLAN Header                            │
│ - VNI: 5001 (Virtual Network ID)       │
│ - Flags                                │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ Inner Ethernet Header                   │
│ - Src MAC: VM1 virtual NIC             │
│ - Dst MAC: VM2 virtual NIC             │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ Inner IP Header                         │
│ - Src IP: 10.0.1.4 (VM1)               │
│ - Dst IP: 10.0.2.5 (VM2)               │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ Inner TCP/UDP Header + Data             │
│ - Original application data            │
└────────────────────────────────────────┘

VXLAN Benefits:
✓ 16 million VNI values (24-bit)
✓ Overlay L2 over L3 network
✓ No physical VLAN limitation
✓ Scalable multi-tenant isolation
✓ Works over any IP network
✓ ECMP load balancing friendly

Example Communication:
VM1 (Host-A, VNI 5001, 10.0.1.4)
  → VM2 (Host-B, VNI 5001, 10.0.2.5)

Encapsulation at Host-A:
1. VM1 sends: 10.0.1.4 → 10.0.2.5
2. VFP adds VXLAN header (VNI 5001)
3. VFP adds outer IP: Host-A → Host-B
4. Physical NIC sends packet

Decapsulation at Host-B:
1. Physical NIC receives packet
2. VFP checks VNI: 5001 ✓
3. VFP removes VXLAN header
4. VFP delivers to VM2: 10.0.1.4 → 10.0.2.5

VM sees only inner packet (overlay network)
Physical network sees only outer packet (underlay)
```

**MAC Address Resolution**:

```
ARP in Azure VNets:

Traditional Network:
VM1 → Broadcast ARP: Who has 10.0.1.5?
All VMs receive broadcast
VM2 responds: I have 10.0.1.5 (MAC: aa:bb:cc...)

Azure VNet (No Broadcast):
VM1 → Gateway: Who has 10.0.1.5?
         ↓
┌────────────────────────────────────────┐
│ Azure ARP Proxy Service                 │
│ - Knows all VM MAC addresses           │
│ - Responds on behalf of VMs            │
│ - No network flooding                  │
└────────────────────────────────────────┘
         ↓
VM1 ← Unicast response: 10.0.1.5 = MAC:...

Benefits:
✓ No broadcast storms
✓ Faster ARP resolution
✓ Works across subnets
✓ Centralized control
✓ No ARP cache poisoning

ARP Table (VM Perspective):
10.0.1.1 (Gateway): 12:34:56:78:9a:bc
10.0.1.5 (VM2): aa:bb:cc:dd:ee:ff
10.0.2.5 (VM3): ff:ee:dd:cc:bb:aa

All MACs are virtual (assigned by Azure)
Physical host MACs are different
Overlay network abstraction
```

### Network Peering Implementation

**Peering Under the Hood**:

```
VNet Peering Mechanism:

Control Plane (When you create peering):
1. Azure validates both VNets exist
2. Checks address space overlap (must not overlap)
3. Creates peering state (Initiated)
4. Waits for reciprocal peering
5. Updates SDN controller
6. Injects routes in both VNets
7. Updates VFP policies on all hosts
8. Peering state: Connected

Data Plane (After peering active):
┌────────────────────────────────────────┐
│ VNet1 Host - VFP                        │
│ Routing Table:                          │
│ 10.0.0.0/16 → Local                    │
│ 10.1.0.0/16 → VNetPeering (VNet2)      │
└────────────────────────────────────────┘

VM1 (VNet1, 10.0.1.4) → VM2 (VNet2, 10.1.2.5)

Step 1: VM1 sends packet
Src: 10.0.1.4
Dst: 10.1.2.5

Step 2: VFP on Host-A
- Checks route: 10.1.0.0/16 → VNetPeering
- Gets VNI for VNet2
- Determines Host-B location (where VM2 is)
- Encapsulates packet:
  * Outer IP: Host-A → Host-B
  * VNI: VNet2's VNI
  * Inner: 10.0.1.4 → 10.1.2.5

Step 3: Physical network
- Packet routed Host-A → Host-B
- Uses Azure backbone (global peering)
- Encrypted in transit

Step 4: VFP on Host-B
- Receives packet
- Checks VNI: VNet2 ✓
- Checks NSG rules ✓
- Decapsulates
- Delivers to VM2

Step 5: VM2 receives packet
Src: 10.0.1.4 (VNet1)
Dst: 10.1.2.5 (local)

Return path: Same process, reverse direction

Peering Advantages:
✓ No gateway needed (unlike VPN)
✓ Lowest latency (direct Azure backbone)
✓ High bandwidth (no gateway bottleneck)
✓ Private connectivity (never internet)
✓ Transitive via NVA (with UDRs)
```

### Service Endpoint Implementation

**Service Endpoint Mechanics**:

```
Without Service Endpoint:
VM (10.0.1.4) → Internet → Storage (public endpoint)

1. VM sends: Dst 52.xxx.xxx.xxx (storage public IP)
2. VFP routes via default route (0.0.0.0/0 → Internet)
3. SNAT applied: Src becomes NAT IP
4. Packet exits via Azure edge
5. Routes through internet
6. Reaches storage public endpoint

With Service Endpoint:
VM (10.0.1.4) → Azure Backbone → Storage (private path)

Configuration:
Subnet: serviceEndpoints: ["Microsoft.Storage"]

Effect on Routing:
┌────────────────────────────────────────┐
│ VFP Routing Table                       │
│ Storage IPs: 52.239.0.0/16 →           │
│   Next Hop: ServiceEndpoint            │
│   (Azure internal backbone)            │
└────────────────────────────────────────┘

Traffic Flow:
1. VM sends: Dst 52.239.x.x (storage)
2. VFP matches service endpoint route
3. No SNAT (source IP preserved: 10.0.1.4)
4. Packet routed via Azure backbone
   - Never leaves Microsoft network
   - Optimized path
   - Lower latency
5. Storage receives from: 10.0.1.4
6. Storage firewall checks: Allow from VNet? ✓

Storage Side Configuration:
┌────────────────────────────────────────┐
│ Storage Account Firewall                │
│ Allow access from:                     │
│ - VNet: MyVNet                         │
│ - Subnet: WebSubnet (10.0.1.0/24)      │
│                                        │
│ Request from 10.0.1.4? → ALLOW ✓       │
│ Request from 1.2.3.4? → DENY ✗         │
└────────────────────────────────────────┘

Benefits:
✓ Traffic never leaves Azure
✓ Source IP preserved (VNet IP)
✓ Service-side firewall (allow specific VNets)
✓ Optimal routing
✓ No additional cost

Implementation Details:
- BGP route injection to VFP
- Service-specific prefix ranges
- Regional service endpoint
- Per-subnet configuration
- Works with service firewall

Limitations:
- Same region only (typically)
- Service must support endpoints
- Subnet-level configuration
- Cannot block specific storage accounts (use policy)
```

### Private Link Architecture

**Private Link Implementation**:

```
Private Endpoint Creation:

Step 1: You create private endpoint
Resource: Storage Account (mystore)
Subnet: PrivateEndpointSubnet (10.0.5.0/24)

Step 2: Azure provisions infrastructure
┌────────────────────────────────────────┐
│ Private Endpoint                        │
│ - NIC created in subnet                │
│ - Private IP allocated: 10.0.5.4       │
│ - Connected to storage via Private Link│
└────────────────────────────────────────┘

Step 3: Azure updates DNS
┌────────────────────────────────────────┐
│ Private DNS Zone (optional)             │
│ privatelink.blob.core.windows.net       │
│                                        │
│ A Record:                              │
│ mystore → 10.0.5.4                     │
└────────────────────────────────────────┘

Traffic Flow with Private Endpoint:

VM (10.0.1.4) → Storage (via private endpoint)

1. Application: Connect to mystore.blob.core.windows.net
2. DNS query: mystore.blob.core.windows.net
   - CNAME: mystore.privatelink.blob.core.windows.net
   - A record: 10.0.5.4 (from Private DNS Zone)
3. VM sends: Dst 10.0.5.4
4. VFP routing: 10.0.5.0/24 → Local subnet
5. Packet delivered to Private Endpoint NIC
6. Private Link tunnel to storage backend
7. Storage processes request
8. Response returns via same path

Physical Implementation:
┌────────────────────────────────────────┐
│ Your VNet (10.0.0.0/16)                 │
│ VM: 10.0.1.4                           │
│ Private Endpoint NIC: 10.0.5.4         │
└────────────────────────────────────────┘
         ↓ (Private Link Service)
┌────────────────────────────────────────┐
│ Microsoft Network                       │
│ Private Link Platform                  │
│ - Secure tunnel                        │
│ - Azure backbone                       │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Storage Account                         │
│ Private connection                     │
│ No public internet path                │
└────────────────────────────────────────┘

Security Benefits:
✓ Storage accessible via private IP only
✓ No public endpoint needed
✓ No internet exposure
✓ NSG rules apply (10.0.5.4)
✓ Data exfiltration prevention
✓ Compliance requirements met

Private Link vs Service Endpoint:
┌──────────────────┬───────────────┬──────────────┐
│ Feature          │ Service EP    │ Private Link │
├──────────────────┼───────────────┼──────────────┤
│ Service IP type  │ Public        │ Private      │
│ DNS resolution   │ Public IP     │ Private IP   │
│ Public disable   │ No            │ Yes          │
│ Cross-region     │ Same region   │ Any region   │
│ Cross-VNet       │ Via peering   │ Direct       │
│ On-premises      │ Via VPN+route │ Via VPN      │
│ Encryption       │ TLS           │ TLS + tunnel │
└──────────────────┴───────────────┴──────────────┘
```

-----

## Layer 5: Mapping to TCP/IP Model

Let’s map Virtual Network operations to the TCP/IP model:

### TCP/IP Layer Overview

```
┌─────────────────────────────────────────────────┐
│ Layer 4: Application Layer                      │
│ - HTTP, HTTPS, SSH, RDP, DNS                    │
│ VNet: Transparent - applications unaware        │
│     ✓ Standard protocols work                   │
│     ✓ No application changes needed             │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 3: Transport Layer                        │
│ - TCP, UDP                                      │
│ VNet: ✓ FULL SUPPORT                           │
│     ✓ TCP connection tracking (NSG stateful)    │
│     ✓ UDP flows                                 │
│     ✓ Port-based routing (load balancer)        │
│     ✓ Connection multiplexing                   │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 2: Internet Layer                         │
│ - IP addressing, routing, ICMP                  │
│ VNet: ✓✓✓ PRIMARY OPERATION LAYER              │
│     ✓ IP address allocation (DHCP)              │
│     ✓ Routing (system + UDR)                    │
│     ✓ Network isolation (VNI)                   │
│     ✓ VNet peering (inter-VNet routing)         │
│     ✓ NAT (outbound internet)                   │
│     ✓ ICMP (ping between VMs)                   │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 1: Network Access Layer                   │
│ - MAC addresses, ARP, Ethernet                  │
│ VNet: ✓ VIRTUALIZED IMPLEMENTATION              │
│     ✓ Virtual MAC addresses                     │
│     ✓ ARP proxy (no broadcast)                  │
│     ✓ VXLAN encapsulation (overlay)             │
│     ✓ Virtual switching (VFP)                   │
└─────────────────────────────────────────────────┘
```

### Layer 1: Network Access (Link Layer)

**Virtual MAC Addresses**:

```
MAC Address in Azure VNets:

Traditional Network:
Physical NIC has burned-in MAC address
Example: 00:15:5D:01:02:03 (real hardware)

Azure Virtual Network:
Virtual NIC gets virtual MAC address
Example: 00:0D:3A:F7:B2:C1 (Azure-assigned)

MAC Address Format:
00:0D:3A:xx:xx:xx → Azure VM NICs
00:15:5D:xx:xx:xx → Hyper-V (underlying host)

VM Perspective:
┌────────────────────────────────────────┐
│ VM Network Interface                    │
│ IP: 10.0.1.4                           │
│ MAC: 00:0D:3A:F7:B2:C1                 │
│ Gateway: 10.0.1.1                      │
│ Gateway MAC: 12:34:56:78:9A:BC         │
└────────────────────────────────────────┘

All MACs are virtual:
- VM's MAC: Virtual
- Gateway MAC: Virtual
- Other VMs' MACs: Virtual

Physical Reality (What Actually Happens):
┌────────────────────────────────────────┐
│ Physical Host NIC                       │
│ MAC: 00:15:5D:A1:B2:C3                 │
│ Receives VXLAN-encapsulated packets    │
│ VFP maps: Virtual MAC ↔ VNI + VM       │
└────────────────────────────────────────┘

MAC Address Resolution (ARP):
VM1 wants to send to: 10.0.1.5 (VM2)

Traditional ARP:
VM1 → Broadcast: "Who has 10.0.1.5?"
All devices receive broadcast
VM2 responds: "I have 10.0.1.5, MAC: aa:bb:cc"

Azure VNet ARP (No Broadcast):
VM1 → Azure ARP Proxy: "Who has 10.0.1.5?"
Azure → VM1: "10.0.1.5 has MAC: 00:0D:3A:12:34:56"
No broadcast, unicast response

Benefits:
✓ No broadcast storms
✓ Faster resolution
✓ Controlled by Azure
✓ Works across subnets
✓ Security (no ARP spoofing)
```

**Virtual Switching (VFP)**:

```
Traditional Ethernet Switch:
- Physical ports
- MAC address table
- Flood unknown destinations
- Spanning tree protocol

Azure Virtual Switch (VFP):
- Software-defined
- Virtual ports (per VM NIC)
- No flooding (centralized knowledge)
- No spanning tree (loop-free by design)

VFP Switch Architecture:
┌────────────────────────────────────────┐
│ Hyper-V Host                            │
│                                        │
│ ┌────────────────────────────────────┐│
│ │ Virtual Switch (VFP)               ││
│ │                                    ││
│ │ Port 1 ← VM1 (00:0D:3A:11:11:11)  ││
│ │ Port 2 ← VM2 (00:0D:3A:22:22:22)  ││
│ │ Port 3 ← VM3 (00:0D:3A:33:33:33)  ││
│ │ Port 4 ← Physical NIC (uplink)     ││
│ │                                    ││
│ │ MAC Table:                         ││
│ │ 00:0D:3A:11:11:11 → Port 1        ││
│ │ 00:0D:3A:22:22:22 → Port 2        ││
│ │ 00:0D:3A:33:33:33 → Port 3        ││
│ │ * (external) → Port 4 (encap)     ││
│ └────────────────────────────────────┘│
└────────────────────────────────────────┘

Packet Forwarding:
VM1 → VM2 (same host):
1. VM1 sends to MAC: 00:0D:3A:22:22:22
2. VFP switch lookup: Port 2
3. Direct delivery to VM2
4. No physical network involved

VM1 → VM3 (different host):
1. VM1 sends to MAC: 00:0D:3A:33:33:33
2. VFP switch lookup: Not local
3. Encapsulate (VXLAN)
4. Send via Port 4 (physical NIC)
5. Destination host decapsulates
6. Deliver to VM3
```

**VXLAN Encapsulation Details**:

```
Layer 2 Frame Encapsulation:

Original Frame (VM1 → VM2):
┌──────────────────────────────────┐
│ Ethernet Header                   │
│ Dst MAC: 00:0D:3A:22:22:22 (VM2) │
│ Src MAC: 00:0D:3A:11:11:11 (VM1) │
│ EtherType: 0x0800 (IPv4)         │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ IP Header                         │
│ Src IP: 10.0.1.4 (VM1)           │
│ Dst IP: 10.0.2.5 (VM2)           │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ TCP/UDP Header + Data             │
└──────────────────────────────────┘

After VXLAN Encapsulation:
┌──────────────────────────────────┐
│ Outer Ethernet Header             │
│ Dst MAC: Physical switch MAC      │
│ Src MAC: Host-A physical NIC     │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ Outer IP Header                   │
│ Src IP: Host-A IP (172.16.1.10)  │
│ Dst IP: Host-B IP (172.16.1.20)  │
│ Protocol: UDP (17)                │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ Outer UDP Header                  │
│ Src Port: 49152 (random)         │
│ Dst Port: 4789 (VXLAN)           │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ VXLAN Header (8 bytes)            │
│ Flags: 0x08 (VNI valid)          │
│ VNI: 5001 (24-bit VNet ID)       │
│ Reserved: 0                       │
└──────────────────────────────────┘
┌──────────────────────────────────┐
│ Inner Ethernet Header             │
│ (Original VM frame)               │
└──────────────────────────────────┘
... (rest of original packet)

Overhead:
- Outer Ethernet: 14 bytes
- Outer IP: 20 bytes
- Outer UDP: 8 bytes
- VXLAN: 8 bytes
Total: 50 bytes overhead

MTU Implications:
Standard VM MTU: 1500 bytes
With VXLAN overhead: 1550 bytes on physical
Azure handles: Automatic (VMs don't see it)

Physical Network Requirements:
- Supports 1550+ byte frames (Azure guarantees)
- Or IP fragmentation (avoided)
- ECMP-friendly (UDP src port varies per flow)
```

### Layer 2: Internet (Network Layer)

**IP Routing in VNets**:

```
VM Routing Table:

VM in 10.0.1.0/24 subnet:

┌────────────────────────────────────────┐
│ Destination      Gateway       Interface│
│ 10.0.1.0/24      On-link       eth0     │← Same subnet
│ 10.0.0.0/16      10.0.1.1      eth0     │← Other VNet subnets
│ 10.1.0.0/16      10.0.1.1      eth0     │← Peered VNet
│ 0.0.0.0/0        10.0.1.1      eth0     │← Default (internet)
│ 168.63.129.16/32 10.0.1.1      eth0     │← Azure DNS
└────────────────────────────────────────┘

Routing Decision Process:

Packet to 10.0.1.5:
- Match: 10.0.1.0/24 (on-link)
- Action: Send directly (same subnet)
- No gateway involved

Packet to 10.0.2.5:
- Match: 10.0.0.0/16 via 10.0.1.1
- Action: Send to gateway (10.0.1.1)
- Gateway: Azure SDN (invisible)

Packet to 10.1.2.5:
- Match: 10.1.0.0/16 via 10.0.1.1
- Action: Send to gateway
- Azure routes via VNet peering

Packet to 8.8.8.8:
- Match: 0.0.0.0/0 via 10.0.1.1
- Action: Send to gateway
- Azure routes to internet (SNAT applied)

Special IP: 168.63.129.16 (Azure DNS):
- Match: 168.63.129.16/32
- Action: Send to gateway
- Azure internal DNS service
- Also used for health probes

Azure Route vs UDR Priority:
Most specific prefix wins:
/32 > /24 > /16 > /0

Example:
System Route: 0.0.0.0/0 → Internet
UDR: 0.0.0.0/0 → VirtualAppliance (10.0.100.4)
Result: UDR wins (same specificity, UDR priority)

System Route: 10.0.0.0/16 → VnetLocal
UDR: 0.0.0.0/0 → VirtualAppliance
Packet to 10.0.2.5:
Result: System route wins (more specific)
```

**ICMP and Connectivity Testing**:

```
Ping (ICMP) in Azure VNets:

Same VNet ICMP:
VM1 (10.0.1.4) → VM2 (10.0.2.5)

┌────────────────────────────────────────┐
│ VM1 sends ICMP Echo Request            │
│ Dst: 10.0.2.5                          │
│ Type: 8 (Echo Request)                 │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Azure VFP                               │
│ - Checks NSG (allows ICMP?)            │
│ - Routes to destination subnet         │
│ - Forwards to VM2 host                 │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VM2 receives ICMP Echo Request          │
│ Sends ICMP Echo Reply                  │
│ Dst: 10.0.1.4                          │
│ Type: 0 (Echo Reply)                   │
└────────────────────────────────────────┘
         ↓
VM1 receives reply: Round-trip time: <1ms

NSG Consideration:
Default NSG: Allows ICMP (VirtualNetwork tag)
Custom NSG: May block ICMP

If ICMP blocked:
$ ping 10.0.2.5
Request timeout
(NSG drops packet silently)

ICMP Across Peered VNets:
Works if:
✓ Peering connected
✓ NSGs allow ICMP
✓ No UDR blocking

ICMP to Internet:
$ ping 8.8.8.8

Outbound ICMP:
✓ Usually allowed (NSG default)
✓ SNAT applied
✓ Reaches internet

Inbound ICMP (ping Azure VM from internet):
❌ Blocked by default (Azure DDoS protection)
❌ Cannot ping public IP of VM
✓ Can RDP/SSH (if NSG allows)

Testing Connectivity:
Instead of ping, use:
- Test-NetConnection (PowerShell)
- telnet/nc for TCP ports
- Azure Network Watcher (Connectivity Check)
```

**NAT Implementation**:

```
Outbound NAT (SNAT):

VM without Public IP:
VM (10.0.1.4) → Internet (8.8.8.8:53)

Traditional SNAT:
┌────────────────────────────────────────┐
│ Azure Load Balancer (implicit)          │
│ SNAT Pool: 40.50.60.70:1024-65535      │
│ Maps: 10.0.1.4:52000 → 40.50.60.70:3000│
└────────────────────────────────────────┘

Port Allocation:
- 64K ports per public IP
- Shared among all VMs in subnet
- Port exhaustion possible under load

With NAT Gateway:
┌────────────────────────────────────────┐
│ NAT Gateway                             │
│ Public IP: 40.50.60.80                 │
│ Maps: 10.0.1.4:52000 → 40.50.60.80:3000│
│ 64K ports per IP                       │
│ Scale to 16 IPs (1M ports)             │
└────────────────────────────────────────┘

SNAT Packet Transformation:
Original Packet (VM → Internet):
Src: 10.0.1.4:52000
Dst: 8.8.8.8:53

After SNAT:
Src: 40.50.60.80:3000 ← NAT IP:Port
Dst: 8.8.8.8:53

Return Packet (Internet → NAT):
Src: 8.8.8.8:53
Dst: 40.50.60.80:3000

After De-NAT:
Src: 8.8.8.8:53
Dst: 10.0.1.4:52000 ← Original VM IP:Port

Connection Tracking Table:
┌──────────────────────────────────────┐
│ 10.0.1.4:52000 ↔ 40.50.60.80:3000   │
│ → 8.8.8.8:53                         │
│ State: ESTABLISHED                   │
│ Timeout: 4 minutes (TCP idle)        │
└──────────────────────────────────────┘

Inbound NAT (DNAT):
Public IP on VM NIC:
External: 40.50.60.90:80
Internal: 10.0.1.4:80

Inbound packet:
Dst: 40.50.60.90:80

Azure DNAT:
Dst: 10.0.1.4:80 ← Translated to private IP

Delivered to VM as:
Dst: 10.0.1.4:80

VM sees only private IP traffic
Public IP transparent to VM
```

### Layer 3: Transport (Transport Layer)

**TCP Connection Handling**:

```
TCP 3-Way Handshake Across VNets:

VM1 (VNet1, 10.0.1.4) → VM2 (VNet2, 10.1.2.5)

Step 1: SYN
VM1 → VM2:
TCP SYN (Seq=1000)
Src: 10.0.1.4:52000
Dst: 10.1.2.5:80

Azure Processing:
1. VM1's NSG: Check outbound rules → Allow
2. VNet1 routing: 10.1.0.0/16 via peering
3. VFP encapsulates (VXLAN, VNI for VNet2)
4. Physical network routes to VNet2 host
5. VFP decapsulates
6. VNet2 NSG: Check inbound rules → Allow
7. Delivers to VM2

Step 2: SYN-ACK
VM2 → VM1:
TCP SYN-ACK (Seq=5000, Ack=1001)
Src: 10.1.2.5:80
Dst: 10.0.1.4:52000

Azure Processing:
1. VM2's NSG: Check outbound (automatic allow - stateful)
2. VNet2 routing: 10.0.0.0/16 via peering
3. Encapsulate and route back to VNet1
4. Deliver to VM1

Step 3: ACK
VM1 → VM2:
TCP ACK (Ack=5001)

Connection ESTABLISHED

NSG Stateful Behavior:
Outbound rule allows: VM1 → VM2:80
Automatic return: VM2:80 → VM1 (stateful)
No explicit inbound rule needed on VM1

Connection Tracking:
┌────────────────────────────────────────┐
│ Connection Table (VM1's VFP)            │
│ 10.0.1.4:52000 ↔ 10.1.2.5:80           │
│ State: ESTABLISHED                     │
│ Direction: Outbound                    │
│ Return traffic: AUTO-ALLOWED           │
└────────────────────────────────────────┘

TCP Connection Timeouts:
Established: No timeout (until FIN/RST)
Idle timeout: Depends on application
NSG: Stateful entry cleanup after connection close
```

**UDP Flows**:

```
UDP Communication:
VM1 (10.0.1.4) → DNS (168.63.129.16:53)

UDP Characteristics:
- Connectionless
- No handshake
- Stateless protocol

Azure NSG for UDP:
- Still stateful tracking!
- Tracks UDP "flows" by 5-tuple:
  * Src IP
  * Src Port
  * Dst IP
  * Dst Port
  * Protocol

UDP Flow Example:
Query:
Src: 10.0.1.4:52000
Dst: 168.63.129.16:53
UDP DNS query

NSG creates flow entry:
┌────────────────────────────────────────┐
│ Flow: 10.0.1.4:52000 → 168.63.129.16:53│
│ Protocol: UDP                          │
│ Direction: Outbound                    │
│ Timeout: 240 seconds                   │
└────────────────────────────────────────┘

Response:
Src: 168.63.129.16:53
Dst: 10.0.1.4:52000
UDP DNS response

NSG matches flow entry → ALLOW
(Even if no explicit inbound rule)

Flow Timeout:
UDP idle timeout: 4 minutes (240 seconds)
If no packets for 4 min → Flow entry removed
Next packet: Evaluated against rules again

UDP Port Behavior:
Source port: Ephemeral (random high port)
- Changes per flow
- Helps with load balancing (ECMP)
- Secure (harder to spoof)
```

### Layer 4: Application Layer

**DNS Resolution**:

```
DNS in Azure VNets:

Azure-Provided DNS (Default):
VM configured with:
DNS Server: 168.63.129.16

DNS Resolution Process:
1. VM queries: www.microsoft.com
   → Sent to 168.63.129.16:53
   
2. Azure DNS service:
   → Forwards to internet DNS
   → Returns: A record (IP address)
   
3. VM receives IP address

Azure Internal DNS:
Automatic hostname registration:
VM name: webserver01
VNet: MyVNet

DNS name: webserver01.internal.cloudapp.net
Resolves to: 10.0.1.4 (private IP)

Query from another VM:
$ nslookup webserver01
Answer: 10.0.1.4

Custom DNS Servers:
VNet → DNS servers: 10.0.10.4, 10.0.10.5

VM uses custom DNS servers instead of Azure
Use case: Active Directory, custom DNS

Private DNS Zones:
┌────────────────────────────────────────┐
│ Private DNS Zone                        │
│ contoso.local                          │
│                                        │
│ Records:                               │
│ web1  A  10.0.1.4                      │
│ web2  A  10.0.1.5                      │
│ db1   A  10.0.2.4                      │
└────────────────────────────────────────┘
         ↓ (Linked to VNet)
┌────────────────────────────────────────┐
│ Virtual Network                         │
│ VMs automatically resolve               │
│ *.contoso.local names                   │
└────────────────────────────────────────┘

Private DNS for Azure Resources:
Private Endpoint creates:
- privatelink.blob.core.windows.net zone
- A record: mystore → 10.0.5.4

Resolution:
mystore.blob.core.windows.net
→ CNAME: mystore.privatelink.blob.core.windows.net
→ A: 10.0.5.4 (private IP in your VNet)

Split-Brain DNS:
Inside VNet: mystore → 10.0.5.4 (private)
From Internet: mystore → 20.x.x.x (public)
```

-----

## Layer 6: Traffic Flow Patterns

### Flow 1: VM to VM (Same Subnet)

```
Scenario: Web tier communication
VM1 (10.0.1.4) → VM2 (10.0.1.5)
Same subnet: 10.0.1.0/24
Same host: NO (different physical servers)

┌────────────────────────────────────────┐
│ VM1 (Host-A)                            │
│ IP: 10.0.1.4                           │
│ MAC: 00:0D:3A:11:11:11                 │
└────────────────────────────────────────┘

Step 1: VM1 sends packet
Dst IP: 10.0.1.5
Check routing table: Same subnet (10.0.1.0/24)
Action: Send directly (no gateway)
Need: MAC address of 10.0.1.5

Step 2: ARP resolution (if not cached)
VM1 → Azure ARP proxy: "Who has 10.0.1.5?"
Azure → VM1: "10.0.1.5 = MAC 00:0D:3A:22:22:22"
VM1 caches: 10.0.1.5 → 00:0D:3A:22:22:22

Step 3: Build Ethernet frame
┌─────────────────────────────────┐
│ Ethernet Header                  │
│ Dst MAC: 00:0D:3A:22:22:22 (VM2)│
│ Src MAC: 00:0D:3A:11:11:11 (VM1)│
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ IP Header                        │
│ Src: 10.0.1.4                   │
│ Dst: 10.0.1.5                   │
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ TCP/UDP + Data                   │
└─────────────────────────────────┘

Step 4: VFP on Host-A processes
- NSG check (outbound from VM1): Allow?
  * Default: Allow VirtualNetwork → VirtualNetwork ✓
- Routing: Destination is local subnet
- Lookup VM2 location: Host-B
- Encapsulate in VXLAN:
  * Outer IP: Host-A → Host-B
  * VNI: 5001 (VNet ID)
  * Inner: Original frame

Step 5: Physical network
Packet routed: Host-A → Host-B
- Top-of-Rack switches
- Spine switches
- Destination Host-B

Step 6: VFP on Host-B receives
- Decapsulates VXLAN
- Checks VNI: 5001 ✓
- NSG check (inbound to VM2): Allow?
  * Default: Allow VirtualNetwork → VirtualNetwork ✓
- Deliver to VM2 virtual NIC

Step 7: VM2 receives packet
Dst IP: 10.0.1.5 ✓ (me!)
Process packet

Return Path:
Same process, reversed direction
NSG stateful: Return traffic auto-allowed

Latency: < 1ms (intra-region)
```

### Flow 2: VM to VM (Different Subnet, Same VNet)

```
Scenario: Web tier → App tier
VM1 (10.0.1.4, WebSubnet) → VM3 (10.0.2.5, AppSubnet)
Different subnets, same VNet: 10.0.0.0/16

┌────────────────────────────────────────┐
│ VNet: 10.0.0.0/16                       │
│ ├─ WebSubnet: 10.0.1.0/24              │
│ │  └─ VM1: 10.0.1.4                    │
│ └─ AppSubnet: 10.0.2.0/24              │
│    └─ VM3: 10.0.2.5                    │
└────────────────────────────────────────┘

Step 1: VM1 sends packet
Dst IP: 10.0.2.5
Check routing table:
- 10.0.1.0/24: On-link (not this)
- 10.0.0.0/16: Via gateway 10.0.1.1 ✓

Action: Send to default gateway
Gateway MAC: 12:34:56:78:9A:BC (Azure virtual gateway)

Step 2: Build frame
┌─────────────────────────────────┐
│ Ethernet Header                  │
│ Dst MAC: 12:34:56:78:9A:BC (GW) │
│ Src MAC: 00:0D:3A:11:11:11 (VM1)│
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ IP Header                        │
│ Src: 10.0.1.4                   │
│ Dst: 10.0.2.5                   │
└─────────────────────────────────┘

Step 3: VFP on Host-A (VM1's host)
- NSG check (VM1 outbound):
  * WebSubnet NSG: Allow outbound to VirtualNetwork ✓
- Routing lookup: 10.0.2.5
  * System route: 10.0.0.0/16 → VnetLocal
  * Action: Forward to AppSubnet
- Determine VM3 location: Host-C
- Encapsulate (VXLAN, VNI 5001)
- Send to Host-C

Step 4: Physical network routing
Host-A → Host-C

Step 5: VFP on Host-C (VM3's host)
- Decapsulate
- NSG check (VM3 inbound):
  * AppSubnet NSG: Check rules
  * Example rule: Allow from 10.0.1.0/24 to 8080 ✓
  * Or: Default VirtualNetwork rule ✓
- Deliver to VM3

Step 6: VM3 receives
IP: 10.0.2.5 ✓
Process packet

Key Points:
✓ VMs in same VNet can communicate by default
✓ NSG can restrict (layer of security)
✓ No gateway VM involved (software routing)
✓ Latency: < 1ms typically

NSG Example (AppSubnet):
Allow: WebSubnet (10.0.1.0/24) → AppSubnet:8080
Deny: All other traffic

Effect: Only web tier can reach app tier
```

### Flow 3: VM to Internet

```
Scenario: VM accessing public website
VM1 (10.0.1.4, no public IP) → 8.8.8.8 (Google DNS)

┌────────────────────────────────────────┐
│ VM1: 10.0.1.4                           │
│ Subnet: 10.0.1.0/24                    │
│ No public IP                           │
│ Outbound: Via NAT Gateway              │
└────────────────────────────────────────┘

Step 1: VM1 sends packet
Dst: 8.8.8.8:53 (DNS query)
Src: 10.0.1.4:52000

Check routing table:
- No specific route for 8.8.8.8
- Match: 0.0.0.0/0 → Gateway 10.0.1.1 ✓
- Default route to internet

Step 2: VFP on VM1's host
- NSG check (outbound):
  * Subnet NSG: Allow outbound to Internet ✓
  * (Default NSG allows outbound internet)
- Routing: 0.0.0.0/0 → Internet
- Check for NAT Gateway on subnet:
  * NAT Gateway exists: 40.50.60.80 ✓
  * Route to NAT Gateway

Step 3: NAT Gateway processing
Input:
Src: 10.0.1.4:52000
Dst: 8.8.8.8:53

SNAT transformation:
Create mapping:
10.0.1.4:52000 ↔ 40.50.60.80:3000 → 8.8.8.8:53

Output:
Src: 40.50.60.80:3000 ← NAT public IP
Dst: 8.8.8.8:53

Step 4: Azure edge
- Route to internet
- Through Azure edge routers
- BGP peering with ISPs

Step 5: Internet
Packet reaches 8.8.8.8
Google DNS sees source: 40.50.60.80:3000

Step 6: Response from internet
Src: 8.8.8.8:53
Dst: 40.50.60.80:3000

Step 7: Azure edge receives
- Routes to NAT Gateway
- NAT Gateway lookup: 40.50.60.80:3000
  * Mapping: → 10.0.1.4:52000 ✓

De-NAT transformation:
Src: 8.8.8.8:53
Dst: 10.0.1.4:52000 ← Original VM IP

Step 8: Route to VM1
VFP delivers packet to VM1

Step 9: VM1 receives response
Application receives DNS response

Without NAT Gateway (Default Azure):
- Uses Azure Load Balancer backend IPs
- Shared SNAT ports
- Can cause port exhaustion
- Non-deterministic public IP

With NAT Gateway:
✓ Dedicated public IP(s)
✓ 64K ports per IP per destination
✓ Scales to 16 IPs (1M ports)
✓ Predictable outbound IP
✓ Better for whitelisting

Outbound Flow Limits:
NAT Gateway: 50 Gbps, 1M concurrent flows
Standard LB SNAT: Varies by VM size
```

### Flow 4: Internet to VM (Inbound with Public IP)

```
Scenario: User accessing web server
Internet (1.2.3.4) → VM1 (Public IP: 40.50.60.90, Private: 10.0.1.4:80)

┌────────────────────────────────────────┐
│ VM1 (Web Server)                        │
│ Private IP: 10.0.1.4                   │
│ Public IP: 40.50.60.90 (on NIC)        │
│ NSG: Allow 80/443 from Internet        │
└────────────────────────────────────────┘

Step 1: Internet user
Browser: http://40.50.60.90/
TCP connection: 1.2.3.4:52000 → 40.50.60.90:80

Step 2: Azure edge network
Packet arrives: Dst 40.50.60.90:80
Lookup: Public IP belongs to VM1 in VNet1
Route to: VNet1 region

Step 3: DDoS Protection (Azure platform)
- Check for DDoS attack patterns
- Rate limiting
- SYN flood protection
- If malicious: Drop
- If legitimate: Allow ✓

Step 4: NSG processing (before VM)
Subnet NSG (if attached to WebSubnet):
┌─────────────────────────────────────┐
│ Inbound Rules:                      │
│ Priority 100: Allow 80 from Internet│
│ Source: 1.2.3.4 (Internet)          │
│ Destination: 40.50.60.90 (VNet)     │
│ Port: 80                            │
│ → MATCH: ALLOW ✓                    │
└─────────────────────────────────────┘

NIC NSG (if attached to VM1-NIC):
(Same evaluation)
Result: ALLOW ✓

Step 5: DNAT (Destination NAT)
Azure translates:
Original packet:
Dst: 40.50.60.90:80 (public IP)

After DNAT:
Dst: 10.0.1.4:80 (private IP)

Src remains: 1.2.3.4:52000

Step 6: VFP routing
- Route to VM1's host
- Encapsulate (VXLAN)
- Send to host

Step 7: VM1 receives
Dst IP: 10.0.1.4:80 ✓
Src IP: 1.2.3.4:52000

VM sees:
- Source: Internet client IP (preserved!)
- Destination: Its private IP (10.0.1.4)
- VM is unaware of public IP

Step 8: VM processes request
Web server handles HTTP request

Step 9: VM sends response
Src: 10.0.1.4:80
Dst: 1.2.3.4:52000

Step 10: VFP reverse DNAT
Azure translates:
Src: 40.50.60.90:80 ← Public IP
Dst: 1.2.3.4:52000

Step 11: NSG (outbound)
Check: Stateful connection exists ✓
Auto-allow return traffic

Step 12: Route to internet
Via Azure edge → Internet

Step 13: Client receives response
From: 40.50.60.90:80

Key Points:
✓ Client sees public IP
✓ VM sees private IP
✓ Source IP preserved (no SNAT inbound)
✓ NSG must allow inbound
✓ DDoS protection automatic
```

### Flow 5: VNet to VNet (via Peering)

```
Scenario: Cross-VNet communication
VNet1 VM (10.0.1.4) → VNet2 VM (10.1.2.5)
Peering: VNet1 ←→ VNet2 (Connected)

┌────────────────────────────────────────┐
│ VNet1: 10.0.0.0/16 (West Europe)        │
│ VM1: 10.0.1.4                          │
└────────────────────────────────────────┘
              ↕ (Peered)
┌────────────────────────────────────────┐
│ VNet2: 10.1.0.0/16 (West Europe)        │
│ VM2: 10.1.2.5                          │
└────────────────────────────────────────┘

Step 1: VM1 sends packet
Dst: 10.1.2.5
Check routing table:
- 10.0.0.0/16: Local ✗
- 10.1.0.0/16: VNetPeering ✓ (auto-injected route)

Gateway: 10.0.1.1 (Azure virtual)

Step 2: VFP on VM1's host
- NSG check (VM1 outbound):
  * VNet1 subnet NSG: Allow outbound to VirtualNetwork ✓
  * (Peered VNets included in VirtualNetwork tag)
- Routing: 10.1.0.0/16 → VNetPeering
- Lookup: VNet2 VNI and VM2 location
- Encapsulate:
  * VNI: VNet2's VNI (not VNet1's!)
  * Outer IP: Host-A → Host-D (VM2's host)
  * Inner: 10.0.1.4 → 10.1.2.5

Step 3: Physical network (Azure backbone)
Host-A → Host-D
- Same region: Intra-datacenter
- Or different region: Microsoft backbone network
- Encrypted in transit (always)

Step 4: VFP on Host-D (VM2's host)
- Receives packet
- Checks VNI: VNet2 ✓
- NSG check (VM2 inbound):
  * VNet2 subnet NSG: Check rules
  * Allow from VirtualNetwork (includes peered VNets) ✓
- Decapsulate
- Deliver to VM2

Step 5: VM2 receives
Src: 10.0.1.4 (VNet1) - preserved
Dst: 10.1.2.5 (me)

Process packet

Step 6: Return traffic
Same process, reverse direction
NSG: Stateful, auto-allow return

Latency:
- Regional peering: <2ms
- Global peering: Varies by distance
  * West Europe ↔ East US: ~80-100ms

Peering Properties Effect:
allowForwardedTraffic: true
- Required for hub-spoke routing
- Allows traffic from other VNets via this VNet

allowGatewayTransit: true (VNet1)
useRemoteGateways: true (VNet2)
- VNet2 can use VNet1's VPN Gateway
- For on-premises connectivity

Billing:
Regional peering:
- Ingress: Free
- Egress: $0.01/GB

Global peering:
- Ingress: $0.035/GB
- Egress: $0.035/GB

(Prices as of 2025, check Azure pricing)
```

### Flow 6: VNet to On-Premises (via VPN Gateway)

```
Scenario: Azure VM accessing on-premises server
VNet VM (10.0.1.4) → On-Prem Server (192.168.1.10)
Connectivity: Site-to-Site VPN Gateway

┌────────────────────────────────────────┐
│ Azure VNet: 10.0.0.0/16                 │
│ VM1: 10.0.1.4                          │
│                                        │
│ GatewaySubnet: 10.0.255.0/27           │
│ VPN Gateway: 10.0.255.4                │
│ Public IP: 40.50.60.100                │
└────────────────────────────────────────┘
         ↕ (IPsec VPN Tunnel)
┌────────────────────────────────────────┐
│ On-Premises Network: 192.168.1.0/24    │
│ VPN Device: 203.0.113.10               │
│ Server: 192.168.1.10                   │
└────────────────────────────────────────┘

Configuration:
Local Network Gateway:
- On-prem address space: 192.168.1.0/24
- VPN device IP: 203.0.113.10

VPN Gateway:
- Type: Route-based
- SKU: VpnGw1
- BGP: Enabled (optional)

Auto-injected Routes:
VM1's routing table:
10.0.0.0/16 → VnetLocal
192.168.1.0/24 → VirtualNetworkGateway

Step 1: VM1 sends packet
Dst: 192.168.1.10
Check routing table:
- 10.0.0.0/16: Local ✗
- 192.168.1.0/24: VirtualNetworkGateway ✓

Gateway: 10.0.1.1 (route to VPN Gateway)

Step 2: VFP on VM1's host
- NSG check: Allow outbound ✓
- Routing: 192.168.1.0/24 → VPN Gateway
- Route to GatewaySubnet: 10.0.255.4
- Encapsulate and forward to Gateway host

Step 3: VPN Gateway receives
Packet arrives at: 10.0.255.4
Src: 10.0.1.4
Dst: 192.168.1.10

VPN Gateway processing:
- Lookup destination: 192.168.1.10
- Match: On-prem network (192.168.1.0/24)
- Select VPN tunnel to: 203.0.113.10
- Encapsulate in IPsec:
  * ESP (Encapsulating Security Payload)
  * Encrypted payload
  * Outer IP: 40.50.60.100 → 203.0.113.10

Step 4: Internet
IPsec packet traverses internet
Encrypted end-to-end

Step 5: On-prem VPN device (203.0.113.10)
- Receives IPsec packet
- Decrypts
- Extract inner packet: 10.0.1.4 → 192.168.1.10
- Routes to local network

Step 6: On-prem server (192.168.1.10)
- Receives packet
- Src: 10.0.1.4 (Azure VM)
- Processes request

Step 7: Return path
192.168.1.10 → 10.0.1.4
- Via on-prem router
- VPN device encrypts
- Sends via IPsec tunnel
- Azure VPN Gateway decrypts
- Routes to VNet
- Delivers to VM1

VPN Gateway SKUs:
┌──────────┬───────────┬────────┬──────────┐
│ SKU      │ Throughput│ Tunnels│ BGP      │
├──────────┼───────────┼────────┼──────────┤
│ Basic    │ 100 Mbps  │ 10     │ No       │
│ VpnGw1   │ 650 Mbps  │ 30     │ Yes      │
│ VpnGw2   │ 1 Gbps    │ 30     │ Yes      │
│ VpnGw3   │ 1.25 Gbps │ 30     │ Yes      │
│ VpnGw1AZ │ 650 Mbps  │ 30     │ Yes + AZ │
└──────────┴───────────┴────────┴──────────┘

ExpressRoute (Alternative):
- Dedicated private connection
- Not over internet
- Higher bandwidth (50 Mbps - 100 Gbps)
- More expensive
- Lower latency
- Predictable performance
```

-----

## Layer 7: Physical Implementation

### Azure Datacenter Networking

**Physical Network Architecture**:

```
Azure Region Datacenter:

┌────────────────────────────────────────┐
│ Internet / External Connectivity        │
│ - BGP Peering with ISPs               │
│ - DDoS Protection                      │
│ - Azure Edge Network                   │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Border/Edge Routers                     │
│ - 100G/400G links                      │
│ - ECMP load balancing                  │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Core/Spine Switches (Clos topology)    │
│ - High-capacity switching fabric       │
│ - Non-blocking architecture            │
│ - Redundant paths                      │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Top-of-Rack (ToR) Switches              │
│ - 10G/25G/40G/100G downlinks           │
│ - VXLAN capable                        │
│ - SDN integration                      │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Compute Servers (Hyper-V Hosts)        │
│ - 10G/25G/40G NICs                     │
│ - SR-IOV support                       │
│ - Hardware offload                     │
│ - VFP (Virtual Filtering Platform)     │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Virtual Machines                        │
│ - Your workloads                       │
└────────────────────────────────────────┘

Physical Isolation:
- No physical VLANs for customer VNets
- All isolation via VXLAN overlay
- Shared physical infrastructure
- Logical separation enforced by SDN
```

**Hyper-V Host Architecture**:

```
Single Compute Server:

┌────────────────────────────────────────┐
│ Physical Server                         │
│                                        │
│ CPU: Dual Xeon (40+ cores)             │
│ RAM: 512 GB - 1 TB                     │
│ NICs: 2-4x 25G/40G (bonded)            │
│ Storage: Local SSD + Remote (SMB3)     │
│                                        │
│ ┌────────────────────────────────────┐│
│ │ Windows Server + Hyper-V            ││
│ │                                    ││
│ │ ┌────────────────────────────────┐││
│ │ │ Virtual Filtering Platform (VFP)│││
│ │ │ - Packet processing             │││
│ │ │ - NSG enforcement               │││
│ │ │ - VXLAN encap/decap             │││
│ │ │ - NAT/Load balancing            │││
│ │ └────────────────────────────────┘││
│ │                                    ││
│ │ ┌──────────┐ ┌──────────┐         ││
│ │ │ VM1      │ │ VM2      │ ...     ││
│ │ │ vCPU: 2  │ │ vCPU: 4  │         ││
│ │ │ RAM: 8GB │ │ RAM: 16GB│         ││
│ │ │ VNet: A  │ │ VNet: B  │         ││
│ │ └──────────┘ └──────────┘         ││
│ │                                    ││
│ │ Hyper-V Virtual Switch:            ││
│ │ - SR-IOV support                   ││
│ │ - VFP extension                    ││
│ │ - Multiple external adapters       ││
│ └────────────────────────────────────┘│
│                                        │
│ Physical NICs:                         │
│ ├─ NIC 1: Management (RDMA)           │
│ ├─ NIC 2: VM traffic (VXLAN)          │
│ └─ NIC 3-4: Storage (SMB Direct)      │
└────────────────────────────────────────┘

Host Networking Stack:
┌────────────────────────────────────────┐
│ VM vNIC                                 │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Hyper-V Virtual Switch                  │
│ - MAC learning                         │
│ - VLAN tagging (host-level)           │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ VFP Extension (Layers)                  │
│ Layer 1: Parsing                       │
│ Layer 2: Stateful firewall (NSG)       │
│ Layer 3: Routing                       │
│ Layer 4: NAT / Load balancing          │
│ Layer 5: Metering                      │
│ Layer 6: Encapsulation (VXLAN)         │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Physical NIC                            │
│ - Hardware offload (checksums, segmentation)│
│ - SR-IOV (bypass hypervisor)           │
│ - RSS (Receive Side Scaling)           │
└────────────────────────────────────────┘
         ↓
     ToR Switch
```

### SDN Controller Architecture

**Control Plane**:

```
Azure SDN Control Plane:

┌────────────────────────────────────────┐
│ Azure Resource Manager (ARM)            │
│ - API Gateway                          │
│ - RBAC enforcement                     │
│ - Configuration validation             │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ Network Resource Provider               │
│ - VNet management                      │
│ - Subnet management                    │
│ - NSG management                       │
│ - Route table management               │
│ - Peering management                   │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ SDN Network Controller (Regional)       │
│ - Maintains network state              │
│ - Computes routing tables              │
│ - Assigns VNIs to VNets                │
│ - Tracks VM locations                  │
│ - Distributes policies to hosts        │
└────────────────────────────────────────┘
         ↓ (Southbound API)
┌────────────────────────────────────────┐
│ Host Agents (On Each Compute Server)    │
│ - Receives policy from controller      │
│ - Programs VFP rules                   │
│ - Reports state back                   │
│ - Monitors health                      │
└────────────────────────────────────────┘

Configuration Distribution:

Step 1: User creates NSG rule (Portal/CLI/ARM)
ARM API receives request
         ↓
Step 2: Network Resource Provider
Validates configuration
Stores in database (geo-replicated)
         ↓
Step 3: Notification to SDN Controller
Controller receives: "NSG updated for VNet X"
         ↓
Step 4: Controller processes
Determines affected hosts
Computes new VFP policies
Generates host-specific configs
         ↓
Step 5: Distribution to hosts
Pushes policy to each affected host
Host agent receives update
         ↓
Step 6: Host applies configuration
Host agent programs VFP
VFP updates packet processing rules
         ↓
Step 7: Confirmation
Host reports success
Controller updates state
ARM shows "Succeeded"

Time: ~1-5 seconds typical
```

**Network State Database**:

```
SDN Controller State (Per Region):

VNet Registry:
┌────────────────────────────────────────┐
│ VNet ID: vnet-12345                     │
│ VNI: 5001                              │
│ Address Space: 10.0.0.0/16             │
│ Region: westeurope                     │
│ Subnets:                               │
│ ├─ subnet-001: 10.0.1.0/24            │
│ │  NSG: nsg-web                       │
│ │  Route Table: rt-web                │
│ ├─ subnet-002: 10.0.2.0/24            │
│ │  NSG: nsg-app                       │
│ └─ subnet-003: 10.0.3.0/24            │
│    NSG: nsg-db                        │
│ Peerings:                              │
│ └─ vnet-67890: Connected              │
└────────────────────────────────────────┘

VM Location Mapping:
┌────────────────────────────────────────┐
│ VM: vm-web01                            │
│ VNet: vnet-12345 (VNI: 5001)           │
│ Subnet: 10.0.1.0/24                    │
│ Private IP: 10.0.1.4                   │
│ MAC: 00:0D:3A:11:11:11                 │
│ Host: compute-node-042                 │
│ Rack: rack-05                          │
│ NIC: vmnic-789                         │
└────────────────────────────────────────┘

Routing Table (Per VNet):
┌────────────────────────────────────────┐
│ VNet: vnet-12345                        │
│                                        │
│ Routes:                                │
│ 10.0.0.0/16 → VnetLocal                │
│ 10.1.0.0/16 → VnetPeering (vnet-67890) │
│ 192.168.0.0/16 → VPN Gateway           │
│ 0.0.0.0/0 → Internet                   │
│                                        │
│ Distributed to: All VMs in this VNet   │
└────────────────────────────────────────┘

This state distributed to every host
running VMs in this VNet!
```

### Physical Network Topology

**Clos Network (Spine-Leaf)**:

```
Azure Datacenter Network Topology:

┌──────────────────────────────────────────────┐
│ Spine Layer (Core)                           │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │
│ │Spine1│ │Spine2│ │Spine3│ │Spine4│         │
│ └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘         │
│     └────────┼────────┼────────┘             │
└──────────────┼────────┼──────────────────────┘
               │        │
┌──────────────┼────────┼──────────────────────┐
│ Leaf Layer (ToR Switches)                    │
│  ┌────┴────┬─┴──────┬─┴──────┬───────┐       │
│  │         │        │        │       │       │
│ ┌┴─┐ ┌───┐ ┌┴─┐ ┌──┐ ┌┴─┐ ┌──┐ ┌──┴┐ ┌───┐   │
│ │L1│ │L2│ │L3│ │L4│ │L5│ │L6│ │L7│ │L8│   │
│ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘   │
└───┼────┼────┼────┼────┼────┼────┼────┼─────┘
    │    │    │    │    │    │    │    │
┌───┴────┴────┴────┴────┴────┴────┴────┴─────┐
│ Compute Servers (Racks)                     │
│ [Srv1][Srv2]...[SrvN] per rack             │
│ Each server: 10-20 VMs                     │
└─────────────────────────────────────────────┘

Key Properties:
✓ Non-blocking: Any server → Any server (full bandwidth)
✓ Redundant paths: Multiple routes between any two servers
✓ ECMP: Equal-Cost Multi-Path for load balancing
✓ Scalable: Add more spines/leaves as needed
✓ No spanning tree: Loop-free by design

Example Packet Path:
Server1 (Leaf1) → Server2 (Leaf3)

Possible Paths:
1. Server1 → Leaf1 → Spine1 → Leaf3 → Server2
2. Server1 → Leaf1 → Spine2 → Leaf3 → Server2
3. Server1 → Leaf1 → Spine3 → Leaf3 → Server2
4. Server1 → Leaf1 → Spine4 → Leaf3 → Server2

ECMP selects path based on 5-tuple hash:
- Source IP
- Destination IP
- Source Port
- Destination Port
- Protocol

Same flow always uses same path (no reordering)
Different flows distributed across paths
```

### High Availability & Redundancy

**Redundancy at Every Layer**:

```
Redundancy in Azure Networking:

Physical Layer:
- Redundant ToR switches (pair)
- Redundant uplinks (LACP bonded)
- Redundant power supplies
- Redundant cooling

Network Layer:
- Multiple spine switches
- ECMP across all paths
- Fast convergence (<1 second)

Host Layer:
- Bonded NICs (LACP)
- Automatic failover
- SR-IOV failover to software

VM Layer:
- Availability Sets (different racks)
- Availability Zones (different datacenters)
- Update domains (planned maintenance)
- Fault domains (physical isolation)

Example Failure Scenarios:

Scenario 1: ToR Switch Failure
┌────────────────────────────────────────┐
│ ToR1 fails (powers off)                 │
│ Servers connected to ToR1:              │
│ - Detect link down (immediate)         │
│ - Failover to ToR2 (bonded pair)       │
│ - Traffic continues (<1 second)        │
│ Impact: None to VMs                    │
└────────────────────────────────────────┘

Scenario 2: Spine Switch Failure
┌────────────────────────────────────────┐
│ Spine1 fails                            │
│ ToR switches detect:                   │
│ - Remove Spine1 from ECMP               │
│ - Redistribute flows to Spine2-4        │
│ - BGP converges (<1 second)            │
│ Impact: None to VMs                    │
│ (3 of 4 spines still available)        │
└────────────────────────────────────────┘

Scenario 3: Host NIC Failure
┌────────────────────────────────────────┐
│ Physical NIC fails on compute server    │
│ Host OS detects:                        │
│ - Link down on NIC1                    │
│ - LACP failover to NIC2                │
│ - VMs continue (transparent)           │
│ Impact: None to VMs                    │
│ (<1 second disruption possible)        │
└────────────────────────────────────────┘

Scenario 4: Entire Host Failure
┌────────────────────────────────────────┐
│ Compute server fails (hardware)         │
│ Azure platform detects:                │
│ - VMs stop sending heartbeat           │
│ - Declare VMs down (60-90 seconds)     │
│ - If Availability Set configured:      │
│   * Restart VMs on different host      │
│   * 2-5 minutes downtime               │
│ - If single VM (no AvSet):             │
│   * Manual restart required            │
└────────────────────────────────────────┘

Scenario 5: Availability Zone Failure
┌────────────────────────────────────────┐
│ Zone 1 entire datacenter offline        │
│ Impact:                                │
│ - VMs in Zone 1: Down                  │
│ - VMs in Zone 2, 3: Unaffected ✓       │
│ - Zone-redundant resources: Continue   │
│   (e.g., zone-redundant Public IPs)    │
│ - VNet peering: Still works            │
│   (between zones)                      │
└────────────────────────────────────────┘

Best Practices for HA:
✓ Use Availability Zones (different datacenters)
✓ Use Availability Sets (different racks)
✓ Use Load Balancers (distribute traffic)
✓ Design stateless apps
✓ Use Azure-managed PaaS (built-in HA)
```

-----

## Layer 8: Practical Examples

### Example 1: Hub-Spoke Topology

**Scenario**: Enterprise hub-spoke network design

```
Topology:
┌────────────────────────────────────────┐
│ Hub VNet (10.100.0.0/16)                │
│ Region: West Europe                    │
│                                        │
│ ├─ GatewaySubnet: 10.100.0.0/27       │
│ │  └─ VPN Gateway (to on-premises)     │
│ │                                      │
│ ├─ FirewallSubnet: 10.100.1.0/26      │
│ │  └─ Azure Firewall (10.100.1.4)      │
│ │                                      │
│ └─ ManagementSubnet: 10.100.2.0/24    │
│    └─ Jump box, monitoring             │
└────────────────────────────────────────┘
         ↕          ↕          ↕
    (Peered)   (Peered)   (Peered)
         ↓          ↓          ↓
┌────────────┬────────────┬────────────┐
│ Spoke 1    │ Spoke 2    │ Spoke 3    │
│ Prod Web   │ Prod App   │ Prod DB    │
│ 10.1.0.0/16│ 10.2.0.0/16│ 10.3.0.0/16│
└────────────┴────────────┴────────────┘

Design Goals:
✓ Centralized security (firewall in hub)
✓ Centralized connectivity (VPN in hub)
✓ Spoke-to-spoke via hub
✓ Internet via hub firewall
✓ Shared services in hub
```

**Configuration**:

```
Hub VNet Peering Settings:
To Spoke1:
{
  "allowVirtualNetworkAccess": true,
  "allowForwardedTraffic": true,    ← Allow spoke-to-spoke
  "allowGatewayTransit": true,      ← Share VPN gateway
  "useRemoteGateways": false
}

Spoke1 Peering Settings:
To Hub:
{
  "allowVirtualNetworkAccess": true,
  "allowForwardedTraffic": true,    ← Accept forwarded traffic
  "allowGatewayTransit": false,
  "useRemoteGateways": true         ← Use hub's VPN gateway
}

Spoke1 Route Table (All subnets):
┌────────────────────────────────────────┐
│ Destination       NextHop               │
├────────────────────────────────────────┤
│ 10.1.0.0/16       VnetLocal            │← Spoke1 local
│ 10.100.0.0/16     VnetPeering          │← Hub
│ 10.2.0.0/16       VirtualAppliance     │← Spoke2 via FW
│                   10.100.1.4           │
│ 10.3.0.0/16       VirtualAppliance     │← Spoke3 via FW
│                   10.100.1.4           │
│ 192.168.0.0/16    VirtualNetworkGateway│← On-prem via VPN
│ 0.0.0.0/0         VirtualAppliance     │← Internet via FW
│                   10.100.1.4           │
└────────────────────────────────────────┘

Azure Firewall Configuration:
Enable IP Forwarding: Yes
Routing: Forward packets between spokes

Traffic Flows:

Spoke1 → Spoke2 (Web tier → App tier):
1. VM in Spoke1: 10.1.1.4
   Wants to reach: 10.2.1.5 (Spoke2)
   
2. Check UDR: 10.2.0.0/16 → 10.100.1.4 (Firewall)
   
3. Packet sent to Hub:
   Via peering to Hub VNet
   
4. Azure Firewall receives:
   Src: 10.1.1.4
   Dst: 10.2.1.5
   
5. Firewall checks rules:
   Network Rule: Allow Spoke1 → Spoke2:443
   → ALLOW
   
6. Firewall forwards to Spoke2:
   Via peering to Spoke2
   
7. Spoke2 VM receives packet

Spoke1 → Internet:
1. VM: 10.1.1.4 wants to reach 8.8.8.8
2. UDR: 0.0.0.0/0 → 10.100.1.4
3. Firewall receives, checks rules
4. SNAT: 10.1.1.4 → Firewall Public IP
5. Send to internet

Spoke1 → On-Premises:
1. VM: 10.1.1.4 wants to reach 192.168.1.10
2. UDR: 192.168.0.0/16 → VPN Gateway
   (Peering allowGatewayTransit enables this)
3. Traffic routed to Hub's VPN Gateway
4. VPN Gateway encrypts and sends via tunnel

Benefits:
✓ Centralized security policy
✓ Single firewall for all spokes
✓ Single VPN gateway (cost savings)
✓ Spoke isolation (no direct spoke-to-spoke)
✓ Easy to add new spokes
✓ Centralized monitoring

Limitations:
- Firewall is single point (use AZ redundancy)
- Additional hop (slight latency)
- Requires UDRs on every spoke subnet
```

### Example 2: Three-Tier Application

**Scenario**: Web-App-Database architecture

```
Application: E-commerce Platform
VNet: 10.0.0.0/16

┌────────────────────────────────────────┐
│ Web Tier (10.0.1.0/24)                  │
│ - Azure Load Balancer (public)         │
│ - VM Scale Set (3 instances)           │
│ - Outbound: App tier only              │
│ - Inbound: Internet (80/443)           │
└────────────────────────────────────────┘
         ↓ (Allow 8080)
┌────────────────────────────────────────┐
│ App Tier (10.0.2.0/24)                  │
│ - Internal Load Balancer               │
│ - VM Scale Set (5 instances)           │
│ - Outbound: DB tier, external APIs     │
│ - Inbound: Web tier only               │
└────────────────────────────────────────┘
         ↓ (Allow 1433)
┌────────────────────────────────────────┐
│ Data Tier (10.0.3.0/24)                 │
│ - SQL Server AlwaysOn AG                │
│ - 3 SQL VMs                            │
│ - Outbound: None (except replication)  │
│ - Inbound: App tier only               │
└────────────────────────────────────────┘
```

**NSG Configuration**:

```
Web Tier NSG (WebNSG):
Inbound Rules:
┌─────────┬──────────┬──────┬───────┬──────────┐
│ Priority│ Name     │ Src  │ Dst   │ Port     │
├─────────┼──────────┼──────┼───────┼──────────┤
│ 100     │ AllowHTTP│ *    │ *     │ 80       │
│ 110     │ AllowHTTPS│ *   │ *     │ 443      │
│ 4096    │ DenyAll  │ *    │ *     │ *        │
└─────────┴──────────┴──────┴───────┴──────────┘

Outbound Rules:
┌─────────┬──────────────┬────────────┬──────────┬──────┐
│ Priority│ Name         │ Src        │ Dst      │ Port │
├─────────┼──────────────┼────────────┼──────────┼──────┤
│ 100     │ AllowAppTier │ *          │10.0.2.0/24│ 8080│
│ 200     │ AllowInternet│ *          │ Internet │ *    │
│ 4096    │ DenyAll      │ *          │ *        │ *    │
└─────────┴──────────────┴────────────┴──────────┴──────┘

App Tier NSG (AppNSG):
Inbound Rules:
┌─────────┬──────────────┬────────────┬────────┬──────┐
│ Priority│ Name         │ Src        │ Dst    │ Port │
├─────────┼──────────────┼────────────┼────────┼──────┤
│ 100     │ AllowWeb     │ 10.0.1.0/24│ *      │ 8080 │
│ 4096    │ DenyAll      │ *          │ *      │ *    │
└─────────┴──────────────┴────────────┴────────┴──────┘

Outbound Rules:
┌─────────┬──────────────┬────────────┬────────────┬──────┐
│ Priority│ Name         │ Src        │ Dst        │ Port │
├─────────┼──────────────┼────────────┼────────────┼──────┤
│ 100     │ AllowDB      │ *          │ 10.0.3.0/24│ 1433 │
│ 110     │ AllowAPIs    │ *          │ Internet   │ 443  │
│ 4096    │ DenyAll      │ *          │ *          │ *    │
└─────────┴──────────────┴────────────┴────────────┴──────┘

Data Tier NSG (DataNSG):
Inbound Rules:
┌─────────┬──────────────┬────────────┬────────┬──────┐
│ Priority│ Name         │ Src        │ Dst    │ Port │
├─────────┼──────────────┼────────────┼────────┼──────┤
│ 100     │ AllowApp     │ 10.0.2.0/24│ *      │ 1433 │
│ 110     │ AllowSQL     │ 10.0.3.0/24│ *      │ 5022 │← Replication
│ 4096    │ DenyAll      │ *          │ *      │ *    │
└─────────┴──────────────┴────────────┴────────┴──────┘

Outbound Rules:
┌─────────┬──────────────┬────────────┬────────────┬──────┐
│ Priority│ Name         │ Src        │ Dst        │ Port │
├─────────┼──────────────┼────────────┼────────────┼──────┤
│ 100     │ AllowSQL     │ *          │ 10.0.3.0/24│ 5022 │
│ 4096    │ DenyAll      │ *          │ *          │ *    │
└─────────┴──────────────┴────────────┴────────────┴──────┘

Traffic Flow Example:
User Request: https://www.contoso.com/products

1. DNS: www.contoso.com → 40.50.60.90 (LB public IP)
2. Azure Load Balancer:
   - Receives: 1.2.3.4:52000 → 40.50.60.90:443
   - Selects backend: Web VM1 (10.0.1.4)
   - DNAT: → 10.0.1.4:443
3. Web VM1 NSG:
   - Inbound rule 110 (AllowHTTPS): ALLOW ✓
4. Web VM1 processes:
   - Static content: Return directly
   - Dynamic content: Call app tier
     * HTTP POST to 10.0.2.4:8080
5. Web VM1 NSG (outbound):
   - Rule 100 (AllowAppTier): ALLOW ✓
6. App VM1 NSG (inbound):
   - Rule 100 (AllowWeb, src 10.0.1.0/24): ALLOW ✓
7. App VM1 processes:
   - Business logic
   - Database query: Connect to SQL
     * TCP to 10.0.3.4:1433
8. App VM1 NSG (outbound):
   - Rule 100 (AllowDB): ALLOW ✓
9. SQL VM1 NSG (inbound):
   - Rule 100 (AllowApp, src 10.0.2.0/24): ALLOW ✓
10. SQL processes query, returns data
11. Return path (NSG stateful):
    - SQL → App: Auto-allowed
    - App → Web: Auto-allowed
    - Web → User: Auto-allowed

Security Benefits:
✓ Web tier: Only 80/443 from internet
✓ App tier: No internet inbound
✓ DB tier: No internet inbound
✓ Lateral movement prevented
✓ Defense in depth
```

### Example 3: Private Endpoint for PaaS Services

**Scenario**: Secure Azure SQL Database with Private Endpoint

```
Requirement:
- Azure SQL Database
- No public internet access
- Accessible from VNet only
- Private IP in VNet

Configuration:
┌────────────────────────────────────────┐
│ VNet: 10.0.0.0/16                       │
│                                        │
│ ├─ AppSubnet: 10.0.1.0/24              │
│ │  └─ VM App Servers                   │
│ │                                      │
│ └─ PrivateEndpointSubnet: 10.0.5.0/24  │
│    └─ Private Endpoint for SQL         │
│       IP: 10.0.5.4                     │
└────────────────────────────────────────┘
         ↓ (Private Link)
┌────────────────────────────────────────┐
│ Azure SQL Database                      │
│ Name: mydb.database.windows.net        │
│ Public endpoint: DISABLED               │
│ Private endpoint: ENABLED               │
└────────────────────────────────────────┘
```

**Step-by-Step Setup**:

```
Step 1: Create Azure SQL Database
Name: mydb
Public network access: Disabled
Firewall: Deny all

Step 2: Create Private Endpoint
Resource: Azure SQL Database (mydb)
Target sub-resource: sqlServer
VNet: MyVNet
Subnet: PrivateEndpointSubnet (10.0.5.0/24)
Private IP: 10.0.5.4 (auto-assigned)

Result:
- NIC created in subnet: 10.0.5.4
- Private Link connection established

Step 3: DNS Configuration (Automatic)
Azure creates Private DNS Zone:
privatelink.database.windows.net

DNS Zone linked to VNet

A Record created:
mydb → 10.0.5.4

Step 4: Test Connectivity
From VM in AppSubnet (10.0.1.4):

$ nslookup mydb.database.windows.net
Server: 168.63.129.16
Address: 168.63.129.16#53

Non-authoritative answer:
mydb.database.windows.net
  canonical name = mydb.privatelink.database.windows.net
mydb.privatelink.database.windows.net
  Address: 10.0.5.4  ← Private IP!

$ sqlcmd -S mydb.database.windows.net -U sqladmin
Connected successfully!

Step 5: Verify from Internet (Should Fail)
From external machine:

$ nslookup mydb.database.windows.net
Address: 20.xxx.xxx.xxx ← Public IP

$ sqlcmd -S mydb.database.windows.net -U sqladmin
Error: Cannot connect
(Public access disabled)

Traffic Flow:
App VM (10.0.1.4) → SQL Database

1. Application: Connect to mydb.database.windows.net
2. DNS resolution:
   - Query: mydb.database.windows.net
   - CNAME: mydb.privatelink.database.windows.net
   - A record: 10.0.5.4 (Private DNS Zone)
3. App VM connects to: 10.0.5.4:1433
4. Packet routing:
   - Dst: 10.0.5.0/24 (PrivateEndpointSubnet)
   - Route: VnetLocal
   - NSG: Check rules (should allow)
5. Private Endpoint NIC receives
6. Private Link tunnel to SQL backend
7. SQL Database processes query
8. Response returns via same path

Benefits:
✓ No public internet exposure
✓ SQL Database fully private
✓ VNet integration (NSG, UDR apply)
✓ Data exfiltration prevention
✓ Compliance requirements met
✓ Works with on-premises (via VPN)

NSG on PrivateEndpointSubnet:
Inbound:
Priority 100: Allow 1433 from 10.0.1.0/24 (AppSubnet)
Priority 4096: Deny All

Outbound:
Default: Allow (return traffic)

Cost:
- Private Endpoint: $7.30/month
- Data processing: $0.01/GB
```

### Example 4: Service Endpoint for Storage

**Scenario**: Restrict Storage Account to VNet

```
Requirement:
- Azure Storage Account
- Allow access only from specific VNet/Subnets
- Optimal routing (Azure backbone)
- No Private Endpoint cost

Configuration:
┌────────────────────────────────────────┐
│ VNet: 10.0.0.0/16                       │
│                                        │
│ ├─ WebSubnet: 10.0.1.0/24              │
│ │  Service Endpoint: Microsoft.Storage │
│ │  VMs: 10.0.1.4, 10.0.1.5             │
│ │                                      │
│ └─ AppSubnet: 10.0.2.0/24              │
│    Service Endpoint: Microsoft.Storage │
│    VMs: 10.0.2.4, 10.0.2.5             │
└────────────────────────────────────────┘
         ↓ (Azure Backbone)
┌────────────────────────────────────────┐
│ Storage Account: mystorageacct          │
│ Networking:                            │
│ - Public endpoint: ENABLED             │
│ - Firewall: Allow from selected VNets  │
│   * MyVNet/WebSubnet                   │
│   * MyVNet/AppSubnet                   │
│ - Deny all other traffic               │
└────────────────────────────​​​​​​​​​​​​​​​​────────────┘

```
**Step-by-Step Configuration**:
```

Step 1: Enable Service Endpoint on Subnets
WebSubnet → Service endpoints → Add
Select: Microsoft.Storage
Region: West Europe (same as storage)

AppSubnet → Service endpoints → Add
Select: Microsoft.Storage
Region: West Europe

Effect:

- Route injected for storage IP ranges
- Traffic routed via Azure backbone
- Source IP preserved (VM private IP)

Step 2: Configure Storage Account Firewall
Storage Account → Networking

Public network access: Enabled from selected VNets

Firewall and virtual networks:

- Selected networks

Virtual networks:

- Add existing virtual network
  VNet: MyVNet
  Subnets: WebSubnet, AppSubnet

Exceptions:
☑ Allow Azure services on the trusted services list

Save

Step 3: Test Access
From VM in WebSubnet (10.0.1.4):

$ az storage blob list  
–account-name mystorageacct  
–container-name mycontainer  
–auth-mode login

Success! ✓
(VM’s private IP 10.0.1.4 allowed by firewall)

From Internet or different VNet:

$ curl <https://mystorageacct.blob.core.windows.net/>

Error: 403 Forbidden
(Not from allowed VNet)

Traffic Flow Analysis:

VM (10.0.1.4) → Storage Account

Without Service Endpoint:

1. VM → Default route (0.0.0.0/0)
1. SNAT applied (VM IP → NAT IP)
1. Via Azure edge → Internet
1. Storage sees: NAT public IP
1. Storage firewall blocks (not in allowed list)

With Service Endpoint:

1. VM → Storage (52.239.x.x)
1. VFP routing: Service endpoint route
1. Via Azure backbone (not internet)
1. Source IP preserved: 10.0.1.4
1. Storage firewall checks:

- Source IP: 10.0.1.4
- VNet: MyVNet
- Subnet: WebSubnet ✓ (in allowed list)

1. Access granted!

Routing Table (WebSubnet VMs):
┌────────────────────────────────────────┐
│ Destination           NextHop           │
├────────────────────────────────────────┤
│ 10.0.0.0/16           VnetLocal         │
│ 52.239.0.0/18         ServiceEndpoint   │← Storage IPs
│ 52.239.64.0/18        ServiceEndpoint   │
│ 0.0.0.0/0             Internet          │
└────────────────────────────────────────┘

Benefits:
✓ Optimal routing (Azure backbone)
✓ Source IP preserved
✓ VNet-level restriction
✓ No additional cost (free!)
✓ Simple configuration

Limitations:

- Storage still has public endpoint
- Same region typically required
- Service-level restriction (all storage accounts)
- Use Service Endpoint Policy for finer control

Service Endpoint Policy (Advanced):
┌────────────────────────────────────────┐
│ Service Endpoint Policy                 │
│ Name: AllowOnlyMyStorage                │
│                                        │
│ Service: Microsoft.Storage              │
│ Allowed Resources:                     │
│ - /subscriptions/{sub}/…              │
│   …/storageAccounts/mystorageacct    │
│                                        │
│ Effect: Block access to other storage  │
│ accounts, even via service endpoint    │
└────────────────────────────────────────┘

Applied to WebSubnet:
VMs can only access mystorageacct
Cannot access otherstorage (blocked by policy)

Use Case: Prevent data exfiltration

- Employee cannot copy data to personal storage
- Only approved storage accounts accessible

```
### Example 5: Multi-Region with Global VNet Peering

**Scenario**: Global application with regional VNets
```

Architecture:
┌────────────────────────────────────────┐
│ West Europe VNet (10.0.0.0/16)          │
│ - Web tier: 10.0.1.0/24                │
│ - App tier: 10.0.2.0/24                │
│ - DB tier: 10.0.3.0/24 (read replica)  │
└────────────────────────────────────────┘
↕ (Global Peering)
┌────────────────────────────────────────┐
│ East US VNet (10.1.0.0/16)              │
│ - Web tier: 10.1.1.0/24                │
│ - App tier: 10.1.2.0/24                │
│ - DB tier: 10.1.3.0/24 (read replica)  │
└────────────────────────────────────────┘
↕ (Global Peering)
┌────────────────────────────────────────┐
│ Southeast Asia VNet (10.2.0.0/16)       │
│ - Web tier: 10.2.1.0/24                │
│ - App tier: 10.2.2.0/24                │
│ - DB tier: 10.2.3.0/24 (read replica)  │
└────────────────────────────────────────┘

User Distribution:

- Europe users → West Europe VNet (low latency)
- US users → East US VNet (low latency)
- Asia users → Southeast Asia VNet (low latency)

Cross-Region Communication:

- DB replication: Between all regions
- Shared services: Via peering
- Management: Via peering

```
**Peering Configuration**:
```

West Europe ←→ East US Peering:

West Europe side:
{
“name”: “WestEurope-to-EastUS”,
“remoteVirtualNetwork”: {
“id”: “…/EastUS-VNet”
},
“allowVirtualNetworkAccess”: true,
“allowForwardedTraffic”: false,
“allowGatewayTransit”: false,
“useRemoteGateways”: false
}

East US side:
{
“name”: “EastUS-to-WestEurope”,
“remoteVirtualNetwork”: {
“id”: “…/WestEurope-VNet”
},
“allowVirtualNetworkAccess”: true,
“allowForwardedTraffic”: false,
“allowGatewayTransit”: false,
“useRemoteGateways”: false
}

Similar peering between:

- West Europe ←→ Southeast Asia
- East US ←→ Southeast Asia

Routing (Auto-Injected):

West Europe VNet routing table:
┌────────────────────────────────────────┐
│ Destination       NextHop               │
├────────────────────────────────────────┤
│ 10.0.0.0/16       VnetLocal            │← Local
│ 10.1.0.0/16       VnetGlobalPeering    │← East US
│ 10.2.0.0/16       VnetGlobalPeering    │← Asia
│ 0.0.0.0/0         Internet              │
└────────────────────────────────────────┘

Traffic Example:
West Europe DB → East US DB (Replication)

1. SQL Server in West Europe: 10.0.3.4
   Replication to: 10.1.3.4 (East US)
1. Check routing: 10.1.0.0/16 → VnetGlobalPeering
1. VFP encapsulates (VXLAN)
   VNI: East US VNet
1. Physical network:

- West Europe datacenter
- Azure Backbone (subsea fiber)
- Across Atlantic Ocean
- East US datacenter

1. Latency: ~80-100ms (geographic distance)
1. East US VFP receives:

- Decapsulates
- Delivers to 10.1.3.4

1. SQL replication data transferred

NSG Considerations:
West Europe DB Subnet NSG:
Outbound:

- Priority 100: Allow 1433 to 10.1.3.0/24 (East US DB)
- Priority 110: Allow 1433 to 10.2.3.0/24 (Asia DB)

East US DB Subnet NSG:
Inbound:

- Priority 100: Allow 1433 from 10.0.3.0/24 (WE DB)
- Priority 110: Allow 1433 from 10.2.3.0/24 (Asia DB)

Data Transfer Costs:

Global Peering Pricing (2025):
┌────────────────┬──────────┬──────────┐
│ Region Pair    │ Ingress  │ Egress   │
├────────────────┼──────────┼──────────┤
│ WE ↔ East US  │ $0.035/GB│ $0.035/GB│
│ WE ↔ SE Asia  │ $0.035/GB│ $0.035/GB│
│ East US ↔ Asia│ $0.035/GB│ $0.035/GB│
└────────────────┴──────────┴──────────┘

Monthly Cost Estimate:
DB Replication: 100 GB/day bidirectional

- 100 GB × 30 days × 2 directions = 6,000 GB/month
- 6,000 GB × $0.035 = $210/month per region pair
- 3 region pairs × $210 = $630/month total

Optimization:

- Compress replication data
- Replicate only changed data
- Use read replicas (not full sync)
- Schedule heavy replication during off-peak

High Availability:
✓ Multi-region presence
✓ Regional failover capability
✓ Global load balancing (Traffic Manager)
✓ Data redundancy across regions

Disaster Recovery:
If West Europe region fails:

1. Traffic Manager detects failure
1. Redirects users to East US or Asia
1. VNets in other regions unaffected
1. Data available (replicated)
1. RPO: Seconds to minutes
1. RTO: Minutes to hours

```
---

## Summary: The Complete Picture

### What You Now Understand About Azure Virtual Network

**Layer 1 (Portal)**:
- Create VNets with address spaces
- Configure subnets with CIDR notation
- Attach NSGs and route tables
- Set up VNet peering
- Enable service endpoints
- Deploy private endpoints

**Layer 2 (Components)**:
- **Address Space**: RFC 1918 private IPs, CIDR planning
- **Subnets**: Network segmentation, Azure reserved IPs (.0-.3, .255)
- **NSGs**: Stateful firewall, priority-based rules, service tags
- **Route Tables**: System routes, UDRs, route priority
- **VNet Peering**: Regional and global, non-transitive by default
- **Service Endpoints**: Optimal routing to Azure services
- **Private Endpoints**: PaaS services with private IPs
- **NAT Gateway**: Scalable outbound internet

**Layer 3 (How It Works)**:
- Software-defined networking (SDN)
- Virtual network isolation via VNI
- Automatic routing between subnets
- ARP proxy (no broadcast)
- Connection tracking (stateful NSG)
- VXLAN encapsulation
- Distributed routing tables

**Layer 4 (Technology)**:
- **VFP (Virtual Filtering Platform)**: Packet processing on every host
- **SDN Controller**: Centralized policy management
- **VXLAN**: Overlay networking with 16M VNIs
- **Azure Backbone**: Microsoft's global network
- **Host Agents**: Policy enforcement on compute nodes

**Layer 5 (TCP/IP Model)**:
- **Layer 1 (Link)**: Virtual MACs, virtual switching
- **Layer 2 (Network)**: IP routing, VNet isolation, peering
- **Layer 3 (Transport)**: TCP/UDP, stateful firewall
- **Layer 4 (Application)**: DNS, standard protocols

**Layer 6 (Traffic Flows)**:
- Intra-subnet: Direct delivery
- Inter-subnet: Via Azure routing
- Inter-VNet: Via peering (encapsulated)
- To Internet: Via NAT or public IP
- From Internet: Via public IP + DNAT
- To On-Prem: Via VPN/ExpressRoute Gateway

**Layer 7 (Physical)**:
- Clos network topology (spine-leaf)
- Hyper-V hosts with VFP
- Redundancy at every layer
- Availability Zones (different datacenters)
- Azure global backbone

### Key Insights

**Azure Virtual Network IS**:
✅ Software-Defined Network (SDN)
✅ Layer 3 (IP) isolation mechanism
✅ Overlay network (VXLAN/NVGRE)
✅ Distributed routing system
✅ Policy-based networking platform
✅ Multi-tenant isolation
✅ Globally connected (via peering)
✅ Highly available by design

**Azure Virtual Network is NOT**:
❌ Physical network hardware
❌ Traditional VLAN-based
❌ Limited to single datacenter
❌ Complex to configure
❌ Expensive to scale

### When to Use Different Networking Features
```

Use Basic VNet When:
✓ Simple VM-to-VM communication
✓ Single region deployment
✓ No complex routing needed
✓ Default internet access sufficient

Use VNet Peering When:
✓ Connect multiple VNets
✓ Hub-spoke topology
✓ Multi-region architecture
✓ Share services across VNets
✓ Need low-latency connectivity

Use Service Endpoints When:
✓ Access Azure PaaS services
✓ Optimal routing desired
✓ Cost-sensitive (free feature)
✓ VNet-level restriction acceptable
✓ Same region typically

Use Private Endpoints When:
✓ Maximum security required
✓ Disable public access
✓ Cross-region PaaS access
✓ Private IP for PaaS needed
✓ Compliance mandates
✓ Worth the cost ($7.30/month)

Use NAT Gateway When:
✓ High outbound connection volume
✓ Predictable outbound IP needed
✓ Port exhaustion concerns
✓ Multiple concurrent flows
✓ Need to whitelist at destination

Use VPN Gateway When:
✓ Connect to on-premises
✓ Site-to-site VPN
✓ Point-to-site VPN (remote users)
✓ Encrypted connection required
✓ Cost-effective hybrid connectivity

Use ExpressRoute When:
✓ Dedicated private connection
✓ High bandwidth required (50Mbps-100Gbps)
✓ Predictable performance
✓ Mission-critical workloads
✓ Compliance requires private WAN

Use Azure Firewall When:
✓ Centralized security
✓ Hub-spoke topology
✓ Application-level filtering
✓ Threat intelligence
✓ Fully managed solution

Use NSG When:
✓ Subnet or NIC-level firewall
✓ Basic traffic filtering
✓ Micro-segmentation
✓ Defense in depth
✓ Free (no additional cost)

```
### Best Practices Summary

**1. Address Space Planning**:
- Use RFC 1918 ranges
- Plan for growth (use /16 or larger)
- Avoid overlaps with on-premises
- Document IP allocation
- Reserve space for future VNets

**2. Subnet Design**:
- Minimum /27 (27 usable IPs)
- /24 for most workloads (251 IPs)
- Dedicated subnets for:
  * GatewaySubnet (/27 minimum)
  * AzureBastionSubnet (/26 required)
  * AzureFirewallSubnet (/26 minimum)
- Don't subnet too small (overhead)

**3. NSG Strategy**:
- Apply at subnet level (simpler)
- Use service tags
- Deny by default, allow explicitly
- Document rules
- Use ASGs for application grouping
- Review regularly

**4. Routing**:
- Understand system routes
- Use UDRs sparingly
- Document custom routes
- Test thoroughly
- Monitor effective routes

**5. VNet Peering**:
- Use hub-spoke for scale
- Enable forwarded traffic correctly
- Plan for non-transitive nature
- Consider global peering costs
- Monitor bandwidth usage

**6. Service Integration**:
- Private Endpoints for maximum security
- Service Endpoints for cost efficiency
- Private DNS Zones for name resolution
- Consider hybrid scenarios

**7. High Availability**:
- Use Availability Zones
- Deploy across regions
- Redundant gateways
- Load balancers
- Regular backups

**8. Security**:
- Defense in depth (NSG + Firewall)
- Least privilege access
- Private endpoints for PaaS
- Disable public access where possible
- Regular security reviews
- Enable NSG flow logs

**9. Monitoring**:
- Enable diagnostic logs
- Use Network Watcher
- Monitor NSG flow logs
- Connection Monitor
- Traffic Analytics
- Regular audits

**10. Cost Optimization**:
- Right-size subnets
- Review peering data transfer
- Use service endpoints (free)
- Private endpoints where justified
- NAT Gateway vs. default SNAT
- Monitor egress costs

---

## Next Steps

You've now completed the Virtual Network deep dive!

**You've learned**:
1. **Network Security Group (NSG)** - Stateful firewall
2. **Azure Load Balancer** - Layer 4 load balancing
3. **Application Gateway** - Layer 7 load balancing
4. **Azure Virtual Network** - Foundation networking

**Suggested next topics**:
1. **Azure Firewall** - Network security appliance
2. **VPN Gateway** - Hybrid connectivity
3. **ExpressRoute** - Dedicated private connection
4. **Azure Front Door** - Global Layer 7 routing
5. **Traffic Manager** - DNS-based load balancing
6. **Azure DNS** - Domain name management
7. **Network Watcher** - Monitoring and diagnostics

**Practice Scenarios**:
1. Build hub-spoke topology
2. Configure service endpoints
3. Deploy private endpoints
4. Create custom route tables
5. Test NSG rules
6. Set up VNet peering
7. Configure NAT Gateway

---

**Created**: November 2025  
**Purpose**: Deep technical understanding for Cloud Engineer role  
**For**: Interview preparation and practical knowledge  
**Part of Series**: NSG → Load Balancer → Application Gateway → **Virtual Network**

This document follows the same 8-layer pattern to help you understand Azure Virtual Network from the Azure Portal down to the physical network implementation. Understanding Virtual Networks is absolutely fundamental—it's the foundation upon which all other Azure networking services are built!

**Key Takeaway**: Azure Virtual Network is not a physical network—it's a sophisticated software-defined overlay network that provides complete isolation, flexible routing, and global connectivity, all running on shared physical infrastructure with zero configuration of physical hardware required from you.​​​​​​​​​​​​​​​​
```

