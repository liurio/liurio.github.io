---
logout: post
title: Flink Table API && Flink SQL分析
tags: [flink,all,sql]
---

## Table API/SQL简介

TableAPI是一种关系型API，类SQL的API，用户可以像操作表一样地操作数据，非常的直观和方便。flink已经拥有了强大的DataStream/DataSet API，满足流计算和批计算中的各种场景需求，Table API还有一个作用是提供一种关系型的API来实现flink API层的流与批的统一。

![introduction](https://gitee.com/liurio/image_save/raw/master/flink/introduction.png)

## Source

source作为Table&SQL API的数据源，同时也是程序的入口。当前Flink的Table&SQL API整体而言支持三种source：Table source、DataSet以及DataStream，它们都通过特定的API注册到Table环境对象。可以从访问的外部的存储系统有数据库，键值存储，消息队列，或文件系统等。

### Table Source

它直接以表对象作为source，这里的表对象可分为两类：

- Flink以Table类定义的关系表对象，通过TableEnvironment的registerTable()方法注册。
- 外部source经过桥接而成的表对象，基础抽象为TableSource，通过具体环境对象的registerTableSource。

下图展示了TableSource被注册时，对应的内部转化图(虚线表示对应关系)：

![tablesource对应关系](https://gitee.com/liurio/image_save/raw/master/flink/tablesource%E5%AF%B9%E5%BA%94%E5%85%B3%E7%B3%BB.png)

由上图可见，不管是直接注册Table对象还是注册外部source，在内部都直接对应了特定的XXXTable对象。当前Flink所支持的TableSource大致上分为两类：

- CsvTableSouce：同时可用于Batch跟Streaming模式
- kafka系列TableSource：包含Kafka的各个版本（0.8，0.9，0.10）以及各种不同的格式（Json、Avro），基本上它们只支持Streaming模式，它们都依赖于各种kafka的connector；KafkaJsonTableSource、KafkaAvroTableSource

```java
// specify JSON field names and types
val typeInfo = Types.ROW(
  Array("id", "name", "score"),
  Array(Types.INT, Types.STRING, Types.DOUBLE)
)

val kafkaTableSource = new Kafka08JsonTableSource(kafkaTopic,kafkaProperties,typeInfo)
tableEnvironment.registerTableSource("kafka-source", kafkaTableSource);

//CsvTableSource的创建方式
val csvTableSource = CsvTableSource
    .builder
    .path("/path/to/your/file.csv")
    .field("name", Types.STRING)
    .field("id", Types.INT)
    .field("score", Types.DOUBLE)
    .field("comments", Types.STRING)
    .fieldDelimiter("#")
    .lineDelimiter("$")
    .ignoreFirstLine
    .ignoreParseErrors
    .commentPrefix("%")
```

### DataSet/DataStream

除了以TableSource作为Table&SQL的source，还支持通过特定的环境对象直接注册DataStream、DataSet。示例如下

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val cust = env.fromElements(...)
val ord = env.fromElements(...)

// register the DataStream cust as table "Customers" with fields derived from the datastream
tableEnv.registerDataStream("Customers", cust)
tableEnv.registerDataSet("Customers", cust)
```

注册DataStream跟DataSet的对应关系如下

![DataStreamSet对应关系](https://gitee.com/liurio/image_save/raw/master/flink/DataStreamSet%E5%AF%B9%E5%BA%94%E5%85%B3%E7%B3%BB.png)

## Sink

sink其实跟source是反向的，一个是将数据源接入进来，另一个是将数据写到外部。因此，我们对比着source来看sink，当你实现一个Table&SQL程序并希望将处理之后的结果输出到外部。外部的存储可以是数据库，键值存储，消息队列，或文件系统（在不同的编码，如CSV，拼接，或ORC）。通常有以下几种方式：

- 在Table对象上调用writeToSink API，它接收一个TableSink的实例
- 将Table再次转换为DataSet/DataStream，然后像输出DataSet/DataStream一样的方式来处理

对于TableSink而言，根据流和批分为两类，一类是BatchTableSink，一类是针对streaming的多种sink(三种)。跟source一样，内置的CsvTableSink同时兼具streaming跟batch的语义

- AppendStreamTableSink：它只支持插入变更，如果Table对象同时有更新和删除的变更，那么将会抛出TableException
- RetractStreamTableSink：它支持输出一个streaming模式的表，该表上可以有插入、更新和删除变更
- UpsertStreamTableSink：它支持输出一个streaming模式的表，该表上可以有插入、更新和删除变更，且还要求表要么有唯一的键字段要么是append-only模式的，如果两者都不满足，将抛出TableException

不管是哪种类型的sink，处理方式都一样，都是先将具体的Table对象转化为DataSet/DataStream，然后输出。

## Table API的执行原理

![Tableapi执行原理](https://gitee.com/liurio/image_save/raw/master/flink/Tableapi%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86.png)

Flink使用基于Apache Calcite这个SQL解析器做SQL语义解析。利用Calcite的查询优化框架与SQL解释器来进行SQL的解析、查询优化、逻辑树生成，得到Calcite的RelRoot类的一颗逻辑执行计划树，并最终生成flink的Table。Table里的执行计划会转化成DataSet或DataStream的计算，经历物理执行计划优化等步骤。但是，Table API和 SQL最终还是基于flink的已有的DataStream API和DataSet API，任何对于DataStream API和DataSet API的性能调优提升都能够自动地提升Table
API或者SQL查询的效率。这两种API的查询都会用包含注册过的Table的catalog进行验证，然后转换成统一Calcite的logical plan。再利用 Calcite的优化器优化转换规则和logical plan。根据数据源的性质(流和批)使用不同的规则进行优化。最终优化后的plan转传成常规的flink DataSet或 DataStream程序。比如：

![tableapi执行计划](https://gitee.com/liurio/image_save/raw/master/flink/tableapi%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92.png)

```java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createCollectionsEnvironment();
    BatchTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);
    DataSet<WC> input = env.fromElements(
        new WC("Hello", 1),
        new WC("Ciao", 1),
        new WC("Hello", 1));
    Table table = tEnv.fromDataSet(input);
    Table filtered = table
        .groupBy("word")
        .select("word, frequency.sum as frequency")
        .filter("frequency = 2");
    DataSet<WC> result = tEnv.toDataSet(filtered, WC.class);
    result.print();
}
```

### DataSet/DataStream

####  创建一个TableEnvironment

TableEnvironment对象是Table API和SQL集成的一个核心，支持以下场景

- 注册一个Table

- 注册一个外部的catalog

- 执行SQL查询

- 注册一个用户自定义的function

- 将DataStream或DataSet转成Table

一个查询中只能绑定一个指定的TableEnvironment，TableEnvironment可以通过配置TableConfig来配置，也可以自定义。

#### 将一个Table注册给TableEnvironment

```java
BatchTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);
```

#### 将DataSet/DataStream注册为Table

```scala
Table table = tEnv.fromDataSet(input);
def fromDataSet[T](dataSet: DataSet[T]): Table = {
    val name = createUniqueTableName()
    registerDataSetInternal(name, dataSet)
    scan(name)
  }
