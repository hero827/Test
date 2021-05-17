# FieldLevel Data Engineering Position Take Home Project

## Purpose

The goal of this project is to help us understand your data engineering abilities.  We do not expect you to spend more than a few hours completing the exercise.

The objective of the exercise is to implement a data stream processor.  The solution should be a dotnet core console application and should be capable to run as a container in docker. 
It needs to process data changes from two different data sources (an OLTP database and an event log database) and produce a transformed result to a third database.


![Complete Setup](img/overview.png)


## Setup

#### Get the project code
1. Import the exercise repository to your personal Github account. Using github's import tool (https://github.com/new/import) import this url: https://github.com/FieldLevel/fieldlevel-data-engineer-take-home/tree/exercise-code
1. Invite our github user "fl-codereview" to be a collaborator on your new repository
1. Get your new repository on your local machine

## Prerequisites

1) docker
2) C# development environment (e.g. Visual Studio, VS Code)
3) Azure Data Studio (optional for querying SQL Server and PostgreSQL)


#### Run the programming environment
1. Create the docker environment.  
```
> docker-compose up -d 
```
The docker compose will create the following containers.  Their purpose is described in more detail below.
* fl-central 
* fl-eventlogs
* datalake 
* data-activity-service



## Requirements

The project requires you to consume changes from two different input data sets, perform a transformation of the data and write the output to another database.  The output should be produced in real-time (or near real-time) based on changes from the input data sets.  

#### Transformation logic
This SQL expression is the psuedo code for how the data input streams should be transformed.  You application needs to perform this function.

```
SELECT a.Sport
      , a.RecruitingClassYear
      , sum( case when datediff( mi, l.dateutc, sysutcdatetime()) < 10 then 1 else 0 end ) as ProvileViewsLast10min
      , count(*) as totalprofileViews
      ,max( l.dateutc ) as last_profileViewTime
FROM dbo.AthleteProfileViewLog l
JOIN dbo.Athlete a on a.athleteId = l.athleteid
GROUP BY a.sport
      ,a.recruitingClassYear

```

The transformed output should land in a table `SportClassYearProfileViewSummary` on the the PostgreSQL database `datalake`.  



## Deliverables

Your applicatiion should TBD include and either run as a stand alone console application, docker container, and/or Visual Studio.





## Background

### fl-central Database

This database is intended to simulate a typical production OLTP database.  
It is running inside a MSSQL docker container `fl-central`  

* rows can be inserted and updated
* an updated row is noted by a change in `modifiedDate` as well as in incremented unique sequence `latestOffset`
* each row represents a single `Athlete`.  The data is maintained by the application and can be altered by the Athlete and/or their team's coaching staff.

#### Athlete 


```
    CREATE TABLE dbo.Athlete(
        athleteId int NOT NULL identity(1,1) ,
        athleteFirstName nvarchar(100) not null ,
        athleteMiddleName nvarchar(100) null ,
        athleteLastName nvarchar(100) not null , 
        sport nvarchar(50) not null ,
        gender char(1) not null ,
        recruitingClassYear smallint NULL,
        enrollmentAcademicLevel tinyint NULL,
        plannedCollegeMajor nvarchar(50) NULL,        
        height decimal(5, 2) NULL,
        weight decimal(5, 2) NULL,
        promotionalCoverLetter nvarchar(max) NULL,
        isMidYearTransfer bit NULL,
        highschoolGraduationYear int NULL,
        hobbiesAndInterests nvarchar(4000) NULL,
        satComposite nvarchar(10) NULL,
        satVerbal nvarchar(10) NULL,
        satMath nvarchar(10) NULL,
        satWriting nvarchar(10) NULL,
        actComposite nvarchar(10) NULL,
        scoutingNotes nvarchar(max) NULL,
        commitmentLevel tinyint NULL,
        recruitingNotes nvarchar(max) NULL,
        recommendedByUserId int NULL,
        gpa decimal(3, 2) NULL,
        latestOffset bigint not null ,
        createDate datetime2 not null ,
        modifiedDate datetime2 not null ,
        CONSTRAINT pk_Athlete PRIMARY KEY NONCLUSTERED (athleteId)  ,
        CONSTRAINT uq_Athlete UNIQUE CLUSTERED ( latestOffset)  
    ) 
```	



### fl-eventlogs Database

This database is intended to simulate a event data store, such as Kafka, log files, etc
For simplicity, it is implemented as a MSSQL docker container `fl-eventlogs`  and the event data is written to a table `AthleteProfileViewLog`.

* data is insert only (no updates or deletes)
* each row represents a coach/recruiter (UserId) that has viewed an athlete's profile in the application.


#### AthleteProfileViewLog 


```
 CREATE TABLE dbo.AthleteProfileViewLog(
	profileviewlogid bigint NOT NULL IDENTITY(1,1),
	userid int NULL,
	athleteid int NULL,
	dateutc datetime NULL,
	clientid uniqueidentifier NULL,
    CONSTRAINT pk_store_VideoInteractionLog PRIMARY KEY CLUSTERED (	profileviewlogid ) 
) 
GO
```	

### datalake Database

This is a PostgreSQL database.  The stream processor needs to deliver the updated output to the table `SportClassYearProfileViewSummary` described below:

```
CREATE TABLE SportClassYearProfileViewSummary (
    Sport varchar(50) not null ,
    RecruitingClassYear int not null  ,
    TotalProfileViews int not null ,
    LastProfileViewTime date not null ,
    constraint pk_SportClassYearProfileViewSummary primary key ( Sport, RecruitingClassYear )
)
```