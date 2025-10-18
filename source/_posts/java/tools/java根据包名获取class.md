---
title: java根据包名获取class
date: 2023-03-03 11:17:00:00
categories: 
    - Java
tags:
    - tools
---


通过指定一个包名获取该包名下所有的类，主要思路是遍历当前jvm依赖的jar包，以及引用的class文件目录，即java.class.path，在java中，我们可以使用System.getProperty获取其值，具体实现为：

```java
private List<Class<?>> getClassByPackageName(String packageName) {
    List<Class<?>> classes = new ArrayList<>();
    String packagePath = packageName.replace(".", "/");
    String[] classPathEntries = System.getProperty("java.class.path")
            .split(System.getProperty("path.separator"));
    for (String classpathEntry : classPathEntries) {
        if (classpathEntry.endsWith(".jar")) {
            File jar = new File(classpathEntry);
            try (FileInputStream fileInputStream = new FileInputStream(jar);
                    JarInputStream jarInputStream = new JarInputStream(fileInputStream)) {
                JarEntry entry;
                while ((entry = jarInputStream.getNextJarEntry()) != null) {
                    String name = entry.getName();
                    if (name.endsWith(".class")) {
                        if (name.contains(packagePath) && name.endsWith(".class")) {
                            String classPath = name.substring(0, entry.getName().length() - 6);
                            classPath = classPath.replaceAll("[|/]", ".");
                            classes.add(Class.forName(classPath));
                        }
                    }
                }
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        } else {
            try {
                File base = new File(classpathEntry + File.separatorChar + packagePath);
                // 使用队列进行文件搜素，也可以使用递归实现
                Queue<File> fileQueue = new LinkedList<>();
                fileQueue.offer(base);
                while (fileQueue.peek() != null) {
                    File currentFile = fileQueue.poll();
                    if (currentFile.isDirectory()) {
                        File[] subFiles = currentFile.listFiles();
                        if (subFiles != null) {
                            for (File subFile : subFiles) {
                                fileQueue.offer(subFile);
                            }
                        }
                    } else if (currentFile.isFile()) {
                        if (currentFile.getName().endsWith(".class")) {
                            String className = currentFile.getPath()
                                    .replace(classpathEntry + "/", "")
                                    .replace("/", ".");
                            className = className.substring(0, className.length() - 6);
                            classes.add(Class.forName(className));
                        }
                    }
                }
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
    return classes;
}
```

如果你的项目引用了spring依赖包，那么也可以使用ClassPathScanningCandidateComponentProvider 获取包名下的class。

```java
private List<Class<?>> getClassByPackageName(String packageName) {
        List<Class<?>> clazzs = new ArrayList<>();
        ClassPathScanningCandidateComponentProvider provider =
                new ClassPathScanningCandidateComponentProvider(false);
        provider.addIncludeFilter((metadataReader, metadataReaderFactory) -> {

            String className = metadataReader.getClassMetadata().getClassName();
            try {
                Class<?> clazz = Class.forName(className);
                clazzs.add(clazz);
            } catch (ClassNotFoundException e) {
                throw new BizCoreException(e.getMessage());
            }

            return true;
        });
        provider.findCandidateComponents(packageName);
        return clazzs;
}
```