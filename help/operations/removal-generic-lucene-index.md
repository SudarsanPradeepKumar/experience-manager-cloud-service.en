---
title: Removal of the generic lucene index
description: Removal of the generic lucene index
exl-id: fe0e00ac-f9c8-43cf-83c2-5a353f5eaeab
---
# Removal of the 'generic lucene' index

Adobe intends to remove the 'generic lucene' index (`/oak:index/lucene-*`) from Adobe Experience Manager as a Cloud Service. This index has been deprecated since AEM 6.5. In this documentation the impact of this decision is described, along with detailled descriptions how to examine if an AEM instance is affected. Finally it contains ways to change queries so they work without this index being present.

## Background

In AEM, 'fulltext' queries as those using the following functions:

* `jcr:contains()` in JCR XPATH
* `CONTAINS` in JCR-SQL2

Such queries cannot return results without using an index. Unlike a query containing only path or property restrictons, a query containing a fulltext restriction for which no index can be found (and thus a traversal is performed) will always return 0 results.

The 'generic lucene' index (`/oak:index/lucene-*`) has existed since AEM 6.0 / Oak 1.0 in order to provide a fulltext search across most of the repository hierarchy (some paths, such as `/jcr:system` and `/var` have always been excluded from this) however this index has largely been superceded by indexes on more specific node types (for example `damAssetLucene-*` for the 'dam:Asset' nodetype) which support both fulltext and property searches.

In AEM 6.5 the 'generic lucene' index was marked as deprecated (indicated that it would be removed in future versions), and since then, a WARN has been logged when the index has been used:

```
org.apache.jackrabbit.oak.plugins.index.lucene.LucenePropertyIndex This index is deprecated: /oak:index/lucene-2; it is used for query Filter(query=select [jcr:path], [jcr:score], * from [nt:base] as a where contains(*, 'search term') and isdescendantnode(a, '/content/mysite') /* xpath: /jcr:root/content/mysite//*[jcr:contains(.,"search term")] */ fullText="search" "term", path=/content/mysite//*). Please change the query or the index definitions.
```

In recent AEM versions, the 'generic lucene' index has been used to support a very small number of features. These are being reworked to use other indexes or otherwise modified to remove the dependency on this index.
For example, 'reference lookup' queries, of the form shown below, should now be using the index at '/oak:index/pathreference' (which indexes only String property values which match a regular expression which looks for JCR paths). 

```
//*[jcr:contains(., '"/content/dam/mysite"')]
```

In order to support larger customer data volumes, Adobe will no longer be creating the 'generic lucene' index on new AEM as a Cloud Service environments and, following this, will begin removing the index from existing repositories. We have already adjusted the index costings (via the 'costPerEntry' and 'costPerExecution' properties) to ensure that other indexes (such as `/oak:index/pathreference`) are used in preference to `/oak:index/lucene-*` wherever possible. 

Customer applications which use queries which are still depending on this index should be updated immediately to leverage other existing indexes (which can be customized if required) or new custom indexes should be added to the customer application. Full instructions for index management in AEM as a Cloud Service can be found at the [indexing documentation](/help/operations/indexing.md).

## How to tell if your application depends on the 'generic lucene' index?

The 'generic lucene' index is currently used as a 'fallback' if no other fulltext index can service a query. When this deprecated index is used, a message like this will be logged at WARN level:

```
org.apache.jackrabbit.oak.plugins.index.lucene.LucenePropertyIndex This index is deprecated: /oak:index/lucene-2; it is used for query Filter(query=select [jcr:path], [jcr:score], * from [nt:base] as a where contains(*, 'test') /* xpath: //*[jcr:contains(.,"test")] */ fullText="test", path=*). Please change the query or the index definitions.
```

In some circumstances Oak might attempt to use another fulltext index (such as `/oak:index/pathreference`) to support the fulltext query, but if the query string does not match the regular expression on the index definition, a message will be logged at WARN level and the query will likely not return results.

```
org.apache.jackrabbit.oak.query.QueryImpl Potentially improper use of index /oak:index/pathReference with queryFilterRegex (["']|^)/ to search for value "test"
```

Once the 'generic lucene' index has been removed, a message as shown below will be logged at WARN level if a fulltext query is not able to locate any suitable index definition:

```
org.apache.jackrabbit.oak.query.QueryImpl Fulltext query without index for filter Filter(query=select [jcr:path], [jcr:score], * from [nt:base] as a where contains(*, 'test') /* xpath: //*[jcr:contains(.,"test")] */ fullText="test", path=*); no results will be returned
```

If any of these are logged, you might need to rework the query to use a different fulltext index, or provide a new index to support the query. Details of the types of dependencies you might see, and how to address them, are provided below.

## Potential dependencies on 'generic lucene' index

### Publish

#### Custom application queries

The most common source of queries using the 'generic lucene' index on Publish will be custom application queries.

In the simplest cases these might be queries with no nodetype specified (implying 'nt:base') or nt:base specified explicitly, such as:

