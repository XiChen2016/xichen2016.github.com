---
layout: post
title: "rom-sql入门"
category: tutorial
tags: ruby, rom-sql
---

### 1. 安装rom-sql

在Gemfile中添加:

	gem 'rom-sql'
	
然后执行:

	bundle install
	

### 2. 初始化，连接数据库

	require 'rom-sql'
	
	db = ROM.setup(
		:sql,
		"postgres://<hostname>/#{dbname}",
		user: <username>,
		password: <password>
	)
	
这里，数据库我们选择了postgres，如果使用其他数据库，自行修改连接字符串中的数据库类型即可。
	
### 3. 定义relation

	db.relation(:companies) do
	  def find(id)
		where(id: id)
	  end
	end
	
	db.relation(:offices) do
	  many_to_one :companies, key: :company_id

	  def find(id)
		where('"offices"."id" = ?', id)
	  end

	  def by_name_and_city(name, city)
		where name: name, city: city
	  end

	  def with_company
		association_left_join(:companies)
	  end
	end

在上面的例子中，我们可以看到，一个company可能包含多个offices，我们期望在查询office的时候可以级联查出所属的company信息。
所以我们在offices中定义了many_to_one的关系。并且添加了with_company方法。

association_left_join是由Sequel提供的(rom的底层是通过Sequel实现的)，会在生成sql语句时对companies表进行left_join。如果在join操作时只希望选出某些字段，我们可以通过select参数进行筛选:

	association_join(:companies, select: [:name])
	
### 4.定义mapping

	db.mappers do
      define(:companies) { model Company }

      define(:offices) { model Office }

      define(:offices_with_company, parent: :offices) do
        model Office
        wrap :companies do
          model Company
          attribute :id, from: :companies_id
          attribute :name, from: :companies_name
          attribute :type, from: :type
        end
      end
    end
    
这里，我们定义了从数据库查出的Hash到Model之间的映射。
为了能够级联查出office所属的company信息，我们定义了offices_with_company这个mapper。并且在其中使用wrap关键字定义了嵌套的Company对象，如果嵌套的不是一个对象而是一个数组，我们可以使用group。
如果仔细看上面的代码，你会发现在定义wrap的Company时，id是通过from关键字从companies_id映射来的，这是由于Office和Company中都包含id这个属性，在生成sql语句时，由于有重名的属性，rom帮我们在company的id和name前自动添加了表名作为前缀。而在映射到model时，我们就需要手动对其进行转换。
而Office中并没有type属性，因此它不存在同名冲突的情况，在查询结果中，type属性并没有被加上表名前缀。

### 5.定义command
	
	db.commands(:offices) do
	  define(:create) do
		result :one
		validator OfficeCreationValidator
	  end
	end
	
	class OfficeCreationValidator
      def self.call(params)
        raise RecordInvalidError, 'Customer Invalid.'
      end
    end
	
result: one 代表我们通过这个command每次只能创建一条记录。validator可以为我们绑定自定义的validator，在调用这个command时会被自动触发。
自定义的validator必须实现call方法。

### 6.finalize

在定义完上面提到的所有元素后，我们需要调用finalize方法来对它们进行注册。

	rom = db.finalize

### 7.如何调用

到这里，我们的rom就配置好了，怎么调用这些方法呢？

要查询所有office记录

	rom.relation(:offices).as(:offices).to_a
要查询某个office并级联查询所属的company信息

	rom.relation(:offices).as(:offices_with_company).with_company.find(id).first
创建office
	
	rom.command(:offices).create.call(attr)
<br>
<br>
更多用法，建议参考rom源代码中的测试代码. :)
