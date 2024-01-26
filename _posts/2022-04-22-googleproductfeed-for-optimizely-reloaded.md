---
title: GoogleProductFeed for Optimizely - Reloaded
author: valdis
date: 2022-04-22 00:15:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source, google product feed, xml, csv]
---

The previous version of GoogleProductFeed package for [EPiServer](https://nuget.optimizely.com/package/?id=Geta.GoogleProductFeed&ref=tech-fellow.eu) / [Optimizely](https://nuget.optimizely.com/package/?id=Geta.Optimizely.GoogleProductFeed&ref=tech-fellow.eu) had some inherited problems. In case you need a few product feed format "exports" of your catalog data - loading multiple times the same catalog content was one of the major issues. There are sites that have thousands of products and variations and loading them takes quite a bit of time.

Mainly performance and time it took to generate feed was the main problem with the previous version of the package. There were also cases when data needs to be "enriched" before it's exposed to the public - like loading prices or checking the availability of the SKU. Sometimes enrichment can also take a long time. Doing this for each feed is a waste of resources if we put it mildly.

## Geta ProductFeed for Optimizely

So taking all known issues into account decision was to reload the package and refactor some of its internals to support a very similar concept for processing pipelines in factories.

With this approach, it's now supported single load of the catalog content, single (or multiple) enrichment steps for loaded data, and one or many product feed endpoint exporters.

We have created a bunch of new packages - [Geta.Optimizely.ProductFeed](https://nuget.optimizely.com/package/?id=Geta.Optimizely.ProductFeed&ref=tech-fellow.eu) and [friends](https://nuget.optimizely.com/?q=geta.optimizely.productfeed&s=Popular&r=10&f=All&ref=tech-fellow.eu).

### Getting Started

Getting started with Geta product feed is pretty easy. You need to install core / shared package:

```dotnetcli
> dotnet install Geta.Optimizely.ProductFeed
```

And just configure it in **Startup.cs** file:

```csharp
services
    .AddProductFeed<T>(options =>
    {
        // setup product feed config
    });
```

Here `T` is an entity to which generic `CatalogContentBase` class will be mapped to.

Feed generation is still implemented as Optimizely scheduled job (look for "ProductFeed - Create feeds").

### Processing Pipeline

During the processing pipeline there are a few key moments to be aware of (you can override any of these mentioned below):

1. Catalog Data Load - loads data from the Optimizely Commerce catalog.
2. Catalog Data Map - loaded data usually comes in `CatalogContentBase` shape. This step allows mapping to `T` data type (mentioned in `AddProductFeed<T>()` method).
3. Enrichments - When loaded data is mapped to a custom entity, the processing pipeline can start the work. Enrichments are responsible for loading some heavy data and adding necessary metadata to the `T` entity.
4. Feed Exporters - exporters are responsible for generating feed content in a specific format and using specific feed entities.
5. Feed entity Converter - converters are responsible for taking the projected entity (`T` mentioned in `AddProductFeed()`) and return a feed entity which will be used to store the actual product feed in the underlying storage.
6. Storage Providers - right now we only have MSSQL storage implementation. But should be quite easy to implement the next one.

### Add Google Xml Product Feed

First you need to install Google Xml product feed package:

```dotnetcli
> dotnet add package Geta.Optimizely.ProductFeed.Google
```

Following code adds Google Xml product feed functionality to your site:

```csharp
services
    .AddProductFeed<MyCommerceProductRecord>(options =>
    {
        options.ConnectionString = _configuration.GetConnectionString("EPiServerDB");
        options.SetEntityMapper<EntityMapper>();

        options.AddEnricher<FashionProductAvailabilityEnricher>();

        options.AddGoogleXmlExport(d =>
        {
            d.FileName = "/google-feed";
            d.SetConverter<GoogleXmlConverter>();
        });
    });
```

Few notes:

1. Loaded commerce catalog data will be mapped to `MyCommerceProductRecord` class.
2. `EntityMapper` will be used to do this mapping.
3. `FashionProductAvailabilityEnricher` will be used to process each `MyCommerceProductRecord` and set SKU availability by some criteria.
4. Google Xml product feed entities will be generated using `GoogleXmlConverter` class.
5. Feed data will be stored in MSSQL database under `"EPiServerDB"` connection string.
6. Google Xml product feed will be mounted to `/google-feed` URL.

Custom mapped entity class (`MyCommerceProductRecord`):

```csharp
public class MyCommerceProductRecord
{
    public string Code { get; set; }
    public string DisplayName { get; set; }
    public string Description { get; set; }
    public string Url { get; set; }
    public string Brand { get; set; }
    public string ImageLink { get; set; }
    public bool IsAvailable { get; set; }
}
```

Custom entity mapper (`EntityMapper.cs`):

```csharp
public class EntityMapper : IEntityMapper<MyCommerceProductRecord>
{
    public MyCommerceProductRecord Map(CatalogContentBase catalogConte
    {
        return ...
    }
}
```

Enricher (`FashionProductAvailabilityEnricher.cs`):

```csharp
public class FashionProductAvailabilityEnricher :
    IProductFeedContentEnricher<MyCommerceProductRecord>
{
    public MyCommerceProductRecord Enrich(
        MyCommerceProductRecord sourceData,
        CancellationToken cancellationToken)
    {
        // enrich SKU availability
    }
}
```

Google Xml feed entity converter (`GoogleXmlConverter.cs`):

```csharp
public class GoogleXmlConverter : IProductFeedConverter<MyCommerceProductRecord>
{
    public object Convert(MyCommerceProductRecord entity)
    {
        return new Geta.Optimizely.ProductFeed.Google.Models.Entry
        {
            // set properties for feed entity
        }
    }
}
```

### Add CSV Product Feed

Adding CSV export is quite easy as well.

```dotnetcli
> dotnet add package Geta.Optimizely.ProductFeed.Csv
```

Then add somewhat similar code to your `Startup.cs` file:

```csharp
services
    .AddProductFeed<MyCommerceProductRecord>(options =>
    {
        options.ConnectionString = _configuration.GetConnectionString("EPiServerDB");
        options.SetEntityMapper<EntityMapper>();

        options.AddEnricher<FashionProductAvailabilityEnricher>();

        options.AddCsvExport(d =>
        {
            d.FileName = "/csv-feed-1";
            d.SetConverter<CsvConverter>();
            d.CsvEntityType = typeof(CsvEntry);
        });
    });
```

All processing pipeline logic is the same as for the Google Xml feed except following changes:

1. CSV product feed entity is set to `CsvEntity`.
2. Feed entity is converted via `CsvConverter`.
3. Feed is mounted to `/csv-feed-1` route.

Custom CSV entity (`CsvEntity.cs`):

```csharp
public class CsvEntry
{
    public string Name { get; set; }
    public string Code { get; set; }
    public decimal Price { get; set; }
    public bool IsAvailable { get; set; }
}
```

CSV converter (`CsvConverter.cs`):

```csharp
public class CsvConverter : IProductFeedConverter<MyCommerceProductRecord>
{
    public object Convert(MyCommerceProductRecord entity)
    {
        return new CsvEntry
        {
            // set properties for the entity
        };
    }
}
```

CSV feed will have a header row generated out-of-the-box based on properties in `CsvEntity` class.

Currently, there is no option to disable this functionality. If you need one - please file an issue in GitHub [repo](https://github.com/Geta/geta-optimizely-productfeed/issues?ref=tech-fellow.eu).

## Under the Hood

General pipeline look & feel is shown in the picture below.

Job starts with **loading** the data from the Optimizely catalog. Then **mapping** to the custom entity. **Enriching** it with some additional data (if needed). And then exporting to different formats via different **exports** and **converters**. Lastly stored feed is **exposed** via mounted route.

![](/assets/img/2022/04/productfeed-1.png)

High level mapping between setup code and the processing pipeline is shown in the image below.

![](/assets/img/2022/04/productfeed-2-2.png)

Happy product feeding!

[*eof*]
