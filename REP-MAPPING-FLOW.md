# Rep Name Mapping Flow

This diagram shows how the rep name mapping system works in the ConnectWise CPQ MCP server.

```mermaid
flowchart TD
    Start([User Query via Teams]) --> Agent[Copilot Studio Agent]
    Agent --> Connector[Power Platform Connector]
    Connector --> AzureFunc[Azure Function]

    AzureFunc --> MCP[MCP Server]
    MCP --> CPQService[CPQ Service]

    CPQService --> API[ConnectWise CPQ API]
    API --> RawData[Raw Quote Data with Rep GUIDs]

    RawData --> RepMapper{Rep Mapping Exists?}

    RepMapper -->|Yes| Lookup[rep-mapping.ts Lookup]
    RepMapper -->|No| ShowGUID[Return GUID as-is]

    Lookup --> RepName[Return Rep Name]

    RepName --> Response[Formatted Response]
    ShowGUID --> Response

    Response --> MCP2[MCP Server Response]
    MCP2 --> AzureFunc2[Azure Function Response]
    AzureFunc2 --> Connector2[Power Platform]
    Connector2 --> Agent2[Copilot Studio]
    Agent2 --> User([User sees Rep Names])

    style RepMapper fill:#ff9999
    style Lookup fill:#99ff99
    style ShowGUID fill:#ffff99
    style RepName fill:#99ff99
```

## Setup Process

```mermaid
flowchart LR
    A[Generate CSV Template] --> B[scripts/generate-rep-csv.ts]
    B --> C[rep-mapping-template.csv]

    C --> D[Manual: Fill in Names]
    D --> E[Open in Excel/Sheets]
    E --> F[Look up GUIDs in CPQ]
    F --> G[Add Names to CSV]

    G --> H[Import CSV]
    H --> I[scripts/import-rep-csv.ts]
    I --> J[src/config/rep-mapping.ts]

    J --> K[Build & Deploy]
    K --> L[npm run build]
    L --> M[func azure functionapp publish]

    M --> N[Rep Names Live in Production]

    style D fill:#ff9999
    style F fill:#ff9999
    style N fill:#99ff99
```

## Cache Warmth Detection

```mermaid
flowchart TD
    Start([Recurring Revenue Query]) --> CacheCheck{Check Query Cache}

    CacheCheck -->|Cache Hit| ReturnCached[Return Cached Results]
    CacheCheck -->|Cache Miss| FetchQuotes[Fetch Quotes from API]

    FetchQuotes --> QuoteCache{Check Quote Cache}
    QuoteCache -->|Cached| LoadQuotes[Load from Cache]
    QuoteCache -->|Not Cached| APIQuotes[Fetch from API]

    APIQuotes --> CacheQuotes[Cache Quotes for 60 min]
    CacheQuotes --> LoadQuotes
    LoadQuotes --> Sample[Sample First 10 Quotes]

    Sample --> CheckItems{Check Item Cache}
    CheckItems --> Count[Count Cached Items]

    Count --> Warmth{Cache Warmth > 50%?}

    Warmth -->|Yes - Warm| Limit300[Check up to 300 Quotes]
    Warmth -->|No - Cold| Limit100[Check up to 100 Quotes]

    Limit300 --> ProcessBatch
    Limit100 --> ProcessBatch[Process in Batches of 20]

    ProcessBatch --> ParallelFetch[Parallel Item Fetch]
    ParallelFetch --> ItemCache{Item Cached?}

    ItemCache -->|Yes| UseCached[Use Cached Items]
    ItemCache -->|No| FetchItems[Fetch from API]

    FetchItems --> CacheItems[Cache Items for 60 min]
    CacheItems --> UseCached

    UseCached --> Filter[Filter Recurring Items]
    Filter --> Calculate[Calculate Recurring Revenue]
    Calculate --> Results[Collect Results]

    Results --> Sort[Sort by Revenue]
    Sort --> CacheResult[Cache Full Result]
    CacheResult --> Return[Return to User]

    ReturnCached --> Return

    style Warmth fill:#ff9999
    style Limit300 fill:#99ff99
    style Limit100 fill:#ffff99
    style UseCached fill:#99ff99
    style FetchItems fill:#ffcccc
```

## 3-Level Caching Strategy

