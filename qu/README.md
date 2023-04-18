nlp
===
# what is nlp

Natural language processing is an important direction in the 
field of computer science and artificial intelligence. With the
development of LLM, the application has been further expanded.
He studies the theory and method of communication between real 
people and computers. Natural language processing is a 
convergent discipline. fusion of language, computer Mathematics 
and many other subjects.

query understander we would call it as QU.
QU is a part of nlp, this part focus on nlp in searching  apps.


Natural language processing is a big topic, and here we focus 
on tasks related to the search part, so QU refers to this part 
of information inspection.

```mermaid
graph LR
    A[自然语言处理]
    A-->Z[基础]
    A-->Y[应用]
    Z-->D[文本挖掘]
    Z-->B[句法语义分词]
    Z-->C[信息抽取]
    Y-->E[机器翻译]
    Y-->F[信息检索]
    Y-->G[智能客服]
    Y-->H[对话系统]
    F-->I[分词]
    F-->J[词权重]
    F-->K[实体识别]
    F-->L[意图识别]
    F-->M[情感分类]
    F-->N[纠错]
    F-->O[query改写]
    F-->P[语义识别]
    F-->Q[知识图谱]
```

# Syntactic Semantic Analysis
For a given sentence, perform word segmentation,
part-of-speech tagging, named entity recognition, 
syntactic analysis, semantic role recognition, and 
polysemy disambiguation.

# information extraction
Extract important information from a given text, such as time, 
place, event, person, reason, result, number, date , currencies,
proper nouns, etc. Speaking human words is to extract when, where, to whom, what things
were done, How is the result. Design to named entity recognition,
causality extraction and other technologies.

# text mining
Including text clustering, classification, information extraction, 
summarization, sentiment analysis, and the expression interface 
(feature engineering) of the mined information and knowledge, and 
the way of interactive expression, mainly based on statistical 
machine learning

* [segment](seg/README.md)


[back](../README.md)