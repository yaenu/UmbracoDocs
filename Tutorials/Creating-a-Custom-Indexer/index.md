---
versionFrom: 8.0.0
needsV8Update: "false"
---


# Tutorial - Creating a Custom Indexer

## Overview

This guide takes you through the steps to setup a simple Custom Index in Umbraco. Indexers are visible in backoffice in ```Examine Management``` section of ```Settings```.

## What is an Index?

In Umbraco an Index is a file structure containing meta data which allow to search content more efficiently.

Basically Umbraco provide tree built-in Indexer (InternalIndex, ExternalIndex, MemberIndex). That means that if you create content based on your own documents type, it will automagically be indexed by InternalIndex and ExternalIndex.

## Why creating a Custom Index ?

Before creating a custom index it's maybe a good idea to think about the data architecture.

Depending on your needs, you have basically two possible architecture:
* Store data as custom document types, manage them with a custom dashboard or through api and using built-in indexer
* Store data into index, create a custom indexer 

If you are in the first case you should maybe read the following Tutorial : [Creating a Custom Dashboard](../Creating-a-Custom-Dashboard)

If you are in the second case you are at the right place.

## Index structure

An Index is composed by following elements:
* CustomIndex : UmbracoExamineIndex
* CustomIndexCreator : LuceneIndexCreator, IIndexCreator
* CustomValueSetBuilder : IValueSetBuilder<CustomData>
* CustomPopulator : IndexPopulator<CustomIndex>

## Let's code

### The Composer and Component
As many other customizations, v8 use now the Composer to define index.


``` csharp
namespace CustomIndexTutorial.Web.App_Start
{
    [RuntimeLevel(MinLevel = RuntimeLevel.Run)]
    public class CustomIndexComposer : ComponentComposer<CustomIndexComponent>, ICoreComposer
    {
        public override void Compose(Composition composition)
        {
            base.Compose(composition);

            composition.Register<CustomIndexPopulator>(Lifetime.Singleton);

            composition.RegisterUnique<CustomIndexCreator>();

            composition.RegisterUnique<CustomIndexValueSetBuilder>();
        }
    }

    public class CustomIndexComponent : IComponent
    {
        private readonly IExamineManager _examineManager;
        private readonly AxisIndexCreator _indexCreator;

        // the default enlist priority is 100
        // enlist with a lower priority to ensure that anything "default" runs after us
        // but greater that SafeXmlReaderWriter priority which is 60
        private const int EnlistPriority = 80;

        public CustomIndexComponent(
            IExamineManager examineManager,CustomIndexCreator indexCreator)
        {
            _indexCreator = indexCreator;
        }

        public void Initialize()
        {
            //we want to tell examine to use a different fs lock instead of the default NativeFSFileLock which could cause problems if the AppDomain
            //terminates and in some rare cases would only allow unlocking of the file if IIS is forcefully terminated. Instead we'll rely on the simplefslock
            //which simply checks the existence of the lock file
            DirectoryFactory.DefaultLockFactory = d =>
            {
                var simpleFsLockFactory = new NoPrefixSimpleFsLockFactory(d);
                return simpleFsLockFactory;
            };

            //create the indexes and register them with the manager
            foreach (var index in _indexCreator.Create())
                _examineManager.AddIndex(index);

        }

        public void Terminate()
        { }
    }
}
```

### The Index

```CustomIndex.cs```
``` csharp
namespace CustomIndexTutorial.Web.Examine
{
    public class CustomIndex : UmbracoExamineIndex
    {
        public AxisIndex(
            string name,
            Directory luceneDirectory,
            FieldDefinitionCollection fieldDefinitions,            
            Analyzer analyzer,
            IProfilingLogger profilingLogger,
            IValueSetValidator validator = null) :
            base(name, luceneDirectory, fieldDefinitions, analyzer, profilingLogger, validator)
        {
        }
    }
}
```

```CustomIndexCreator.cs```
``` csharp
namespace CustomIndexTutorial.Web.Examine
{
    public class CustomIndexCreator : LuceneIndexCreator, IIndexCreator
    {
        public CustomIndexCreator(IProfilingLogger profilingLogger)
        {
            ProfilingLogger = profilingLogger;
        }

        public const string CustomIndexName = "CustomIndex";
        public const string CustomIndexPath = "CustomIndex";

        protected IProfilingLogger ProfilingLogger { get; }

        public override IEnumerable<IIndex> Create()
        {
            var customIndex = new CustomIndex(
                    CustomIndexName,
                    CreateFileSystemLuceneDirectory(CustomIndexPath),
                    new FieldDefinitionCollection(),
                    new CultureInvariantWhitespaceAnalyzer(),
                    ProfilingLogger,
                    null  //optionnaly it's possible to define a new CustomIndexValueSetValidator() 
                );

            return new[] { customIndex };
        }
    }
}
```

```CustomValueSetBuilder.cs```
``` csharp
namespace CustomIndexTutorial.Web.Examine
{
    public class CustomIndexValueSetBuilder : IValueSetBuilder<CustomData>
    {
        public  IEnumerable<ValueSet> GetValueSets(params CustomData[] peoples)
        {
            foreach (var p in peoples)
            {
                var indexValues = new Dictionary<string, object>
                {
                    ["firstname"] = p.FirstName,
                    ["lastname"] = p.LastName,
                    ["email"] = p.Email,
                    ["__rawPeople"] = new JavaScriptSerializer().Serialize(p)
                };
        
                var valueSet = new ValueSet(p.Email, "people", "people", indexValues);
                yield return valueSet;
            }
        }
    }
}
```

```CustomIndexPopulator.cs```
``` csharp
namespace CustomIndexTutorial.Web.Examine
{
    public class CustomIndexPopulator : IndexPopulator<CustomIndex>
    {
        private readonly CustomIndexValueSetBuilder _valueSetBuilder;

        public AxisIndexPopulator(CustomIndexValueSetBuilder valueSetBuilder)
        {
            _valueSetBuilder = valueSetBuilder;
        }
        protected override void PopulateIndexes(IReadOnlyList<IIndex> indexes)
        {
            if (indexes.Count == 0) return;

            IEnumerable<CustomData> people = new CustomData[] {
                new CustomData(){FirstName = "John", LastName = "Doe", Email = "john.do@do.com"},
                new CustomData(){FirstName = "Peter", LastName = "Doe", Email = "peter.do@do.com"},
                new CustomData(){ FirstName ="Giacomo", LastName = "Alfonso", Email = "giacomo.alfonso@do.com"}
            };

            if (people.Count() > 0)
            {
                foreach (IIndex index in indexes)
                {
                    index.IndexItems(_valueSetBuilder.GetValueSets(people.ToArray()));
                }
            }
        }
    }
}
```

### The Data
```CustomData.cs```
``` csharp
namespace CustomIndexTutorial.Web.Examine
{
    public class CustomData
    {
        public string FirstName {get; set;}
        public string LastName {get; set;}
        public string Email {get; set;}
    }
}
```

## Result
