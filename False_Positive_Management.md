# False Positive Management


## Summary

Anchore identifies vulnerable packages by comparing the package CPEs, as determined by Anchore, to CPEs listed in vulnerability feeds. A discrepancy between the CPEs assigned by Anchore to a particular package and the CPEs listed in the vulnerability feed may result in Anchore generating a false positive or a false negative when an image is scanned. The false-positive correction feature introduced in Anchore Enterprise 3.0 allows users to manually edit package CPEs in Anchore to eliminate false positives and false negatives. This feature requires the use of Anchore’s API or CLI. The current implementation of false-positive management applies to only package artifacts in container images.


## Introduction

As Anchore analyzes a container image, it inspects all available metadata for each package in order to assign a Common Platform Enumerator (CPE). The CPE is an industry standard structured naming scheme for software packages. When a vulnerability is published, affected packages are identified by their CPEs. When scanning an image for vulnerabilities, Anchore compares the CPEs of packages found in the container with CPEs in vulnerability feeds to determine whether vulnerabilities are present.

Anchore makes a best effort to assign correct CPEs to each package based on available metadata, but in some situations the CPEs assigned by Anchore may not match the package CPEs as they appear in the [CPE dictionary](https://nvd.nist.gov/products/cpe/search) maintained by the [National Vulnerability Database](https://nvd.nist.gov/) (NVD). Any discrepancy between the CPEs assigned by Anchore and those published in the NVD may result in Anchore reporting a false positive or false negative.


## CPE Naming Specification

A CPE is often presented as a colon-delimited string, such as


```
cpe:2.3:a:microsoft:internet_explorer:8.0.6001:beta:*:*:*:*:*:*
```


A CPE in this form can be generalized as


```
cpe:2.3: part : vendor : product : version : update : edition :  language : sw_edition : target_sw : target_hw : other
```


The first two elements indicate that the string is a CPE and conforms to version 2.3 of the CPE specification.

The remaining elements, in order, correspond to the following attributes:



*   **Part** – ”a” for applications, “o” for operating systems, “h” for hardware devices
*   **Vendor** – the person or organization that created the product
*   **Product** – the common and recognizable name of the product
*   **Version** – a vendor-specific alphanumeric string characterizing the particular release version of the product
*   **Update** –  a vendor-specific alphanumeric strings characterizing the particular update, service pack, or point release of the product
*   **Edition** – deprecated by CPE v2.3
*   **Language** – the language supported in the user interface of the product being described
*   **SW_Edition** – how the product is tailored to a particular market or class of end users
*   **Target_SW** – the software computing environment within which the product operates
*   **Target_HW** – the instruction set architecture (e.g., x86) on which the product being described operates
*   **Other** – any other general descriptive or identifying information which is vendor- or product-specific and which does not logically fit in any other attribute value

Any attribute may be assigned the logical value of ANY, indicated by “*”, or the logical value of NA (not applicable), indicated by “-”.

Complete CPE documentation can be found [here](https://nvlpubs.nist.gov/nistpubs/Legacy/IR/nistir7695.pdf).


## Managing Corrections with the API


### Correction Example – API

For the Java package Spring Core version 5.1.4, Anchore assigns the following CPEs and metadata:


```
{
    "cpes": [
        "cpe:2.3:a:*:spring-core:5.1.4.RELEASE:*:*:*:*:*:*:*",
        "cpe:2.3:a:*:spring-core:5.1.4.RELEASE:*:*:*:*:java:*:*",
        "cpe:2.3:a:*:spring-core:5.1.4.RELEASE:*:*:*:*:maven:*:*",
        "cpe:2.3:a:spring-core:spring-core:5.1.4.RELEASE:*:*:*:*:*:*:*",
        "cpe:2.3:a:spring-core:spring-core:5.1.4.RELEASE:*:*:*:*:java:*:*",
        "cpe:2.3:a:spring-core:spring-core:5.1.4.RELEASE:*:*:*:*:maven:*:*"
    ],
    "implementation-version": "5.1.4.RELEASE",
    "location": "/app.jar:BOOT-INF/lib/spring-core-5.1.4.RELEASE.jar",
    "maven-version": "N/A",
    "origin": "N/A",
    "package": "spring-core",
    "specification-version": "N/A",
    "type": "JAVA-JAR"
}
```


However, the CPE that appears in the NVD is:


```
cpe:2.3:a:pivotal_software:spring_security:5.1.4:*:*:*:*:*:*:*
```


There is a mismatch between the product attributes in the two CPEs (“spring-core” in Anchore, “spring_security” in the NVD), so the Spring Core package is not flagged as vulnerable. 

For Anchore to flag vulnerable packages, we need to modify the CPE. Using the API, we make an HTTP POST request to `<anchore-server>/v1/enterprise/corrections` with the following JSON data payload:


```
{
  "description": "Update Spring Core CPE",
  "match": {
    "type": "java",
    "field_matches": [
        {
            "field_name": "package",
            "field_value": "spring-core"
        },
        {
            "field_name": "implementation-version",
            "field_value": "5.1.4.RELEASE"
        }
    ]
  },
  "replace": [
      {
          "field_name": "cpes",
          "field_value": "cpe:2.3:a:pivotal_software:spring_security:5.1.4:*:*:*:*:*:*:*"
      }
  ],
  "type": "package"
}
```



### JSON Payload Reference



*   **description** – a brief description of why the correction was made
*   **match** – the criteria determining which packages will be modified by the correction. Specify the package type (java, gem, python, npm, os, go, nuget) the correction will apply to. Next, specify any fields associated with the package(s) that should be used for matching. Include the field name and field value for each desired field.
*   **replace** – the field(s) you wish to modify. Specify the field name and its new value. Note that if you specify a field that does not currently exist in the package metadata it will be created.
*   **type** – the type of image content (package, files, binary, malware) that will be modified. Currently only the “package” type is supported. 


### CPE Templating

CPE replacement can be templated based on the other fields of the package metadata. In the above example, the version attribute of the CPE could be templated as follows:


```
{
  "field_name": "cpes",
  "field_value": "cpe:2.3:a:pivotal_software:spring_security:{implementation-version}:*:*:*:*:*:*:*" 
}
```


Templated fields must be enclosed in curly braces "{}". The templated field will be replaced by the actual value for each package that matches the correction. Note that templated fields can only be used in the "cpes" field value.


### Viewing Corrections

To view a list of all corrections that have been made in your Anchore account using the API, make an HTTP GET request to `<anchore-server>/v1/enterprise/corrections` .

To view details about a specific correction, make an HTTP GET request to `<anchore-server>/v1/enterprise/corrections/{correction_id}` where {correction_id} is the UUID for the desired correction.


<table>
  <tr>
  </tr>
</table>



```
{
   "created_at": "2021-01-28T18:53:01Z",
   "description": "Correct NPM Redis Package in Anchore",
   "last_updated": "2021-01-28T18:53:01Z",
   "match": {
       "field_matches": [
           {
               "field_name": "path",
               "field_value": "/home/node/aui/build/node_modules/redis/package.json"
           }
       ],
       "type": "npm"
   },
   "replace": [
       {
           "field_name": "cpes",
           "field_value": "mis-identified-node-pacakge"
       }
   ],
   "type": "package",
   "uuid": "93e78b9e-8372-4ed7-961b-47d44ca82fba"
}
```



### Deleting a Correction

To delete a correction and restore the original CPEs for a package using the API, make an HTTP DELETE request to `<anchore-server>/v1/enterprise/corrections/{correction_id}` where {correction_id} is the UUID for the correction you wish to delete.


## Managing Corrections using the Anchore CLI


### Adding a Correction

anchore-cli enterprise corrections add [OPTIONS]

Options:

  -m, --match TEXT         [required]

  -p, --package_type TEXT  [required]

  -r, --replace TEXT       [required]

The CLI version of the API POST request made above is shown below. Note that the -m or -r flag must be included with each field=value pair.


```
anchore-cli enterprise corrections add \
-m package=spring-core \
-m implementation-version="5.1.4.RELEASE" \
-p java \
-r cpes="cpe:2.3:pivotal_software:spring_security:5.1.4:*:*:*:*:*:*:*" \
-r description="Update Spring Core CPE"
```


As with the API, CPE attributes can be templated by enclosing the name of a package metadata field in curly braces:


```
-r cpes="cpe:2.3:pivotal_software:spring_security:{implementation-version}:*:*:*:*:*:*:*"
```



### Viewing Corrections

The command to retrieve a list of existing corrections:


```
anchore-cli enterprise corrections list
```




The command to retrieve a list of existing corrections is:


```
anchore-cli enterprise corrections delete {correction_id}
```


where` {correction_id}` is the UUID of the correction you wish to delete.


## Frequently Asked Questions


### Is false positive management available in Anchore Engine?

No. False positive management is only available in Anchore Enterprise.


### Does false positive management address all sources of false positives in Anchore?

No. The current implementation of this feature only addresses false positives resulting from a mismatch between package metadata in Anchore and package metadata in vulnerability feeds. However, this feature will be further developed to address additional sources of false positives.


### Is false positive management limited to correcting false positives?

No. Mismatches between package metadata in Anchore and package metadata in vulnerability feeds may also result in false _negatives_. The false positive management feature can correct false negatives by ensuring consistency between information in Anchore and in vulnerability feeds.


### Don't Anchore's existing allowlist and denylist features already give me the tools I need to correct false positives and false negatives?

The false positive management feature is a more effective solution than creating allowlist and denylist entries because it provides accurate information and applies to all packages, not just specific policies or bundles.


### Will Anchore provide access to false positive management through the graphical user interface?

Yes—stay tuned!
