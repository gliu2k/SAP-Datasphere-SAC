# Roadmap to SAP Datasphere

SAP is encouraging customers to transition to Datasphere from BW. 

![alt text](/Roadmap/images/AsIs.png?raw=true)


![alt text](/Roadmap/images/Target.png?raw=true)

> [!Note]
> **Impacts**:
> 
> 1. BW-BPC: Use S/4HANA embedded BPC or the **planning** function in SAP SAC
> 
> 2. Bex Query: Gone


# 1. Migration

The path has been laid out to ensure a smooth migration and protect the investment in the BW system.

[Introducing the Migration from SAP BW to SAP Datasphere, SAP BW Bridge](https://learning.sap.com/learning-journeys/modernizing-your-data-warehouse-landscape-from-sap-bw-to-sap-datasphere/introducing-the-migration-from-sap-bw-to-sap-datasphere-sap-bw-bridge)

SAP recommends moving to BW Bridge first. It operates on SAP HANA Cloud platform but still includes the **ABAP stack**, enabling a seamless migration of BW objects.

![alt text](/Roadmap/images/Path.png?raw=true)

SAP Cloud is on **HANA(XSA) + NODEJS** stack. Python runtime environmant may be different.

## 1.1. Migrate to BW Bridge and Datasphere

![alt text](/Roadmap/images/Bridge.png?raw=true)

- There are two ways to migrate the contents from on-promise BW system to BW Bridge.

  - Shell Conversion: 
Only the metadata of the objects are transferred. The data are reloaded from the source systems (S/4HANA or ERP).

  - Remote Conversion: 
Both the metadata and data are transferred. This is for the scenario where the historical data are not available in the source systems. After the conversion, the new data will be loaded from the source systems into BW Bridge.

- There are limitations of BW Bridger(See SAP Note 3117800). 
  - Queries are not supported.
  - No support for the OLAP engine and functionality dependent on the OLAP engine, e.g., analysis authorizations, query as InfoProvider, query execution, calculation of non-cumulative key figures (aka inventory key figures).
  - ....

The ADOs tables are replciated from BW bridge and the models are rebuilt in DataSphere(DWC = Datasphere). See section  **2.1** and **4.2**. The new reports are created in SAC.
![alt text](/Roadmap/images/Future.png?raw=true) 


## 1.2. Load data from BW Bridge to DataSphere

You can find the details in this [blog](https://community.sap.com/t5/technology-blogs-by-sap/delta-extraction-of-adso-from-sap-bw-bridge-into-sap-datasphere-via/ba-p/13651788) about how to load the delta data from the changelog table of ADSO in BW bridge. However, it does not include the case when 0RecordMode = 'A'. 

Alternatively, you can use openHub and delta-DTP to do the same thing.

Using BW Bridge(ADSOs) will heavily rely on the models and processes built in the BW system.

> [!Note]
> **BW Bridge** is a tempoary solution.


# 2. Greenfield 

I think this approach is better and offers greater flexibility. [Link](https://learning.sap.com/learning-journeys/modernizing-your-data-warehouse-landscape-from-sap-bw-to-sap-datasphere/introducing-the-greenfield-approach)

## 2.1. Load data into DataSphere from the source systems(S/4HANA, ERP)

Below are the approaches(**containers**) and their drawbacks

- ABAP Tables (SLT)
  - Remodeling, without using the Pre-built Business Content in both Datashphere and SAP S/4HANA systems.

- ABAP CDSViews (SDI/DP Aent-ABAP)
  
  **Below is excerpted from "First Guide: SAP Data Integratoin for ABAP Source System", DWC = Datasphere**. All the CDSViews are seemed as **Extractors** in DP Agent (**Remote Table** import). This method is not flexible in maintenance and troubleshooting.
  
  ![alt text](/Roadmap/images/CDSV_DPAgent.png?raw=true)

- ABAP CDS View Extractor (ODP/ODQ)
  - Fewer customers develop the custom CDSView with the delta mechanism. 
  [Example](https://github.com/SAP-samples/teched2022-DA281/blob/main/exercises/dd1/README.md)

  - The doable way is to load the init/delta data from SAP standard CDSViews into the local tables in Datasphere, which can be imported from the Datashere Contents.
  [HowTo](https://community.sap.com/t5/enterprise-resource-planning-blogs-by-sap/cds-based-data-extraction-part-ii-delta-handling/ba-p/13425761)  And then, we build the DataSphere **VIEWS** on top of these local tables, which have the same structures of SAP standard CDSViews.
  - When we want to make enhancements(to the CDSViews- the datasource in S/4HANA), we need to create the custom CDSViews by referring(copying) to the SAP standard ones though SAP's Best Practice is to extend the standard CDSViews.

- BW SAPI Extractor(ODP)
  - The new Business Contents in Datasphere are based on CDS Views, the new datasources, in S/4HANA. The only reason to use this method is that you want to keep  the current BW architecture, using the **View** in Datashere to replace the old objects and logics - BW models and processes in the BW Bridge system before getting rid of it.

> [!Important]
> **The ABAP CDS Views, the second and the third approach, are [SAP Best Practice](https://community.sap.com/t5/technology-blogs-by-sap/sap-datasphere-sap-s-4hana-your-guide-to-seamless-data-integration/ba-p/13662817)**.
>
> **Remoet Tables** vs **Replication Flow**:
> 
> **Remoet Tables** (mapping to the **ABAP Tables** or **ABAP CDS Views** via DPAgent): can replicate the snapshot of the master data(dimension/text/hierarchy) and do realtime replication(delta loading) of the transaction data. Only the CDSViews with **@Analytics.dataExtraction.enabled** + **CDC**(delta.changeDataCapture) are supported for the realtime replication. But for those CDSViews that use **Generic Date/Timestamp Delta**, delta data loading is not supported because the delta changes cannot be pushed to the target(**CDC = PUSH; TIMESTAMP = PULL**).
>
> **Replication Flow**: is mainly used to replicate the master data and the transaction data through **ABAP CDS View Extractor (ODP/ODQ )** with **@Analytics.dataExtraction.enabled**. SAP also use **Replication Flow** in the Datasphere Business Contents. See section **3**.
>
> You can find the comparison of using **Replication Flow** and **Remoet Tables** in SAP Datasphere Expert [Content](https://help.sap.com/docs/SUPPORT_CONTENT/datasphere/4181116697.html).

## 2.2 ODP/ODQ (SLT & CDS View)

![alt text](/Roadmap/images/ODP1.png?raw=true) 

### 2.2.1. ODP/SLT
![alt text](/Roadmap/images/ODP2.png?raw=true) 

- **SLT: Need to install SLT server**

### 2.2.2. ODP/ABAP CDSView

- Full extraction and query-type access (direct access)
  - The following annotations make it possible for an ABAP CDS view to be available for full/init extraction or for direct access:
    - @Analytics.dataCategory
    - @Analytics.dataExtraction.enabled

- Delta Extraction
  - For delta extraction, the following additional annotations are available:
    - Analytics.dataExtraction.delta.byElement.name
    - Analytics.dataExtraction.delta.byElement.maxDelayInSeconds
    - Analytics.dataExtraction.delta.byElement.detectDeletedRecords

- **Fewer custom CDSView has such a good design to capture the delta. Some custom CDSViews may have input parameter or may be too complex.**

#### 2.2.2.1. Generic Date/Timestamp Delta

![alt text](/Roadmap/images/CDC1.png?raw=true) 

```
@Analytics:{
  dataCategory: #FACT,
  dataExtraction: {
	enabled: true,
	delta.byElement : {
	  name: 'LastChangeDateTime',
	  maxDelayInSeconds : 1800
	  }
	}
}
```
**The ODP frame work stores key data and hashes of all records belonging to the safety interval for finding changed records.**

![alt text](/Roadmap/images/CDC2.png?raw=true)

It uses the **"PULL"** method.

#### 2.2.2.2. Change Data Capture Delta (CDC = delta.changeDataCapture)

![alt text](/Roadmap/images/CDC3.png?raw=true)

This is a SAP standard CDS View **C_SalesDocumentItemDEX_1** for the data extraction of the header and line-item of S/4HANA sales documents.

![alt text](/Roadmap/images/CDC4.png?raw=true)

Whenever a record is inserted, updated or deleted in the underlying table, a record with the respective table key is stored in a generated logging table. Based on this info the scheduled job selects the data record from the CDS view and pushes it into the ODQ. It uses the **"PUSH"** method.

```
@AbapCatalog.sqlViewName: 'CSDSLSDOCITMDX1'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl:{
    authorizationCheck: #CHECK,
    personalData.blocking: #('TRANSACTIONAL_DATA')
}
@EndUserText.label: 'Data Extraction for Sales Document Item'
@ClientHandling.algorithm: #SESSION_VARIABLE
@ObjectModel.usageType.dataClass: #TRANSACTIONAL
@ObjectModel.usageType.serviceQuality: #D
@ObjectModel.usageType.sizeCategory: #XL
@ObjectModel.representativeKey: 'SalesDocumentItem'
@VDM.viewType: #CONSUMPTION
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.modelingPattern: #NONE
@ObjectModel.supportedCapabilities:  [ #EXTRACTION_DATA_SOURCE ]

@Analytics: {
    dataCategory: #FACT,
    dataExtraction: {
        enabled: true,
        delta.changeDataCapture: {
            mapping:[
                {
                    table: 'vbap', role: #MAIN,
                    viewElement: ['SalesDocument', 'SalesDocumentItem'],
                    tableElement: ['vbeln', 'posnr']
                },
                {
                    table: 'vbak', role: #LEFT_OUTER_TO_ONE_JOIN,
                    viewElement: ['SalesDocument'],
                    tableElement: ['vbeln']}
            ]
        }
    }
 }

define view C_SalesDocumentItemDEX_1
as select from I_SalesDocumentItem as SalesDocumentItem
left outer to one join I_SalesDocument              as SalesDocument          on SalesDocumentItem.SalesDocument = SalesDocument.SalesDocument
left outer to one join I_CompanyCode                as CompanyCode            on  SalesDocument.BillingCompanyCode = CompanyCode.CompanyCode
//Extensibility
association [0..1] to E_SalesDocumentItemBasic      as _ExtensionItem         on  $projection.SalesDocument     = _ExtensionItem.SalesDocument
                                                                              and $projection.SalesDocumentItem = _ExtensionItem.SalesDocumentItem
association [0..1] to E_SalesDocumentBasic          as  _ExtensionHeader      on  $projection.SalesDocument     = _ExtensionHeader.SalesDocument
{
    // Key
    @ObjectModel.foreignKey.association: '_SalesDocument'
    key SalesDocumentItem.SalesDocument,
    key SalesDocumentItem.SalesDocumentItem,
```

## 2.3. Hierarchy & Others

Please refer to this good [blog](https://community.sap.com/t5/enterprise-resource-planning-blogs-by-sap/cds-based-data-extraction-part-iii-miscellaneous/ba-p/13452148) for hierarchy extraction.

Leverage CONVERT_UNIT to do [Unit Conversion](https://community.sap.com/t5/technology-blogs-by-sap/sap-datasphere-unit-conversion/ba-p/13567259)

The followings show how the **Currency** tables are improted from S/4HANA.

![alt text](/Roadmap/images/Curr1.png?raw=true)

The currency related Remote Tables, DataFlow, Local Tables, Views for are imported. [Blog](https://community.sap.com/t5/technology-blogs-by-members/currency-translation-in-sap-datasphere/ba-p/13688008)

![alt text](/Roadmap/images/Curr2.png?raw=true)

</BR></BR>

> [!Tip]
> Check the CDS Extractor in this table/SE16 and make sure the CDSView is released as **C1** (**BTW, in the embedded analsyis, we need to release the Cube or Query CDSViews so that they can be inserted into the "Analysis for Office" or "PowerBI" as a dataSource**)

![alt text](/Roadmap/images/CDS_EX1.png?raw=true)


# 3. Business Content in Datasphere (IMPORTANT)

This is SAP Business Content in Datasphere, which is similar to SAP BW Content in BW system. Their data are loaded from the corresponding CDSViews, the datasources, in S/4HANA system. The CDSView name is part of the name of the local table in the business content.

![alt text](/Roadmap/images/BC1.png?raw=true)

They can be imported into the space.
![alt text](/Roadmap/images/BC2.png?raw=true)

These are the Sales SAP Business Contents imported into the space of my trial Datasphere environemnt. The data in the local tables are loaded from the CDSViews in the on-premise S/4HANA system.
![alt text](/Roadmap/images/BC3.png?raw=true)

- Transaction Data:
	- Datashphere Local Table: SAP_SD_IL_**C_SALESDOCUMENTITEMDEX_1**
 	- S/4HANA CDSView: **C_SalesDocumentItemDEX_1**

- Master Data:
 	- Datashphere Local Table: SAP_LO_IL_**I_CUSTOMER**
 	- S/4HANA CDSView: **I_Customer** [SAP Help](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/ee6ff9b281d8448f96b4fe6c89f2bdc8/a11849401368469fb9e7dfb34f44e9c7.html)

> [!Note]
> From the Busienss Content, you can see that SAP use the **Replication Flows** to replicate both master and transaction data from S/4HANA(CDSViews) to **Local Tables**.

![alt text](/Roadmap/images/BC4.png?raw=true)


> [!Important]
> **It is highly recommendeded to leverage the Datasphere Business Content(and CDSViews) to alleviate the workload in the implementation and future maintenance. SAP has desgined and developed a comprehensive suite of standard objects in both Datasphere and S/4HANA(CDSViews) systems for reuse. We should follow the best practise so that we don't need to worry about table relationship, data redundant, record duplication and the issues in the future system upgrade.** 

# 4. Delta Data Loading Within DataSphere
Generally，this topic should be included in the Flow section in DataBuilder. In fact, the Delta feature is quite simple and straight-forward within SAP DataSphere. We need to consider more when we integrate it with other system like S/4HANA and BW Bridge.

Let's take a quick at the **Delta** within Datasphere. We have talked about the settings in the "Model" and "Flow" in DataBuilder section. We need to turn on "delta capture" in 
local table and use Transformation Flow to do dalta data loading.

### 4.1. Full Data Loading 

Local Source Table - Non Delta

Local Target Table - Delta

Transformation flow - We can only choose "Initial Only" in "load type"

![alt text](/Roadmap/images/LoadType_Error.png?raw=true)

In the first execution of **Transformation Flow**, three records are loaded into target.

![alt text](/Roadmap/images/AD1.png?raw=true)

We changed one record in the source table and run the **Transformation Flow**. All three records are loaded again into the target.

![alt text](/Roadmap/images/AD2.png?raw=true)

The same is shown in the log(three records are loaded).
![alt text](/Roadmap/images/log.png?raw=true)


### 4.2. Delta Data Loading 

Local Source Table - Delta

Local Target Table - Detla

Transformation Flow - select "Delta Table" of the source and "Initial and Delta" type

In the first execution of **Transformation Flow**, three records are loaded into the target.
![alt text](/Roadmap/images/DD1.png?raw=true)

We changed one record in the source table and run the **Transformation Flow** again. Only the changed record is loaded into the target.
![alt text](/Roadmap/images/DD2.png?raw=true)

The same is shown in the log (one record is loaded).
![alt text](/Roadmap/images/log2.png?raw=true)

Detla can be reset.
![alt text](/Roadmap/images/ResetWaterMark.png?raw=true)

> [!Important]
> - **Summary:**
>   - We CANNOT choose "Initial and Detla" type in Transformation Flow if the source table does not support "Detla".
>   - We CANNOT use the Active Table of the source table (if the source table supports "Delta" mode as well) and select "Initial and Delta" type in Transformation Flow. "Initial Only" is the only option.
>   - We can choose "Initial Only" type in the first execution of the Transformation Flow but need to change it to "Initial and Delta" type in the consequential execution. (Why not select "Initial and Delta" from the very beginning.)

# 5. Limits & Integration(with non-SAP)

According to my understanding, SAP Datasphere is an integrated data Fabric platform. It should comprise advantage (cloud) services to processes structured, unstructured and realtime data.

So far, compared to Azure Fabric product, I think it has the following limits.

- No specific service for stream data(like KQL in Azure)
- No vector DB 
- No ML and AI services. Datasphere does not support python notebook. All the services like machine learning are still in **SAP Data Intelligence Cloud**. 
  
![alt text](/Roadmap/images/Arch.png?raw=true)

In the current landscape, it shows that the **Integration** is only at the **DATA** level not at the Application(service) level. However, it says SAP will integrate the services from **SAP Data Intelligence Cloud** into **Datasphere**. According to this [Roadmap](https://community.sap.com/t5/technology-blogs-by-sap/sap-datasphere-and-sap-data-intelligence-cloud-what-does-this-mean-for-me/ba-p/13603947), **Datashpere** will become SAP's Data Fabric as **Azure Data Faric** for **Microsoft** in the future.

There are already embedded Data Intelligence cloud services in the DataSphere to achieve some functionalities in "Replication Flow". Below screenshot is from the "Replication Flow". Apache **Parquet** file format is used in Dataphere instead of HANA data file. It will be easier to integarte services like park, Databrick for Machine Learning.

![alt text](/Roadmap/images/Parquet.png?raw=true)
