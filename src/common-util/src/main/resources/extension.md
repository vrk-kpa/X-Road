# Security Server targeting extension for the X-Road message protocol

Version: 1.0  
Doc. ID: PR-TARGETSS

| Date      | Version  | Description                                                                  | Author             |
|-----------|----------|------------------------------------------------------------------------------|--------------------|
| 1.3.2017 | 1.0       | Initial version                                                              | Olli Lindgren     |


## Table of Contents
<!-- toc -->

  * [License](#license)
  * [Overview](#overview)
  * [References](#references)
- [Components](#components)
  * [Monitoring metaservice (proxymonitor add-on)](#monitoring-metaservice-proxymonitor-add-on)
  * [Monitoring service (xroad-monitor)](#monitoring-service-xroad-monitor)
  * [Central monitoring client](#central-monitoring-client)
  * [Central monitoring data collector](#central-monitoring-data-collector)
  * [Central server admin user interface](#central-server-admin-user-interface)
- [Monitoring in action](#monitoring-in-action)
  * [Pull messaging model](#pull-messaging-model)
  * [Modified X-Road message protocol](#modified-x-road-message-protocol)
  * [Access control](#access-control)

<!-- tocstop -->

## License

This document is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/3.0/.


## Introduction

This specification describes an extension of the X-Road protocol for targeting the message to a specific security server.

The original X-Road 4.0 message protool (PS-MESS) has the SOAP header element _service_ to define the recipient of a message.
In a clustered security server configuration, one service can be served from multiple security servers. When X-Road routes the message to such a service,
it picks the target security server based on which server establishes a connection the quickest.
There is no guarantee about the actual target server, it can be any of the clustered servers. There are use cases,
like environmental monitoring (ARC-ENVMON), where targeting of a specific security server is needed.

## Format of messages

### Target security server
This section describes XML-based data formats for expressing the target security server. The data
structures and elements defined in this section will be located under namespace http://x-road.eu/xsd/xroad.securityserver.xsd.
The complete XML Schema is shown in section XML Schema securityserver.xsd.
The following listing shows the header of the schema definition

```xml
 <xs:schema elementFormDefault="qualified" 
         targetNamespace="http://x-road.eu/xsd/xroad.xsd"
         xmlns="http://x-road.eu/xsd/xroad.xsd"
         xmlns:id="http://x-road.eu/xsd/identifiers"
         xmlns:xs="http://www.w3.org/2001/XMLSchema">
     <xs:import namespace="http://www.w3.org/XML/1998/namespace"
             schemaLocation="http://www.w3.org/2001/xml.xsd"/>
     <xs:import id="id" namespace="http://x-road.eu/xsd/identifiers"
             schemaLocation="http://x-road.eu/xsd/identifiers.xsd"/>
</xs:schema>
``` 

A new `securityServer` element was added to identify the specific target security server.

```xml
 <xs:element name="securityServer" type="id:XRoadSecurityServerIdentifierType">
        <xs:annotation>
            <xs:documentation>Identifies a specific security server</xs:documentation>
        </xs:annotation>
    </xs:element>
```
  The element is of the type `XRoadSecurityServerIdentifierType`, which is one of the identifiers already defined
  in the X-Road Message Protocol v 4.0 [PR-MESS] section 2.1. The whole XML schema for the identifier types is defined in
  Annex A of the same document. The relevant part is listed below for convenience.
  
```xml
<xs:complexType name="XRoadSecurityServerIdentifierType">
    <xs:complexContent>
        <xs:restriction base="XRoadIdentifierType">
            <x:sequence>
                <xs:element ref="xRoadInstance"/>
                <xs:element ref="memberClass"/>
                <xs:element ref="memberCode"/>
                <xs:element ref="serverCode"/>
            </xs:sequence>
            <xs:attribute ref="objectType" use="required" fixed="SERVER"/>
        </xs:restriction>
    </xs:complexContent>
</xs:complexType>
```
 
 ## Message headers
 This section describes additional SOAP headers that are used by the X-Road system.
 
 | Field          | Type                              | Mandatory/Optional | Description                              |
 |----------------|-----------------------------------|--------------------|------------------------------------------|
 | securityServer | XRoadSecurityServerIdentifierType | Optional           | The security server this message is for. |
 
 

 ## XML Schema representation
 ```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema elementFormDefault="qualified"
        targetNamespace="http://x-road.eu/xsd/xroad.xsd" 
        xmlns="http://x-road.eu/xsd/xroad.xsd" 
        xmlns:id="http://x-road.eu/xsd/identifiers" 
        xmlns:xs="http://www.w3.org/2001/XMLSchema"> 
    <xs:import namespace="http://www.w3.org/XML/1998/namespace" 
            schemaLocation="http://www.w3.org/2001/xml.xsd"/> 
    <xs:import id="id" namespace="http://x-road.eu/xsd/identifiers" 
            schemaLocation="http://x-road.eu/xsd/identifiers.xsd"/> 
  
    <!-- Header elements -->
    <xs:element name="securityServer" type="id:XRoadSecurityServerIdentifierType">
        <xs:annotation>
            <xs:documentation>Identifies security server</xs:documentation>
        </xs:annotation>
    </xs:element>
</xs:schema>
```

## Example
Below are examples from a request and response related to the Environmental Monitoring [ARC-ENVMON] service `getSecurityServerMetrics`
which uses the `securityServer` element.

### Request
```xml
<SOAP-ENV:Envelope
    xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:id="http://x-road.eu/xsd/identifiers"
    xmlns:xrd="http://x-road.eu/xsd/xroad.xsd"
    xmlns:m="http://x-road.eu/xsd/monitoring">
    <SOAP-ENV:Header>
        <xrd:client id:objectType="MEMBER">
            <id:xRoadInstance>fdev</id:xRoadInstance>
            <id:memberClass>GOV</id:memberClass>
            <id:memberCode>1710128-9</id:memberCode>
        </xrd:client>
        <xrd:service id:objectType="SERVICE">
            <id:xRoadInstance>fdev</id:xRoadInstance>
            <id:memberClass>GOV</id:memberClass>
            <id:memberCode>1710128-9</id:memberCode>
            <id:serviceCode>getSecurityServerMetrics</id:serviceCode>
        </xrd:service>
        <xrd:securityServer id:objectType="SERVER">
            <id:xRoadInstance>fdev</id:xRoadInstance>
            <id:memberClass>GOV</id:memberClass>
            <id:memberCode>1710128-9</id:memberCode>
            <id:serverCode>fdev-ss1.i.palveluvayla.com</id:serverCode>
        </xrd:securityServer>
        <xrd:id>ID11234</xrd:id>
        <xrd:protocolVersion>4.0</xrd:protocolVersion>
    </SOAP-ENV:Header>
    <SOAP-ENV:Body>
        <m:getSecurityServerMetrics/>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```
### Response
```xml
<SOAP-ENV:Envelope
    xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:id="http://x-road.eu/xsd/identifiers"
    xmlns:m="http://x-road.eu/xsd/monitoring"
    xmlns:xrd="http://x-road.eu/xsd/xroad.xsd">
   <SOAP-ENV:Header>
      <xrd:client id:objectType="MEMBER">
         <id:xRoadInstance>fdev</id:xRoadInstance>
         <id:memberClass>GOV</id:memberClass>
         <id:memberCode>1710128-9</id:memberCode>
      </xrd:client>
      <xrd:service id:objectType="SERVICE">
         <id:xRoadInstance>fdev</id:xRoadInstance>
         <id:memberClass>GOV</id:memberClass>
         <id:memberCode>1710128-9</id:memberCode>
         <id:serviceCode>getSecurityServerMetrics</id:serviceCode>
      </xrd:service>
      <xrd:securityServer id:objectType="SERVER">
         <id:xRoadInstance>fdev</id:xRoadInstance>
         <id:memberClass>GOV</id:memberClass>
         <id:memberCode>1710128-9</id:memberCode>
         <id:serverCode>fdev-ss1.i.palveluvayla.com</id:serverCode>
      </xrd:securityServer>
      <xrd:id>ID11234</xrd:id>
      <xrd:protocolVersion>4.0</xrd:protocolVersion>
      <xrd:requestHash algorithmId="http://www.w3.org/2001/04/xmlenc#sha512">mChpBRMvFlBBSNKeOxAJQBw4r6XdHZFuH8BOzhjsxjjOdaqXXyPXwnDEdq/NkYfEqbLUTi1h/OHEnX9F5YQ5kQ==</xrd:requestHash>
   </SOAP-ENV:Header>
   <SOAP-ENV:Body>
      <m:getSecurityServerMetricsResponse>
         <m:metricSet>
            <m:name>SERVER:fdev/GOV/1710128-9/fdev-ss1.i.palveluvayla.com</m:name>
            <m:stringMetric>
               <m:name>proxyVersion</m:name>
               <m:value>6.7.7-1.20151201075839gitb72b28e</m:value>
            </m:stringMetric>
            <m:metricSet>
               <m:name>systemMetrics</m:name>
               <m:stringMetric>
                  <m:name>OperatingSystem</m:name>
                  <m:value>Linux version 3.13.0-70-generic</m:value>
               </m:stringMetric>
               <m:numericMetric>
                  <m:name>TotalPhysicalMemory</m:name>
                  <m:value>2097684480</m:value>
               </m:numericMetric>
               <m:numericMetric>
                  <m:name>TotalSwapSpace</m:name>
                  <m:value>2097684480</m:value>
               </m:numericMetric>
            </m:metricSet>
            ...          
         </m:metricSet>
      </m:getSecurityServerMetricsResponse>
   </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

<a name="refsanchor"></a>
## References

| Code||
| ------------- |-------------|
| PR-MESS | Cybernetica AS.X-Road: Message Protocol v4.0      |
| ARC-ENVMON | ARC-ENVMON |



