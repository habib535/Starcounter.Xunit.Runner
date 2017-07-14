# Starcounter2.3_xunit
Proof of concept for testing Starcounter Apps and Databases in runtime. Import `ScXunitRunner` to a normal Starcounter version 2.3.1.X App and start write the xunit tests.

## How to use it
* Create a normal Starcounter App, version 2.3
* Add reference to `ScXunitRunner` (nuget package is coming soon)
* Add [references the Xunit and Xunit.Console.Runner libraries](https://xunit.github.io/docs/getting-started-desktop.html)
* Starcounter.dll, Starcounter.Internal.dll and Starcounter.XSON.dll references needs to be copied to the output directory, i.e. `Copy Local = True`
* Create an instance of `StarcounterXunitRunner` in `Main()`
* Write Xunit tests
* Any time the Starcounter database is accessed, the code needs to be wrapped around `Scheduling.ScheduleTask(() => { /* Test code here */}, waitForCompletion: true);`. Make sure to set `waitForCompletion = true` because `Xunit` may execute tests in parallell.
* Compile and deploy the Starcounter test App as any other Starcounter App
* Execute the tests by calling the Url `/ScXunitRunner/<CallingAssemblyName>`

## Future implementation
* nuget support for the `ScXunitRunner` library
* Test output in JSON format, using `Xunit.Runners.*Info` classes format

## ScXunitRunner
Using:
* .NET Framework 4.5.2
* Starcounter version 2.3.1.X. 
* xunit 2.2.0
* xunit.runner.console 2.2.0

Adds a Starcounter handler, `/ScXunitRunner/<CallingAssemblyName>`, which executes `Xunit` tests as a xunit console application in the same AppDomain as the Starcounter database. `<CallingAssemblyName>` may be overwritten in the `StarcounterXunitRunner` constructor. There is also a public string getter `StarcounterXunitRunner.XunitRunnerUrl` which executes all the `Xunit` from the calling assembly in the same AppDomain as the Starcounter database.

## Demo\ScTestApp
A Starcounter version 2.3.1.X test App which imports `ScXunitRunner`. It illustrates how to use `ScXunitRunner`. 

First off, it create an instance of ScXunitRunner in order to generate test UriHandler.
```c#
public class Program
{
    static void Main()
    {
        StarcounterXunitRunner runner = new StarcounterXunitRunner();
    }
}
```

When the handler is called, the CallingAssembly, i.e. this app, is executed in a `Xunit.Console.Runner` and the output is given as a http response.

The example below creates `ScApp.ScAppDb` rows for verifying the proof of concept:
```c#
[Fact]
public void TestCase_AddingNewRow()
{
    Scheduling.ScheduleTask(() => {
        int entriesBeforeCount = Db.SQL<ScAppDb>("SELECT x FROM ScApp.ScAppDb x").Count();

        Db.Transact(() =>
        {
            ScAppDb entry = new ScAppDb();
        });

        int entriesAfterCount = Db.SQL<ScAppDb>("SELECT x FROM ScApp.ScAppDb x").Count();

        Assert.True(int.Equals(entriesBeforeCount + 1, entriesAfterCount), $"ScApp.ScAppDb beforeCount={entriesBeforeCount} is not one less than afterCount={entriesAfterCount}");
    }, waitForCompletion: true);
}
```

Test result from running `/ScXunitRunner/ScTestApp` handler.
```
[1/9] ScTestApp.TestSetDoNotSaveContext.TestSetDoNotSaveContext_Test.................. PASSED. Execution time: 0,000s
[2/9] ScTestApp.TestSetDoNotSaveContext.TestSetDoNotSaveContext_Test2................. PASSED. Execution time: 0,000s
[3/9] ScTestApp.TestSetDoNotSaveContext.TestSetDoNotSaveContext_Test3................. PASSED. Execution time: 0,000s
[4/9] ScTestApp.TestSetAddingDatabaseEntries.TestCase_AddingNewRow_3.................. PASSED. Execution time: 0,028s
[5/9] ScTestApp.TestSetAddingDatabaseEntries.TestCase_AddingNewRow_1.................. PASSED. Execution time: 1,397s
[6/9] ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_1......................... FAILED. Execution time: 1,425s
This assertion will always fail
Expected: True
Actual:   False   at Xunit.Assert.True(Nullable`1 condition, String userMessage)
   at ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_1() in C:\GitHub\home\Starcounter2.3_xunit\Demo\ScTestApp\TestSetAlwaysFailing.cs:line 13
[7/9] ScTestApp.TestSetAddingDatabaseEntries.TestCase_AddingNewRow_2.................. PASSED. Execution time: 0,648s
[8/9] ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_2......................... FAILED. Execution time: 0,648s
This assertion will always fail
Expected: True
Actual:   False   at Xunit.Assert.True(Nullable`1 condition, String userMessage)
   at ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_2() in C:\GitHub\home\Starcounter2.3_xunit\Demo\ScTestApp\TestSetAlwaysFailing.cs:line 19
[9/9] ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_3......................... FAILED. Execution time: 0,768s
This assertion will always fail
Expected: True
Actual:   False   at Xunit.Assert.True(Nullable`1 condition, String userMessage)
   at ScTestApp.TestSetAlwaysFailing.TestCase_AlwaysFailing_3() in C:\GitHub\home\Starcounter2.3_xunit\Demo\ScTestApp\TestSetAlwaysFailing.cs:line 25

Total execution time: 2,848s, Passed: 6, Failed: 3, Skipped: 0
```

## Demo\ScApp
A dummy Starcounter version 2.3.1.X App which only has one database-table. Used for testing purposes only.
```c#
[Database]
public class ScAppDb
{
    public string Name { get; set; }
    public int Integer { get; set; }
    public DateTime DateCreated { get; set; }
}
```

It has handlers for the following
* `/ScApp/CreateNewEntry` - create a new row with random data
* `/ScApp/CreateNewEntry/<# of new entries>` - create multiple new rows with random data
* `/ScApp/DeleteAll` - delete all rows
* `/ScApp/ListAll` - list all rows in text format