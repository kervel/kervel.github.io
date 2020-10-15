---
layout: post
title:  "Downloading a maven artifact including all its dependencies (recursively)"
date:   2020-10-15 20:45:21 +0200
categories: spark
---

Often, when working with apache spark in an airgapped environment, i need to
be able to quickly add a maven artifact as dependency for a spark job i'm
working on. Since my installation is airgapped, i can't use --packages which
would resolve the packages from the internet.

So i created a small python script that does two things (it uses maven):

* it downloads all jars for a certain artifact to the current working directory

* it creates an assembly jar (= 1 single big fat jar that contains the artifact and all its dependencies)


The script can be called like this:

```shell
python3 download_artifact.py org.apache.hadoop:hadoop-aws:3.2.0
```
This will download a lot of jars to the current working directory, but also a file called
*target/my-app-1.0-SNAPSHOT-jar-with-dependencies.jar* which is the fat jar.


and the source code goes as follows (download_artifact.py)


```python3
#!/usr/bin/python3

def write_pom(artifacts):
  preamble = """
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>com.mycompany.app</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0-SNAPSHOT</version>
  <build>
      <plugins>
      <plugin>
        <!-- NOTE: We don't need a groupId specification because the group is
             org.apache.maven.plugins ...which is assumed by default.
         -->
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id> <!-- this is used for inheritance merges -->
            <phase>package</phase> <!-- bind to the packaging phase -->
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      </plugins>
  </build>
  
  <dependencies>
  """
  postamble = """
  </dependencies>
  </project>
  """
  deps = ""
  for a in artifacts:
      descriptor = a.split(":")
      deps += "<dependency><groupId>" + descriptor[0] + "</groupId><artifactId>" + descriptor[1] + "</artifactId><version>" + ":".join(descriptor[2:]) + "</version></dependency>"
  import os
  if os.path.exists("pom.xml"):
      raise Exception("pom.xml already exists")
  with open("pom.xml", "w") as f:
      f.write(preamble + deps + postamble)

def run_maven():
    import os
    os.system("mvn dependency:copy-dependencies -DoutputDirectory=./")
    os.system("mvn package -DoutputDirectory=./")

def remove_pom():
    import os
    os.unlink("pom.xml")

import sys
write_pom([sys.argv[1]])
run_maven()
remove_pom()
```