```mermaid
flowchart TD
    Query[User Query] --> L1{Level 1: Query Cache}

    L1 -->|Hit| Return1[Return Full Results]
    L1 -->|Miss| L2[Level 2: Quote Metadata Cache]

    L2 --> Key2["Cache Key: quotes + dateFrom + dateTo"]
    Key2 --> Check2{Cached?}

    Check2 -->|Hit| LoadQuotes[Load Quote Metadata]
    Check2 -->|Miss| FetchQuotes[Fetch from API]

    FetchQuotes --> Store2[Store in Cache - 60 min TTL]
    Store2 --> LoadQuotes

    LoadQuotes --> L3[Level 3: Quote Items Cache]

    L3 --> ProcessQuotes[For Each Quote]
    ProcessQuotes --> Key3["Cache Key: items + quoteId"]

    Key3 --> Check3{Items Cached?}

    Check3 -->|Hit| LoadItems[Load Items from Cache]
    Check3 -->|Miss| FetchItems[Fetch Items from API]

    FetchItems --> Store3[Store in Cache - 60 min TTL]
    Store3 --> LoadItems

    LoadItems --> FilterRecurring[Filter Recurring Items]
    FilterRecurring --> CalcRevenue[Calculate Revenue]

    CalcRevenue --> AllDone{All Quotes Processed?}
    AllDone -->|No| ProcessQuotes
    AllDone -->|Yes| FinalResults[Compile Results]

    FinalResults --> StoreL1[Store in Level 1 Cache]
    StoreL1 --> Return2[Return to User]

    style L1 fill:#ffcccc
    style L2 fill:#ffeecc
    style L3 fill:#ffffcc
    style Check2 fill:#ccffcc
    style Check3 fill:#ccffcc
    style Return1 fill:#99ff99
    style Return2 fill:#99ff99
```

## Priority Mapping Order

```mermaid
graph TD
    Start[48 Unique Rep GUIDs] --> Top10[Top 10 Reps]

    Top10 --> R1[Rep 1: 490 quotes - 90% coverage]
    Top10 --> R2[Rep 2: 143 quotes]
    Top10 --> R3[Rep 3: 133 quotes]
    Top10 --> R4[Rep 4: 94 quotes]
    Top10 --> R5[Rep 5: 68 quotes]
    Top10 --> R6[Rep 6: 57 quotes]
    Top10 --> R7[Rep 7: 46 quotes]
    Top10 --> R8[Rep 8: 37 quotes]
    Top10 --> R9[Rep 9: 34 quotes]
    Top10 --> R10[Rep 10: 29 quotes]

    R1 --> MapFirst[Map These First]
    R2 --> MapFirst
    R3 --> MapFirst
    R4 --> MapFirst
    R5 --> MapFirst
    R6 --> MapFirst
    R7 --> MapFirst
    R8 --> MapFirst
    R9 --> MapFirst
    R10 --> MapFirst

    MapFirst --> Coverage[90% Quote Coverage]

    Start --> Remaining[Remaining 38 Reps]
    Remaining --> MapLater[Map Gradually as Needed]
    MapLater --> LowerCoverage[10% Quote Coverage]

    Coverage --> Deploy[Deploy Mapping]
    LowerCoverage --> Deploy

    style R1 fill:#ff9999
    style MapFirst fill:#99ff99
    style Coverage fill:#99ff99
```

## Key Insights

### Why Manual Mapping is Required
- ConnectWise CPQ API doesn't have a user/rep lookup endpoint
- Only quote objects contain rep GUIDs (e.g., `primaryRep`, `insideRep`, `solutionsArchitect`)
- No way to programmatically resolve GUID → Name

### Cache Benefits
- **Level 1** (Query Cache): Instant responses for repeated queries (same date range + filters)
- **Level 2** (Quote Metadata): Avoid re-fetching quotes for same date range
- **Level 3** (Item Cache): Massively reduces API calls when checking multiple quotes

### Performance with Caching
- **Cold cache**: 100 quotes in ~5-8 seconds (100 API calls for items)
- **Warm cache**: 300 quotes in ~5-8 seconds (0 API calls for cached items)
- **Full cache hit**: Instant response (no API calls)

### Priority Mapping Strategy
- Focus on top 10 reps first (they appear in 90% of quotes)
- Remaining 38 reps can be added gradually
- Unmapped GUIDs display as-is until mapped
