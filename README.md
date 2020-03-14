# The Tip of an Iceberg: A Story of Code Authors

## Project summary

The authors of source files are important in many applications, but are often not recorded. In the literature, various approaches (e.g., [1]) have been proposed to identify authors of source files. However, as a review [2] pointed out, there is no good benchmark, which hinders the exploration of this research line. In addition, although researchers analyze the challenges in this research line [2], their analysis is built upon their personal experience without sold empirical evidences. To the best of our knowledge, there is no empirical study to reveal the challenges

In this project, we propose a benchmark called CodA. In our benchmark, we accurately extracted the code authors for source files. In a source file, we marked authors line by line. Furthermore, based on our benchmark, we conduct an empirical study to reveal the challenges of identifying code authors. 

Here is the list of our [benchmark](https://anonymous.4open.science/repository/643bb230-7da2-4b2c-858f-6ee267f7db9f/benchmark/).

An example:
```Java
1:09c2697:     ValueNode bindExpression(
1:71c8e86:     FromList fromList, SubqueryList subqueryList, List<AggregateNode> aggregates)
1:eac0369:         throws StandardException
...

author:Richard N. Hillegas
commit:71c8e86
/////////////////////////////////////////////////////////////////////////
1:     FromList fromList, SubqueryList subqueryList, List<AggregateNode> aggregates)
commit:09c2697
/////////////////////////////////////////////////////////////////////////
1:     ValueNode bindExpression(
0:     FromList fromList, SubqueryList subqueryList, List aggregates)
1:         super.bindExpression(fromList, subqueryList, aggregates);
/////////////////////////////////////////////////////////////////////////
...
author:Daniel John Debrunner
commit:eac0369
...
1: 			throws StandardException
```

## Reference
[1] Aylin Caliskan-Islam, Richard Harang, Andrew Liu, Arvind Narayanan, Clare Voss, Fabian Yamaguchi, and Rachel Greenstadt. 2015. De-anonymizing programmers via code stylometry. In Proc. USENIX Security. 255¨C270.

[2] Vaibhavi Kalgutkar, Ratinder Kaur, Hugo Gonzalez, Natalia Stakhanova, and Alina Matyukhina. 2019. Code Authorship Attribution: Methods and Challenges. Comput. Surveys 52, 1 (2019), 3.
