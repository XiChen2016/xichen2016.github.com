---
layout: post
title: "如何构建一个服务"
category: tutorial
tags: microservice
---

### 概述

目前，微服务变得越来越火，关于微服务架构的讨论也是层出不穷。今天，我就和大家探讨一下，如何从代码出发，实现一个最简单的服务。

俗话说‘不积跬步，无以至千里’，当我们明白了如何构建一个简单的服务，就可以通过不断的扩展，不断的衍生，对复杂的业务进行解耦、拆分，从而完成更多的服务的构建。

#### 构建一个服务

对于微服务而言，因为本身的粒度不大（由团队决定），因此选用什么样的编程语言，可以根据具体的业务需求来决策。在初期，编程语言的选型需要做，但并不是最重要的。

那最重要的部分是什么呢？最重要的部分就是如何基于软件交付的生命周期，从工程实践出发，考虑代码、考虑配置、考虑如何测试、部署、监控，从而有效体现微服务的独立性、可部署性，并让团队能够在开发、测试、部署、运维的整个生命周期驾驭微服务。
   
下面，我就以一个产品服务作为例子，来谈谈如何实现第一个微服务。

### 场景分析

假定某电商系统，随着业务的发展，需要将产品的模块，从现有的单块架构逻辑中剥离出来，形成一个用于专门提供产品数据的服务，便于维护和扩展。

作为产品部分，通常具有如下的属性：产品名称、产品分类、价格、创建时间、修改时间等。当然，在真实的情况下，其相关的属性可能会更复杂。

首先，让我们来分析一下如果需要构建这样一个提供产品数据的服务，其应该具备哪些功能或者接口。

如下的代码描述了对于当前这个例子，基于RESTful所定义服务提供的接口。

> ## 关于REST

> 表述性状态转移（英文：Representational State Transfer，简称REST）是Roy Fielding博士在2000年他的博士论文中提出的一种软件架构风格。它是一种针对分布式应用系统的设计和开发方式，能有效降低系统的复杂性，提高可伸缩性。
> 同传统的SOAP或者XML-RPC相比，REST是一种更简洁和轻量级的架构风格，已经有越来越多的WEB服务开始采用REST风格设计和实现相关系统。需要注意的是，REST是设计风格而不是标准。REST通常使用HTTP、URI和XML以及JSON这些现有的广泛流行的协议和标准。
> REST以资源为中心，将外部访问的所有信息定义成不同的资源，并使用HTTP协议的verbs，映射为操作资源的行为。
> 如果想了解更多关于REST的知识，请参考Roy Fielding博士的文章《Architectural Styles and the Design of Network-based Software Architectures》

* 创建一个新的产品

        Path:          /products
        Verb:          POST
        Content-Type:  application/json   
        Body:          { 
                            'name':'Docker In Action',
                            'category':'Book',
                            'price':56.00
                       }

* 获取现有的产品列表

        Path:          /products
        Verb:          GET
        Accept:        application/json   

* 获取某个产品的明细

        Path:          /products/:id
        Verb:          GET
        Accept:        application/json   

* 修改某个产品的明细

        Path:          /products/:id
        Verb:          PUT
        Body:          {
                            'name':'Docker In Action',
                            'category':'Book',
                            'price':56.00
                       }

* 删除某个产品

        Path:          /products/:id
        Verb:          DELETE

### 代码实现

这里，我们就以获取某个产品明细这个接口为例给大家介绍如何一步一步地去实现一个最简单的服务。编程语言我们选择使用Ruby，其他语言的实现也非常类似。实际上，笔者在编写这个示例时使用了TDD(测试驱动开发)的方式，由于TDD不是本文的重点，因此在这里不做赘述。

为了让大家在阅读详细步骤之前对我们要做的事情有个结构性的认识，我们首先进行一个简单的tasking，也就是列出to-do list，步骤如下：

<img src="/assets/article_images/2015-05-22-how-to-build-a-service/todo.jpg" />