protected def registerDataSetInternal[T](name: String, dataSet: DataSet[T]): Unit = {
    val (fieldNames, fieldIndexes) = getFieldInfo[T](dataSet.getType)
    val dataSetTable = new DataSetTable[T](
      dataSet,
      fieldIndexes,
      fieldNames
    )
    registerTableInternal(name, dataSetTable)
  }
protected def registerTableInternal(name: String, table: AbstractTable): Unit = {
      rootSchema.add(name, table)
  }
```

从源码中可以看出通过TableEnvironment对象调用fromDataSet函数，会new一个DataSetTable，将DataSet转化为Table。在注册Table的函数里，会调用rootSchema方法中的createRootSchema为该TableEnvironment的catalog注册一个Calcite.

### Table --> TreeNode

Table API是一个Scala和Java的集成查询序言。与SQL不同的是，Table API的查询不是一个指定的sql字符串，而是调用指定的API方法。Table API中的每一个方法输入都是一个Table，输出也是一个新的Table。

该部分的作用是把Table API表达的计算逻辑表示成逻辑树，用TreeNode表示。首先通过TableEnvironment对象调用fromDataSet函数，在该函数中调用scan()方法，该方法的参数是tablePath，根据表的路径浏览表的内容，然后创建new一个CatalogNode，生成了一个TreeNode节点。该节点对所有的sql操作而言，一般都会有。

```scala
def scan(tablePath: String*): Table = {
    scanInternal(tablePath.toArray) match {
      case Some(table) => table
      case None => throw new TableException(s"Table '${tablePath.mkString(".")}' was not found.")
    }
  }
  private[flink] def scanInternal(tablePath: Array[String]): Option[Table] = {
    require(tablePath != null && !tablePath.isEmpty, "tablePath must not be null or empty.")
    val schemaPaths = tablePath.slice(0, tablePath.length - 1)
    val schema = getSchema(schemaPaths)
    if (schema != null) {
      val tableName = tablePath(tablePath.length - 1)
      val table = schema.getTable(tableName)
      if (table != null) {
        return Some(new Table(this, CatalogNode(tablePath, table.getRowType(typeFactory))))
      }
    }
  }
