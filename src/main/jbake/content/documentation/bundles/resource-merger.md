title=Resource Merger (org.apache.sling.resourcemerger)		
type=page
status=published
~~~~~~
[TOC]


# Introduction
**This documentation only applies to versions >= 1.2 (due to major changes done in [SLING-3423](https://issues.apache.org/jira/browse/SLING-3423))**

The Resource Merger bundle provides two resource provider factories

* `MergingResourceProvider` (only for reading)
* `CRUDMergingResourceProvider` (for reading and writing)

Those providers give access to virtual resources which provide a merged view on multiple other resources.

Each `MergedResourcePicker` service implementation in the system provides one unique mount point/root path (usually starting with `/mnt`) exposing a view on merged resources from at least two different locations. 

The exact merging mechanism depends on the resource picker implementation (see below). The order of the merging is important as the overlying resource may overwrite properties/child resources from the underlying resource (but not vice-versa). 
It is possible to

* remove existing resource/properties from the underlying resources,
* modify existing properties/child resources of the underlying resources and
* add new properties/child resources

You may use any of the merge properties described below to influence the merging behaviour. All those special properties are not exposed below the mount point. 

The `CRUDMergingResourceProvider` not only gives read-access to the merged resources but even allows to write. This provider always writes inside the topmost resource path (being returned as last item by the resource picker). Currently both resource pickers shipped with the Sling Resource Merger bundle do not allow to write though. Further details can be found in the description of [SLING-3909](https://issues.apache.org/jira/browse/SLING-3909).


# Merge Properties

Property Name | Type | Description
------------- | ---- | ---------
sling:hideProperties | String[] | Hides the properties with the given names. `*` hides all properties (since version 1.3.2 the wildcard only affects underlying properties and no longer local ones, see also [SLING-5468](https://issues.apache.org/jira/browse/SLING-5468)).
sling:hideChildren| String[] | Hides the child resources with the given names. `*` hides all child resources (since version 1.3.2 the wildcard only affects underlying child resources and no longer local ones, see also [SLING-5468](https://issues.apache.org/jira/browse/SLING-5468)). If one value starts with `!` this is a negation (which means the property with the given value should not be hidden). Since by default nothing is hidden the negation is only useful if you specify multiple values for this property. E.g. giving the values `[!child1,*]` hides all children except for the one with the name `child1`. If you have a resource name starting with `!` you must escape it with an additional `!` in front if you want to reference it in `sling:hideChildren`, e.g. `!!child1` means that the resource with the name `!child1` should be hidden.
sling:hideResource | Boolean | If `true` then the resource with the name which contains this property should not be exposed!
sling:orderBefore | String | Contains the name of the preceding sibling resource. This is influencing the order of resources when calling e.g. `Resource.listChildren()` or `Resource.getChildren()` on the merged resource. This is only necessary if the default child resource order is not sufficient (see below).

# Child Resource Order

For a merged resource the order of its child resources is the following:
First the ones from the underlying resource, then the ones from the overlying resources.

In case the same child is defined in more than one resource, its position is taken from the highest overlaying resource (since version 1.3.2, see also [SLING-4915](https://issues.apache.org/jira/browse/SLING-4915), for some more examples for more complicated merges look at [SLING-6956](https://issues.apache.org/jira/browse/SLING-6956)).
For example:
    
    base/
    	+--child1
    	+--child3
    	+--child5
    	
    overlay/
    	+--child2
    	+--child3
    	+--child4
    	
    resulting order: child1, child5, child2, child3, child4

You can overwrite that behaviour by leveraging the `sling:orderBefore` property.

# Resource Pickers

## Merging Resource Picker (Overlay approach)

 Description | Merged Resource Path | Merging Order | Read-Only | Related Ticket 
------------ | -------------------- | ------------- | --------- | ---------------
Merging based on the resource resolver's search path | `/mnt/overlay/<relative resource path>` | Last resource resolver search path first (Order = Descending search paths). | `true` |  [SLING-2986](http://issues.apache.org/jira/browse/SLING-2986) 

This picker is thought for globally overlaying content by placing the diff-resource in "/apps" without actually touching anything in "/libs" (since this is only thought for Sling itself). This should be used whenever some deviation of content is desired which is initially shipped with Sling. The client (i.e. the code using the merged resource) does not have to know if there is an overlay in place.
 

### Example

    /libs/sling/example (nt:folder)
         +-- sling:resourceType = "some/resource/type"
         +-- child1 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child1"
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child3"
         
    
    /apps/sling/example (sling:Folder)
         +-- property1 = "property added in apps"
         +-- child1 (nt:folder)
         |   +-- sling:hideResource = true
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /apps/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property2 = "property from /apps/sling/example/child3"
    
    
    /mnt/overlay/sling/example (sling:Folder)
         +-- sling:resourceType = "some/resource/type"
         +-- property1 = "property added in apps"
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /apps/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child3"
         |   +-- property2 = "property from /apps/sling/example/child3"



## Overriding Resource Picker (Override approach)

Description | Merged Resource Path | Merging Order | Read-Only | Related Ticket
----------- | -------------------- | ------------- | --------- | --------------
Merging based on the `sling:resourceSuperType` | `/mnt/override/<absolute resource type>` | The topmost resource type (having itself no `sling:resourceSuperType` defined) is the base. The resources are overlaid in the order from the most generic one to the most specific one (which is the one having the most inheritance levels). | `true` | [SLING-3675](https://issues.apache.org/jira/browse/SLING-3657)

This picker is thought for extending content of another already existing resource (which should not be modified). The client (i.e. the code using the merged resource) must directly reference the most specific resource type (i.e. it must be aware of the sub resource type).

### Example

    /apps/sling/base (nt:folder)
         +-- child1 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child1"
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child3"
         
    
    /apps/sling/example (sling:Folder)
    	 +-- sling:resourceSuperType = "/apps/sling/base"
         +-- property1 = "property added in /apps/sling/example"
         +-- child1 (nt:folder)
         |   +-- sling:hideResource = true
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /apps/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property2 = "property from /apps/sling/example/child3"
    
    
    /mnt/override/apps/sling/example (sling:Folder)
         +-- sling:resourceSuperType = "/apps/sling/base"
         +-- property1 = "property added in /apps/sling/example"
         +-- child2 (nt:folder)
         |   +-- property1 = "property from /apps/sling/example/child2"
         +-- child3 (nt:folder)
         |   +-- property1 = "property from /libs/sling/example/child3"
         |   +-- property2 = "property from /apps/sling/example/child3"


