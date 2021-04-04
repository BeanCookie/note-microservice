#### 跨语言方式
##### 什么是RPC
RPC的全称是Remote Procedure Call(远程过程调用)，也就是说两台服务器A和B，一个部署在A服务器上的应用，想要调用部署在B服务器上某个应用提供的函数，由于不在一个内存空间不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。

##### 如何使用gRPC
##### 消息约定
##### Protobuf

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.zeaho.proto.water";
option java_outer_classname = "WaterProto";
option objc_class_prefix = "HLW";

package water;

// The greeting service definition.
service WaterCompany {
  // Sends a greeting
  rpc BuyStreamWater (WaterRequest) returns (stream WaterReply) {}
  rpc BuyWater (WaterRequest) returns (WaterReply) {}
}

// The request message containing the user's name.
message WaterRequest {
  float price = 1;
}

// The response message containing the greetings
message WaterReply {
  string message = 1;
}
```
##### 选择语言
##### Python
```shell
# 安装依赖
python -m pip install grpcio
python -m pip install grpcio-tools
```
##### 服务端
```python
from concurrent import futures
import logging
import grpc

import water_pb2
import water_pb2_grpc

class WaterCompany(water_pb2_grpc.WaterCompanyServicer):

    def BuyWater(self, request, context):
        return water_pb2.WaterReply(message='Buy water, %.1f!' % request.price)

    def BuyStreamWater(self, request, context):
        for i in range(100):
            yield water_pb2.WaterReply(message='Buy water, %d!' % i)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    water_pb2_grpc.add_WaterCompanyServicer_to_server(WaterCompany(), server)
    server.add_insecure_port('[::]:50052')
    server.start()
    server.wait_for_termination()


if __name__ == '__main__':
    logging.basicConfig()
    serve()
```
##### 客户端
```python
from __future__ import print_function
import logging

import grpc

import water_pb2
import water_pb2_grpc


def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = water_pb2_grpc.WaterCompanyStub(channel)
        response = stub.BuyWater(water_pb2.WaterRequest(price=14.5))
    print("WaterCompany client received: " + response.message)


if __name__ == '__main__':
    logging.basicConfig()
    run()
```

##### Java
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zeaho.grpc-java</groupId>
    <artifactId>grpc-java</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <grpc.version>1.29.0</grpc.version>
        <lombok.version>1.18.12</lombok.version>
        <slf4j.version>1.7.30</slf4j.version>
        <javaOutputDirectory>
            ${project.basedir}/src/main/java-proto
        </javaOutputDirectory>
        <protocPluginOutputDirectory>
            ${project.basedir}/src/main/java-grpc
        </protocPluginOutputDirectory>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>protoc-gen-grpc-java</artifactId>
            <version>${grpc.version}</version>
            <type>pom</type>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.11.0:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.29.0:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>test-compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
##### 服务端
```java
package com.zeaho.grpc.server;

import com.zeaho.proto.water.WaterCompanyGrpc;
import com.zeaho.proto.water.WaterReply;
import com.zeaho.proto.water.WaterRequest;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;

import java.io.IOException;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

/**
 * @author lzzz
 * @since 2020-04-25 17:17
 */
public class WaterServer {
    private static final Logger logger = Logger.getLogger(WaterServer.class.getName());

    private Server server;