```

接着是对相应的sql操作生成对应的TreeNode节点，对于groupBy操作而言，和sql中的groupby操作是一样的，都是根据key进行分组，接下来的Aggregate操作是在按照key分组之后的group进行操作的。在逻辑树上的每个节点的计算逻辑都是用Expression来表示的，从源码可以看出，groupby首先会调用parseExpressionList()将计算逻辑进行解析，如果存在多个关键字，就用‘,’隔开，因为此时只有一个‘word’，所以解析后的结果如下。然后把结果作为参数传递给groupby函数，接着会new一个GroupedTable，会为每个解析后的key分别创建对应的Table。

![treenode-1](https://gitee.com/liurio/image_save/raw/master/flink/treenode-1.jpg)

```scala
def groupBy(fields: String): GroupedTable = {
    val fieldsExpr = ExpressionParser.parseExpressionList(fields)
    groupBy(fieldsExpr: _*)
  }
lazy val expressionList: Parser[List[Expression]] = rep1sep(expression, ",")
```

接着分析select操作，同样的Flink会把计算逻辑进行解析，以Expression形式存在，解析后的结果如下， 

> Expression的格式：Expression(condition,child)

![treenode-2](https://gitee.com/liurio/image_save/raw/master/flink/treenode-2.jpg)

然后会根据Expression的逻辑生成正确的表达式，即调用replaceAggFunctionCall()方法，生成的结果如下，

![treenode-3](https://gitee.com/liurio/image_save/raw/master/flink/treenode-3.jpg)

```scala
def select(fields: String): Table = {
    val fieldExprs = ExpressionParser.parseExpressionList(fields)
    //get the correct expression for AggFunctionCall
    val withResolvedAggFunctionCall = fieldExprs.map(replaceAggFunctionCall(_, table.tableEnv))
    select(withResolvedAggFunctionCall: _*)
  }
```

接下来会把解析之后的逻辑表达式结果作为参数传递给select函数，首先会提取出表达式以及相应的属性，接着会进行创建`Project -> Aggregate -> Project`.

```scala
def select(fields: Expression*): Table = {
    val expandedFields = expandProjectList(fields, table.logicalPlan, table.tableEnv)
    val (aggNames, propNames) = extractAggregationsAndProperties(expandedFields, table.tableEnv)

    new Table(table.tableEnv,
      Project(projectsOnAgg,
        Aggregate(groupKey, aggNames.map(a => Alias(a._1, a._2)).toSeq,
          Project(projectFields, table.logicalPlan).validate(table.tableEnv)
        ).validate(table.tableEnv)
      ).validate(table.tableEnv))
  }
```

然后通过validate函数结合catalog，调用resolveExpressions()方法将树的每个节点的Unresolved Expression进行绑定，生成Resolved Expression，同时会调用postOrderTransform()方法，递归的调用Expression中的child，并创建TreeNode节点。例如

![treenode-4](https://gitee.com/liurio/image_save/raw/master/flink/treenode-4.jpg)

```scala
def validate(tableEnv: TableEnvironment): LogicalNode = {
    val resolvedNode = resolveExpressions(tableEnv)
    resolvedNode.expressionPostOrderTransform
  }
