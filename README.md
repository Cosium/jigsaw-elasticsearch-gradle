# jigsaw-elasticsearch

Run `mvn clean package` and it will fail with:


```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.7.0:compile (default-compile) on project jigsaw-elasticsearch: Compilation failure
[ERROR] /srv/git/cosium/jigsaw-elasticsearch/src/main/java/module-info.java:[7,14] module not found: elasticsearch
```

Debugging `maven-compiler-plugin` lead to `jdk.internal.module.ModulePath`.
It fails in `jdk.internal.module.ModulePath` at line 557:
```java
// scan the names of the entries in the JAR file
        Map<Boolean, Set<String>> map = VersionedStream.stream(jf)
                .filter(e -> !e.isDirectory())
                .map(JarEntry::getName)
                .filter(e -> (e.endsWith(".class") ^ e.startsWith(SERVICES_PREFIX)))
                .collect(Collectors.partitioningBy(e -> e.startsWith(SERVICES_PREFIX),
                                                   Collectors.toSet()));

        Set<String> classFiles = map.get(Boolean.FALSE);
        Set<String> configFiles = map.get(Boolean.TRUE);

        // the packages containing class files
        Set<String> packages = classFiles.stream()
                .map(this::toPackageName)
                .flatMap(Optional::stream)
                .distinct()
                .collect(Collectors.toSet());

        // all packages are exported and open
        builder.packages(packages);

        // map names of service configuration files to service names
        Set<String> serviceNames = configFiles.stream()
                .map(this::toServiceName)
                .flatMap(Optional::stream)
                .collect(Collectors.toSet());

// parse each service configuration file
        for (String sn : serviceNames) {
            JarEntry entry = jf.getJarEntry(SERVICES_PREFIX + sn);
            List<String> providerClasses = new ArrayList<>();
            try (InputStream in = jf.getInputStream(entry)) {
                BufferedReader reader
                    = new BufferedReader(new InputStreamReader(in, "UTF-8"));
                String cn;
                while ((cn = nextLine(reader)) != null) {
                    if (cn.length() > 0) {
                        String pn = packageName(cn);
                        if (!packages.contains(pn)) {
                            String msg = "Provider class " + cn + " not in module";
                            throw new InvalidModuleDescriptorException(msg); // <-- FAILING HERE
                        }
                        providerClasses.add(cn);
                    }
                }
            }
            if (!providerClasses.isEmpty())
                builder.provides(sn, providerClasses);
        }
```

The issue comes from the fact that no elasticsearch packages match the following declared Java services:
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.Codec`
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.DocValuesFormat`
- `server/src/main/resources/META-INF/services/org.apache.lucene.codecs.PostingsFormat` containing `org.apache.lucene.search.suggest.document.Completion50PostingsFormat`

Adding fake classes to `org.apache.lucene.codecs` and `org.apache.lucene.search.suggest.document` fix the issue:
- https://github.com/Cosium/elasticsearch/blob/e4cac84d72413d1abfaa0b975d8f0b6959310db5/server/src/main/java/org/apache/lucene/search/suggest/document/JigsawMarker.java
- https://github.com/Cosium/elasticsearch/blob/e4cac84d72413d1abfaa0b975d8f0b6959310db5/server/src/main/java/org/apache/lucene/codecs/JigsawMarker.java
