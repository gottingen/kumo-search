engine design
===

# what it to be
lambda design to be the kernel of search. analogous to `lucene`, it
provide the most basic search capability. In fact, it provides a 
framework of a basic search engine, and continues to expand upwards
to provide business-customized search services, such as distributed,
fragmented, and replica consistency. Downward, continuously 
optimize optimization components such as scoring components, 
sorting components, and continuously improve local effects.

# architecture

```mermaid
graph BT
    A[api]
    B[sercher]-->A
    P[table manager]-->A
    C[query]-->A
    E[scorer]-->B
    F[sortor]-->B
    D[comparator]-->G[table]
    G-->B
    H[storage]-->G
    I[relevance scorer]-->E
    J[user filter plugin ...]-->E
    K[invert list storage]-->H
    L[tight forward storage]-->H
    M[raw doc storage]-->H
    N[column forward storage]-->H
    O[vector storage]-->H
    Q[table reader] --> B
    R[table builder] -->P
    G-->Q
    G-->R
    S[relevance sortor]-->F
    T[ctr sortor]-->F
    U[table snapshot sync]-->P
```