    private void start() throws IOException {
        /* The port on which the server should run */
        int port = 50051;
        server = ServerBuilder.forPort(port)
                .addService(new WaterServer.WaterCompanyImpl())
                .build()
                .start();
        logger.info("Server started, listening on " + port);
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                // Use stderr here since the logger may have been reset by its JVM shutdown hook.
                System.err.println("*** shutting down gRPC server since JVM is shutting down");
                try {
                    WaterServer.this.stop();
                } catch (InterruptedException e) {
                    e.printStackTrace(System.err);
                }
                System.err.println("*** server shut down");
            }
        });
    }

    private void stop() throws InterruptedException {
        if (server != null) {
            server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
        }
    }

    /**
     * Await termination on the main thread since the grpc library uses daemon threads.
     */
    private void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }

    /**
     * Main launches the server from the command line.
     */
    public static void main(String[] args) throws IOException, InterruptedException {
        final WaterServer server = new WaterServer();
        server.start();
        server.blockUntilShutdown();
    }

    static class WaterCompanyImpl extends WaterCompanyGrpc.WaterCompanyImplBase {

        @Override
        public void buyWater(WaterRequest req, StreamObserver<WaterReply> responseObserver) {
            WaterReply reply = WaterReply.newBuilder().setMessage("Gave your water " + req.getPrice()).build();
            responseObserver.onNext(reply);
            responseObserver.onCompleted();
        }

        @Override
        public void buyStreamWater(WaterRequest request, StreamObserver<WaterReply> responseObserver) {
            for (int i = 0; i < 100; i++) {
                WaterReply reply = WaterReply.newBuilder().setMessage("Gave your water " + i).build();
                responseObserver.onNext(reply);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            responseObserver.onCompleted();
        }
    }
}
```
##### 客户端
```java
# 阻塞方式
package com.zeaho.grpc.client;

import com.zeaho.grpc.util.BuildClientUtils;
import com.zeaho.proto.water.WaterCompanyGrpc;
import com.zeaho.proto.water.WaterReply;
import com.zeaho.proto.water.WaterRequest;
import io.grpc.Channel;
import io.grpc.ManagedChannel;
import io.grpc.StatusRuntimeException;

import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * @author lzzz
 * @since 2020-04-25 17:09
 */
public class WaterBlockingClient {
    private static final Logger logger = Logger.getLogger(WaterBlockingClient.class.getName());

    private final WaterCompanyGrpc.WaterCompanyBlockingStub futureStub;

    /** Construct client for accessing HelloWorld server using the existing channel. */
    public WaterBlockingClient(Channel channel) {
        // 'channel' here is a Channel, not a ManagedChannel, so it is not this code's responsibility to
        // shut it down.

        // Passing Channels to code makes code easier to test and makes it easier to reuse Channels.
        futureStub = WaterCompanyGrpc.newBlockingStub(channel);
    }

    /** Say hello to server. */
    public void butWater(float price) {
        logger.info("Will try to buy water " + price + " ...");
        WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
        WaterReply waterReply;
        try {
            waterReply = futureStub.buyWater(request);
            logger.info("WaterCompany: " + waterReply.getMessage());

        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
        }
    }

    /**
     * Greet server. If provided, the first element of {@code args} is the name to use in the
     * greeting. The second argument is the target server.
     */
    public static void main(String[] args) throws Exception {
        BuildClientUtils buildClientUtils = new BuildClientUtils(3.5f, args).invoke();
        float price = buildClientUtils.getPrice();
        ManagedChannel channel = buildClientUtils.getChannel();
        try {
            WaterBlockingClient client = new WaterBlockingClient(channel);
            client.butWater(price);
        } finally {
            // ManagedChannels use resources like threads and TCP connections. To prevent leaking these
            // resources the channel should be shut down when it will no longer be used. If it may be used
            // again leave it running.
            channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
        }
    }
}
```
```java
# 异步方式
package com.zeaho.grpc.client;

import com.zeaho.grpc.util.BuildClientUtils;
import com.zeaho.proto.water.WaterCompanyGrpc;
import com.zeaho.proto.water.WaterReply;
import com.zeaho.proto.water.WaterRequest;
import io.grpc.Channel;
import io.grpc.ManagedChannel;
import io.grpc.StatusRuntimeException;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * @author lzzz
 * @since 2020-04-25 17:09
 */
public class WaterFutureClient {
    private static final Logger logger = Logger.getLogger(WaterFutureClient.class.getName());

    private final WaterCompanyGrpc.WaterCompanyFutureStub futureStub;

