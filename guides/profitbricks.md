---
layout: page
title: "ProfitBricks: Getting Started Guide"
permalink: /guides/profitbricks/
---

[jclouds](http://jclouds.apache.org/) is an open source multi-cloud toolkit for the Java platform that gives you the freedom to create applications that are portable across clouds while giving you full control to use cloud-specific features.

This guide will show you how to programmatically use the ProfitBricks provider in jclouds to perform common management tasks available in the ProfitBricks Data Center Designer.

## Table of Contents

* [Concepts](#concepts)
* [Getting Started](#getting-started)
* [How to: Create a Data Center](#how-to-create-a-data-center)
* [How to: Delete a Data Center](#how-to-delete-a-data-center)
* [How to: Create a Server](#how-to-create-a-server)
* [How to: List Available Disk and ISO Images](#how-to-list-available-disk-and-iso-images)
* [How to: Create a Storage Volume](#how-to-create-a-storage-volume)
* [How to: Update Cores, Memory, and Disk](#how-to-update-cores-memory-and-disk)
* [How to: Attach and Detach a Storage Volume](#how-to-attach-and-detach-a-storage-volume)
* [How to: List Servers, Volumes, and Data Centers](#how-to-list-servers-volumes-and-data-centers)
* [Example](#example)
* [pom.xml](#pom.xml)
* [Support and Feedback](#support-and-feedback)

--------------

## Concepts

The jclouds library wraps the [ProfitBricks API](https://devops.profitbricks.com/api/soap/). All operations are performed over SSL and authenticated using your ProfitBricks portal credentials. The API can be accessed within an instance running in ProfitBricks or directly over the Internet from any application that can send an HTTPS request and receive an HTTPS response.


## Getting Started

Before you begin you will need to have [signed-up](https://www.profitbricks.com/signup) for a ProfitBricks account. The credentials you setup during sign-up will be used to authenticate against the API.
 
### Installation

jclouds has some pre-requisities before you're able to use it. You will need to: 

* Ensure you are using the [Java Development Kit (JDK)](http://www.oracle.com/technetwork/java/javase/downloads/index.html) version 6 or later. You can check this by running: `javac -version`.
* Ensure you are using [Maven version 3](http://maven.apache.org/guides/getting-started/maven-in-five-minutes.html) or later. You can check this by running: `mvn -version`.

Now that you have validated the pre-requisities, you will want to do the following: 

* Create a directory to try out jclouds. This can be done by doing: 

`mkdir jclouds`
`cd jclouds`

* Make a local copy of the pom.xml file below in the jclouds directory.

`mvn dependency:copy-dependencies "-DoutputDirectory=./lib"`

You should now have a directory with the following structure:

	jclouds/
		pom.xml
		lib/		
			*.jar    

The ProfitBricks provider is currently available as part of the jclouds labs project [here](https://github.com/jclouds/jclouds-labs).


### Authentication

Connecting to ProfitBricks can be done by creating a compute connection with the ProfitBricks provider.


    pbApi = ContextBuilder.newBuilder("profitbricks")
            .credentials(username, apiKey)
            .buildApi(ProfitBricksApi.class);

**Caution:** You will want to ensure you follow security best practices when using credentials within your code or stored in a file.

# How To's
## How to: Create a Data Center

ProfitBricks introduces the concept of Virtual Data Centers. These are logically separated from one another and allow you to have a self-contained environment for all servers, volumes, networking, snapshots, and so forth. The goal is to give you the same experience as you would have if you were running your own physical data center.

The following code example shows you how to programmatically create a data center: 

     DataCenter dc = api.dataCenterApi().createDataCenter(
              DataCenter.Request.CreatePayload.create("JClouds", Location.DE_FKB)
      );

This responds with the datacenter object once created.


## How to: Delete a Data Center

You will want to exercise a bit of caution here. Removing a data center will **destroy** all objects contained within that data center -- servers, volumes, snapshots, and so on.

The code to remove a data center is as follows. This example assumes you want to remove previously datacenter: 

    api.dataCenterApi().deleteDataCenter(dc.id());


## How to: Create a Server

The server create method has a list of required parameters followed by a hash of optional parameters. The optional parameters are specified within the "options" hash and the variable names match the [SOAP API](https://devops.profitbricks.com/api/soap/) parameters.

The following example shows you how to create a new server in the virtual datacenter created above:

	String serverId = api.serverApi().createServer(Server.Request.creatingBuilder()
			.dataCenterId(dc.id())
			.name("jclouds-node")
			.cores(1)
			.ram(1024)
			.build());

The server can take time to provision. The "waitUntilAvailable" server object method will wait until the server state is available before continuing. This is useful when chaining requests together that are dependent on one another.

    waitUntilAvailable = Predicates2.retry(
		  new ProvisioningStatusPollingPredicate(api, ProvisioningStatusAware.SERVER, ProvisioningState.AVAILABLE),
		  2l * 60l, 2l, TimeUnit.SECONDS);

## How to: List Available Disk and ISO Images

A list of disk and ISO images are available from ProfitBricks for immediate use. These can be easily viewed and selected. The following shows you how to get a list of images. This list represents both CDROM images and HDD images.

         List<Image> images = api.imageApi().getAllImages();

## How to: Create a Storage Volume

ProfitBricks allows for the creation of multiple storage volumes that can be attached and detached as needed. It is useful to attach an image when creating a storage volume. The storage size is in gigabytes.

		String storageId = api.storageApi().createStorage(
				Storage.Request.creatingBuilder()
				.dataCenterId(dc.id())
				.name("hdd-1")
				.size(2f)
				.build());


## How to: Update Cores, Memory, and Disk

ProfitBricks allows users to dynamically update cores, memory, and disk independently of each other. This removes the restriction of needing to upgrade to the next size available size to receive an increase in memory. You can now simply increase the instances memory keeping your costs in-line with your resource needs.

**Note:** The memory parameter value must be a multiple of 256, e.g. 256, 512, 768, 1024, and so forth.

The following code illustrates how you can update cores and memory: 

		api.serverApi().updateServer(
				Server.Request.updatingBuilder()
				.id(serverId)
				.name("apache-node")
				.cores(2)
				.ram(2 * 1024)
				.build());
					
The server object may need to be refreshed in order to show the new configuration.

    Server server = api.serverApi().getServer(createdServerId);

 This is how you would update the storage volume size:

		api.storageApi().updateStorage(
				Storage.Request.updatingBuilder()
				.id(storageId)
				.name("hdd-2")
				.build());

## How to: Attach and Detach a Storage Volume

ProfitBricks allows for the creation of multiple storage volumes. You can detach and reattach these on the fly. This allows for various scenarios such as re-attaching a failed OS disk to another server for possible recovery or moving a volume to another location and spinning it up. 

The following illustrates how you would attach and detach a volume from a server:

    String requestId = api.storageApi().disconnectStorageFromServer(storageId, serverId);

## How to: List Servers, Volumes, and Data Centers

jclouds provides standard functions for retrieving a list of volumes, servers, and datacenters. 

The following code illustrates how to pull these three list types: 

    List<Storage> storages = api.storageApi().getAllStorages();

    List<Server> servers = api.serverApi().getAllServers();
 
    List<DataCenter> dataCenters = api.dataCenterApi().getAllDataCenters();


## Example:


	package com.profitbricks.example;

	import com.google.common.base.Predicate;
	import java.util.List;
	import java.util.concurrent.TimeUnit;
	import org.jclouds.ContextBuilder;
	import org.jclouds.profitbricks.ProfitBricksApi;
	import org.jclouds.profitbricks.compute.internal.ProvisioningStatusAware;
	import org.jclouds.profitbricks.compute.internal.ProvisioningStatusPollingPredicate;
	import org.jclouds.profitbricks.domain.DataCenter;
	import org.jclouds.profitbricks.domain.Image;
	import org.jclouds.profitbricks.domain.Location;
	import org.jclouds.profitbricks.domain.ProvisioningState;
	import org.jclouds.profitbricks.domain.Server;
	import org.jclouds.profitbricks.domain.Storage;
	import org.jclouds.util.Predicates2;

	public class App {

		private static Predicate<String> waitUntilAvailable;
		private static final String provider = "profitbricks";
		private static final String username = "username";
		private static final String apikey = "apikey";

		public static void main(String[] args) {

			ProfitBricksApi api = ContextBuilder.newBuilder(provider)
					.credentials(username, apikey)
					.buildApi(ProfitBricksApi.class);

			/*
			 * CreateDataCenterRequest. 
			 * The only required field is DataCenterName. 
			 * If location parameter is left empty data center will be created in the default region of the customer
			 */
			DataCenter dc = api.dataCenterApi().createDataCenter(
					DataCenter.Request.CreatePayload.create("JClouds", Location.DE_FKB)
			);

			/*  
			 * DataCenterId: Defines the data center wherein the server is to be created.
			 * AvailabilityZone: Selects the zone in which the server is going to be created (AUTO, ZONE_1, ZONE_2).
			 * Cores: Number of cores to be assigned to the specified server. Required field.
			 * InternetAccess: Set to TRUE to connect the server to the Internet via the specified LAN ID.
			 * OsType: Sets the OS type of the server.
			 * Ram: Number of RAM memory (in MiB) to be assigned to the server.
			 */
			String serverId = api.serverApi().createServer(Server.Request.creatingBuilder()
					.dataCenterId(dc.id())
					.name("jclouds-node")
					.cores(1)
					.ram(1024)
					.build());

			/*
			 * The "waitUntilAvailable" server object method will wait until the server state is available before continuing
			 */
			waitUntilAvailable = Predicates2.retry(
					new ProvisioningStatusPollingPredicate(api, ProvisioningStatusAware.SERVER, ProvisioningState.AVAILABLE),
					2l * 60l, 2l, TimeUnit.SECONDS);

			/*
			 * Get list of all images.
			 */
			List<Image> images = api.imageApi().getAllImages();

			/* dataCenterId: Defines the data center wherein the storage is to be created. If left empty, the storage will be created in a new data center
			 * name: names the new volume.
			 * Size: Storage size (in GiB). Required Field.
			 */
			String storageId = api.storageApi().createStorage(
					Storage.Request.creatingBuilder()
					.dataCenterId(dc.id())
					.name("hdd-1")
					.size(2f)
					.build());

			/*
			 * ServerId: Id of the server to be updated.
			 * ServerName: Renames target virtual server
			 * Cores: Updates the amount of cores of the target virtual server
			 * Ram: Updates the RAM memory (in MiB) of the target virtual 
			 */
			api.serverApi().updateServer(
					Server.Request.updatingBuilder()
					.id(serverId)
					.name("apache-node")
					.cores(2)
					.ram(2 * 1024)
					.build());

			/*
			 * id: identifier of a storage to be updated
			 * name: changes the name of the storage
			 */
			api.storageApi().updateStorage(
					Storage.Request.updatingBuilder()
					.id(storageId)
					.name("hdd-2")
					.build());

			/*
			 * Disconnects storage from the server.
			 */
			api.storageApi().disconnectStorageFromServer(storageId, serverId);

			/*
			 * Fetches list of all DataCenters
			 */
			List<DataCenter> dataCenters = api.dataCenterApi().getAllDataCenters();

			/*
			 * Fetches list of all Volumes
			 */
			List<Storage> storages = api.storageApi().getAllStorages();

			/*
			 * Fetches list of all Servers
			 */
			List<Server> servers = api.serverApi().getAllServers();

			api.dataCenterApi().deleteDataCenter(dc.id());
		}
	}
	
## pom.xml

	<?xml version="1.0" encoding="UTF-8"?>
	<!--

		Licensed to the Apache Software Foundation (ASF) under one or more
		contributor license agreements.  See the NOTICE file distributed with
		this work for additional information regarding copyright ownership.
		The ASF licenses this file to You under the Apache License, Version 2.0
		(the "License"); you may not use this file except in compliance with
		the License.  You may obtain a copy of the License at

			http://www.apache.org/licenses/LICENSE-2.0

		Unless required by applicable law or agreed to in writing, software
		distributed under the License is distributed on an "AS IS" BASIS,
		WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
		See the License for the specific language governing permissions and
		limitations under the License.

	-->
	<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
		<modelVersion>4.0.0</modelVersion>
		<parent>
			<groupId>org.apache.jclouds.labs</groupId>
			<artifactId>jclouds-labs</artifactId>
			<version>2.0.0-SNAPSHOT</version>
		</parent>

	  
		<!-- TODO: when out of labs, switch to org.jclouds.api -->
		<artifactId>profitbricks</artifactId>
		<name>jclouds ProfitBricks api</name>
		<description>jclouds components to access an implementation of ProfitBricks</description>
		<packaging>bundle</packaging>

		<properties>
			<test.profitbricks.endpoint>https://api.profitbricks.com/1.3</test.profitbricks.endpoint>
			<test.profitbricks.identity>FIXME</test.profitbricks.identity>
			<test.profitbricks.credential>FIXME</test.profitbricks.credential>
			<test.profitbricks.api-version>1.3</test.profitbricks.api-version>
			<test.profitbricks.template />
			<jclouds.osgi.export>org.jclouds.profitbricks*;version="${project.version}"</jclouds.osgi.export>
			<jclouds.osgi.import>
				org.jclouds.labs*;version="${project.version}",
				org.jclouds*;version="${jclouds.version}",
				*
			</jclouds.osgi.import>
		</properties>
	  
		<dependencies>
			<dependency>
				<groupId>org.apache.jclouds</groupId>
				<artifactId>jclouds-core</artifactId>
				<version>${jclouds.version}</version>
			</dependency>
			<dependency>
				<groupId>org.apache.jclouds</groupId>
				<artifactId>jclouds-compute</artifactId>
				<version>${jclouds.version}</version>
			</dependency>
			<dependency>
				<groupId>com.google.auto.service</groupId>
				<artifactId>auto-service</artifactId>
				<scope>provided</scope>
			</dependency>
			<dependency>
				<groupId>com.google.auto.value</groupId>
				<artifactId>auto-value</artifactId>
				<scope>provided</scope>
			</dependency>
			<!-- Test dependencies -->
			<dependency>
				<groupId>org.apache.jclouds</groupId>
				<artifactId>jclouds-core</artifactId>
				<version>${jclouds.version}</version>
				<type>test-jar</type>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>org.apache.jclouds</groupId>
				<artifactId>jclouds-compute</artifactId>
				<version>${jclouds.version}</version>
				<type>test-jar</type>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>org.apache.jclouds.driver</groupId>
				<artifactId>jclouds-sshj</artifactId>
				<version>${jclouds.version}</version>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>com.squareup.okhttp</groupId>
				<artifactId>mockwebserver</artifactId>
				<scope>test</scope>
				<exclusions>
					<!-- Already provided by jclouds-sshj -->
					<exclusion>
						<groupId>org.bouncycastle</groupId>
						<artifactId>bcprov-jdk15on</artifactId>
					</exclusion>
				</exclusions>
			</dependency>
			<dependency>
				<groupId>org.apache.jclouds.driver</groupId>
				<artifactId>jclouds-slf4j</artifactId>
				<version>${jclouds.version}</version>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>ch.qos.logback</groupId>
				<artifactId>logback-core</artifactId>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>ch.qos.logback</groupId>
				<artifactId>logback-classic</artifactId>
				<scope>test</scope>
			</dependency>
			<dependency>
				<groupId>${project.groupId}</groupId>
				<artifactId>profitbricks</artifactId>
				<version>${project.version}</version>
			</dependency>
		</dependencies>
	  
		<profiles>
			<profile>
				<id>live</id>
				<build>
					<plugins>
						<plugin>
							<groupId>org.apache.maven.plugins</groupId>
							<artifactId>maven-surefire-plugin</artifactId>
							<executions>
								<execution>
									<id>integration</id>
									<phase>integration-test</phase>
									<goals>
										<goal>test</goal>
									</goals>
									<configuration>
										<threadCount>1</threadCount>
										<systemPropertyVariables>
											<test.profitbricks.endpoint>${test.profitbricks.endpoint}</test.profitbricks.endpoint>
											<test.profitbricks.identity>${test.profitbricks.identity}</test.profitbricks.identity>
											<test.profitbricks.credential>${test.profitbricks.credential}</test.profitbricks.credential>
											<test.profitbricks.api-version>${test.profitbricks.api-version}</test.profitbricks.api-version>
											<test.profitbricks.template>${test.profitbricks.template}</test.profitbricks.template>
										</systemPropertyVariables>
									</configuration>
								</execution>
							</executions>
						</plugin>
					</plugins>
				</build>
			</profile>
		</profiles>
	</project>

## Support and Feedback
Your feedback is welcome! If you have comments or questions regarding using ProfitBricks via jclouds, please reach out to us at [DevOps Central](https://devops.profitbricks.com).
