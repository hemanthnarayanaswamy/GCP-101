# Understanding Google Cloud Regions, Zones & Cluster (GCP)

A Region is a geographical location where Google Cloud resources live. 
Think of it like a city where your cloud data center is located.

```ini
us-central1 -> lowa
us-east1 -> South Carolina
us-east4 -> North Virginia
```

![img](https://k21academy.com/wp-content/uploads/2023/03/regions-768x379.png)

### Why Do We Need Regions ?
* High Availability -> Deploy workloads across multiple regions to ensure uptime.
* Compliance -> Meet local data residency laws for specific countries/regions.
* Low Latency -> Host apps closer to customers for faster performance. 

### Where Do we use Regions in GCP ?
As a cloud admin, you choose a region when creating resources such as:
* VM instances
* Cloud SQL databases
* Storage buckets.

### What is a Zone ?
A Zone is a deployment area within a Region. Regions are collections of zones. Each region typically has 3 or more zones.

Zones have high-bandwidth, low-latency network connections to other zones in the same region. 
*In order to deploy fault-tolerant applications that have high availability, Google recommends deploying applications across multiple zones and multiple regions. This helps protect against unexpected failures of components, up to and including a single zone or region.*

- Zones which are specialized for AI and ML workloads are called **AI zones**. 
- Non-AI zones are referred to as **standard zones or just zones**.

1. **Standard Zone**
A standard zone name contains two parts: the region and the zone in the region. For example, the fully qualified name for zone a in region us-central1 is `us-central1-a`.
2. **AI Zone**
For an AI zone, the <zone> variable consists of three parts: the string ai (to identify it as an AI zone), a number (indicating its deployment group), and a letter (indicating the shared software update schedule). For example, the fully qualified name for AI zone ai2b in region us-west4 is `us-west4-ai2b`.

![zones](https://download.huihoo.com/google/gdgdevkit/DVD1/developers.google.com/compute/images/zones_diagram.png)

#### Why multiple zones?
* Fault tolerance → If one zone goes down, others keep running.
* Low-latency interconnect → Zones within a region have high-speed links.

### Network Edge Locations
Network edge locations provide connectivity to Google’s network from the internet via peering such as 3rd parties. As of today, there are 176 network edge locations.

Google Cloud Platform (GCP) network edge locations are entry points and caching nodes distributed across over 200 countries and territories to bring services closer to users and reduce latency

* Zonal resources operate in a single zone and can be affected by zonal outages such as a virtual machine.
* Regional resources span multiple zones for redundancy similar to a static IP address.
* Multi-regional supports redundancy across various regions in case of a single region failure.

1. **Points of Presence** Places where Google's private network connects with external networks and Internet Service Providers (ISPs)
2. **Google Global Cache (Edge Nodes)** Infrastructure placed inside local operator networks across over 1300 cities to cache content locally.
3. **Cloud Interconnect Locations** Dedicated facilities where enterprise networks physically hook up to Googles backbone.

### What are Clusters in a Zone ?
Inside each zone, there are clusters - sets of physical infrastructure (compute, networking, storage) in a data center. 

Example Mapping:
```ini
Organization A in asia-east1-a → Cluster Z
Organization B in asia-east1-a → Cluster Y
```
Google ensures all projects in your organization follow a consistent zone-to-cluster mapping for predictability.

> Zone to Cluster Mapping is not visible to customers. Google Cloud manages it internally

![img](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fwgb2tnlqto6797d2uf5e.png)