# AMWA INFO-XXX: NMOS Implementation Guide for DNS-SD \[Work In Progress\]

[![Lint Status](https://github.com/AMWA-TV/nmos-dns-sd-implementation-guide/workflows/Lint/badge.svg)](https://github.com/AMWA-TV/nmos-template/actions?query=workflow%3ALint)
[![Render Status](https://github.com/AMWA-TV//workflows/Render/badge.svg)](https://github.com/AMWA-TV/nmos-template/actions?query=workflow%3ARender)

This repository holds the source for this Implementation Guide, part of the family of [Networked Media Open Specifications](https://specs.amwa.tv/nmos) from the [Advanced Media Workflow Association](https://amwa.tv)

<!-- INTRO-START -->

### What does it do?

-   DNS-DS provides a simple and flexible method to discover NMOS RDS Services.
### Why does it matter?

- The NMOS RDS provides all information about NMOS nodes, devices, APIs. It must be located in order for a NNMOS component to register itself for discovery by other NMOS components or for discovering other NMOS components on a network. 

### How does it work?

- DNS-DS adds in Service Lookup to a DNS Server. This allows RDS(s) to be listed under names in a configuration file and clients can query the DNS for the particular Service required. In the present case this would be a query for the IP address of RDS Primary and potentially secondary services.

## Getting started

See the list of documents below.

There is more information about the NMOS Specifications and their GitHub repos at <https://specs.amwa.tv/nmos>.
