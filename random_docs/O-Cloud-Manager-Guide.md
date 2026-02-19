# O-RAN O2IMS Implementation Guide

---

## ⚠️ Disclaimer

**This is NOT an official documentation.**

This guide has been created by **Jose Gato** through interactions with Claude AI to understand and document the [O-RAN O-Cloud Manager project](https://github.com/openshift-kni/oran-o2ims). It is intended as a personal learning resource and quick reference guide.

For official documentation, please refer to:
- **Official Repository:** https://github.com/openshift-kni/oran-o2ims
- **Official Docs:** [docs/](https://github.com/openshift-kni/oran-o2ims/tree/main/docs) directory in the repository
- **O-RAN Specifications:** https://specifications.o-ran.org/

---

## Table of Contents

- [O-RAN O2IMS Implementation Guide](#o-ran-o2ims-implementation-guide)
  - [Table of Contents](#table-of-contents)
  - [1. Introduction](#1-introduction)
  - [2. Architecture Overview](#2-architecture-overview)
    - [2.1 Dual Interface Architecture](#21-dual-interface-architecture)
    - [2.2 Why Both Kubernetes CRDs and REST APIs?](#22-why-both-kubernetes-crds-and-rest-apis)
  - [3. Main REST APIs](#3-main-rest-apis)
  - [4. Custom Resources (CRs)](#4-custom-resources-crs)
    - [4.1 Provisioning \& Orchestration (`clcm.openshift.io`)](#41-provisioning--orchestration-clcmopenshiftio)
      - [4.1.1 ProvisioningRequest (Short: `pr`)](#411-provisioningrequest-short-pr)
      - [4.1.2 ClusterTemplate (Short: `ct`)](#412-clustertemplate-short-ct)
    - [4.2 Hardware Management (`clcm.openshift.io`)](#42-hardware-management-clcmopenshiftio)
      - [4.2.1 HardwareTemplate (Short: `hwtmpl`)](#421-hardwaretemplate-short-hwtmpl)
      - [4.2.2 HardwareProfile (Short: `hwprofile`)](#422-hardwareprofile-short-hwprofile)
      - [4.2.3 HardwarePlugin (Short: `hwplugin`)](#423-hardwareplugin-short-hwplugin)
    - [4.3 Hardware Plugin Interface (`plugins.clcm.openshift.io`)](#43-hardware-plugin-interface-pluginsclcmopenshiftio)
      - [4.3.1 NodeAllocationRequest (Short: `nar`)](#431-nodeallocationrequest-short-nar)
      - [4.3.2 AllocatedNode (Short: `an`)](#432-allocatednode-short-an)
    - [4.4 Inventory \& Configuration (`ocloud.openshift.io`)](#44-inventory--configuration-ocloudopenshiftio)
      - [4.4.1 Inventory (Short: `inv`)](#441-inventory-short-inv)
    - [4.5 CR Relationships Flow](#45-cr-relationships-flow)
    - [4.6 Summary by Use Case](#46-summary-by-use-case)
  - [5. Deployment Workflow: Single Node OpenShift Example](#5-deployment-workflow-single-node-openshift-example)
    - [5.1 Scenario](#51-scenario)
    - [5.2 Prerequisites](#52-prerequisites)
    - [5.3 Step-by-Step CR Creation](#53-step-by-step-cr-creation)
      - [Step 1: Configure Inventory (One-time setup)](#step-1-configure-inventory-one-time-setup)
      - [Step 2: Create HardwarePlugin](#step-2-create-hardwareplugin)
      - [Step 3: Create HardwareProfile](#step-3-create-hardwareprofile)
      - [Step 4: Create HardwareTemplate](#step-4-create-hardwaretemplate)
      - [Step 5: Create ClusterTemplate](#step-5-create-clustertemplate)
      - [Step 6: Create ProvisioningRequest](#step-6-create-provisioningrequest)
    - [5.4 Automatic Workflow After ProvisioningRequest](#54-automatic-workflow-after-provisioningrequest)
    - [5.5 CR Relationship Diagram](#55-cr-relationship-diagram)
    - [5.6 What You Create (Summary)](#56-what-you-create-summary)
    - [5.7 Monitoring Progress](#57-monitoring-progress)
    - [5.8 Expected Timeline](#58-expected-timeline)
    - [5.9 Key Points](#59-key-points)
  - [6. Adding Bare-Metal Servers to Resource Pool](#6-adding-bare-metal-servers-to-resource-pool)
    - [6.1 Key Concepts](#61-key-concepts)
    - [6.2 Workflow Steps](#62-workflow-steps)
    - [6.3 Understanding the Labels](#63-understanding-the-labels)
    - [6.4 Verify Servers are Available](#64-verify-servers-are-available)
    - [6.5 How Server Selection Works](#65-how-server-selection-works)
    - [6.6 Summary](#66-summary)

---

## 1. Introduction

The **O-RAN O-Cloud Manager** is an implementation of the O-RAN O2 IMS (Infrastructure Management Service) API built on top of OpenShift and Advanced Cluster Management (ACM). It provides O-RAN compliant infrastructure management and orchestration for telco cloud environments, managing the complete lifecycle of cluster provisioning from hardware to configuration.

**Key Features:**
- O-RAN O2 IMS compliant REST APIs
- Hardware management integration (Metal3)
- Cluster provisioning orchestration
- Inventory and alarm management
- Template-based deployment workflows
- 100% O-RAN SC O2 IMS Automated Test Compliance

---

## 2. Architecture Overview

### 2.1 Dual Interface Architecture

The project provides **both** Kubernetes-native interfaces and external REST APIs to serve different consumers:

**1. Internal Kubernetes Interface (CRDs + Controllers)**
- For **internal** OpenShift/ACM workflows
- Custom Resources like:
  - `ProvisioningRequest`
  - `ClusterTemplate`
  - `HardwarePlugin`
  - `Inventory`
- Kubernetes controllers that watch and reconcile these CRs

**2. External REST APIs (O2IMS Compliant)**
- For **external** consumers (SMO systems, orchestrators, external management tools)
- O-RAN standardized interfaces
- OAuth2 authenticated endpoints

### 2.2 Why Both Kubernetes CRDs and REST APIs?

```
┌─────────────────────────────────────────────────────────────┐
│                  SMO (External System)                      │
│         Service Management & Orchestration                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ O2IMS REST APIs (OAuth2)
                      │ https://o2ims.apps.cluster/o2ims-infrastructure*
                      │
┌─────────────────────▼───────────────────────────────────────┐
│              OpenShift/Kubernetes Cluster                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  O-Cloud Manager (oran-o2ims namespace)                │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  REST API Servers (External Interface)           │  │ │
│  │  │  - resource-server                               │  │ │
│  │  │  - provisioning-server                           │  │ │
│  │  │  - alarms-server                                 │  │ │
│  │  │  - cluster-server                                │  │ │
│  │  │  - artifacts-server                              │  │ │
│  │  │  (Exposed via OpenShift Routes)                  │  │ │
│  │  └──────────────────┬───────────────────────────────┘  │ │
│  │                     │                                   │ │
│  │  ┌──────────────────▼───────────────────────────────┐  │ │
│  │  │  Controllers (Internal Kubernetes Logic)         │  │ │
│  │  │  - ProvisioningRequest Controller                │  │ │
│  │  │  - ClusterTemplate Controller                    │  │ │
│  │  │  - HardwarePlugin Controller                     │  │ │
│  │  │  - Inventory Controller                          │  │ │
│  │  └──────────────────┬───────────────────────────────┘  │ │
│  │                     │                                   │ │
│  │  ┌──────────────────▼───────────────────────────────┐  │ │
│  │  │  Kubernetes CRs (Custom Resources)               │  │ │
│  │  │  - ProvisioningRequest                           │  │ │
│  │  │  - ClusterTemplate                               │  │ │
│  │  │  - HardwarePlugin                                │  │ │
│  │  └──────────────────┬───────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│           │                                                  │
│  ┌────────▼──────────────────────────────────────────────┐  │
│  │  OpenShift/ACM Infrastructure                         │  │
│  │  - Metal3 (bare metal provisioning)                   │  │
│  │  - SiteConfig (cluster installation)                  │  │
│  │  - ACM Policy Engine (configuration)                  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Why This Design?**

1. **O-RAN Standard Compliance:** O-RAN specifications require REST APIs (O2IMS interface)
2. **Separation of Concerns:** External systems use REST, internal orchestration uses Kubernetes
3. **Security Boundary:** External systems get OAuth2-authenticated REST access without direct Kubernetes API access
4. **Interoperability:** External telco management systems (SMO) can interact using O-RAN standard interfaces

---

## 3. Main REST APIs

The O-RAN O2IMS implementation provides **5 main REST APIs** for external SMO systems. These are compliant with the O-RAN O2IMS Interface Specification.

> **Note:** These REST APIs are primarily for external Service Management and Orchestration (SMO) systems. For internal cluster provisioning workflows, you'll use the Kubernetes Custom Resources (CRs) documented in Section 4.

| **API** | **Endpoint** | **Purpose** | **Main Operations** |
|---------|--------------|-------------|---------------------|
| **Infrastructure Inventory** | `/o2ims-infrastructureInventory` | Resource and infrastructure inventory management | List deployment managers, query resource types/pools/resources, manage subscriptions |
| **Infrastructure Provisioning** | `/o2ims-infrastructureProvisioning` | Cluster provisioning lifecycle management | Create/update/delete provisioning requests, monitor status |
| **Infrastructure Monitoring** | `/o2ims-infrastructureMonitoring` | Alarm and event management | Query/acknowledge alarms, configure alarm service, subscribe to notifications |
| **Infrastructure Cluster** | `/o2ims-infrastructureCluster` | Kubernetes cluster resource management | Query clusters and cluster resources, get cluster types, manage subscriptions |
| **Infrastructure Artifacts** | `/o2ims-infrastructureArtifacts` | Infrastructure template management | List templates, get template details and defaults |

**Common Features:**
- OAuth2 authentication with role-based access (admin/reader roles)
- Filtering, pagination, and sorting on collections
- Event subscriptions with callback mechanisms

**Base URL Pattern:**
```
https://<o2ims-host>/o2ims-infrastructure{Service}/v1/...
```

For detailed REST API usage, refer to the [official O-RAN O2IMS specification](https://specifications.o-ran.org/).

---

## 4. Custom Resources (CRs)

The O-RAN O2IMS implementation defines **8 main Custom Resources** organized by functional area.

### 4.1 Provisioning & Orchestration (`clcm.openshift.io`)

#### 4.1.1 ProvisioningRequest (Short: `pr`)

**Purpose:** The main orchestration CR for end-to-end cluster provisioning lifecycle

**Key Fields:**
- `templateName` / `templateVersion` - References which ClusterTemplate to use
- `templateParameters` - Input data conforming to template's OpenAPI schema
- `extensions` - Additional configuration key-value pairs

**What it does:**
- Triggers the complete provisioning workflow: hardware → cluster install → configuration
- Tracks status through multiple phases: pending → progressing → fulfilled/failed
- Orchestrates NodeAllocationRequest (hardware), ClusterInstance (install), and ACM Policies (config)
- Stores cluster details, ZTP status, policy compliance

**Location:** `api/provisioning/v1alpha1/provisioningrequest_types.go:17`

**Example:**
```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: ProvisioningRequest
metadata:
  name: my-cluster-provision
  namespace: oran-o2ims
spec:
  name: "Edge Cluster Site 01"
  templateName: sno-template
  templateVersion: v1.0.0
  templateParameters:
    clusterName: edge-cluster-01
    baseDomain: example.com
    # ... more parameters
```

---

#### 4.1.2 ClusterTemplate (Short: `ct`)

**Purpose:** Defines reusable cluster deployment templates with versioning

**Key Fields:**
- `name` / `version` / `release` - Template identity (metadata.name must be `<name>.<version>`)
- `templates` - References to HardwareTemplate, ClusterInstanceDefaults, PolicyTemplateDefaults
- `templateParameterSchema` - OpenAPI v3 schema defining required input parameters

**What it does:**
- Acts as a "blueprint" for cluster provisioning
- Defines what hardware, cluster config, and policies to apply
- Enforces parameter validation via OpenAPI schema
- Immutable once validated (spec changes blocked)

**Location:** `api/provisioning/v1alpha1/clustertemplate_types.go:17`

**Example:**
```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: ClusterTemplate
metadata:
  name: sno-template.v1.0.0  # Must be: <spec.name>.<spec.version>
  namespace: oran-o2ims
spec:
  name: sno-template
  version: v1.0.0
  release: "4.20"
  templates:
    hwTemplate: sno-hardware-template
    clusterInstanceDefaults: sno-cluster-defaults
    policyTemplateDefaults: sno-policy-defaults
  templateParameterSchema:
    type: object
    required:
      - clusterName
      - baseDomain
    properties:
      clusterName:
        type: string
      baseDomain:
        type: string
```

---

### 4.2 Hardware Management (`clcm.openshift.io`)

#### 4.2.1 HardwareTemplate (Short: `hwtmpl`)

**Purpose:** Defines hardware requirements and node group specifications

**Key Fields:**
- `hardwarePluginRef` - Which HardwarePlugin to use
- `nodeGroupData[]` - Array of node groups (master/worker) with:
  - `name`, `role` (master/worker), `hwProfile`
  - `resourcePoolId`, `resourceSelector`
- `hardwareProvisioningTimeout` - Timeout for hardware provisioning

**What it does:**
- Specifies how many nodes needed and their hardware profiles
- Maps to node groups (e.g., 3 masters with profile "dell-r750", 2 workers with "dell-r650")
- Used by ProvisioningRequest to create NodeAllocationRequest

**Location:** `api/hardwaremanagement/v1alpha1/hardwaretemplate_types.go:35`

**Example:**
```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwareTemplate
metadata:
  name: sno-hardware-template
  namespace: oran-o2ims
spec:
  hardwarePluginRef: metal3-plugin
  hardwareProvisioningTimeout: 90m
  nodeGroupData:
    - name: master
      role: master
      hwProfile: hpe-proliant-e910-profile
      resourceSelector:
        site: edge-site-01
```

---

#### 4.2.2 HardwareProfile (Short: `hwprofile`)

**Purpose:** Defines hardware configuration settings (BIOS, firmware)

**Key Fields:**
- `bios.attributes` - BIOS settings key-value pairs
- `biosFirmware` - BIOS firmware version and URL
- `bmcFirmware` - BMC firmware version and URL
- `nicFirmware[]` - NIC firmware versions and URLs

**What it does:**
- Specifies desired hardware settings for servers
- Applied to BaremetalHosts through Metal3 integration
- Ensures consistent hardware configuration across nodes

**Location:** `api/hardwaremanagement/v1alpha1/hardwareprofile_types.go:35`

**Example:**
```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwareProfile
metadata:
  name: hpe-proliant-e910-profile
  namespace: oran-o2ims
spec:
  bios:
    attributes:
      WorkloadProfile: Virtualization
      ProcVirtualization: Enabled
      ProcHyperthreading: Enabled
  biosFirmware:
    version: "U32 v2.80"
    url: "http://firmware-server/hpe-e910-bios-2.80.bin"
  bmcFirmware:
    version: "iLO 6 v1.50"
    url: "http://firmware-server/hpe-ilo6-1.50.bin"
```

---

#### 4.2.3 HardwarePlugin (Short: `hwplugin`)

**Purpose:** Registers and configures external hardware management plugins

**Key Fields:**
- `apiRoot` - Base URL of the hardware plugin server
- `caBundleName` - Custom CA certificates for mTLS
- `authClientConfig` - OAuth2 config for plugin authentication

**What it does:**
- Pluggable architecture for different hardware backends (Metal3, Dell, others)
- Provides API endpoint for hardware inventory and provisioning
- Abstracts hardware-specific operations behind standardized interface

**Location:** `api/hardwaremanagement/v1alpha1/hardwareplugin_types.go:33`

**Example:**
```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwarePlugin
metadata:
  name: metal3-plugin
  namespace: oran-o2ims
spec:
  apiRoot: http://metal3-hardwareplugin-server.oran-o2ims.svc.cluster.local:8000
  # Optional: caBundleName, authClientConfig
```

---

### 4.3 Hardware Plugin Interface (`plugins.clcm.openshift.io`)

#### 4.3.1 NodeAllocationRequest (Short: `nar`)

**Purpose:** Requests allocation of specific hardware nodes from hardware plugin

**Key Fields:**
- `clusterId` - ID of the cluster being provisioned
- `locationSpec` - Geographic location (site)
- `hardwarePluginRef` - Which plugin to use
- `nodeGroup[]` - Requested node groups with size and hardware profile
- `callback` - Webhook URL for status notifications
- `configTransactionId` - Tracks configuration changes

**What it does:**
- Created by ProvisioningRequest controller
- Sent to HardwarePlugin to allocate physical servers
- Tracks hardware provisioning state machine
- Results in AllocatedNode CRs when servers are allocated

**Location:** `api/hardwaremanagement/plugins/v1alpha1/node_allocation_requests.go:51`

**Note:** This CR is automatically created by the operator - users don't create it manually.

---

#### 4.3.2 AllocatedNode (Short: `an`)

**Purpose:** Represents an individual allocated hardware node

**Key Fields:**
- Node identity, BMC details, network interfaces
- Reference to source NodeAllocationRequest
- Allocation status

**What it does:**
- Created by HardwarePlugin when a server is allocated
- Maps to a BaremetalHost in Metal3
- Used by cluster installation to know which hardware is available

**Location:** `api/hardwaremanagement/plugins/v1alpha1/allocated_nodes.go`

**Note:** This CR is automatically created by the hardware plugin - users don't create it manually.

---

### 4.4 Inventory & Configuration (`ocloud.openshift.io`)

#### 4.4.1 Inventory (Short: `inv`)

**Purpose:** Global configuration for the O-Cloud Manager instance

**Key Fields:**
- `smoConfig` - SMO registration endpoint and OAuth config
- `ingressConfig` - Custom FQDN and TLS for API endpoints
- `resourceServerConfig` - Resource server settings
- `alarmServerConfig` - Alarm server settings
- `clusterServerConfig` - Cluster server settings
- `artifactsServerConfig` - Artifacts server settings
- `provisioningServerConfig` - Provisioning server settings

**What it does:**
- Singleton CR (typically one per cluster)
- Configures all O2IMS REST API servers
- Manages SMO registration and OAuth2 settings
- Controls ingress routes for external API access

**Location:** `api/inventory/v1alpha1/inventory_types.go:18`

**Example:**
```yaml
apiVersion: ocloud.openshift.io/v1alpha1
kind: Inventory
metadata:
  name: o2ims-inventory
  namespace: oran-o2ims
spec:
  smoConfig:
    url: https://smo.example.com
    registrationEndpoint: /o2ims-registration
    oauth:
      url: https://keycloak.example.com/realms/oran
      tokenEndpoint: /protocol/openid-connect/token
      clientSecretName: smo-oauth-client
      scopes:
        - openid
        - roles
  resourceServerConfig: {}
  alarmServerConfig: {}
  clusterServerConfig: {}
  artifactsServerConfig: {}
  provisioningServerConfig: {}
```

---

### 4.5 CR Relationships Flow

```
Inventory (Global Config)
    │
    ├── Configures REST API Servers
    └── Registers with SMO

ClusterTemplate                HardwareTemplate       HardwareProfile
      │                              │                      │
      └──────────────┬───────────────┘                      │
                     │                                      │
              ProvisioningRequest ◄────(references)─────────┘
                     │
                     ├─► NodeAllocationRequest ──► HardwarePlugin ──► Metal3
                     │                                    │
                     │                                    ▼
                     │                              AllocatedNode
                     │
                     ├─► ClusterInstance (SiteConfig) ──► Cluster Install
                     │
                     └─► ACM Policies ──► Day-2 Configuration
```

---

### 4.6 Summary by Use Case

| **To...** | **Use CR** |
|-----------|------------|
| Configure O-Cloud instance & SMO connection | `Inventory` |
| Define cluster blueprint | `ClusterTemplate` |
| Provision a new cluster | `ProvisioningRequest` |
| Specify hardware requirements | `HardwareTemplate` |
| Configure BIOS/firmware settings | `HardwareProfile` |
| Integrate hardware backend | `HardwarePlugin` |
| Request physical servers | `NodeAllocationRequest` (auto-created) |
| Track allocated servers | `AllocatedNode` (auto-created) |

---

## 5. Deployment Workflow: Single Node OpenShift Example

This section provides a detailed, step-by-step workflow for deploying a Single Node OpenShift (SNO) cluster.

### 5.1 Scenario

**Hardware:**
- 3 HPE ProLiant e910 bare-metal servers

**Goal:**
- Deploy OpenShift 4.20 as a Single Node cluster
- No Day-2 policies or configurations
- One server will be used for SNO, other two remain available

**Prerequisites:**
- OpenShift hub cluster with ACM installed
- O-Cloud Manager operator deployed
- Metal3 Baremetal Operator running
- 3 HPE ProLiant e910 servers registered as `BareMetalHost` CRs

---

### 5.2 Prerequisites

Before starting, ensure these components are already deployed on your hub cluster:

✅ **OpenShift with ACM** (Advanced Cluster Management)
✅ **O-Cloud Manager operator** (oran-o2ims)
✅ **Metal3 Baremetal Operator**
✅ **Your 3 HPE ProLiant e910 servers** registered as `BareMetalHost` CRs in Metal3

---

### 5.3 Step-by-Step CR Creation

#### Step 1: Configure Inventory (One-time setup)

**Purpose:** Global O-Cloud configuration

```yaml
apiVersion: ocloud.openshift.io/v1alpha1
kind: Inventory
metadata:
  name: o2ims-inventory
  namespace: oran-o2ims
spec:
  # Optional: SMO configuration (skip for now)
  # smoConfig: ...

  # Optional: Custom ingress settings
  # ingressConfig: ...

  # Server configs - usually defaults are fine
  resourceServerConfig: {}
  alarmServerConfig: {}
  clusterServerConfig: {}
  artifactsServerConfig: {}
  provisioningServerConfig: {}
```

**When:** Once per hub cluster (likely already exists)
**Created by:** Cluster administrator
**Status:** ✅ Skip if already exists

---

#### Step 2: Create HardwarePlugin

**Purpose:** Register Metal3 as the hardware management backend

```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwarePlugin
metadata:
  name: metal3-plugin
  namespace: oran-o2ims
spec:
  # URL where Metal3 plugin API is running
  apiRoot: http://metal3-hardwareplugin-server.oran-o2ims.svc.cluster.local:8000

  # Optional: If using custom CA certificates
  # caBundleName: metal3-ca-bundle

  # Optional: OAuth2 if plugin requires authentication
  # authClientConfig: ...
```

**When:** Once per hardware backend type
**What happens:** O-Cloud Manager can now communicate with Metal3 for hardware operations
**Status:** ✅ Often pre-installed with the operator

---

#### Step 3: Create HardwareProfile

**Purpose:** Define BIOS and firmware settings for HPE ProLiant e910

```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwareProfile
metadata:
  name: hpe-proliant-e910-profile
  namespace: oran-o2ims
spec:
  # BIOS settings specific to HPE ProLiant e910
  bios:
    attributes:
      # Example BIOS settings (adjust for your needs)
      WorkloadProfile: Virtualization
      ProcVirtualization: Enabled
      ProcHyperthreading: Enabled
      ThermalConfig: OptimalCooling

  # Optional: BIOS firmware version
  biosFirmware:
    version: "U32 v2.80"
    url: "http://firmware-server/hpe-e910-bios-2.80.bin"

  # Optional: BMC firmware version
  bmcFirmware:
    version: "iLO 6 v1.50"
    url: "http://firmware-server/hpe-ilo6-1.50.bin"

  # Optional: NIC firmware
  # nicFirmware:
  #   - version: "22.5.13"
  #     url: "http://firmware-server/intel-e810-22.5.13.bin"
```

**When:** Once per server hardware model
**What it controls:**
- BIOS settings applied to servers
- Firmware versions to flash
- Applied when servers are allocated

---

#### Step 4: Create HardwareTemplate

**Purpose:** Specify hardware requirements for SNO (1 master node)

```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: HardwareTemplate
metadata:
  name: sno-hpe-e910-hwtemplate
  namespace: oran-o2ims
spec:
  # Which hardware plugin to use
  hardwarePluginRef: metal3-plugin

  # Timeout for hardware provisioning (optional)
  hardwareProvisioningTimeout: 90m

  # Node groups - For SNO, we need 1 master node
  nodeGroupData:
    - name: master
      role: master  # master role (in SNO this is also the worker)
      hwProfile: hpe-proliant-e910-profile  # References HardwareProfile above

      # Optional: Specify which resource pool
      # resourcePoolId: "datacenter-1-pool"

      # Optional: Label selectors to pick specific servers
      resourceSelector:
        site: edge-site-01
        rack: rack-A
        # Metal3 will pick 1 server matching these labels
```

**Key points for SNO:**
- Only ONE node group with role `master`
- SNO = Single Node = master + worker combined
- Metal3 will allocate 1 HPE e910 server from your 3 available

---

#### Step 5: Create ClusterTemplate

**Purpose:** Complete cluster blueprint combining hardware + installation config

**5a. Create ClusterInstance Defaults ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sno-cluster-defaults-v1
  namespace: oran-o2ims
data:
  clusterinstance-defaults.yaml: |
    clusterImageSetNameRef: openshift-4.20  # OpenShift version
    pullSecretRef:
      name: pull-secret  # Must exist in target namespace
    templateRefs:
      - name: ai-cluster-templates-v1
        namespace: siteconfig-operator
    nodes:
      - hostname: "node1"
        templateRefs:
          - name: ai-node-templates-v1
            namespace: siteconfig-operator
        # Node-specific config
        ironicInspect: "enabled"
```

**5b. Create Policy Defaults ConfigMap** (Empty for this example):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sno-policy-defaults-v1
  namespace: oran-o2ims
data:
  policy-defaults.yaml: |
    # Empty - no Day-2 policies for this example
    {}
```

**5c. Create ClusterTemplate CR:**

```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: ClusterTemplate
metadata:
  # Name MUST be: <spec.name>.<spec.version>
  name: sno-hpe-e910.v1.0.0
  namespace: oran-o2ims
spec:
  name: sno-hpe-e910
  description: "Single Node OpenShift template for HPE ProLiant e910"
  version: v1.0.0
  release: "4.20"  # OpenShift version

  # Optional metadata
  characteristics:
    profile: single-node
    vendor: hpe
    model: proliant-e910

  # References to templates
  templates:
    hwTemplate: sno-hpe-e910-hwtemplate  # From Step 4
    clusterInstanceDefaults: sno-cluster-defaults-v1
    policyTemplateDefaults: sno-policy-defaults-v1
    # upgradeDefaults: upgrade-defaults-v1  # Optional

  # OpenAPI v3 schema - defines what parameters are required
  templateParameterSchema:
    type: object
    required:
      - clusterName
      - baseDomain
      - nodeHostname
      - machineCIDR
      - apiVIP
      - ingressVIP
    properties:
      # Cluster-level parameters
      clusterName:
        type: string
        description: "Name of the SNO cluster"

      baseDomain:
        type: string
        description: "Base domain (e.g., example.com)"

      nodeHostname:
        type: string
        description: "Hostname for the single node"

      # Network parameters
      machineCIDR:
        type: string
        description: "Machine network CIDR (e.g., 192.168.1.0/24)"

      apiVIP:
        type: string
        description: "API VIP address (e.g., 192.168.1.100)"

      ingressVIP:
        type: string
        description: "Ingress VIP address (e.g., 192.168.1.101)"

      # Optional parameters
      ntpServers:
        type: array
        items:
          type: string
        description: "NTP server addresses"

      sshPublicKey:
        type: string
        description: "SSH public key for node access"
```

**What this defines:**
- Blueprint combining hardware + installation config
- OpenAPI schema validates ProvisioningRequest parameters
- References all the previous CRs (HardwareTemplate, ConfigMaps)

---

#### Step 6: Create ProvisioningRequest

**Purpose:** 🚀 Actually trigger the cluster provisioning

```yaml
apiVersion: clcm.openshift.io/v1alpha1
kind: ProvisioningRequest
metadata:
  name: my-sno-cluster-provision
  namespace: oran-o2ims
spec:
  name: "SNO Cluster at Edge Site 01"
  description: "Single Node OpenShift for edge site"

  # Reference the ClusterTemplate
  templateName: sno-hpe-e910
  templateVersion: v1.0.0

  # Provide actual values matching the templateParameterSchema
  templateParameters:
    # These must match the schema from ClusterTemplate
    clusterName: sno-edge-01
    baseDomain: edge.example.com
    nodeHostname: sno-node-01.edge.example.com

    # Network configuration
    machineCIDR: 192.168.100.0/24
    apiVIP: 192.168.100.10
    ingressVIP: 192.168.100.11

    # Optional
    ntpServers:
      - time.google.com
      - pool.ntp.org

    sshPublicKey: "ssh-rsa AAAAB3NzaC1yc2E... admin@example.com"

  # Optional: Additional extensions
  # extensions: {}
```

**This triggers the magic! 🎉**

---

### 5.4 Automatic Workflow After ProvisioningRequest

Once the `ProvisioningRequest` is created, the following workflow happens **automatically**:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. ProvisioningRequest Controller (Validation)                 │
│    - Validates templateParameters against schema               │
│    - Checks ClusterTemplate exists                             │
│    - Checks HardwareTemplate exists                            │
│    - Status: pending → progressing                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Create NodeAllocationRequest (Hardware Provisioning)        │
│    apiVersion: plugins.clcm.openshift.io/v1alpha1              │
│    kind: NodeAllocationRequest                                 │
│    spec:                                                       │
│      clusterId: my-sno-cluster-provision                       │
│      hardwarePluginRef: metal3-plugin                          │
│      nodeGroup:                                                │
│        - name: master                                          │
│          size: 1                                               │
│          hwProfile: hpe-proliant-e910-profile                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Metal3 Plugin (Hardware Allocation)                         │
│    - Searches available BareMetalHosts                         │
│    - Finds 1 HPE e910 matching labels                          │
│    - Applies HardwareProfile (BIOS settings)                   │
│    - Creates AllocatedNode CR                                  │
│    - Powers on server, provisions OS image                     │
│    - Status: Hardware provisioning (5-15 mins)                 │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Create AllocatedNode (Hardware Ready)                       │
│    apiVersion: plugins.clcm.openshift.io/v1alpha1              │
│    kind: AllocatedNode                                         │
│    metadata:                                                   │
│      name: allocated-node-xyz123                               │
│    spec:                                                       │
│      bmcAddress: 192.168.1.50                                  │
│      bmcCredentials: ...                                       │
│      bootMACAddress: aa:bb:cc:dd:ee:ff                         │
│      nodePoolRef: my-sno-cluster-provision                     │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Create ClusterInstance (SiteConfig Operator)                │
│    apiVersion: siteconfig.open-cluster-management.io/v1alpha1  │
│    kind: ClusterInstance                                       │
│    spec:                                                       │
│      clusterName: sno-edge-01                                  │
│      baseDomain: edge.example.com                              │
│      nodes:                                                    │
│        - hostname: sno-node-01.edge.example.com                │
│          bmcAddress: 192.168.1.50                              │
│          bootMACAddress: aa:bb:cc:dd:ee:ff                     │
│          role: master                                          │
│    - Merges defaults from ConfigMap                            │
│    - Merges user parameters from ProvisioningRequest           │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. SiteConfig Operator (Cluster Installation)                  │
│    - Creates AgentClusterInstall                               │
│    - Creates InfraEnv                                          │
│    - Boots server with discovery ISO                           │
│    - Agent registers and starts installation                   │
│    - Installs OpenShift 4.20                                   │
│    - Status: Installing cluster (30-60 mins)                   │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Cluster Installation Complete                               │
│    - Cluster is running and healthy                            │
│    - ManagedCluster CR created in ACM                          │
│    - ProvisioningRequest status: fulfilled                     │
│    - You can access: https://api.sno-edge-01.edge.example.com  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 5.5 CR Relationship Diagram

```
┌────────────────────┐
│   Inventory        │  (Global config - already exists)
│ (ocloud.openshift) │
└────────────────────┘

┌────────────────────┐
│ HardwarePlugin     │──┐
│ metal3-plugin      │  │
└────────────────────┘  │
                        │
┌────────────────────┐  │
│ HardwareProfile    │  │
│ hpe-e910-profile   │  │
└──────────┬─────────┘  │
           │            │
           └────────┐   │
                    ▼   ▼
          ┌───────────────────────┐
          │ HardwareTemplate      │
          │ sno-hpe-e910-hwtempl  │
          │  - 1 master node      │
          │  - profile: e910      │
          └──────────┬────────────┘
                     │
          ┌──────────┴────────────┐
          │                       │
┌─────────▼─────────┐   ┌─────────▼─────────┐
│ ConfigMap:        │   │ ConfigMap:        │
│ cluster-defaults  │   │ policy-defaults   │
└─────────┬─────────┘   └─────────┬─────────┘
          │                       │
          └──────────┬────────────┘
                     ▼
          ┌──────────────────────┐
          │ ClusterTemplate      │
          │ sno-hpe-e910.v1.0.0  │
          │  - hwTemplate ─────► │
          │  - defaults ────────►│
          │  - param schema      │
          └──────────┬───────────┘
                     │
                     │ (User creates)
                     ▼
          ┌──────────────────────────┐
          │ ProvisioningRequest      │
          │ my-sno-cluster-provision │
          │  - template: sno-hpe...  │
          │  - params: clusterName,  │
          │           network, etc.  │
          └──────────┬───────────────┘
                     │
          (Operator creates automatically)
                     │
      ┌──────────────┴──────────────┐
      ▼                             ▼
┌─────────────────────┐   ┌──────────────────┐
│NodeAllocationRequest│   │ ClusterInstance  │
│  - requests 1 node  │   │  (SiteConfig)    │
└─────────┬───────────┘   └──────────────────┘
          │
          ▼
┌─────────────────────┐
│ AllocatedNode       │
│  - BMC: 192.168.1.50│
│  - MAC: aa:bb:cc... │
└─────────────────────┘
```

---

### 5.6 What You Create (Summary)

| **Step** | **CR** | **Who Creates** | **Purpose** |
|----------|--------|-----------------|-------------|
| 0 | `Inventory` | Admin (once) | Global O-Cloud config |
| 1 | `HardwarePlugin` | Admin (once per backend) | Register Metal3 |
| 2 | `HardwareProfile` | Admin (once per server model) | BIOS/firmware for HPE e910 |
| 3 | `HardwareTemplate` | Template designer | Define 1 master node needed |
| 4 | `ConfigMaps` (2x) | Template designer | Installation defaults |
| 5 | `ClusterTemplate` | Template designer | Complete blueprint |
| **6** | **`ProvisioningRequest`** | **End user** | **🚀 Trigger provisioning** |

**What gets auto-created:**
- `NodeAllocationRequest` → requests hardware
- `AllocatedNode` → hardware allocated
- `ClusterInstance` → cluster installation starts

---

### 5.7 Monitoring Progress

```bash
# Watch ProvisioningRequest status
oc get provisioningrequest my-sno-cluster-provision -n oran-o2ims -w

# Check detailed status
oc describe provisioningrequest my-sno-cluster-provision -n oran-o2ims

# Status phases you'll see:
# - pending: Validation in progress
# - progressing: Hardware provisioning
# - progressing: Cluster installing
# - fulfilled: ✅ Cluster ready!
# - failed: ❌ Check .status.conditions for errors
```

**Example status output:**

```yaml
status:
  conditions:
    - type: ProvisioningRequestValidated
      status: "True"
      reason: Completed
    - type: HardwareProvisioned
      status: "True"
      reason: Completed
    - type: ClusterProvisioned
      status: "True"
      reason: Completed
  provisioningPhase: fulfilled
  extensions:
    clusterDetails:
      name: sno-edge-01
      ztpStatus: ZTP Done
    allocatedNodeHostMap:
      allocated-node-xyz123: sno-node-01.edge.example.com
```

---

### 5.8 Expected Timeline

1. **Hardware provisioning:** 5-15 minutes
   - Server selection, BIOS config, power on, network boot
2. **Cluster installation:** 30-60 minutes
   - Discovery ISO boot, agent registration, OpenShift install
3. **Total:** ~45-75 minutes for complete SNO deployment

---

### 5.9 Key Points

- **Templates are reusable:** Create ClusterTemplate once, use many times for different clusters
- **ProvisioningRequest is the trigger:** This is what you create for each new cluster deployment
- **Only 1 server used:** SNO needs 1 master (other 2 HPE e910s stay available for future deployments)
- **No Day-2 policies:** We left policyTemplateDefaults empty as requested
- **Fully automated:** After creating ProvisioningRequest, everything happens automatically
- **Declarative:** You describe desired state, operators handle the implementation
- **Status tracking:** All status updates reflected in ProvisioningRequest CR

---

## 6. Adding Bare-Metal Servers to Resource Pool

This section describes how to register bare-metal servers with Metal3 so they become available in O2IMS resource pools for cluster provisioning.

### 6.1 Key Concepts

- **Resource Pool** = A Kubernetes namespace containing BareMetalHost CRs
- **Site** = Logical grouping of servers (e.g., datacenter, edge location)
- **Labels** = Used by O2IMS for resource discovery and server selection
- Servers are discovered automatically by O2IMS when BareMetalHosts are created

### 6.2 Workflow Steps

#### Step 1: Create Resource Pool Namespace

Each resource pool is a separate namespace:

```bash
# Create namespace for your resource pool
export POOL_NAME="hpe-proliant-pool"  # Choose your pool name

oc create namespace ${POOL_NAME}
```

**Naming convention:**
- Use descriptive names: `dell-r740-pool`, `hpe-proliant-pool`, `edge-site-01-pool`
- Namespace name = Resource Pool ID

---

#### Step 2: Create BMC Credentials Secrets

For each server, create a secret with BMC credentials:

```bash
# Create BMC credentials for each server
oc create secret generic bmc-secret-server-01 \
  -n ${POOL_NAME} \
  --from-literal=username=admin \
  --from-literal=password='YourBMCPassword'

oc create secret generic bmc-secret-server-02 \
  -n ${POOL_NAME} \
  --from-literal=username=admin \
  --from-literal=password='YourBMCPassword'

oc create secret generic bmc-secret-server-03 \
  -n ${POOL_NAME} \
  --from-literal=username=admin \
  --from-literal=password='YourBMCPassword'
```

**Alternative (YAML):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bmc-secret-server-01
  namespace: hpe-proliant-pool
type: Opaque
data:
  username: YWRtaW4=           # base64 encoded "admin"
  password: WW91clBhc3N3b3Jk   # base64 encoded password
```

---

#### Step 3: Create Network Data Secrets (Optional)

Define static network configuration for each server if not using DHCP:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: network-data-server-01
  namespace: hpe-proliant-pool
type: Opaque
stringData:
  nmstate: |
    interfaces:
      - name: eno1
        type: ethernet
        state: up
        ipv4:
          enabled: true
          dhcp: true
        ipv6:
          enabled: false
    dns-resolver:
      config:
        server:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
oc apply -f network-data-server-01.yaml
```

---

#### Step 4: Create BareMetalHost CRs

This is the key step! Create BareMetalHost with O2IMS-specific labels:

**Example: HPE ProLiant e910 Server**

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: hpe-e910-server-01
  namespace: hpe-proliant-pool

  # O2IMS REQUIRED LABELS
  labels:
    # Site location (e.g., datacenter, edge site)
    resources.clcm.openshift.io/siteId: edge-site-01

    # Resource pool ID (MUST match namespace name)
    resources.clcm.openshift.io/resourcePoolId: hpe-proliant-pool

    # SELECTABLE LABELS - Used in HardwareTemplate resourceSelector
    resourceselector.clcm.openshift.io/server-model: e910
    resourceselector.clcm.openshift.io/server-vendor: hpe
    resourceselector.clcm.openshift.io/server-id: hpe-e910-server-01
    resourceselector.clcm.openshift.io/rack: rack-A
    resourceselector.clcm.openshift.io/subnet: "192.168.100.0"

    # INTERFACE LABELS - Network interface names
    interfacelabel.clcm.openshift.io/boot-interface: eno1
    interfacelabel.clcm.openshift.io/data-interface: eno2

  # ANNOTATIONS - Additional metadata (optional)
  annotations:
    bmac.agent-install.openshift.io/allow-provisioned-host-management: ""
    resourceinfo.clcm.openshift.io/description: "HPE ProLiant e910 at edge site 01"
    resourceinfo.clcm.openshift.io/partNumber: "P12345"
    resourceinfo.clcm.openshift.io/serialNumber: "SN123456"
    resourceinfo.clcm.openshift.io/groups: "production, edge"

spec:
  # Server is powered off initially
  online: false

  # BMC connection details
  bmc:
    # BMC address - format depends on vendor (HPE iLO example below)
    address: redfish-virtualmedia+https://192.168.1.10/redfish/v1/Systems/1
    credentialsName: bmc-secret-server-01
    disableCertificateVerification: true

  # Boot MAC address (PXE boot interface)
  bootMACAddress: "aa:bb:cc:dd:ee:01"

  # Network configuration secret (optional)
  preprovisioningNetworkDataName: network-data-server-01
```

**Apply the BareMetalHost:**
```bash
oc apply -f hpe-e910-server-01.yaml
```

Repeat this step for each server you want to add to the pool.

---

### 6.3 Understanding the Labels

#### Required Labels (O2IMS)

| Label | Purpose | Example Value | Notes |
|-------|---------|---------------|-------|
| `resources.clcm.openshift.io/siteId` | Geographic site identifier | `edge-site-01` | Used for multi-site deployments |
| `resources.clcm.openshift.io/resourcePoolId` | Pool membership | `hpe-proliant-pool` | **Must match namespace** |

#### Selector Labels (HardwareTemplate matching)

These labels are used in `HardwareTemplate.spec.nodeGroupData[].resourceSelector`:

| Label Prefix | Purpose | Example |
|--------------|---------|---------|
| `resourceselector.clcm.openshift.io/*` | Custom selectable attributes | `server-model: e910`<br>`rack: rack-A`<br>`subnet: "192.168.100.0"` |

**You can define any custom selector labels** - just prefix them with `resourceselector.clcm.openshift.io/`

#### Interface Labels

| Label | Purpose | Example |
|-------|---------|---------|
| `interfacelabel.clcm.openshift.io/boot-interface` | PXE boot NIC name | `eno1` |
| `interfacelabel.clcm.openshift.io/data-interface` | Data NIC name | `eno2` |

---

### 6.4 Verify Servers are Available

```bash
# List all BareMetalHosts in pool
oc get baremetalhosts -n hpe-proliant-pool

# Check specific server details
oc describe baremetalhost hpe-e910-server-01 -n hpe-proliant-pool
```

**Expected Status:**
```yaml
Status:
  Provisioning State: available  # Server is ready to be allocated
  Hardware Profile: unknown
  Power Status: off
```

---

### 6.5 How Server Selection Works

When you create a **ProvisioningRequest**, the **HardwareTemplate** specifies selection criteria:

```yaml
# In HardwareTemplate
spec:
  nodeGroupData:
    - name: master
      role: master
      hwProfile: hpe-e910-profile

      # These match the resourceselector labels!
      resourceSelector:
        server-model: e910
        rack: rack-A
        # O2IMS will find servers with matching labels
```

**Matching Logic:**
1. O2IMS finds all BareMetalHosts with `resourcePoolId: hpe-proliant-pool`
2. Filters by `resourceSelector` labels
3. Allocates the required number of servers
4. Creates `AllocatedNode` CRs for selected servers

---

### 6.6 Summary

**To add servers to O2IMS resource pool:**

1. ✅ Create namespace (= Resource pool)
2. ✅ Create BMC secrets (Server credentials)
3. ✅ Create network secrets (Optional - Static network config)
4. ✅ Create BareMetalHost CRs with:
   - `siteId` label
   - `resourcePoolId` label (= namespace)
   - `resourceselector.*` labels (for matching)
   - `interfacelabel.*` labels (for NICs)
   - BMC address and credentials
5. ✅ Verify servers show as "available"

The servers are now in the resource pool and ready to be allocated by O2IMS when you create a ProvisioningRequest!

---

## Document Information

**Last Updated:** 2026-02-19
**Document Version:** 1.1
