## Anchore Enterprise SBOM


## Introduction

In Anchore Enterprise, the Software Bill of Materials (SBOM) is a comprehensive inventory of individual components, or artifacts, found in a container image (or filesystem directory). These artifacts include software packages, binary executables, and files. An extensive set of metadata is collected for each artifact and stored in the SBOM. In this way the SBOM is not merely a list of container artifacts but a detailed descriptive anatomy of the container image.

A detailed and accurate SBOM enables Anchore Enterprise to flag security issues by identifying matches between image contents and published CVEs (Common Vulnerabilities and Exposures). In addition, SBOM data can be evaluated to ensure compliance with organizational policies.


## SBOM Creation and Storage

To create an SBOM, Anchore Enterprise unpacks the container image and then scans its contents. This process is called image analysis. During analysis, every package, software library and file is inspected and the resulting metadata is stored in the SBOM. Anchore has developed a number of analyzer modules, each designed to extract data from specific types of image content. Sources of analyzed data include:



*   Image metadata
*   Image layers
*   Operating system package data (RPM, DEB, APKG)
*   File data
*   Ruby Gems
*   Node.JS NPMs
*   Java archives
*   Python packages
*   .NET NuGet packages
*   File contents

Every SBOM created is stored in the Anchore catalog for reuse. The SBOM remains available for evaluation as new CVEs are published or new security policies are implemented. 


## Metadata

Anchore Enterprise attempts to collect as much metadata as possible about each artifact it discovers. Types of metadata recorded in the SBOM include:



*   Version
*   File system path
*   Permissions
*   Language
*   Author
*   License
*   Source


## CPE Assignment

As CVEs identify vulnerable software by CPE (Common Platform Enumerator), it is important that Anchore Enterprise include CPEs for each package in the SBOM. However, CPEs are not provided explicitly in package metadata so Anchore Enterprise must infer the proper CPEs from the metadata that is available. In most cases, Anchore Enterprise will assign multiple CPEs to each package in the SBOM to ensure packages match appropriate CVEs.
