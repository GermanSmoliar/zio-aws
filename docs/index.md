---
id: index
title: "Overview"
---

[ZIO AWS](https://zio.dev/zio-aws) is AWS wrapper for [ZIO](https://zio.dev) for _all_ AWS services using the AWS Java SDK v2.

@PROJECT_BADGES@

## Introduction

The goal is to have access to all AWS functionality for cases when only a simple, direct access is needed from a ZIO application, or to be used as a building block for higher level wrappers around specific services.

Key features of ZIO AWS:

- Common configuration layer
- ZIO module layer per AWS service
- Wrapper for all operations on all services
- HTTP service implementations for functional Scala HTTP libraries, injected through ZIO’s module system
- ZStream wrapper around paginated operations
- Service-specific extra configuration
- More idiomatic Scala request and response types wrapping the Java classes

## Installation

There are tones of artifacts [published](artifacts.md) for each AWS service. We can pick whichever services we need.

## Example

The following example uses the _ElasticBeanstalk_ and _EC2_ APIs:

```scala
libraryDependencies += "dev.zio" %% "zio-aws-core"             % "@VERSION@"
libraryDependencies += "dev.zio" %% "zio-aws-ec2"              % "@VERSION@"
libraryDependencies += "dev.zio" %% "zio-aws-elasticbeanstalk" % "@VERSION@"
libraryDependencies += "dev.zio" %% "zio-aws-netty"            % "@VERSION@"
```

And here is the example code:

```scala mdoc:compile-only
import zio.aws.core.AwsError
import zio.aws.core.config.AwsConfig
import zio.aws.ec2.Ec2
import zio.aws.ec2.model._
import zio.aws.ec2.model.primitives._
import zio.aws.elasticbeanstalk.ElasticBeanstalk
import zio.aws.elasticbeanstalk.model._
import zio.aws.elasticbeanstalk.model.primitives._
import zio.aws.netty.NettyHttpClient
import zio._
import zio.stream._

object ZIOAWSExample extends ZIOAppDefault {
  val program: ZIO[Ec2 & ElasticBeanstalk, AwsError, Unit] =
    for {
      appsResult <- ElasticBeanstalk.describeApplications(
        DescribeApplicationsRequest(applicationNames = Some(List(ApplicationName("my-service"))))
      )
      app <- appsResult.getApplications.map(_.headOption)
      _ <- app match {
        case Some(appDescription) =>
          for {
            applicationName <- appDescription.getApplicationName
            _ <- Console
              .printLine(
                s"Got application description for $applicationName"
              )
              .ignore

            envStream = ElasticBeanstalk.describeEnvironments(
              DescribeEnvironmentsRequest(applicationName =
                Some(applicationName)
              )
            )

            _ <- envStream.run(ZSink.foreach { env =>
              env.getEnvironmentName.flatMap { environmentName =>
                (for {
                  environmentId <- env.getEnvironmentId
                  _ <- Console
                    .printLine(
                      s"Getting the EB resources of $environmentName"
                    )
                    .ignore

                  resourcesResult <-
                    ElasticBeanstalk.describeEnvironmentResources(
                      DescribeEnvironmentResourcesRequest(environmentId =
                        Some(environmentId)
                      )
                    )
                  resources <- resourcesResult.getEnvironmentResources
                  _ <- Console
                    .printLine(
                      s"Getting the EC2 instances in $environmentName"
                    )
                    .ignore
                  instances <- resources.getInstances
                  instanceIds <- ZIO.foreach(instances)(_.getId)
                  _ <- Console
                    .printLine(
                      s"Instance IDs are ${instanceIds.mkString(", ")}"
                    )
                    .ignore

                  reservationsStream = Ec2.describeInstances(
                    DescribeInstancesRequest(instanceIds = Some(instanceIds.map(id => zio.aws.ec2.model.primitives.InstanceId(ResourceId.unwrap(id)))))
                  )
                  _ <- reservationsStream.run(ZSink.foreach { reservation =>
                    reservation.getInstances
                      .flatMap { instances =>
                        ZIO.foreach(instances) { instance =>
                          for {
                            id <- instance.getInstanceId
                            typ <- instance.getInstanceType
                            launchTime <- instance.getLaunchTime
                            _ <- Console.printLine(s"  instance $id:").ignore
                            _ <- Console.printLine(s"    type: $typ").ignore
                            _ <- Console
                              .printLine(
                                s"    launched at: $launchTime"
                              )
                              .ignore
                          } yield ()
                        }
                      }
                  })
                } yield ()).catchAll { error =>
                  Console
                    .printLineError(
                      s"Failed to get info for $environmentName: $error"
                    )
                    .ignore
                }
              }
            })
          } yield ()
        case None =>
          ZIO.unit
      }
    } yield ()

  override def run: URIO[ZIOAppArgs, ExitCode] = {
    val httpClient = NettyHttpClient.default
    val awsConfig = httpClient >>> AwsConfig.default
    val aws = awsConfig >>> (Ec2.live ++ ElasticBeanstalk.live)

    program
      .provideLayer(aws)
      .either
      .flatMap {
        case Left(error) =>
          Console.printLineError(s"AWS error: $error").ignore.as(ExitCode.failure)
        case Right(_) =>
          ZIO.unit.as(ExitCode.success)
      }
  }
}
```