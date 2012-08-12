---
layout: body
title: Research
---

### Research Statement

I am motivated by the challenges that today’s networked services face in
reliably generating and delivering content to end-users. My research follows
an “end-to-end” philosophy: I strive to improve and optimize the user
experience wherever in the stack those optimizations might be. When I look
at today’s Internet I see two broad challenges: The first is creating
user-specific content through efficient exploration of massive data-sets.
The second is delivering high-bandwidth content performantly over
increasingly heterogeneous networked infrastructure (disparate connection
types, a proliferation of content distributors, etc.). My work studies and
addresses these challenges.

### Projects
 * <span class="proj_title"> 
     Decentralized Scheduling in Computing Clusters
   </span> 
   <span class="proj_dates">(Fall 2011 - Present)</span>
   <br>
    The majority of cluster computing frameworks put an emphasis on
    high-bandwidth parallel computation. These frameworks trade scheduler
    latency (typically accepting delays on the order of seconds) for improved
    utilization and throughput. This project asks whether a decentralized,
    uncoordinated scheduler might help reduce latency for short-lived,
    latency-sensitive queries.  By drawing parallels to link sharing protocols,
    such as Ethernet, we hope to develop scheduling techniques which better
    share the cluster between short and long-lived tasks.
 * <span class="proj_title">
   HTTP As a Narrow Waist </span> 
   <span class="proj_dates">(Fall 2011) </span> 
    <br/>
    Today’s Internet is increasingly dominated by HTTP traffic. This has arisen
    largely due to practical concerns, such as circumventing firewalls, and is
    also fostered by HTTP’s vast extensibility. This project suggests that HTTP
    has become the new “narrow waist” in the network stack, and posits that
    future Internet architecture research should consider HTTP, rather than IP,
    for deploying new network functionality. As a flagship example, it proposes
    an HTTP-based relay service, designed to standardize the budding use of
    HTTP for channel-based communication (eg, facebook chat). The relay service
    borrows from clean-slate proposals, but is built on top of existing HTTP
    infrastructure, easily traverses today’s middleboxes, and incurs only
    modest overhead compared with lower-layer alternatives.
 * <span class="proj_title">Flash Crowds </span> 
   <span class="proj_dates">(Spring 2011 - Fall 2011)</span> 
    <br />
    Handling flash crowds poses a difficult task for web services. Content
    distribution networks (CDNs), hierarchical web caches, and peer-to-peer
    networks have all been proposed as mechanisms for mitigating the effects
    of these sudden spikes in traffic to underprovisioned origin sites.  In
    this project, we characterize and quantify the behavior of thousands of
    flash crowds on CoralCDN, an open content distribution network running
    at several hundred POPs.  Our analysis considers over four years of CDN
    traffic, comprising more than 33 billion HTTP requests.  We draw
    conclusions in several areas, including (i) the potential benefits
    of cooperative vs. independent caching by CDN nodes, (ii) the
    efficacy of elastic redirection and resource provisioning, and (iii)
    the ecosystem of portals, aggregators, and social networks that drive
    traffic to third-party websites.
 * <span class="proj_title">DONAR</span> 
   <span class="proj_dates">(Summer 2009 - Fall 2010)</span>
    <br />
    With the advent of cloud computing and the growth of popular Web services,
    many networked services are replicated at multiple geographic locations.
    Such distributed services face the challenge of server selection — that is,
    directing an incoming client request to the appropriate server or data
    center, in the hope of reducing network latency or carefully tuning server
    loads. To meet these potentially conflicting goals, existing approaches use
    heuristics or rely on central coordination to perform server selection.

    DONAR is a distributed system that provides name resolution and server 
    selection for replication services.  It  applies optimization theory to 
    derive a simple, provably optimal, fully distributed solution to the 
    server-selection problem. DONAR defines a global objective for a 
    mapping service, and shows that decentralized mapping nodes performing 
    small amounts of local computation and sharing limited information, 
    can achieve the global objective.
