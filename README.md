# jigsaw-elasticsearch-gradle

https://github.com/Cosium/jigsaw-elasticsearch

https://github.com/elastic/elasticsearch/issues/28984

Run `./gradlew build` and it will fail with:


```
> Task :compileJava FAILED
/srv/git/cosium/jigsaw-elasticsearch-gradle/src/main/java/module-info.java:7: error: module not found: elasticsearch
    requires elasticsearch;
             ^
1 error
```

The issue comes from the fact that no elasticsearch packages match the following declared Java services:
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.Codec`
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.DocValuesFormat`
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.PostingsFormat` containing `org.apache.lucene.search.suggest.document.Completion50PostingsFormat`

Adding fake classes to `org.apache.lucene.codecs` and `org.apache.lucene.search.suggest.document` fix the issue:
- https://github.com/Cosium/elasticsearch/blob/e4cac84d72413d1abfaa0b975d8f0b6959310db5/server/src/main/java/org/apache/lucene/search/suggest/document/JigsawMarker.java
- https://github.com/Cosium/elasticsearch/blob/e4cac84d72413d1abfaa0b975d8f0b6959310db5/server/src/main/java/org/apache/lucene/codecs/JigsawMarker.java
