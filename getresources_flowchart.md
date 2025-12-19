# GetResources() Flow Chart

```mermaid
flowchart TD
    Start([Start: GetResources]) --> Init[Initialize variables:<br/>- resources = empty array<br/>- shouldHaveDiscoveredResources = false]
    
    Init --> CheckFilters{Does service have<br/>ResourceFilters?}
    
    CheckFilters -->|Yes| SetFlag1[shouldHaveDiscoveredResources = true]
    SetFlag1 --> BuildFilters[Build filter list from<br/>svc.ResourceFilters]
    
    BuildFilters --> CheckSearchTags{Does job have<br/>SearchTags?}
    CheckSearchTags -->|Yes| BuildTagFilters[Build TagFilter array<br/>with SearchTag keys]
    CheckSearchTags -->|No| CreateInput
    BuildTagFilters --> CreateInput[Create GetResourcesInput<br/>with filters and TagFilters]
    
    CreateInput --> CreatePaginator[Create paginator for<br/>AWS GetResources API]
    
    CreatePaginator --> HasMore{paginator.<br/>HasMorePages?}
    HasMore -->|Yes| CallAPI[Call NextPage API<br/>Increment counter]
    
    CallAPI --> CheckError1{Error?}
    CheckError1 -->|Yes| ReturnError1([Return nil, error])
    CheckError1 -->|No| ProcessPage[Process ResourceTagMappingList]
    
    ProcessPage --> LoopResources[For each resource mapping:<br/>- Create TaggedResource<br/>- Copy ARN, Namespace, Region<br/>- Copy all tags]
    
    LoopResources --> FilterTags{resource.FilterThroughTags<br/>matches SearchTags?}
    FilterTags -->|Yes| AddResource[Append to resources array]
    FilterTags -->|No| LogSkip[Log: Skipping resource]
    
    AddResource --> HasMore
    LogSkip --> HasMore
    
    HasMore -->|No| LogDebug1[Log: GetResourcesPages finished]
    
    CheckFilters -->|No| CheckExtension
    LogDebug1 --> CheckExtension{ServiceFilters has<br/>extension for namespace?}
    
    CheckExtension -->|Yes| CheckResourceFunc{Extension has<br/>ResourceFunc?}
    CheckExtension -->|No| CheckShouldHave
    
    CheckResourceFunc -->|Yes| SetFlag2[shouldHaveDiscoveredResources = true]
    SetFlag2 --> CallResourceFunc[Call ext.ResourceFunc]
    
    CallResourceFunc --> CheckError2{Error?}
    CheckError2 -->|Yes| ReturnError2([Return error])
    CheckError2 -->|No| AppendNew[Append newResources to resources]
    
    AppendNew --> LogDebug2[Log: ResourceFunc finished]
    
    CheckResourceFunc -->|No| CheckFilterFunc
    LogDebug2 --> CheckFilterFunc{Extension has<br/>FilterFunc?}
    
    CheckFilterFunc -->|Yes| CallFilterFunc[Call ext.FilterFunc<br/>with resources]
    CheckFilterFunc -->|No| CheckShouldHave
    
    CallFilterFunc --> CheckError3{Error?}
    CheckError3 -->|Yes| ReturnError3([Return error])
    CheckError3 -->|No| ReplaceResources[Replace resources with<br/>filteredResources]
    
    ReplaceResources --> LogDebug3[Log: FilterFunc finished]
    LogDebug3 --> CheckShouldHave
    
    CheckShouldHave{shouldHaveDiscoveredResources<br/>is true AND<br/>len(resources) == 0?}
    
    CheckShouldHave -->|Yes| ReturnExpectedError([Return ErrExpectedToFindResources])
    CheckShouldHave -->|No| ReturnSuccess([Return resources, nil])
    
    style Start fill:#90EE90
    style ReturnSuccess fill:#90EE90
    style ReturnError1 fill:#FFB6C6
    style ReturnError2 fill:#FFB6C6
    style ReturnError3 fill:#FFB6C6
    style ReturnExpectedError fill:#FFB6C6
```

## Key Logic Points

1. **Resource Discovery Phase 1 - AWS Tagging API**
   - If service has ResourceFilters, query AWS Resource Groups Tagging API
   - Apply SearchTags as filters to reduce API response size
   - Paginate through all results
   - Filter results locally using FilterThroughTags

2. **Resource Discovery Phase 2 - Custom ResourceFunc**
   - If service has custom ResourceFunc, call it to get additional resources
   - Append these to existing resources

3. **Filtering Phase - Custom FilterFunc**
   - If service has custom FilterFunc, apply it to all resources
   - Replaces the resources array with filtered results

4. **Validation**
   - If we expected to find resources but found none, return error
   - Otherwise return the discovered resources

