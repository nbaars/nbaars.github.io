---
layout: post
title: Keep vulnerable libraries out!
categories: [Security]
---

Modern applications development is a mix of custom code and many pieces of open source. The developer is normally 
very knowledgeable about their custom code but less familiar with the potential risk of the libraries/components 
they use. 

A study from Black Duck which covers more than 200 applications shows that 95% of the projects use open
source libraries ([see open source security analysis](https://info.blackducksoftware.com/rs/872-OLS-526/images/OSSAReportFINAL.pdf)). 
Important side note is we only use a fraction of all the 
libraries imported into a project. Last couple of months some [critical issues](https://cwiki.apache.org/confluence/display/WW/S2-045) were found in Struts 2 which 
enabled attackers to perform a remote-code-execution through a malicious content-type. One way to track whether 
you are using vulnerable components is to use the [OWASP Dependency-Check](https://www.owasp.org/index.php/OWASP_Dependency_Check). 
This tool uses the [National Vulnerability Database](https://nvd.nist.gov/vuln/search) to search components for well known published vulnerabilities.
 
 
## Example
 
 Let’s take a look at the well known Spring Pet Clinic project and integrate OWASP Dependency-Check, 
 first we add the following plugin to the Maven `pom.xml`:

```xml
<plugin>
 <groupId>org.owasp</groupId>
 <artifactId>dependency-check-maven</artifactId>
 <version>2.1.1</version>
   <configuration>
     <failBuildOnCVSS>8</failBuildOnCVSS>
   </configuration>
 <executions>
   <execution>
     <goals>
       <goal>check</goal>
     </goals>
   </execution>
 </executions>
</plugin>
```

The plugin supports all kind of configuration items, in this example our build will fail if the Common Vulnerability Scoring 
System (CVSS) score if above 8.

>CVSS is a free and open industry standard for assessing the severity of computer system security
>vulnerabilities. CVSS attempts to assign severity scores to vulnerabilities, allowing
>responders to prioritize responses and resources according to threat. Scores are calculated
>based on a formula that depends on several metrics that approximate ease of exploit and the impact of exploit. Scores range from
>0 to 10, with 10 being the most severe. While many utilize only the CVSS Base score for determining severity, temporal and
>environmental scores also exist, to factor in availability of mitigations and how widespread vulnerable systems are within
>an organization, respectively.

If we run dependency checker the build will fail due to:

```shell
[ERROR] Failed to execute goal org.owasp:dependency-check-maven:2.1.1:check (default) on project spring-petclinic:
[ERROR]
[ERROR] One or more dependencies were identified with vulnerabilities that have a CVSS score greater than '8.0':
[ERROR]
[ERROR] mysql-connector-java-5.1.42.jar: CVE-2016-6662, CVE-2016-0705, CVE-2016-0639, CVE-2014-6507, CVE-2012-3163
[ERROR]
```

The first time Dependency Checker it might take some time to because it will download
all the databases.
If we look at the report we will see the mysql connector has more than 400 CVEs and the build fails due to the CVSS score
above 8 (in this case there is even a CVE with CVSS score 10). Based on this score this library should be replaced with a more
up-to-date version.

## Integrations
   
Dependency checker supplies many ways to integrate into your build pipeline. For example it support Maven,
Gradle, SBT, you can run it using Docker. There is also a Sonar plugin which makes it even
more visible to your team.


## Don't forget

The plugin is only triggered during the build phase. It is also important to keep tracking your software after
it has been put into production. Make sure you run the plugin at least once a week and create some organisational
rules how to deal with new found vulnerabilities.

## Other tools

OWASP Dependency Checker is one of the tools you can use to scan your project for vulnerable components, there are
also a lot of commercial tools available, like Sonatype and snyk.io.

## References

1. https://cve.mitre.org/
2. https://nvd.nist.gov/vuln
3. https://nvd.nist.gov/vuln/data-feeds
4. https://jeremylong.github.io/DependencyCheck/index.html
5. https://github.com/spring-projects/spring-petclinic
6. https://nvd.nist.gov/vuln-metrics/cvss
7. https://info.blackducksoftware.com/rs/872-OLS-526/images/OSSAReportFINAL.pdf

