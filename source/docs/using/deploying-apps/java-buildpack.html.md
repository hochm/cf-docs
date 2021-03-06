---
title: Java Buildpack
---
For detailed information about using, configuring, and extending the Cloud Foundry Java buildpack, see <https://github.com/cloudfoundry/java-buildpack>.

## <a id='design'></a>Design ##
The Java Buildpack is designed to convert artifacts that run on the JVM into executable applications.  It does this by identifying one of the supported artifact types (Grails, Groovy, Java, Play Framework, Spring Boot, and Servlet) and downloading all additional dependencies needed to run.  The collection of services bound to the application is also analyzed and any dependencies related to those services are also downloaded.

As an example, pushing a WAR file that is bound to a PostgreSQL database and New Relic (for performance monitoring) would result in the following:

<pre class="terminal">
Initialized empty Git repository in /tmp/buildpacks/java-buildpack/.git/
-----> Java Buildpack source: https://github.com/cloudfoundry/java-buildpack#0928916a2dd78e9faf9469c558046eef09f60e5d
-----> Downloading Open Jdk JRE 1.7.0_51 from http://.../openjdk/lucid/x86_64/openjdk-1.7.0_51.tar.gz (0.0s)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.9s)
-----> Downloading New Relic Agent 3.4.1 from http://.../new-relic/new-relic-3.4.1.jar (0.4s)
-----> Downloading Postgresql JDBC 9.3.1100 from http://.../postgresql-jdbc/postgresql-jdbc-9.3.1100.jar (0.0s)
-----> Downloading Spring Auto Reconfiguration 0.8.7 from http://.../auto-reconfiguration/auto-reconfiguration-0.8.7.jar (0.0s)
       Modifying /WEB-INF/web.xml for Auto Reconfiguration
-----> Downloading Tomcat 7.0.50 from http://.../tomcat/tomcat-7.0.50.tar.gz (0.0s)
       Expanding Tomcat to .java-buildpack/tomcat (0.1s)
-----> Downloading Buildpack Tomcat Support 1.1.1 from http://.../tomcat-buildpack-support/tomcat-buildpack-support-1.1.1.jar (0.1s)
-----> Uploading droplet (57M)
</pre>

## <a id='configuration'></a>Configuration ##
The buildpack is designed such that it will "just work" without any configuration.  If you are new to Cloud Foundry you should make your first attempts without modifying the buildpack configuration.  However, if you find that some configuration is required, it can be done with [a fork of the buildpack][f].

## <a id='extension'></a>Extension ##
The Java Buildpack is also designed to be easily extended.  It creates abstractions for [three types of components][a] (containers, frameworks, and JREs) in order to allow users to easily add (and contribute back) functionality.  In addition to these abstractions, there are a number of [utility classes][e] for simplifying typical buildpack behaviors.

As an example, the New Relic framework looks like the following:

```ruby
class NewRelicAgent < JavaBuildpack::Component::VersionedDependencyComponent

  # @macro base_component_compile
  def compile
    FileUtils.mkdir_p logs_dir

    download_jar
    @droplet.copy_resources
  end

  # @macro base_component_release
  def release
    @droplet.java_opts
    .add_javaagent(@droplet.sandbox + jar_name)
    .add_system_property('newrelic.home', @droplet.sandbox)
    .add_system_property('newrelic.config.license_key', license_key)
    .add_system_property('newrelic.config.app_name', "'#{application_name}'")
    .add_system_property('newrelic.config.log_file_path', logs_dir)
  end

  protected

  # @macro versioned_dependency_component_supports
  def supports?
    @application.services.one_service? FILTER, 'licenseKey'
  end

  private

  FILTER = /newrelic/.freeze

  def application_name
    @application.details['application_name']
  end

  def license_key
    @application.services.find_service(FILTER)['credentials']['licenseKey']
  end

  def logs_dir
    @droplet.sandbox + 'logs'
  end

end
```

[a]: https://github.com/cloudfoundry/java-buildpack/blob/master/docs/design.md
[e]: https://github.com/cloudfoundry/java-buildpack/blob/master/docs/extending.md
[f]: https://github.com/cloudfoundry/java-buildpack#configuration-and-extension
