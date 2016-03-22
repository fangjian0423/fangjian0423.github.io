title: Scala持久层框架Slick介绍
date: 2015-08-18 23:22:33
tags:
- scala
- frm
- orm
categories:
- jvm
description: 近看到了一个FRM的框架Slick。 FRM的意思是Functional Relational Mapping， 一种基于函数式的ORM ...

---------------

## FRM介绍 ##

最近看到了一个FRM的框架Slick。 FRM的意思是Functional Relational Mapping， 一种基于函数式的ORM。

举一个最简单的例子：

    val queryResult = db.query(queryStr)

    queryResult.onSuccess { result =>
        result.doSomething ...
    }

数据库db查询一条sql语句。查询成功的时候使用闭包完成处理。 看到这段代码的第一反应就是js的ajax处理，代码几乎是一样的，也发现之前在学校里写nodejs的时候查询db也是这样的语法。

	var jqxhr = $.ajax( {
    	url: 'url',
        method: 'GET',
        data: [user: 'format']
    });
    
    jqxhr.success(function() {
    	...
    });
    
FRM相比ORM最明显的优势就是FRM基于多线程的Future的数据查询，而ORM是单线程的线性执行。

FRM构造sql查询也是相当简单的：

	// 构造查询
	val newQuery = students.filter(_.age > 24).sortBy(_.name)
    // 执行查询
    db.run(newQuery)
    