private[flink] def postOrderTransform(rule: PartialFunction[A, A]): A = {
    def childrenTransform(rule: PartialFunction[A, A]): A = {
      var changed = false
      val newArgs = productIterator.map {
        case arg: TreeNode[_] if children.contains(arg) =>
          val newChild = arg.asInstanceOf[A].postOrderTransform(rule)
          if (!(newChild fastEquals arg)) {
            changed = true
            newChild
          } else {
            arg
          }
```

最后分析filter语句，同样为filter的计算逻辑进行解析并且以Expression形式存在，然后把传递的解析的参数传递给filter()函数，接着就会new创建一个Filter的TreeNode，调用validate，将未绑定的Expression进行绑定变成Resoved Expression。最后的TreeNode逻辑树的结果如图。

![Treenode逻辑树](https://gitee.com/liurio/image_save/raw/master/flink/Treenode%E9%80%BB%E8%BE%91%E6%A0%91.png)

### TreeNode --> Logical Plan

该部分的作用是依次遍历树的每个节点，调用construct方法将原先用treeNode表达的节点转成用calcite 内部的数据结构relNode 来表达。即生成了LogicalPlan, 用relNode表示.

```java
public RelBuilder construct(RelBuilder relBuilder) {
    this.child().construct(relBuilder);
    return relBuilder.filter(new RexNode[]{this.condition().toRexNode(relBuilder)});
}
public final RelNode toRelNode(RelBuilder relBuilder) {
    return this.construct(relBuilder).build();
}
```

`Construct()`依次遍历TreeNode的每个节点，生成对应的Logical Plan，比如scan节点

```java
public static LogicalTableScan create(RelOptCluster cluster, RelOptTable relOptTable) {
    final Table table = (Table)relOptTable.unwrap(Table.class);
    RelTraitSet traitSet =
        cluster.traitSetOf(Convention.NONE).replaceIfs(RelCollationTraitDef.INSTANCE, new Supplier<List<RelCollation>>() {
        public List<RelCollation> get() {
            return (List)(table != null?table.getStatistic().getCollations():ImmutableList.of());
        }
    });
    return new LogicalTableScan(cluster, traitSet, relOptTable);
}
```

### Logical Plan  --> Optimized Plan
Calcite内部提供了两套planner，HepPlanner和VolcanoPlanner，HepPlanner主要是一种贪婪式的Planner，而VolcanoPlanner是一种启发式的Planner。对于HepPlanner而言，每次rule产生的result当作下一次迭代的开始，也认为是当作这次最优的plan，则丢掉了其他优化可能，所以plan可能不是最优，而VolcanoPlanner具有条件下推，剪枝等优化规则，能够得到最优的plan。优化过程中会把两者结合使用，HepPlanner在Calcite中主要是用来提前剪枝，方便给VolcanoPlanner提供一种次优plan后，继续优化。整个优化的部分都在optimize()函数中，该函数源码为：

```scala
private[flink] def optimize(relNode: RelNode): RelNode = {
   
 // 0. convert sub-queries before query decorrelation
    val convSubQueryPlan = runHepPlanner(
      HepMatchOrder.BOTTOM_UP, FlinkRuleSets.TABLE_SUBQUERY_RULES, relNode, relNode.getTraitSet)
   
 // 0. convert table references转换带有表引用的LogicalTableScan转化为新的RelNode
    val fullRelNode = runHepPlanner(
      HepMatchOrder.BOTTOM_UP, FlinkRuleSets.TABLE_REF_RULES, convSubQueryPlan, relNode.getTraitSet)
  
  // 1. Decorrelate去除关联子查询
    val decorPlan = RelDecorrelator.decorrelateQuery(fullRelNode)
    
// 2. normalize the logical plan规范化逻辑计划，比如一个Filter它的过滤条件都是true的话,那么我们可以直接将这个filter去掉
    val normRuleSet = getNormRuleSet
    val normalizedPlan = if (normRuleSet.iterator().hasNext) {
      runHepPlanner(HepMatchOrder.BOTTOM_UP, normRuleSet, decorPlan, decorPlan.getTraitSet)
    } else { decorPlan }

    // 3. optimize the logical Flink plan优化成flink 的logicalplan
    val logicalOptRuleSet = getLogicalOptRuleSet
    val logicalOutputProps = relNode.getTraitSet.replace(FlinkConventions.LOGICAL).simplify()
    val logicalPlan = if (logicalOptRuleSet.iterator().hasNext) {
      runVolcanoPlanner(logicalOptRuleSet, normalizedPlan, logicalOutputProps)
    } else { normalizedPlan }

    // 4. optimize the physical Flink plan将flinklogicalplan转化为flinkphysicalplan，实际上，没有做任何改变，调用的是同一套代码
    val physicalOptRuleSet = getPhysicalOptRuleSet
    val physicalOutputProps = relNode.getTraitSet.replace(FlinkConventions.DATASET).simplify()
    val physicalPlan = if (physicalOptRuleSet.iterator().hasNext) {
      runVolcanoPlanner(physicalOptRuleSet, logicalPlan, physicalOutputProps)
    } else { logicalPlan }
    physicalPlan
  }
```

#### HepPlanner

优化的入口是optimize()函数，首先经过HepPlanner优化，该planner的入口是runHepPlanner函数，该函数的源码如下：

```scala
protected def runHepPlanner(hepMatchOrder: HepMatchOrder,
    ruleSet: RuleSet, input: RelNode, targetTraits: RelTraitSet): RelNode = {
    val builder = new HepProgramBuilder   //新建一个HepProgram
    builder.addMatchOrder(hepMatchOrder)	// 倒序的优化规则
    val planner = new HepPlanner(builder.build, frameworkConfig.getContext)
    planner.setRoot(input)     //创建Graph
    if (input.getTraitSet != targetTraits) {
      planner.changeTraits(input, targetTraits.simplify)
    }
    planner.findBestExp		//返回该部分的最优计划
  }
```

首先会构建一个Graph，将所有的relNode Tree(Logical Plan)转化为HepRelVertex，将所有的relNode节点关系使用Vertex表示。首先根据每个HepRelVertex构建了从其本身出发到其指向其他的孩子节点的source->dest的关系。构建后的Graph为：

![heprelvertex-graph](https://gitee.com/liurio/image_save/raw/master/flink/heprelvertex-graph.jpg)

HepPlanner为了更好地采用Greedy算法，将每次运行的rule使用HepProgram进行组装。HepProgram由各种不同的HepInstruction组成，常用的HepInstruction有MatchLimit、MatchOrder、Rules。

- MatchLimit表示这次HepProgram优化的次数限制，如果不设置，则为无穷
- MatchOrder则表示每次rule执行的顺序，包括ARBITRARY(rule每次Apply是从当前relNode节点开始，直到没有vertex执行后，采用root节点重新apply)、BOTTOM_UP(逆序)、TOP_DOWN(与bottom_up相反)三种方式，其中ARBITRARY被认为是最高效的apply方式，也是默认方式
- Rules表示需要运行的优化规则

这部分的优化规则(Flink所有的规则定义在类FlinkRuleSets中)如下FF1A

```scala
val DATASET_NORM_RULES: RuleSet = RuleSets.ofList(
    // simplify expressions rules
    ReduceExpressionsRule.FILTER_INSTANCE,
    ReduceExpressionsRule.PROJECT_INSTANCE,
    ReduceExpressionsRule.CALC_INSTANCE,
    ReduceExpressionsRule.JOIN_INSTANCE,
    ProjectToWindowRule.PROJECT,
    // Transform grouping sets
    DecomposeGroupingSetRule.INSTANCE,
    // Transform window to LogicalWindowAggregate
    DataSetLogicalWindowAggregateRule.INSTANCE,
    WindowPropertiesRule.INSTANCE,
    WindowPropertiesHavingRule.INSTANCE
  )
```

HepPlanner的处理方式很简单，就是顺序执行每个HepInstruction，然后当遇到transformation发生的次数达到一定的值时，就会清理一些中间结果的RelNode。

```java
while(var3.hasNext()) {
    HepInstruction instruction = (HepInstruction)var3.next();
    instruction.execute(this);
    int delta = this.nTransformations - this.nTransformationsLastGC;
    if(delta > this.graphSizeLastGC) {
        this.collectGarbage();
    }
}
```

针对每一个集合的rules时，HepPlanner会调用applyRules()函数，处理过程比较简单，会根据root节点构建整棵树的RelVertex迭代器，遍历每一个HepRelVertex，同时与给定的rules进行match并apply，如果发现rule产生了更优的result，即发生了transformation，然后继续按照MatchOrder构建HepRelVertex迭代器，从而继续apply，直到整颗树不再有transformatin产生或者达到了MatchLimit的限制.

```scala
HepRelVertex vertex = (HepRelVertex)iter.next();
      Iterator var8 = rules.iterator();
      while(var8.hasNext()) {
           RelOptRule rule = (RelOptRule)var8.next();
           HepRelVertex newVertex = this.applyRule(rule, vertex, forceConversions);
           if(newVertex != null) {
                ++nMatches;
                if(nMatches >= this.currentProgram.matchLimit) {return;}
                if(fullRestartAfterTransformation) {
                      iter = this.getGraphIterator(this.root);
                } else {
                      iter = this.getGraphIterator(newVertex);
                      fixpoint = false;
                }
                break;
            }
         }
      }
  }
} while(!fixpoint);
```

#### VolcanoPlanner

VolcanoPlanner的作用有两个：一个是将logical转化为flinklogicalplan和将flinklogicalplan转化为flinkphysicalplan。该部分是基于代价计算的优化CBO

```java
public RelNode run(RelOptPlanner planner, RelNode rel, RelTraitSet requiredOutputTraits, List<RelOptMaterialization> materializations, List<RelOptLattice> lattices) {
    planner.clear();
    Iterator var6 = this.ruleSet.iterator();

    //将所有的rule添加到planner中
    while(var6.hasNext()) {
        RelOptRule rule = (RelOptRule)var6.next();
        planner.addRule(rule);
    }

    //开始所有关系节点的注册，以及寻找最优的plan
    planner.setRoot(rel);
    return planner.findBestExp();
}

