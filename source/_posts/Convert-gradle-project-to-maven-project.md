title: Convert gradle project to maven project ( pom.xml )
date: 2015-12-27 08:49:41
tags: [gradle, maven]
category: gradle
---

## 1. Add this to build.gradle
```
apply plugin: 'maven'

group = 'com.company.root'
// artifactId is taken by default, from folder name
version = '0.0.1-SNAPSHOT'

task writeNewPom << {
    pom {
        project {
            inceptionYear '2014'
            licenses {
                license {
                    name 'The Apache Software License, Version 2.0'
                    url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    distribution 'repo'
                }
            }
        }
    }.writeTo("pom.xml")
}
```
## 2. Run
```shell
	$ gradle writeNewPom 
```

## Reference
* http://stackoverflow.com/questions/12888490/gradle-build-gradle-to-maven-pom-xml