```
/jcr:root/content/mysite//*[jcr:contains(., 'search term')]
/jcr:root/content/mysite//element(*, nt:base)[jcr:contains(., 'search term')]
```

Action Required : These queries can be modified to use an appropriate nodetype - for example, to return results matching pages (or any of the 'aggregates' beneath the cq:Page node) the query could become:

```
/jcr:root/content/mysite//element(*, cq:Page)[jcr:contains(., 'search term')]
```

In other cases, a query might specify a nodetype but contain a fulltext restriction that cannot be handled by another fulltext index, such as:

```
/jcr:root/content/dam//element(*, dam:Asset)[jcr:contains(jcr:content/metadata/@cq:tags, 'NewsTopics:cateogries/domestic'))]
```

In this case the query has the 'dam:Asset' nodetype, but contains a fulltext restriction on the relative `jcr:content/metadata/@cq:tags` property.

This property is not marked as 'analyzed' in the damAssetLucene index (the fulltext index most commonly used for queries against the 'dam:Asset' nodetype) so this index cannot be used for this query.

As such, the query falls back on the 'generic fulltext' index where all the included properties are marked as analysed by the wildcard match at `/oak:index/lucene-2/indexRules/nt:base/properties/prop`.

Customer Action Required : marking the `jcr:content/metadata/@cq:tags` property as 'analyzed' in a custom version of the damAssetLucene index will result in this query being handled by this index, and no WARN will be logged.

### Author 

In addition to queries in customer application servlets, OSGI components and rendering scripts there can be a number of author-specific usages of the 'generic lucene' index. 

#### Reference search

Historically the 'generic lucene' index has been used to support 'reference search' (searching for content which contains references to another content path). Such queries should already have moved to use the new '/oak:index/pathreference' index.
Customer Action Required : no customer action required.


#### Pathfield picker search

AEM includes a custom dialog component (Sling resource type 'granite/ui/components/coral/foundation/form/pathfield') which provides a browser (picker) for selecting another AEM path.  The default pathfield picker (used when no custom 'pickerSrc' property defined in content structure) renders a search bar in the popup dialog box.
The node types against which to search-against can be specified using the 'nodeTypes' property.

At present, if no 'nodeTypes' property present, the underlying search query will use the 'nt:base' nodetype, and thus is likely to use the 'generic lucene' index (typically logging the WARN messages shown below).

```
20.01.2022 18:56:06.412 *WARN* [127.0.0.1 [1642704966377] POST /mnt/overlay/granite/ui/content/coral/foundation/form/pathfield/picker.result.single.html HTTP/1.1] org.apache.jackrabbit.oak.plugins.index.lucene.LucenePropertyIndex This index is deprecated: /oak:index/lucene-2; it is used for query Filter(query=select [jcr:path], [jcr:score], * from [nt:base] as a where contains(*, 'test') and isdescendantnode(a, '/content') /* xpath: /jcr:root/content//element(*, nt:base)[(jcr:contains(., 'test'))] order by @jcr:score descending */ fullText="test", path=/content//*). Please change the query or the index definitions.
```

Prior to removal of 'generic lucene', the pathfield component will be updated so that the search box is hidden for components using the default picker, which do not provide a 'nodeTypes' property.

|Pathfield Picker with Search|Pathfield Picker without Search|
|---|---|
|![Pathfield Picker with Search](assets/index-pathfield-picker-with-search.png)|![Pathfield Picker without Search](assets/index-pathfield-picker-without-search.png)|


Customer Action Required : If no search is required, no action is required from the customer.

If the customer would like to retain the Search functionality within the pathfeld picker, a property 'nodeTypes' should be provided listing the node types against which they would like to query. These can be specified as a comma-separated list of nodetypes in a String property.


  >[!NOTE]
  >The Content Fragment Model Editor uses a specialised pathfields with Sling Resource Type 'dam/cfm/models/editor/components/contentreference'.
  > * At present these perform queries without node types specified - resulting in a WARN being logged due to usage of the 'generic lucene' index.
  > * Instances of these components will soon automatically default to using 'cq:Page' and 'dam:Asset' nodetypes without further customer action.
  > * The 'nodeTypes' property can be added to override these default nodetypes. 




## Timeline for removing 'generic lucene'

Adobe will be taking a 2-phase approach to the removal of the 'generic lucene' index.

**Phase 1** (planned start January 31st, 2022): No longer create '/oak:index/lucene-*' on new AEM as a Cloud Service environments.

**Phase 2** (planned start March 31st, 2022) : Remove '/oak:index/lucene-*' index from new AEM as a Cloud Service environnments.

Adobe will monitor the log messages noted above and will attempt to contact customers who remain dependant on the 'generic lucene' index.

As a short term mitigation, if necessary Adobe will add custom index definitions directly to customer systems to prevent functional or performance issues as a result of the removal of the 'generic lucene' index.

In such cases, the customer will be provided with the updated index definition and advised to include this in future releases of their application via Cloud Manager.