```

SetRoot()的作用是根关系节点和后裔关系节点的注册过程。FindBestExp()是寻找最优计划的过程。其中具体是根据cost代价计算来确定最优计划的。每次在RelSet中添加新的关系节点时cost代价会动态变化从而确定最优计划，有三个指标，rowCount、cpu、io。

```java
if(this.root.bestCost.isLe(targetCost)) {
    if(firstFiniteTick < 0) {
        firstFiniteTick = cumulativeTicks;
        this.clearImportanceBoost();
    }

    if(!this.ambitious) {
        break;
    }

    targetCost = this.root.bestCost.multiplyBy(0.9D);
    ++splitCount;
    if(this.impatient) {
        if(firstFiniteTick < 10) {
            giveUpTick = cumulativeTicks + 25;
        } else {
            giveUpTick = cumulativeTicks + Math.max(firstFiniteTick / 10, 25);
        }
    }
}
```

在优化过程中会调用getLogicalOptRuleSet获取需要优化的逻辑规则(52个)，上述例子的规则如图

![volcanoPlanner规则](https://gitee.com/liurio/image_save/raw/master/flink/volcanoPlanner%E8%A7%84%E5%88%99.jpg)

Flink会依次遍历这52个规则，比如FilterToCalcRule、ProjectToCalcRule，当匹配到这类规则后，会触发每个规则类中的onMatch()函数，会把原先的filter Node、Project Node都变成LogicalCalc节点。

```java
public void onMatch(RelOptRuleCall call) {
    LogicalProject project = (LogicalProject)call.rel(0);
    RelNode input = project.getInput();
    RexProgram program = RexProgram.create(input.getRowType(), project.getProjects(), (RexNode)null, project.getRowType(), project.getCluster().getRexBuilder());
    LogicalCalc calc = LogicalCalc.create(input, program);
    call.transformTo(calc);
}
```

最终会调用`findBestExp()`中的`buildCheapestPlan()`构建最优的plan，此时的结果为：

![volcanoPlanner最优计划](https://gitee.com/liurio/image_save/raw/master/flink/volcanoPlanner%E6%9C%80%E4%BC%98%E8%AE%A1%E5%88%92.jpg)

![物理计划生成](https://gitee.com/liurio/image_save/raw/master/flink/%E7%89%A9%E7%90%86%E8%AE%A1%E5%88%92%E7%94%9F%E6%88%90.png)

### Optimized Plan --> Physical Plan

该部分仍然是优化的部分，基于Flink提供的规则将`optimized LogicalPlan`转成Flink的物理执行计划。该部分的源码如下

```scala
// 4. optimize the physical Flink plan
val physicalOptRuleSet = getPhysicalOptRuleSet
val physicalOutputProps = relNode.getTraitSet.replace(FlinkConventions.DATASET).simplify()
val physicalPlan = if (physicalOptRuleSet.iterator().hasNext) {
    runVolcanoPlanner(physicalOptRuleSet, logicalPlan, physicalOutputProps)
} else {
    logicalPlan
}
```

上述例子的规则如下，

![物理计划规则](https://gitee.com/liurio/image_save/raw/master/flink/%E7%89%A9%E7%90%86%E8%AE%A1%E5%88%92%E8%A7%84%E5%88%99.jpg)

该部分会生成对应的Flink物理节点。

![物理计划节点](https://gitee.com/liurio/image_save/raw/master/flink/%E7%89%A9%E7%90%86%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.jpg)

![生成可执行的物理计划](https://gitee.com/liurio/image_save/raw/master/flink/%E7%94%9F%E6%88%90%E5%8F%AF%E6%89%A7%E8%A1%8C%E7%9A%84%E7%89%A9%E7%90%86%E8%AE%A1%E5%88%92.png)

### Physical Plan --> Flink Plan

最后一部分是将物理执行计划转成Flink Execution Plan: 就是调用相应的tanslateToPlan方法转换。该部分是利用codegen机制，生成DataSet的Function代码，以字符串的形式存在。

该部分函数入口源码为

```scala
Main: DataSet<WC> result = tEnv.toDataSet(table, WC.class);