FRM其他的优势可以参考[官方文档](http://slick.typesafe.com/doc/3.0.1/introduction.html#functional-relational-mapping)。

## Slick实例 ##

下面以一个Students和Classrooms的实例来说明一下Slick的使用。

首先是创建对应的domain，学生与教室的关系是1对多。

Students domain(使用Option类型说明该列是可为空的)：

	class Student(tag: Tag) extends Table[(Int, String, Int, Int, Option[Date])](tag, "Students") {

      def id: Rep[Int] = column[Int]("id", O.PrimaryKey, O.AutoInc)
  	def name: Rep[String] = column[String]("name")
	  def age: Rep[Int] = column[Int]("age")
  	def birthDate: Rep[Option[Date]] = column[Option[Date]]("birth_date")
  	def classroomId = column[Int]("classroom_id")

 	  def * : ProvenShape[(Int, String, Int, Int, Option[Date])] = (id, name, age, classroomId, birthDate)

	  def classroom: ForeignKeyQuery[Classroom, (Int, String)] = foreignKey("FK_CLASSROOM", classroomId, TableQuery[Classroom])(_.id)
	}


Classrooms domain：

	class Classroom(tag: Tag) extends Table[(Int, String)](tag, "Classrooms") {
      def id = column[Int]("id", O.PrimaryKey, O.AutoInc)
      def name = column[String]("name")

      def * = (id, name)
	}

各个db操作，schema创建，sql插入，sql查询等操作如下，加了几句备注，具体的代码就不分析了：

    object SampleSlickDemo extends App {

      val db = Database.forConfig("h2mem1")

      try {

        val classrooms = TableQuery[Classroom]
        val students = TableQuery[Student]


        val setupAction: DBIO[Unit] = DBIO.seq(
          // create student and classroom table in database
          (classrooms.schema ++ students.schema).create,
          // insert some rows in classroom
          classrooms += (1, "classroom1"),
          classrooms += (2, "classroom2"),
          classrooms += (3, "classroom2")
        )

        val setupFuture = db.run(setupAction)

        val f = setupFuture.flatMap { _ =>

          val insertAction: DBIO[Option[Int]] = students ++= Seq (
            (1, "format1", 11, 1, new Date(System.currentTimeMillis())),
            (2, "format2", 22, 2, new Date((System.currentTimeMillis()))),
            (3, "format3", 33, 3, new Date((System.currentTimeMillis())))
          )

          val insertAndPrintAction = insertAction.map { studentResult =>
            studentResult.foreach { numRows =>
              println(s"inserted $numRows students")
            }
          }

          db.run(insertAndPrintAction)
        }.flatMap { _ =>

          // print All Classrooms
          db.run(classrooms.result).map { classroom =>
            classroom.foreach(println);
          }

          // print All Students
          db.run(students.result).map { studnet =>
            studnet.foreach(println);
          }

          // condition search
          val studentQuery = students.filter(_.age > 20).sortBy(_.name)
          db.run(studentQuery.result).map { student =>
            student.foreach(println)
          }
        }
        Await.result(f, Duration.Inf)
      } finally db.close()
    }

## 数据库配置 ##

在配置文件application.conf里配置数据库配置信息：

	h2mem1 = {
      url = "jdbc:h2:mem:test1"
      driver = org.h2.Driver
      connectionPool = disabled
      keepAliveConnection = true
    }

然后就可使用Database初始化数据库，参数就是配置文件里对应的数据库name：

	val db = Database.forConfig("h2mem1")

## DBIOAction介绍 ##

DBIOAction就是数据库的一个操作，比如Insert，Update，Delete，Query等操作。

可以使用上面分析的数据库配置变量db进行操作。

db有个run方法使用DBIOAction作为参数，返回Future类型的返回值。

DBIO是一个单例对象，它的seq方法可以传入多个DBIOAction，然后返回一个新的DBIOAction。 += 方法返回的也是DBIOAction。

	val setupAction: DBIO[Unit] = DBIO.seq(
      (classrooms.schema ++ students.schema).create,
      classrooms += (1, "classroom1"),
      classrooms += (2, "classroom2"),
      classrooms += (3, "classroom2")
    )

++=方法跟+=方法一样会返回DBIOAction，只不过它的参数是个Iterable：

	val insertAction: DBIO[Option[Int]] = students ++= Seq (
        (1, "format1", 11, 1, new Date(System.currentTimeMillis())),
        (2, "format2", 22, 2, new Date((System.currentTimeMillis()))),
        (3, "format3", 33, 3, new Date((System.currentTimeMillis())))
      )

DBIOAction提供许多好用的方法：

map方法：参数是个函数，这个函数可以返回任意类型的值，返回是个DBIOAction。 所以可以使用map关联起来多个DBIOAction。

flatMap方法：参数是个函数，这个函数的返回值必须是个DBIOAction，返回值是个DBIOAction。作用跟map类似，只不过函数参数的返回值不一样。

filter方法：参数是个函数，这个函数的返回值必须是个Boolean，返回值是个DBIOAction。过滤作用。

andThen方法：参数是个DBIOAction，返回值是个DBIOAction。在Action完成后执行另外一个Action。

## 增删改查操作 ##

### 查询 ###

Slick的查询可以直接通过TableQuery操作，使用TableQuery提供的filter可以实现过滤操作，使用drop和take完成分页操作，使用sortBy完成排序操作。
	
    students.filter(_.classroomId === 1)
    students.drop(1).take(2)
    students.sortBy(_.age.desc)
    
可以使用map方法找出需要的列。 

多列：

	students.map { student =>
      (student.name, student.age)
    }
    
一列：

	students.map(_.name)
    
Join方法：

cross join操作：

	val crossJoin = for {
      (s, c) <- students join classrooms
    } yield (s.name, c.name)
    
inner Join操作：

	val innerJoin = for {
      (s, c) <- students join classrooms on (_.classroomId === _.id)
    } yield (s.name, c.name)
    
另外一个inner join：

	val innerJoin = for {
      s <- students
      c <- classrooms if c.id === s.classroomId
    } yield (s.name, c.name)

left join操作：

	val leftJoin = for {
      (s, c) <- students joinLeft classrooms on (_.classroomId === _.id)
    } yield (s.name, c.map(_.name))

right join操作：

	val rightJoin = for {
      (s, c) <- students joinRight classrooms on (_.classroomId === _.id)
    } yield (s.map(_.name), c.name)

### 新增 ###

所有列都有值：

	val insertAction = DBIO.seq(
      students += (4, "format4", 44, 3, new Date(System.currentTimeMillis())),
      students += (5, "format5", 55, 3, new Date(System.currentTimeMillis())),

      students ++= Seq (
        (6, "format6", 66, 3, new Date(System.currentTimeMillis())),
        (7, "format7", 77, 3, new Date(System.currentTimeMillis()))
      )
    )
    
部分列有值：

	students.map(s => (s.name, s.age, s.classroomId)) += ("format8", 88, 3)
	
### 删除 ###

删除classroomId为3的所有数据：

	val q = students.filter(_.classroomId === 3)
    val affectedRowsCountFuture = db.run(q.delete)
    affectedRowsCountFuture.map { rows =>
      println(rows)
    }

### 修改 ###

修改单列：

	val q = students.filter(_.id === 2).map(_.name)
    val updateSql = q.update("format2222")
    db.run(updateSql)

修改多列：

	val q = students.filter(_.id === 2).map(s => (s.name, s.age))
    val updateSql = q.update(("format2222", 222))
    db.run(updateSql)


## CaseClass的使用 ##

之前的例子都是使用Tuple构造domain。 还有一种更方便的方式，那就是使用CaseClass。

	case class People(id: Long, name: String, age: Int)
    
例子：

    private class PeopleTable(tag: Tag) extends Table[People](tag, "people") {
      val id = column[Long]("id", O.PrimaryKey, O.AutoInc)
      val name = column[String]("name")
      val age = column[Int]("age")

      def * = (id, name, age) <> ((People.apply _).tupled, People.unapply)
    }

    val db = Database.forConfig("h2mem1")

    try {

      val people = TableQuery[PeopleTable]

      val setupAction = DBIO.seq(
          people.schema.create,
          people += People(1, "format1", 11)
      )

      val setupFuture = db.run(setupAction);

      val f = setupFuture.flatMap { _ =>
        db.run(people.result).map { p =>
          p.foreach(println)
        }
      }

      Await.result(f, Duration.Inf)

    } finally db.close()