    /** Construct client for accessing HelloWorld server using the existing channel. */
    public WaterFutureClient(Channel channel) {
        // 'channel' here is a Channel, not a ManagedChannel, so it is not this code's responsibility to
        // shut it down.

        // Passing Channels to code makes code easier to test and makes it easier to reuse Channels.
        futureStub = WaterCompanyGrpc.newFutureStub(channel);
    }

    /** Say hello to server. */
    public void butWater(float price) {
        logger.info("Will try to buy water " + price + " ...");
        WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
        Future<WaterReply> waterReplyFuture;
        try {
            waterReplyFuture = futureStub.buyWater(request);
            logger.info("WaterCompany: " + waterReplyFuture.get().getMessage());

        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }

    /**
     * Greet server. If provided, the first element of {@code args} is the name to use in the
     * greeting. The second argument is the target server.
     */
    public static void main(String[] args) throws Exception {
        BuildClientUtils buildClientUtils = new BuildClientUtils(3.5f, args).invoke();
        float price = buildClientUtils.getPrice();
        ManagedChannel channel = buildClientUtils.getChannel();
        try {
            WaterFutureClient client = new WaterFutureClient(channel);
            client.butWater(price);
        } finally {
            // ManagedChannels use resources like threads and TCP connections. To prevent leaking these
            // resources the channel should be shut down when it will no longer be used. If it may be used
            // again leave it running.
            channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
        }
    }
}
```
```java
# 流式调用
package com.zeaho.grpc.client;

import com.zeaho.grpc.util.BuildClientUtils;
import com.zeaho.proto.water.WaterCompanyGrpc;
import com.zeaho.proto.water.WaterReply;
import com.zeaho.proto.water.WaterRequest;
import io.grpc.Channel;
import io.grpc.ManagedChannel;
import io.grpc.StatusRuntimeException;
import io.grpc.stub.StreamObserver;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * @author lzzz
 * @since 2020-04-25 17:09
 */
public class WaterStreamClient {
    private static final Logger logger = Logger.getLogger(WaterStreamClient.class.getName());

    private final WaterCompanyGrpc.WaterCompanyStub futureStub;

    final CountDownLatch latch = new CountDownLatch(1);

    /** Construct client for accessing HelloWorld server using the existing channel. */
    public WaterStreamClient(Channel channel) {
        // 'channel' here is a Channel, not a ManagedChannel, so it is not this code's responsibility to
        // shut it down.

        // Passing Channels to code makes code easier to test and makes it easier to reuse Channels.
        futureStub = WaterCompanyGrpc.newStub(channel);
    }

    /** Say hello to server. */
    public void butWater(float price) {
        logger.info("Will try to buy water " + price + " ...");
        WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
        try {
            futureStub.buyStreamWater(request, new StreamObserver<WaterReply>() {
                @Override
                public void onNext(WaterReply waterReply) {
                    logger.info("WaterCompany: " + waterReply.getMessage());
                }

                @Override
                public void onError(Throwable throwable) {
                    logger.log(Level.WARNING, throwable.getMessage());
                }

                @Override
                public void onCompleted() {
                    logger.info("WaterCompany: completed");
                    latch.countDown();
                }
            });
            latch.await();

        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    /**
     * Greet server. If provided, the first element of {@code args} is the name to use in the
     * greeting. The second argument is the target server.
     */
    public static void main(String[] args) throws Exception {
        BuildClientUtils buildClientUtils = new BuildClientUtils(23.5f, args).invoke();
        float price = buildClientUtils.getPrice();
        ManagedChannel channel = buildClientUtils.getChannel();
        try {
            WaterStreamClient client = new WaterStreamClient(channel);
            client.butWater(price);
        } finally {
            // ManagedChannels use resources like threads and TCP connections. To prevent leaking these
            // resources the channel should be shut down when it will no longer be used. If it may be used
            // again leave it running.
            channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
        }
    }
}
```
##### 跨语言调用