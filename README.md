# GCP_SQLServer_Dataflow_template

Extraction example of the data from sqlserver (VM) to BigQuery via a simple Dataflow Template

Microsoft SQL Server to Google Big Query, Data Transfer using a Dataflow Template. 

Background:
A few months ago we moved into Google Cloud. Previously we were a Microsoft stack house. The way migration progressed was mostly focused on front end and microservices. Data department was a troublesome cousin best left alone. So the solution was to move all the systems into GCP... onto Microsoft based VMs. Yes. I Know. 
Databases and the Data Warehouse with all of its ETL glory, were on Sql Server, on premises. Now they are on SQL Server Virtual Machines in GCP. 

The ETL written (?) in SSIS is a mature and complex product. Rewriting it in a new tool would take a lot of time and would require a very good amount of trainng in that said new tool. It was down to me to teach myself about GCP and learn whatever I needed to learn. Test, experiment, fail, fail, suceed. I learned docker, python, Apache Beam and a bunch of other stuff. The learning and experimentation resulted in small new data products concepts now living in production and a data lake working on GCP. 
Organiclly it seemed most effective to now get all the Data Warehouse data into Big Query, post SSIS processing. This meant one batch operation, once per day. 
It is probalby not a surprise at this point that all of the cost should be kept to a minimum. This means no 3rd parties working towards locking you in. I just happaned to be reading a just released book 'Google BigQuery: The Definitive Guide' by Lakshman and Tigani. In it I came across a use of Dataflow templates and really wanted to test it out for my use case. It turned out to be more difficult then expected and I had though luck googling it. So here it is, so you dont have to goo through all the madness alone. 

The Detail: 

After many failures and debugging it is most sensible to begin with as small and simple example and build on top of it. 

Steps:
1. Create a SQL Server instance in Compute Engine
2. SQL Server Configuration and setup 
3. Pre Dataflow test 
4. Big Query Set up 
5. Dataflow Template creation and run 


1.	Create a SQL Server instance in Compute Engine
		
	In your GCP project Create a new VM instance with 4 vCPUs and 15 GB Ram. GCP will complain with lesser settings. We will only needed for an hour or two so cost should stay below a couple of $. Pay attention to the Region and select the same one your live server is on, and potenially is the same as your Big Query instance unless you want to deal with data transfers. You dont. 
	
	We will select an available Application image provided by Google. Select one which is a closes representation of your production server. 
	(Screenshot made) x 2
	Go to a Firewall section and select both HTTP and HTTPS traffic (should work without these but at least one less thing to worry about). Leave everything else as default. 
	(Screenshot made) 
	Create the VM. 	
	It will take a few minutes for the instance to be initialised. once its complete click on it as listed in the 'VM instances' list
	(screenshot)
	Find and click on 'Set Windows password', this will provide you with credentials to remotelly connect to the machine. Copy and save your username , click Set and then copy and save your password. You are now ready to log into the Server and set up the database and login. 
		(screenshot)
		
		
		
		
2. SQL Server Configuration and setup 	
	Remote Connection 
		I tried using "Chrome RDP" extension for my Chromium browser but it would work to RDP to the machine. It can be installed by following a link in the pop pop once pressed on 'RDP' button. 
	(screenshot)
	Instead I downloaded the RDP file and connected using Remmina ( from Linux )
	Bear in ming that it may not work right away. In 2 out of 4 cases I had to give it 5 minutes before I retried the connection. 
	
	Connect to the sql server using SSMS. 
	Go to Start and start typing 'SQL Server Managment Studio' you should see the search result we are after. Open it. 
	(screenshot) 
	
	Our next step are :	
		create database
		create a table
		insert data 
		create login
		check server accepts sql login 
		
		(script?) 
	
	By default you should see that the SSMS is giving you and option of connecting to SQL via Windows Authentication. Click Connect. 
		I have created a database 'mssqltogcp'
		In it a table 'testtable' with one row of data: 
			ID:		1
			Text:	'this is data from a sql server database'
		I have created a new login 'gcp' set it up as a 'SQL  Server authentication', entered password and deselected the 'Enforce password policy'. 
		In the 'Server Roles' I gave it 'sysadmin' rights and Mapped it to my database with 'datareader' role. 
			
	One last step was to check that I can connect with SQL serever account. Right click on the the SQL Server instance and go tp Properties > Security. Select the 'SQL Server and Windows Authentication mode'  if it is not selected already.  
	Once all this is completed we have to restart the server to make sure that all the changes are applied. 
	
	In Start menu begin typing 'Configuration Manager' and look out for 'SQL Serever 20XX Configuration Manager' result. Open it and go to 'SQL Server Services' and Right Click on the 'SQL Server (my server)' to select 'Restart'. 
	(Screenshot) 
	
	