BatchTableEnvironment: 
 def toDataSet[T](table: Table, clazz: Class[T]): DataSet[T] = {
    translate[T](table)(TypeExtractor.createTypeInfo(clazz))
  }
logicalPlan match {
      case node: DataSetRel =>
        val plan = node.translateToPlan(this)
. . . 
```

Flink会依次遍历生成的Flink物理节点，每一个节点上都会实现translateToPlan()方法，每一种方法都会产生一个`FunctionCodeGenerator`代码生成器的对象，然后调用`generateFunction()`方法产生对应操作的Function。例如`Aggregate`操作，flink会调用`transformToAggregateFunctions()`函数判断操作的类型`LongSumAggFunction`；对于`DataSet Calc`，flink会new一个`FlatMapRunner`,生成flatmap的对应操作。

该部分的功能是将CalCite优化后的flink节点，生成flink可以直接执行的function，Tree中的每个节点都会调用对应的generateFunction，生成对应的code，以字符串的形式存在。比如DataSet Scan节点，会生成DataSource，然后生成对应的DataSource的类。在运行之前，flink都会先编译再提交运行。

![flink代码生成器](https://gitee.com/liurio/image_save/raw/master/flink/flink%E4%BB%A3%E7%A0%81%E7%94%9F%E6%88%90%E5%99%A8.jpg)

![生成Flink计划](https://gitee.com/liurio/image_save/raw/master/flink/%E7%94%9F%E6%88%90Flink%E8%AE%A1%E5%88%92.png)

## CBO—基于代价的优化

CBO的实现主要包括

1. 采集原始表基本信息

2. 定义核心算子的基数推导规则

3. 核心算子的实际代价计算

4.  选择最优执行路径（代价最小执行路径）

### 原始表基本信息采集

最重要的是定义需要采集的指标，如行级的rowCount等。这里用的是rowCount，指每个LogicalPlan节点输出数据总条数，例如LogicalTableScan节点，会遍历整个表，输出表中所有行的数据。

### 定义核心算子的基数推导规则

规则推导意思是说在当前子节点统计信息的基础上，计算父节点相关统计信息的一套推导规则。对于不同算子，推导规则必然不一样，比如fliter、group by、limit等等的评估推导是不同的。这里以filter为例进行讲解。先来看看这样一个SQL：select * from A , C where A.id = C.c_id and C.c_id > N ，经过RBO之后的语法树如下图所示：

![基数推导规则](https://gitee.com/liurio/image_save/raw/master/flink/%E5%9F%BA%E6%95%B0%E6%8E%A8%E5%AF%BC%E8%A7%84%E5%88%99.png)

### 核心算子的实际代价计算

一条执行路径的总代价就是这条路径上所有节点的代价累加之和。每个系统对代价的定义是不一致的，通常来讲，节点实际执行代价主要从两个维度来定义：CPU Cost以及IO Cost。一般的计算规则是:

>CPU Cost: RowCount * cpu(需要的cpu资源)
>
>IO Cost:  RowCount * io(需要的输入输出的花费)

例如：查询语句包括三张事实表: store_sales (29 亿行纪录), store_returns (2.88 亿行纪录) 和catalog_sales (14.4 亿行纪录). 同时也包括三张维度表: date_dim(7.3万行纪录), store (1K 行纪录) 和 item (300K 行纪录).

```sql
SELECT 
    i_item_id, 
    i_item_desc, 
    s_store_id, 
    s_store_name, 
    sum(ss_net_profit) AS store_sales_profit, 
    sum(sr_net_loss) AS store_returns_loss, 
    sum(cs_net_profit) AS catalog_sales_profit
FROM 
store_sales, store_returns, catalog_sales, date_dim d1, date_dim d2, date_dim d3, store, item
WHERE 
d1.d_moy = 4   AND d1.d_year = 2001   
    AND d1.d_date_sk = ss_sold_date_sk   
    AND i_item_sk = ss_item_sk   
    AND s_store_sk = ss_store_sk   
    AND ss_customer_sk = sr_customer_sk   
    AND ss_item_sk = sr_item_sk   
    AND ss_ticket_number = sr_ticket_number   
    AND sr_returned_date_sk = d2.d_date_sk   
    AND d2.d_moy BETWEEN 4 AND 10   
    AND d2.d_year = 2001   
    AND sr_customer_sk = cs_bill_customer_sk   
    AND sr_item_sk = cs_item_sk   
    AND cs_sold_date_sk = d3.d_date_sk   
    AND d3.d_moy BETWEEN 4 AND 10   
    AND d3.d_year = 2001
GROUP BY i_item_id, i_item_desc, s_store_id, s_store_name
ORDER BY i_item_id, i_item_desc, s_store_id, s_store_name
LIMIT 100
```

首先看没有代价优化CBO的join树如图2所示，一般这种树也叫做左线性树。这里, join #1 和 #2 是大的事实表与事实表join，join了3张事实表store_sales, store_returns, 和catalog_sales，并产生大的中间结果表。这两个join都以shuffle join的方式执行并会产生大的输出，其中join #1输出了1.99亿行纪录。总之，关闭CBO,查询花费了241秒。

![无优化](https://gitee.com/liurio/image_save/raw/master/flink/%E6%97%A0%E4%BC%98%E5%8C%96.png)

另一方面，用了CBO, 创建了优化方案可以减小中间结果（如图3）。在该案例中，创建了浓密树而不是左-深度树。在CBO规则下,先join 的是事实表对应的维度表 (在尝试直接join事实表前)。避免大表join意味着避免了大开销的shuffle。在这次查询中，中间结果大小缩小到原来的1/6（相比之前）。最后，查询只花了71秒，性能提升了3.4倍。

![CBO优化](https://gitee.com/liurio/image_save/raw/master/flink/CBO%E4%BC%98%E5%8C%96.png)

参考链接

SPARK CBO: https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html

HIVE CBO: https://hortonworks.com/blog/hive-0-14-cost-based-optimizer-cbo-technical-overview/

PAPER: [Introducing Cost Based Optimizer to Apache Hive](file:///C:/Users/Administrator/Desktop/CBO-2.pdf)