由上图可以看出，第一步，我们需要定义model，model通常是用来存放业务模型，譬如对当前例子而言，模型很简单，就是一个Product类，具有一些属性，如下所示：

    class Product
      include Virtus.model
    
      attribute :id, Integer
      attribute :name, String
      attribute :price, Float
      attribute :category, String
    end

在这个例子中，我们使用[Virtus](https://github.com/solnic/virtus)来定义模型的属性，Virtus支持对属性的类型，默认值进行定义，并且可以对属性添加一些约束条件，甚至可以进行内嵌对象或者集合的映射。

有了用来存放业务模型的Product类，第二步，我们自然要进行数据的读取，repositories通常用来封装存储，读取以及查找行为的逻辑，在笔者的实践中，该部分通常都会遵循模型存储模式(Repository Pattern)。模型存储模式是由埃里克·伊文思在他的《领域驱动设计》一书中提到，可以算是使用相当广泛的应用架构设计模式之一。

我们知道，大部分应用程序都需要存储的功能。譬如电商系统中的用户信息、产品信息、图片信息等都需要存储。应用场景不同，存储的类型也存在很大差异，譬如关系型数据库适合存储持久化的结构化信息，NoSQL适合存储非结构化信息，文件系统通常存储临时信息，如日志文件、锁文件等，而云存储（非云计算）则适合用来存放图片、资源等静态文件。

对于这些场景，模型存储模式的本质及优势在于，其屏蔽了模型的存储以及获取的实现细节，让调用者更关注接口。

在当前的例子中，笔者使用[rom-sql](https://github.com/rom-rb/rom-sql)来完成Product的存储以及获取。由于是示例，省去了关于数据连接以及配置等部分的处理。关于更详细的模型存储的例子，请读者参考案例篇里描述的其他场景的模型存储的实现方式。读取Product的代码如下：

    class ProductRepository
      class << self
        def find(id)
          relation.as(:products).find(id).first || raise(RecordNotFoundError, 'Product not found')
        end
    
        private
    
        def relation
          Database.db.relation(:products)
        end
      end
    end

这里，`Database.db.relation(:products)`能够以Hash的形式读取数据库products表中的所有数据，`as(:products)`会帮我们把Hash映射成对应的Product对象。当然，我们需要事先对rom-sql的relation以及数据库和model之间的映射进行配置，基本配置如下：
    
    def self.setup_relations(rom)
      rom.relation(:products) do
        def find(id)
          where(id: id)
        end
      end
    end
    
    def self.setup_mappings(rom)
      rom.mappers do
        define(:products) do
          model ProductService::Product
        end
      end
    end

对于repositories，复杂的情况还应该考虑是否在应用程序内部需要实现命令查询职责分离(Command Query Responsibility Segration)。

> ## 什么是命令查询职责分离（Command Query Responsibility Separation）？

> 在三层架构中，通常是通过数据访问层来修改或者查询数据，在这种情况下，对数据的读写针对的都是同一个业务模型。随着系统业务逻辑变得复杂，访问量的增加，这种设计可能会出现性能、安全、可伸缩性等问题。虽然我们可以在数据库层面进行读写分离的设计，来解决类似的问题，但如果只是数据库读写分离，业务上读写依旧混合的话，随着业务的复杂度以及业务变化频率的加快，依然会出现难以维护、灵活性不高以及性能问题。因此，操作和查询的分离能有效的从业务上将数据的修改和数据的获取职责分离，在业务逻辑清晰、分工明确的基础之上，提高系统的性能、可扩展性和安全性。
> 更多关于CQRS的细节，请参考马丁.福勒的这篇[文章](http://martinfowler.com/bliki/CQRS.html)

> ## 模型存储模式（Repository Pattern）与数据访问层（Data Access Layer）的区别

> 模型存储模式是领域驱动设计中的概念，它强调如何基于业务的需求实现模型的存储，它是一种抽象的设计方法。例如，模型有可能被存储到关系型数据库、NoSQL、文件系统、云存储等，也有可能作为其他服务的输入，继续被处理。
> 模型存储模式中定义的功能要体现领域模型的意图和约束。使用模型存储模式，隐含着一种思想，就是领域模型需要什么，它才提供什么，不需要的功能、不该提供的功能就不要提供，一切都是以业务需求为核心。
> 数据访问层则更关注数据的存储方式以及存储功能的实现，并不严格受限于业务逻辑。使用数据访问层，其意图在于其能够提供数据访问的所有接口，业务逻辑层需要用哪个数据接口，可以由业务层根据场景来自由选则。
> 使用模型存储模式的另外一个优势在于，当依赖的环境构建或者访问成本很高时候，能有效的对数据的存储或者获取的行为做Mock。几个月前，笔者就遇到类似的问题，由于业务需求，需要将某些模型的数据存储到亚马逊的云存储S3上，但由于网络延迟、安全、权限等因素，访问该环境所耗费的成本非常高。为了不让这部分功能影响团队本地的开发，团队就使用Mock的方式，构建一个假的模型存储，使得调用者能够有效的访问该接口并且迅速获取数据。
> 如果想了解更多关于数据访问层（DAL）与模型存储模式（Repository Pattern）的区别，推荐读者仔细阅读《领域驱动设计》中关于模型存储模式的部分。

> ## 模型存储模式（Repository Pattern）与对象关系映射（Object Relation Mapping）的区别

> 对象关系映射，是随着面向对象的开发方法和关系型数据库的发展，而诞生的一种将对象和关系型数据库进行映射的开发方法。我们知道，对象和关系数据是业务模型的两种表现形式，业务模型在内存中表现为对象，在数据库中表现为关系数据。因此，对象-关系映射一般以中间适配的形式存在，完成对象到关系数据库数据、或者关系数据库数据到对象的转换。
> 因此，对象关系映射可以理解成基于模型存储模式，对关系数据库访问的一种实现方式。

上面，我们通过ProductRepository读取了Product记录，怎么将其转换成表现层需要的格式？

representers通常是用来描述业务模型在应用层的表现形式。大多数情况下，Representer负责两部分内容:

1. 业务模型里的数据如何在应用层以数据的方式表现出来。这部分和几年前提出的DTO(Data Transfer Object)类似。通过适配，将业务模型的数据转换成表现层需要的数据。
2. 定义消费者（服务）如何同生产者（服务）交互，其协议和格式遵循什么样的规范。例如在HTTP协议中，使用什么样的Header，Content-type，Accept-type等。

在当前的例子中，笔者使用[roar](https://github.com/apotonick/roar)对数据进行转换和渲染，并使用基于REST之上的HAL协议作为服务间（也就是消费者与生产者间）通信的规范，关于更多使用HAL的实践，请参考《不仅仅是REST一章》。ProductRepresenter的代码实现如下：

    module ProductRepresenter
      include Roar::JSON::HAL
      include Roar::Hypermedia
      include Grape::Roar::Representer
    
      property :id
      property :name
      property :price
      property :category
    
      link :self do |opts|
        request = Grape::Request.new(opts[:env])
        "#{request.base_url}/products/#{id}"
      end
    end

由于在上面的representer中我们声明了id, name, price, category以及self link, 因此，最终生成的json结构如下:

    {
         "id": 1,
         "name": "Lavina Schultz",
         "price": 123.4,
         "category": “Book",
         "_links": {
              "self": {
              "href": "http://localhost:9292/products/1"
              }
         }
    }

最后，我们需要通过API将服务暴露出来，API是处理请求和响应的部分。这和我们熟悉的WEB MVC框架中controller作用类似，譬如Spring MVC的controller，Rails框架中的controller，负责接收请求，调度业务处理逻辑，并返回结果。
在当前的例子中，笔者使用Grape（由Ruby语言实现的基于Rack的，REST风格的框架）来实现api相关的功能，具体实现如下：

    module ProductService
      class API < Grape::API
        content_type :json, 'application/hal+json'
        format :json
        formatter :json, Grape::Formatter::Roar
    
        rescue_from RecordNotFoundError do |error|
          Rack::Response.new({ errors: error.message }.to_json, 404).finish
        end
    
        resource :products do
          route_param :id do
            get do
              present ProductRepository.find(params[:id]), with: ProductRepresenter
            end
          end
        end
      end
    end

API测试如下:

    describe 'GET /products/:id' do
      subject { get "/products/#{id}" }
    
      context 'when the product does not exist' do
        let(:id) { 12345 }
        it 'should return a 404' do
          subject
          expect(last_response.status).to eq 404
        end
      end
    
      context 'when the product does exist' do
        let(:product) { ProductRepository.create ModelFactory.build_product_attributes }
        let(:id) { product.id }
        it 'should return the details of the product identified by the id provided' do
          subject
          expect(last_response.status).to eq 200
          expect(json_response).to be_json_of_product product
        end
      end
    end

这里我们看到，API将我们之前定义的ProductRepository和ProductRepresenter串了起来。当没有查到指定Id的记录时，ProductRepository会抛出RecordNotFoundError并且被API捕获，最终返回404错误。

### 代码静态检查

随着敏捷开发的日渐流行，文档的地位不断被削弱，此时代码的可读性和质量往往成为直接决定项目是否健壮可维护的关键。这就要求我们的代码具有良好的风格（包括注释、命名等）、代码格式标准、程序没有非法调用和低级bug，以及用以对功能进行解释的单元测试能够尽可能多地覆盖核心功能。如果每次我们都手动去依次检查这些点是否达标，那持续集成将变得无比缓慢。更遑论持续交付了。

所以我们通常会使用一些自动化的代码检查工具来保证我们的代码质量，在本例中，笔者使用了rubocop来进行ruby代码的静态检查，部分配置如下：
   
    EmptyLinesAroundBlockBody:
      Exclude:
        - 'db/schema.rb'
    
    LineLength:
      Max: 120
      Exclude:
        - 'spec/**/*'
        - 'config/initializers/*'
    
    MethodLength:
      Max: 20
      Exclude:
        - 'db/migrate/*'
    
    Metrics/ClassLength:
      Max: 105
    
    Metrics/PerceivedComplexity:
      Max: 10
    
    WordArray:
      MinSize: 2

上面的配置文件中，规定了在block体前后必须有空行，每行代码最长不能超过120个符号，方法不能超过20行等，更多配置，请参考rubocop的官方文档。我们通常希望将代码检查工具和rake集成，这样我们就可以在使用rake运行自动化测试时，同时进行代码质量的检查，为了达到这个目的，我们只需要为它创建一个rake task，并把它加入rakefile的默认任务中，代码如下:

    namespace :quality do
      begin
        require 'rubocop/rake_task'
    
        RuboCop::RakeTask.new(:rubocop) do |task|
          task.patterns = %w{
            app/**/*.rb
            config/**/*.rb
            lib/**/*.rb
            spec/**/*.rb
            --rails
          }
        end
      rescue LoadError
        warn "rubocop not available, rake task not provided."
      end
    end

我们可以看到，在这个rake task中，我们只需要将期望被检查的文件加到task.patterns中即可，其他的任务就交给Rubocop帮我们处理吧。

### 自动化构建与发布

上文中提到，每个服务都是一个可独立部署的业务单元，经过静态检查、代码度量、单元测试、接口测试等阶段后，构建符合需求的部署包，部署包存在的形式是多种多样的。

本例中，笔者选用了docker作为部署的工具，作为一种新兴的虚拟化方式，Docker 跟传统的虚拟化方式相比具有众多的优势。首先，Docker 容器的启动可以在秒级实现，这相比传统的虚拟机方式要快得多。 其次，Docker 对系统资源的利用率很高，一台主机上可以同时运行数千个 Docker 容器。容器除了运行其中应用外，基本不消耗额外的系统资源，使得应用的性能很高，同时系统的开销尽量小。传统虚拟机方式运行 10 个不同的应用就要起 10 个虚拟机，而Docker 只需要启动 10 个隔离的应用即可。具体说来，Docker 在如下几个方面具有较大的优势。

* 更快速的交付和部署

对开发和运维（devop）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。

开发者可以使用一个标准的镜像来构建一套开发容器，开发完成之后，运维人员可以直接使用这个容器来部署代码。 Docker 可以快速创建容器，快速迭代应用程序，并让整个过程全程可见，使团队中的其他成员更容易理解应用程序是如何创建和工作的。 Docker 容器很轻很快！容器的启动时间是秒级的，大量地节约开发、测试、部署的时间。

* 更高效的虚拟化

Docker 容器的运行不需要额外的 hypervisor 支持，它是内核级的虚拟化，因此可以实现更高的性能和效率。

* 更轻松的迁移和扩展

Docker 容器几乎可以在任意的平台上运行，包括物理机、虚拟机、公有云、私有云、个人电脑、服务器等。 这种兼容性可以让用户把一个应用程序从一个平台直接迁移到另外一个。

* 更简单的管理

使用 Docker，只需要小小的修改，就可以替代以往大量的更新工作。所有的修改都以增量的方式被分发和更新，从而实现自动化并且高效的管理。

说了这么多好处，我们来看看使用docker怎么去构建并发布包含工程运行环境的镜像包。首先需要在项目的根目录创建一个Dockerfile:

    FROM ruby:2.2.0
    MAINTAINER YOUR NAME <YOUR EMAIL>
    
    RUN mkdir /app
    
    RUN apt-get update -y
    RUN apt-get install -y libsqlite3-dev freetds-dev
    
    WORKDIR /app
    
    ADD Gemfile /app/Gemfile
    ADD Gemfile.lock /app/Gemfile.lock
    RUN bundle install
    
    ADD . /app
    
    EXPOSE 80
    CMD bundle exec unicorn -c config/unicorn.rb -p 80

Docker能够自动读取Dockerfile中的指令来构建镜像包。 Dockerfile是一个文本文档，它实际上包含了你手动构建一个Docker镜像包的所有指令。通过在命令行执行docker build，Docker就会一步一步地执行Dockerfile中定义的指令。从上面的Dockerfile中可以看出，我们要构建的镜像包是基于一个基础的镜像包“ruby:2.2.0”，在它的基础上，首先清除/app文件夹，然后使用apt-get安装依赖的包，接着用bundler安装项目中用到的所有gem包，最后使用unicorn运行服务并开放80端口。整个过程一气呵成，Dockerfile帮我们省掉了许多手动操作的步骤。

既然有了Dockerfile，接下来要做的就是编写一个脚本将我们项目的构建以及发布自动化起来，因此就有了下面的build.sh：

    #!/bin/bash
    
    set -e 
    set -x
    
    DOCKER_REGISTRY_URL="YOUR REGISTRY URL"
    DOCKER_REGISTRY_USER_NAME="USER NAME"
    APP_NAME=${APP_NAME:-product-service}
    BUILD_NUMBER=${BUILD_NUMBER:-dev}
    VERSION=$(cat MAJOR_VERSION).$BUILD_NUMBER
    
    FULL_TAG=${DOCKER_REGISTRY_URL:="localhost"}/$DOCKER_REGISTRY_USER_NAME/$APP_NAME:$VERSION
    
    echo "Build and push Docker image to Registry"
    echo "Building Docker image"
    docker build -t $FULL_TAG .
    
    if [ $DOCKER_REGISTRY_URL != "localhost" ]; then
      echo "Pushing Docker image to Registry"
      docker push $FULL_TAG
    fi

build.sh实际上只做了两件事，首先通过执行docker build构建一个包含项目代码以及运行环境的镜像包，再使用docker push将其发布到某个docker registry中，这个docker registry可以是第三方的docker hub等，也可以自己搭建一个安全又私有的registry。

### 部署

在前一小节中介绍了如何自动化构建一个能够运行微服务的docker镜像包并将其发布到了docker registry上，这时我们就可以在任何一台安装了docker的机器上运行它了。由于是示例，笔者在这里编写了一个简单的脚本，你可以使用ssh登录想要部署的机器，并且执行下面的脚本，当然，我们也让该脚本在机器启动时自动执行。

    #! /bin/bash
    set -e
    
    [[ -z $BUILD_NUMBER ]] && echo "Need BUILD_NUMBER" && exit 1
    [[ -z $PRODUCT_DB_URL ]] && echo "Need PRODUCT_DB_URL" && exit 1
    [[ -z $PRODUCT_DB_USERNAME ]] && echo "Need PRODUCT_DB_USERNAME" && exit 1
    [[ -z $PRODUCT_DB_PASSWORD ]] && echo "Need PRODUCT_DB_PASSWORD" && exit 1
    [[ -z $ENV ]] && echo "Need ENV" && exit 1
    
    DOCKER_REGISTRY_URL="YOUR REGISTRY URL"
    DOCKER_REGISTRY_USER_NAME="USER NAME"
    APP_NAME=${APP_NAME:-product-service}
    APP_VERSION=$(cat MAJOR_VERSION).$BUILD_NUMBER
    FULL_TAG=${DOCKER_REGISTRY_URL:="localhost"}/$DOCKER_REGISTRY_USER_NAME/$APP_NAME:$APP_VERSION
    
    echo "Pulling Docker image from Registry"
    docker pull $FULL_TAG
    
    echo "Launching Dokcer Container"
    docker run -d -p 80:80 --name product-service -e PRODUCT_DB_URL=$PRODUCT_DB_URL -e PRODUCT_DB_USERNAME=$PRODUCT_DB_USERNAME -e PRODUCT_DB_PASSWORD=$PRODUCT_DB_PASSWORD -e RACK_ENV=$ENV $FULL_TAG

这个脚本会自动将我们之前发布的docker镜像包pull到本地，并且通过docker run为我们启动一个docker container，在执行docker run时，-e可以通过参数设置需要的环境变量，-p 80:80指的是将docker container的80端口映射到docker host的80端口，到这里，我们就可以通过docker host的80端口访问我们的服务了。

### 监控

上文中提到，由于每个服务都是一个可以独立运行的业务单元，同时每个服务都运行在不同的独立节点上。因此，需要为服务建立独立的监控、告警、快速分析和定位问题的机制，我们将它们统一归纳为服务的运维。其中，监控是整个运维环节，乃至服务生命周期中非常重要的一环。关于监控，目前业界已经有许多成熟的产品，譬如Zabbix、NewRelic、OneAPM等。这里，我们以NewRelic为例给大家介绍如何配置服务的监控。

由于本例中我们使用Grape来实现API，所以，首先在项目的Gemfile中添加

    gem 'newrelic-grape'
    
    执行bundle install。
    
    登陆[NewRelic官网](http://newrelic.com/)注册账户，并且创建应用，网站会自动为你的项目生成newrelic.yml，其中包含了需要的license_key。
    
    common: &default_settings
      license_key: ed95188a2872e7d8fee549a43bf128c7d6908aca
      app_name: Product-Service
      log_level: info
    
    development:
      <<: *default_settings
      app_name: Product-Service (Development)
      developer_mode: true
    
    test:
      <<: *default_settings
      monitor_mode: false
    
    staging:
      <<: *default_settings
      app_name: Product-Service (Staging)
    
    production:
      <<: *default_settings

将newrelic.yml保存至工程的config目录下，重新启动应用，喝杯咖啡等待几分钟，这时就能通过NewRelic的dashboard看到你应用的监控信息了。

### 告警

告警是运维环节另外一个重要的部分。当系统出现异常时，通过合适的告警机制，能及时、有效的通知相关负责人，做到早发现、早分析，早修复。针对每个服务，都应该提供有效的告警机制，确保当某服务出现异常时，能够准确有效的通知到责任人，并及时解决问题。目前比较流行的告警软件是PagerDuty。

PagerDuty是一款能够在服务器出问题时发送提醒的软件。在发生问题时，提醒的方式包括屏幕显示、电话呼叫、短信通知、电邮通知等，而且在无人应答时还会自动将提醒级别提高。上一小节介绍了用NewRelic进行监控，那么如何将PagerDuty和NewRelic集成起来呢？

登录NewRelic的控制台，点击 “Browser” -> “Alerts” -> “Channels and groups”。

<img src="/assets/article_images/2015-05-22-how-to-build-a-service/newrelic.jpg" />

接下来，点击“Create Channel”并选择“PagerDuty”，通过一系列的配置即可完成PagerDuty和NewRelic的集成，从此以后，NewRelic的任何报警信息都会被发送到PagerDuty，并且通过邮件或者电话的方式对服务的负责人进行提醒。

### 日志聚合

最后，我们来谈谈日志聚合，它也是运维部分必不可少的一环。由于微服务架构本质上是基于分布式系统之上的软件应用架构方式，随着服务的增多、节点的增多，登录节点、查看日志、分析日志的工作将会耗费更高的成本。通过日志聚合的方式，能有效将不同节点的日志聚合到集中的地方，便于分析和可视化。目前，业界最著名的日志聚合工具是Splunk和Logstash，其中笔者在项目中使用较多的是Splunk，Splunk是一个功能强大的日志管理工具，它不仅可以用多种方式来添加日志，生产图形化报表，最厉害的是它的搜索功能 - 被称为“Google for IT”。

这里我们就对如何配置Splunk做一简单介绍，Splunk可以单机部署，也可以分布式部署，在微服务架构中，由于不同的服务可能部署在不同的主机上，而我们希望对日志进行集中管理，因此通常使用的是分布式部署。分布式部署时，每个部署有微服务的主机上都会安装一个转发器(forwarder)，日志通过转发器被转发到Splunk服务器上，负责接收的这台Splunk实例被称作接收器，他通常也作为Splunk索引器(indexer)。

Splunk服务器配置非常简单，只需要下载相应的安装包，根据向导一步步的进行就好，安装完成后便可通过8000端口在浏览器中进行访问。我们主要介绍的是Splunk forwarder，也就是客户端的配置。

首先需要下载[Splunk Universal Forwarder](http://download.splunk.com/products/splunk/releases/6.1.1/universalforwarder/windows/splunkforwarder-6.1.1-207789-x64-release.msi)并运行，一直点击”Next”直到”Receiving Indexer”对话框，在这一页，需要配置indexer的ip地址和端口号，端口号默认为9997，填写好后继续点击”Next”直到安装完成。

配置Splunk Universal Forwarder

打开文件"C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf”，如果你在安装时选择了其他路径，请打开对应目录下的inputs.conf。
在文件末尾添加以下代码：
 
    [monitor://日志路径]
    ignoreOlderThan = 10d
    disabled = 0
    followTail = 0
    index = [index name]
    sourcetype = [source type]

ignoreOlderThan代表忽略10天前的历史日志，也就是从近十天的日志开始转发，index和sourcetype用来在服务端对日志进行搜索，具体参数请参考[Splunk官方文档](http://docs.splunk.com/Documentation/Splunk/6.1/admin/Inputsconf)

在命令行执行以下命令重启服务

    cd "C:\Program Files\SplunkUniversalForwarder\bin"
    splunk restart

到这里Splunk Universal Forwarder就配置完成了，但是由于服务端还没创建对应的index，所以我们需要登录服务端控制台，根据刚才在inputs.conf中设置的index name创建对应的index。这样，就可以通过浏览器访问Splunk indexer查看和搜素日志信息了。

### 持续集成

之前已经谈到对于服务本身，应该有独立的持续集成环境(CI)，以及在CI上配置好的相关运行的项目。从代码库而言，不同的服务，也需要有相应的脚本，来支持如何快速、有效安装并配置CI代理节点（Agent或者Slave）。所谓CI代理节点，通常是指能够从指定的版本库上下载最新源码，运行测试、构建包，执行部署的这样一类机器，可以是虚拟机，也可是物理机。

在笔者的项目中，通常都会在持续集成环境定义如下三个阶段：

譬如，在笔者从事的项目中，每个服务的持续集成流水线通常会分成如下三个阶段：

1. 构建

    a) 静态检查          （代码规范检查)
    
    b) 测试验证          （运行各种类型的测试）
    
    c) 依赖包升级检测（检查当前Gem的待更新情况）
    
    d) 构建RPM包      （生成应用的软件包）
    

2. 发布

    a) 创建AMI映像 （根据生成的RPM包，构建新的基于AWS的映像）
    
    b) 打标签           （标签的目的是为了限制当前包，能够被部署在什么样的环境上。譬如对于类生产环境，应该期望仅当标              
                               记了staging后，才可以被部署到类生产环境）

3. 部署

    a) 部署到测试环境     （通常情况下，测试环境主要是团队内部使用，因此每当有映像生成后，可以自动完成部署）
    
    b) 部署到类生产环境 （通常其存在的目的是为业务团队做演示，一般是按需部署）
     
    c) 部署到生产环境       (产品环境一般为手动部署)