3. Pre Dataflow test 	
	
	It is important to be confident that you can actually use that account and connect the the sever. If you followed the above steps, or used the script you can close the configuration manager now and go back to SSMS. 
	
	Go to File>'Connect...' 
	Change the Authentication to 'SQL Server Authentication'
	Enter your credentials (my case, or if you executed the script: gcp and password) and Connect. Create a new query tab and execute this query, which later we will use in the Dataflow template to server a data extraction example:
		
		SELECT [ID] ,[text] FROM [mssqltogcp].[dbo].[testtable];

	(Screenshot)  x2
	
	At this if it worked, try to connect the the SQL Server instance from your own machine. Instead of the SQL server name you have to provide the VMs IP address (both worked for me). Execute the query as above.	
	

		JDBC drivers 
		We will need these drivers for our template to work. You can download them from here: 
		https://www.microsoft.com/en-us/download/details.aspx?id=57175
		
		once Downloaded you will need to upload them into Google Cloud Storage bucket and reffer to their location in the template.

	
	
	
4. Big Query Set up 
	In order to put table data into Big Query we have to prepare a table in Big Query first. In real (production) scenario you can 'script to' a table in SSMS and then make adjustments to create it in Big Query via one of the available options. Here we will do it manually via the GCP Console. 
	
	Open you your GCP Project and go to Big Query. 
	Create a new dataset and a table in it. My setup: 
		Dataset 	mssql_2_bigq 
		Table 		test 
					ID: INTEGER NOT REQUIRED
					text: STRING NOT REQUIRED
	
	
	
5. Dataflow Template creation and run 	
	
	In GCP go to Dataflow and click on '+ Create Job From Template'
	(Screenshot) 
	type in your job name and select an apropriate region (same as in the VM creation step above) 
	in the template dropdown select a 'Jdbc to BigQuery' option. 
	(Screenshot) 
	
	Now for the rest: 
	
	Jdbc connection URL string: 
		jdbc:sqlserver://<IP>:1433
		example: jdbc:sqlserver://10.164.0.24:1433
	
	
	Jdbc driver class name:
		com.microsoft.sqlserver.jdbc.SQLServerDriver
		
	
	Jdbc source SQL query:
		SELECT [ID] ,[text] FROM [mssqltogcp].[dbo].[testtable];
		
	
	BigQuery output table:
		<project_name>:dataset_name.table_name
		example projectX:mssql_2_bigq.test
		
		
	GCS paths for Jdbc drivers: 
		gs://temporary_bucket/path/to/driver/mssql-jdbc-7.0.0.jre8.jar
		example gs://temporary111/jdbc/sqljdbc_7.0/enu/mssql-jdbc-7.0.0.jre8.jar
			
gs://lh_dataflow_drivers/jdbc/sqljdbc_7.0/enu/mssql-jdbc-7.0.0.jre8.jar
	
	Temporary directory for BigQuery loading process:
		gs://temporary111/dataflowtemp
		gs://throwaway_temp/bigq
	
	Temporary location: 
		gs://temporary111/dataflowtemp
		gs://throwaway_temp/dataflow
	
	Below options are in 'optional paramaters' 
	
	
	Username
			SQL Login
			example 'gcp'
			
	Password 
			SQL Login Password 
			example 'password'
			Warning! this will be visible to anyone with the rights to view your dataflow jobs list in the jobs propreries 
	
	
	Click 'Run'  and wait a little. In my case it takes about 2 minutes for the process to really kick off. 
	(screenshot) 
	I hope that if you followed the steps there are no warning messages when you run it. 
	
	
Extras: 

	JDBC drivers 
		We will need these drivers for our template to work. You can download them from here: 
	https://www.microsoft.com/en-us/download/details.aspx?id=57175
	once Downloaded you will need to upload them into Google Cloud Storage bucket and reffer to their location in the template.
	
Whats next? 

	Add layers, remove http and https. 
	Test your AD group for connection Authentication if thats what your dba is using 
	Create a dataflow in another project B and connect to to the sql in project A 
	Try to get as close as possible to your production environment setup as possible
	
	Cross project dataflow will require that the project with the template has its driver files copy on its gcs and some way of allowing access between projects. But thats for next time. Once I get it to work I shall post that too. 
		Some cross platform account/vpc.
		cross account gcs? 
		
	

	Scheduling execution of the templates. Reusability, deploy in production.
	

	
	

	
	
	
	
	
	
	
	
	
	
	
	 
	
