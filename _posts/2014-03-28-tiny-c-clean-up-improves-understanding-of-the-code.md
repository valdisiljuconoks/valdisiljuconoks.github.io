---
title: Tiny C# clean-up improves understanding of the code
author: valdis
date: 2014-03-28 23:40:00 +0200
categories: [.NET, C#]
tags: [.net, c#]
---

I’m starting to collect small fragments from everyday life to understand and increase understanding and readability of code I’m coming across.
First piece in this category is gone which I came across while contributing to one of the open source modules.

Original fragment:

```csharp
{
    IEnumerable btypes = GetBlockTypes();
    List blockproperties = new List();
    int x = 0;

    foreach (BlockType type in btypes)
    {
        x = 0;

        foreach (PropertyDefinition def in type.PropertyDefinitions)
        {
            x++;
            blockproperties.Add(new TypePropertyResultItem {
                TypeName = x == 1 ? type.Name : "",
                PropertyName = def.Name,
                Description = def.TranslateDescription(),
                DisplayName = def.TranslateDisplayName()
            });
        }
    }

    return blockproperties;
}
```

Modified code fragment:

```csharp
{
    var types = GetBlockTypes();
    var blockproperties = new List();

    foreach (var type in types)
    {
        blockproperties.AddRange(type.PropertyDefinitions.Select((def, i) =>
            new TypePropertyResultItem
            {
                TypeName = i == 0 ? type.Name : string.Empty,
                PropertyName = def.Name,
                Description = def.TranslateDescription(),
                DisplayName = def.TranslateDisplayName()
            }));
    }

    return blockproperties;
}
```

Happy reading not your code! :)

[*eof*]
