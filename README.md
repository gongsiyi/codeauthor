# Code Authors Hidden in File Revision Histories: An Empirical Study

## Project summary

The authors of source files are important in many applications, but are often not recorded. Even if they are recorded in a code repository, many authors are hidden in revision histories. In the literature, various approaches (e.g., [1]) have been proposed to identify authors of source files. However, the true authors of code lines are still largely unknown, and many fundamental questions along with code authors are still open. For example, a recent review [2] complains that most approaches assume
that a source file is written by only an author, but source files in the wild are typically written by multiple programmersas. In addition, researchers analyze the challenges in this research line [2], although we agree that their visions are insightful, they did not provide any empirical evidences to support their listed challenges. To deepen the understanding on code authorship, there is a strong need for an empirical study.

To meet the timely need, we conducted the first empirical study on code authors. To assist our study, we implemented a tool called CODA. It extracts authors and their modifications from code repositories and matches modifications with the latest source files to determine the true author of each line.

Here is the list of our [dataset](https://anonymous.4open.science/repository/643bb230-7da2-4b2c-858f-6ee267f7db9f/benchmark/).

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
