Chinese Word Segmentation -- CWS
===

# what is `cws`
`cws` It refers to dividing a sequence of Chinese characters 
into a sequence of words, which is converted from one sequence 
to another.


# CWS algorithm

```mermaid
graph LR
A[CWS]

A --> B[vocabulary and rules]
A --> C[Statistical Model]
A --> D[Sequence Tags]
B-->E[Forward Maximum Match]
B-->F[inverse maximum matching]
B-->G[two-way maximum matching]
C-->H[n-gram]
D-->I[HMM]
D-->J[CRF]
D-->K[Deep Learning End-to-End]
```

# tools

## [libtext](https://github.com/gottingen/libtext)

this repo using jieba c++ as a baseline.