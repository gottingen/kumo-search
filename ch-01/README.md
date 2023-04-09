searching architecture
====
```mermaid
sequenceDiagram
    participant User
　　 participant Searcher
    participant QU
    participant Ranker
    participant Engine
　　 User->>Searcher: 1
　　 Searcher->>QU: 2
    QU->>Searcher: 3
    Searcher->>Engine: 4
    Engine->>Searcher: 5
    Searcher->>Ranker: 6
    Ranker->>Searcher: 7
    Searcher->>User: 8
```