对于这些阶段，在实际的项目中，我们可以使用不同的rake任务来实现。例如： `rake quality`，`rake spec`等。

通过这些事先存在的rake任务，我们只需要定义一个通用的调用脚本，安装运行环境，就能帮助我们在持续集成节点（譬如Jenkins、Bamboo）中有效的执行任务。

    ci.sh
    #!/bin/bash
    set -e
    
    source ./ci-setup.sh
    
    # Delegate to rake
    bundle exec rake "$@"
    
    ci-setup.sh
    #!/bin/bash
    set -e
    
    # Check the directory
    if [ -z "$PROJECT_ROOT_DIR" ]; then
      PROJECT_ROOT_DIR="."
    fi
    
    # Get Ruby version
    RUBY_VERSION=$(cat ${PROJECT_ROOT_DIR}/.ruby-version | cut -d. -f1,2)
    echo "Using ruby version ${RUBY_VERSION}"
    
    # Install/Updgrade ruby
    sudo yum install -y chruby-ruby-${RUBY_VERSION}
    
    # Use ruby specified in .ruby-version file
    source /usr/share/chruby/chruby.sh
    chruby ruby-${RUBY_VERSION}
    type ruby
    ruby -v
    
    # Bootstrap bundler
    BUNDLER_VERSION=$(cat ${PROJECT_ROOT_DIR}/.bundler-version)
    gem specification bundler -v "$BUNDLER_VERSION" >/dev/null 2>&1 || gem install --no-rdoc --no-ri bundler -v "$BUNDLER_VERSION"
    
    bundle -v
    
    bundle install --jobs 8 --retry 3

譬如，如果执行代码静态检查，则只需要在持续集成服务器上定义项目，设置运行命令为 ci.sh 'quality’即可。当然，我们也可以直接调用shell脚本，比如之前提到的用来构建docker镜像包的build.sh。

### 总结

综上所述，从微服务架构的角度而言，每个服务都是一个独立的、可部署的业务单元。因此，当谈论到代码结构和实现上，我们也希望代码库本身是一个独立的、自组织的单元。在每个服务的代码库里，应该包括但不限于业务逻辑的实现代码、持续集成环境的配置脚本、部署环境的定义脚本、部署脚本、监控与告警机制的配置脚本等。通过这样一个代码库，能有效促进团队技能的全面发展，同时减少团队和其他部门之间的非必要的沟通成本，让团队有更大的自主性和灵活性，也更容易形成以做产品为目标的全功能团队。




