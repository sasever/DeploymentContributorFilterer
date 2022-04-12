# DeploymentContributorFilterer
Generic implementation of a DacFx deployment contributor in .net. Use this tool to filter sql objects during dacpac deployment process by `SqlPackage.exe`.

Original documentation and discussion adapted from:
*https://the.agilesql.club/2015/01/howto-filter-dacpac-deployments/*

You can also refer to the microsoft documentation on [developing  and using deployment contributors](https://docs.microsoft.com/sql/ssdt/use-deployment-contributors-to-customize-database-build-and-deployment?view=sql-server-ver15).

## Basic Usage
Download the latest release from Github or build yourself. Put the `AgileSqlClub.SqlPackageFilter.dll` file that you will find under `DeploymentContributorFilterer\SqlPackageFilter\bin\Debug` into the same folder as `SqlPackage.exe`, and add these command line parameters to your deployment:

```
/p:AdditionalDeploymentContributors=AgileSqlClub.DeploymentFilterContributor /p:AdditionalDeploymentContributorArguments="SqlPackageFilter=IgnoreSchema(BLAH)"
```

This will neither deploy, drop or alter anything in the BLAH schema.

## Bootstrapping custom filters with SqlPackage.exe
Ok so the way the DacFx api works is that you need to put the dll that contains the contributor into the same folder as `sqlpackage.exe`. Once the dll is in the same folder as `sqlpackage.exe` you need to tell it to load the contributor which you do using this argument:

```
/p:AdditionalDeploymentContributors=AgileSqlClub.DeploymentFilterContributor
```

## Types of Filters
There are two types of filters: **Keep** and **Ignore**.

**Keep** filters stop objects being dropped when they are in the dacpac but not the destination, if they are in the dacpac and not in the destination *or are different* then they will be created or altered. 

In  other words  if the original deployment intends to remove an object covered with KEEP specification scope in the destination, deployment gets edited with the intention of protecting (keeping) the destination object. It does not take any action in changing the deployment of the other objects existing in the dacpac that is not covered in the KEEP specification 

Keep are really only ever used in combination with `/p:DropObjectsInSource=True` otherwise they wouldn’t be dropped anyway.

**Ignore** filters stop any sort of operation, create, alter or drop so there is some flexibility. 

In other words if the original deployment intends to add an object covered with IGNORE specification scope  in the destination, deployment gets edited with the intention of not changing  the destination objects hence ignoring the deployment for IGNORE specification  scope.

Once you know what type of filter you want you need to decide what you will filter on, your choices are: **Name**, **SchemaName** and **object type** (stored procedure, function, table, user, role, rolemembership etc etc).

* Name filters work on an objects name, pretty straight forward. You can also specify a comma-separated set of names to match multipart identifiers. See the examples section for more details. Note that parts are matched from right to left.
* Schema filters work on the name of the schema so you can keep or ignore everything in a specific schema
* Object type filters work on the type of the object as the DacFx api sees it, these types are all documented as properties of the ModelSchema class: [link](http://msdn.microsoft.com/library/microsoft.sqlserver.dac.model.modelschema.aspx)

The object types are all fields, so the list starts Aggregate, ApplicationRole etc etc. Once you have decided how you will filter you specify the filter itself which is a regex, but don’t be scared it doesn’t have to be complex.

## Examples
Because of the way we pass the arguments to SqlPackage.exe and it then parses them and passes them onto the deployment contributor it is a little rigid, but essentially the filter itself look like:

**To keep everything in dbo in target environment:**
```
KeepSchema(dbo)
```

**To ignore all Tables in the dacpac and not to deploy:**
```
IgnoreType(Table)
```

**To keep a table called MyTable or MyExcellentFunnyTable you can use:**
```
KeepName(.*yTabl.*)
```

Note that since we are using the KeepName filter it will filter the objects through the object name, hence this will match the right-most part of a multipart identifier, so this matches: `[dbo].[MyTable]`, `[dbo].[MyExcellentFunnyTable]`,
`[dev].[MyTable]`, `[dbo].[SomeOtherTable].[MyTabletColumnId]`, etc. And protect all these objects from dropping.

You can match against multiple parts of a multipart identifier, matching from right part to left, by specifying multiple
values (regexes) separated by a comma:
```
KeepName(dbo,.*yTabl.*)
```

Using the example above, this instead matches only `[dbo].[MyTable]` or `[dbo].[MyExcellentFunnyTable]`.

Behind the scenes, matching relies on regex using the default .Net options for the Match method.

**To only deploy to the dbo schema (ie exclude non-dbo objects):**
```
IgnoreSchema(^(?!\b(?i)dbo\b).*)
```

If we summarize the example behavior in a table with the `/p:DropObjectsInSource=True` option of `SqlPackage.exe`:

| Source        | Destination   | DropObjectsNotInSource | Filter Type     | Generates  | Result |
| ------------- |:-------------:| :---------------------:| --------------- | ---------- | ------------ |
| dbo.Table     | Missing       | TRUE                   | KeepSchema(dbo) | Create     | Leave in deployment |
| etl.Table     | Missing       | TRUE                   | KeepSchema(dbo) | Create     | Leave in deployment |
| Missing       | dbo.Table     | TRUE                   | KeepSchema(dbo) | Drop       | Remove from deploy |
| Missing       | etl.Table     | TRUE                   | KeepSchema(dbo) | Drop       | Leave in deployment |
| dbo.Table     | Missing       | TRUE                   | IgnoreSchema(dbo) | Create   | Remove from deploy  |
| etl.Table     | Missing       | TRUE                   | IgnoreSchema(dbo) | Create   | Leave in deployment |
| Missing       | dbo.Table     | TRUE                   | IgnoreSchema(dbo) | Drop     | Remove from deploy |
| Missing       | etl.Table     | TRUE                   | IgnoreSchema(dbo) | Drop     | Leave in deployment |

## How to use with `SqlPackage.exe`
When you have decided on the filter you use need to pass it to `SqlPackage.exe` using:
```
/p:AdditionalDeploymentContributors=AgileSqlClub.DeploymentFilterContributor/p:AdditionalDeploymentContributorArguments="SqlPackageFilter=KeepSecurity"
```

You can specify multiple filters by separating them with a semi colon  and adding a unique identifier  to the end of each arg name:

```
/p:AdditionalDeploymentContributorArguments="SqlPackageFilter0=KeepSecurity;SqlPackageFilter1=IgnoreSchema(dev)"
```

An example of how the final command in combination with the `SqlPAckage.ex`e when deploying a remote environment should look like:

```
./SqlPackage.exe /Action:Publish /SourceFile:"<Drive>:\<Path of your dacpac>\<Name of your dacpac>.dacpac"  /TargetConnectionString:"Data Source=<Server URL>;Initial Catalog=<Database Name>;User ID=<user with sufficient rights to deploy dacpac>;Password=<password of the user>;Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False" /p:AdditionalDeploymentContributors=AgileSqlClub.DeploymentFilterContributor /p:AdditionalDeploymentContributorArguments="SqlPackageFilter0=KeepSecurity;SqlPackageFilter1=IgnoreSchema(PII)
```

## Important

Using `Ignore` filters may cause deployment to break if you have an object in your deployment which has a dependency on another object/change that currently does not exist in target and should be created within this deployment but somehow got into the `Ignore` scope and hence ignored. Such deployments would fail adn any created objects during the deployments remain deployed. Would advice to do an empty database deployment as a **PreProduction** deployment test to make sure the filtered deployment does not break any dependencies.

### Contributing

If you would like to contribute and want to run the tests, create a sql local db instance called Filter - "sqllocaldb c Filter" - all tests are hardcoded to that (sorry ha ha).