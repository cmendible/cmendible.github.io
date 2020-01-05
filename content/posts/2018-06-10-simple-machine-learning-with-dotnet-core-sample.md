---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2018-06-10T20:39:00Z"
description: Simple Machine Learning with .NET Core Sample
image: /wp-content/uploads/2017/07/dotnetcore.png
published: true
tags: ["MachineLearning", "Titanic", "BinaryClassifier", "Classifier"]
title: Simple Machine Learning with .NET Core Sample
---

I think that more than a year and a half ago I read ["Real-World Machine Learning"](https://www.manning.com/books/real-world-machine-learning) by Henrik Brink, Joseph
Richards, and Mark Fetherolf. A book that is easy to read and goes "to the point"!!! I'm sure you know what I mean.

At the time the only thing that prevented me from really enjoying the book samples was that there was no easy way to "translate" them to .NET Core.

Well well, times have changed and now [ML.NET](https://www.microsoft.com/net/learn/apps/machine-learning-and-ai/ml-dotnet/get-started/windows) (aka Microsoft.ML) exists: An open source and cross-platform machine learning framework for .NET.

The following sample will show you how to train a model with a binary classifier algorithm to determine if a passenger aboard the Titanic survived or not (Based on the sample provided in chapter 3 of the book).

## 1. Create your project
---
Open a command prompt and run:

``` shell
md titanic.ml
cd titanic.ml
dotnet new console
```

## 2. Add a reference to ML.NET
---

The nuget package for ML.NET is: [Microsoft.ML](https://www.nuget.org/packages/Microsoft.ML/) so add a reference to it:

``` powershell
dotnet add package Microsoft.ML -v 0.2.0
dotnet restore
```

## 3. Replace the content of Program.cs
---
Replace the content of Program.cs with the following contents:

``` csharp
using Microsoft.ML;
using Microsoft.ML.Data;
using Microsoft.ML.Runtime.Api;
using Microsoft.ML.Trainers;
using Microsoft.ML.Transforms;
using System;
using System.Threading.Tasks;

namespace Titanic.ML
{
    class Program
    {
        static async Task Main(string[] args)
        {
            // Create a learning pipeline
            var pipeline = new LearningPipeline();

            // Load training data and add it to the pipeline
            string dataPath = @".\data\titanic.training.csv";
            var data = new TextLoader(dataPath).CreateFrom<TitanicData>(useHeader: true, separator: ',');
            pipeline.Add(data);

            // Transform any text feature to numeric values
            pipeline.Add(new CategoricalOneHotVectorizer(
                "Sex",
                "Ticket",
                "Fare",
                "Cabin",
                "Embarked"));

            // Put all features into a vector
            pipeline.Add(new ColumnConcatenator(
                "Features",
                "Pclass",
                "Sex",
                "Age",
                "SibSp",
                "Parch",
                "Ticket",
                "Fare",
                "Cabin",
                "Embarked"));

            // Add a learning algorithm to the pipeline.
            // This is a classification scenario (Did this passenger survive?)
            pipeline.Add(new FastTreeBinaryClassifier() { NumLeaves = 5, NumTrees = 5, MinDocumentsInLeafs = 2 });

            // Train your model based on the data set
            Console.WriteLine($"Training Titanic.ML model...");
            var model = pipeline.Train<TitanicData, TitanicPrediction>();

            // Optional: Save the model to a file. You can use teh model in another program!!!
            var modelPath = @".\data\titanic.model";
            await model.WriteAsync(modelPath);

            // Use your model to make a prediction. Let's predict what happened to this passenger
            var prediction = model.Predict(new TitanicData()
            {
                Pclass = 3f,
                Name = "Braund, Mr. Owen Harris",
                Sex = "male",
                Age = 31,
                SibSp = 0,
                Parch = 0,
                Ticket = "335097",
                Fare = "7.75",
                Cabin = "",
                Embarked = "Q"
            });

            Console.WriteLine($"Did this passenger survive? {(prediction.Survived ? "Yes" : "No")}");

            // Evaluate the model using the test data. In other words let's test the model 
            // to see how accurate it is.
            Console.WriteLine($"Evaluating Titanic.ML model...");
            dataPath = @".\data\titanic.csv";
            data = new TextLoader(dataPath).CreateFrom<TitanicData>(useHeader: true, separator: ',');
            var evaluator = new Microsoft.ML.Models.BinaryClassificationEvaluator();
            var metrics = evaluator.Evaluate(model, data);

            Console.WriteLine($"Accuracy: {metrics.Accuracy:P2}");
            Console.WriteLine($"Auc: {metrics.Auc:P2}");
            Console.WriteLine($"F1Score: {metrics.F1Score:P2}");
        }
    }
}
```

**Note:** I splitted the original dataset from the [book](https://github.com/brinkar/real-world-machine-learning) for this post.

Please find the code and data files [here](https://github.com/cmendible/dotnetcore.samples/tree/master/titanic.ml).

Hope it helps!