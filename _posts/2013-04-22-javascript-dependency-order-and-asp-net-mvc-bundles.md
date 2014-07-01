---
layout: post
title: Javascript dependency order and ASP.NET MVC Bundles
tags:
- C#
- software development
- web optimization
status: publish
type: post
published: true
meta:
  _publicize_pending: '1'
author: 
---
Inevitably you find yourself working on contract on a "simple" project that has become increasingly complex, growing from a few javascript files to hundreds.

#### ***side note***

This post is targeted at solely .NET environments. Yes there is AMD with [requirejs](http://requirejs.org/) and [commonjs ](http://wiki.commonjs.org/wiki/Modules)and the various module loaders and build tools, but sometimes not every tool is available and find yourself working within a <del>stubborn </del>locked down environment operated by the [BOFH](http://en.wikipedia.org/wiki/Bastard_Operator_From_Hell), limiting whats available for the job.

## tl;dr;

*The problem*: "I am using System.Web.Optimization but I want my javascript in my bundle ordered by dependency!!!"

*The solution*: Custom IBundleOrderer implementation with regular expressions and a topological sort.

## Continuing on

With ASP.NET MVC 4 (and 3)  you should be taking advantage of System.Web.Optimization and [bundle and minify](http://www.asp.net/mvc/tutorials/mvc-4/bundling-and-minification) those scripts to keep your page times snappy.

When creating your bundle you can include files individually or by folder using a wildcard.

Adding files individually allows you to control the order that files appear in the bundle, but for your massive project who wants to maintain that through refactors and new features.

Adding via wildcard is so much simpler but the files are added alphabetically which hurts when abc.js refers to components in xyz.js.

So to control the order of the wildcard bundle you could create a custom IBundleOrderer implementation and specify which files should appear first, which can lead you back towards the maintenance problem mentioned above.

## JavascriptDependencyOrderer: A smarter IBundleOrderer

Our implementation will apply a [topological sorting algorithm](http://en.wikipedia.org/wiki/Topological_sorting) to the list of files supplied by the bundle, ...err... sorting the files according to their dependencies.

To use the topsort however we need to create a adjacency graph describing the files and their relationships.

Each vertex in the graph is represented by the individual file names provided to the bundle.

Each edge is represented by the file name of each file's dependency.

### But how do we discover the dependencies??

First we need to add a little bit of ceremony to our javascript files and list the required dependencies.

Here we borrow some similar syntax used by client side module loaders using a "require" statement which is wrapped in some comments as to not give us any client script errors.

So an example ModelA.js might be....

````js
/* require('ModelB.js') */
(function (mynamespace) {
  var mynamespace.ModelA = mynamespace.ModelB.extend({});
} (window.myuni = window.myuni || {} ));

````

I like the require statement because if in the future I ever could use a more appropriate tool, part of the work is kinda already done. But you could use any pattern you desired.

The commented section is also removed by the bundle when it is minified.

Listing the dependency here makes the it more explicit on what components this bit of code is using and frees us from having to manage not only this file's dependencies but also how their order may impact the dependency order on other files.

Next, a regular expression is then used as each file is processed to grab the dependencies from the require statement and build up the graph edges.

### What about 3rd party libraries and their dependencies?

A good example is Backbone.js having a dependency on underscore.js

Not wanting to modify the source of these external libraries with our extra "require" ceremony, we add some extra logic to our [IBundleOrderer ](http://msdn.microsoft.com/en-us/library/system.web.optimization.ibundleorderer.aspx)implementation to explicitly add these files first and in the order they are listed.

These explicit items are then excluded from the topological sort as they are assumed to be already added.

### Building the topological sort

I chose to use the [QuickGraph](http://quickgraph.codeplex.com/) nuget package but you can also [roll your own](http://tawani.blogspot.com.au/2009/02/topological-sorting-and-cyclic.html) for finer control over performance, error handling, and debugging.

By using the QuickGraph package some extra guard clauses and exception handling was needed to better identify potential sources of error.

### Putting it together

````csharp
public class JavascriptDependencyOrderer : IBundleOrderer
{
  private readonly string dependencyRegex;
  private readonly List<Field> fields;
  public List<string> ExplicitOrder { get; set; }

  public JavascriptDependencyOrderer()
    : this(@"(?<=require\(['\""]).*?(?=['\""]\))")
  {}

  public JavascriptDependencyOrderer(string dependencyRegex)
  {
    this.dependencyRegex = dependencyRegex;
    fields = new List<Field>();
    ExplicitOrder = new List<string>();
  }

  #region IBundleOrderer Members

  public IEnumerable<FileInfo> OrderFiles(BundleContext context, IEnumerable<FileInfo> files)
  {
    FindDependecies(files);
    List<FileInfo> result = 
          ExplicitOrder
            .Select(fileName => files.FirstOrDefault(x => x.Name == fileName))
            .Where(f => f != null).ToList();
     
    result.AddRange(BuildAndSortDependencies()
          .Select(sortedVertex => files.FirstOrDefault(x => x.Name == sortedVertex)));

    return result;
  }

  #endregion

  private void FindDependecies(IEnumerable<FileInfo> files)
  {
    var regex = new Regex(dependencyRegex);
    foreach (FileInfo fileInfo in files)
    {
      if(!ExplicitOrder.Contains(fileInfo.Name))
      {
        var file = new StreamReader(fileInfo.OpenRead());
        MatchCollection matches = regex.Matches(file.ReadToEnd());

        fields.Add(new Field{
                    Name = fileInfo.Name,
                    DependsOn = matches.OfType<Match>()
                      .Select(match =>
                      {
                        if (files.Any(x => x.Name == match.Groups[0].Value))
                        {
                          return match.Groups[0].Value;
                        }
                        else
                        {
                          throw new Exception(
                            string.Format("Dependency {0} for {1} could not be found in supplied list of files",
                              match.Groups[0].Value,
                              fileInfo.Name));
                        }
                      }).ToArray()
                });
      }
    }
  }

  private IEnumerable<string> BuildAndSortDependencies()
  {
    try
    {
      var adjacencyGraph = new AdjacencyGraph<string, Edge<string>>();

      foreach (Field field in fields)
      {
        adjacencyGraph.AddVertex(field.Name);

        foreach (string dependecy in field.DependsOn)
        {
           adjacencyGraph.AddEdge(new Edge<string>(field.Name, dependecy));
        }

      }

      var topSort = new TopologicalSortAlgorithm<string, Edge<string>>(adjacencyGraph);
      topSort.Compute();

      return topSort.SortedVertices.Reverse();
    }
    catch(NonAcyclicGraphException cyclicException)
    {
      throw new Exception("Circular reference detected while processing javascript dependency order",cyclicException);
    }
    catch(KeyNotFoundException keyNotFoundException)
    {
      throw new Exception("Dependency could not be found. Check that the file names match.", keyNotFoundException);
    }
  }

  #region Nested type: Field

  private class Field
  {
    public string Name { get; set; }
    public string[] DependsOn { get; set; }
    public override string ToString(){ return Name; }
  }
  #endregion
}
````

### And now we use it

Finally we can get our global.asax / bundle config back under control and more maintainable using the wildcard search and our orderer

````csharp
// Site Javascript Bundle

Bundle bundle = new ScriptBundle("~/Scripts/site.js");
bundle.Transforms.Add(new JsMinify());
bundle.IncludeDirectory("~/scripts", "*.js", true);

var jsOrder = new JavascriptDependencyOrderer();
jsOrder.ExplicitOrder.Add("jquery-1.8.0.js");
jsOrder.ExplicitOrder.Add("underscore.js");
jsOrder.ExplicitOrder.Add("backbone.js");

bundle.Orderer = jsOrder;

BundleTable.Bundles.Add(bundle);
````
##Last words...

There are still a few improvements I would like to make, such as

* allowing the dependencies to be case insensitive
* having the vertex key be more unique by including the part of the path
* or perhaps some type of shim for 3rd party files

Either way, for our project in our environment its working well and removed some heartache.

As always comments, criticisms, and corrections are eagerly welcomed.