# Manage CosmosDB objects (Stored Procedure, Functions, Triggers ...) with this Nuget package

In my Toss project I decided to use CosmosDB as the main data store. Document oriented Database are fun to work with as you have to change the way you see data and processing compared to relationnal database. NoSQL database are often badly framed as "schemaless", [but there isn't such thing as schemaless](https://martinfowler.com/articles/schemaless/), the schema is just defined elsewhere : not on the database but on your application code.

There are 2 problems with this approach for the developer :
- the data are persisted, so when you change your schema (property rename), you have to think of the existing data
- there is some specific database object such as functions and triggers that needs to be defined somewhere

That's why I created this package, it'll help you to manage your database object and schema evolution along with your application code on your repository and it'll be applied when you want (on app startup mostly).

## Reading he embedded ressources

I chose to use embedded js file in assembly for defining objets definitions :
- Because they are defined in javascript it's better to use .js file, the IDE will help the developer
- If it's embedded then the package user won't have to think about securing the folder containing the definitions

Reading the embedded ressource is fairly simple in C# / netstandard :

```cs
//read all the migration embbeded in CosmosDB/Migrations
var ressources = migrationAssembly.GetManifestResourceNames()
    .Where(r => r.Contains(".CosmosDB.Migrations.") && r.EndsWith(".js"))
    .OrderBy(r => r)
    .ToList();
//for each migration
foreach (var migration in ressources)
{
    string migrationContent;
    using (var stream = migrationAssembly.GetManifestResourceStream(migration))
    {
        using (var reader = new StreamReader(stream))
        {
            migrationContent = await reader.ReadToEndAsync();
        }
    }
    // do something
}
```

- migrationAssembly is sent as a parameter of my method, so the user can add its migrations to the app ressources or to a library ressources.
- I don't really think about performance here as this code is supposed to be ran only once per app lifecycle, so I prefer to keep it clear.

## Applying the migrations

I decided to implement the strategy pattern : for each type of objects there is one strategy, so user will be able to implement their own strategies.

This piece of code goes on the "//do something" from the previous code sample :

```cs
var parsedMigration = new ParsedMigrationName(migration);
var strategy = strategies.FirstOrDefault(s => s.Handle(parsedMigration));
if (strategy == null)
{
    throw new InvalidOperationException(string.Format("No strategy found for migration '{0}", migration));
}

await client.CreateDatabaseIfNotExistsAsync(parsedMigration.DataBase);
if (parsedMigration.Collection != null)
{
    await client.CreateDocumentCollectionIfNotExistsAsync(UriFactory.CreateDatabaseUri(parsedMigration.DataBase.Id), parsedMigration.Collection);
}

await strategy.ApplyMigrationAsync(client, parsedMigration, migrationContent);
```

- Parsed migration is the class that checks and reads the ressource name. Each ressource must respect the documented convention : the "Test.js" file located in "CosmosDB/Migrations/TestDataBase/StoredProcedure/" is the content of the stored procedure called "Test" located in the database "TestDataBase" and in the collection
- Here I also create the required database and collection

## Strategy implementation

For each kind of object I want to handle I have to create an implementation of the strategy, here is the one for the triggers :

```cs 
internal class TriggerMigrationStrategy : IMigrationStrategy
{
    public async Task ApplyMigrationAsync(IDocumentClient client, ParsedMigrationName migration, string content)
    {
        var nameSplit = migration.Name.Split('-');

        Trigger trigger = new Trigger()
        {

            Body = content,
            Id = nameSplit[2],
            TriggerOperation = (TriggerOperation)Enum.Parse(typeof(TriggerOperation), nameSplit[1]),
            TriggerType = (TriggerType)Enum.Parse(typeof(TriggerType), nameSplit[0])
        };
        await client.UpsertTriggerAsync(
                        UriFactory.CreateDocumentCollectionUri(migration.DataBase.Id, migration.Collection.Id),
                        trigger);
    }

    public bool Handle(ParsedMigrationName migration)
    {
        return migration.Type == "Trigger";
    }
}
```

- For the triggers I need more information than the name : the type and the operation, so I created an other convention for setting those on the file name : {Type}-{Operation}-{Name}

## Using the library

For using the library you need to :
- Install the package "Install-Package RemiBou.CosmosDB.Migration -Version 0.0.0-CI-20190126-145359" (it's still a pre release, I'll try o have a better versionning strategy later).
- create the folders "CosmosDB" and "CosmosDB/Migrations"
- Add "<EmbeddedResource Include="CosmosDB\Migrations\**\*.js" />" to your project csproj
- Add your migration respecting the convention
- Add this code when you want to run the migrations (I guess on App Startup.)
```cs 
await new CosmosDBMigration(documentClient).MigrateAsync(this.GetType().Assembly);// add .Wait() if your are in a not in an async context like the Startup
```
- Run your app

I'll try later to automate the step 2 and 3 when installing the package.

## Conclusion

This package was fun to build and design and I think I got something nice. Now I have many features to do [read here the todo list](https://github.com/RemiBou/RemiBou.CosmosDB.Migration)), but before that I have to setup the test suite so the user will be reassured that the package is stable. 