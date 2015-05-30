---
layout: post
title: "消费者驱动契约测试"
category: tutorial
tags: microservice, contracts test, pact
---

### 概述

目前，有很多文章在讨论微服务的优缺点，微服务的一些优势是显而易见的：

* 每个服务都很简单，只关注于一个业务功能。
* 每个微服务可以由不同的团队独立开发。
* 微服务是松散耦合的。
* 微服务可以通过不同的编程语言与工具进行开发。

这些优势使得微服务看起来是非常完美的解决方案，不过Contino的CTO Benjamin Wootton正在基于微服务对一个系统进行架构设计，他遇到了不少困难，并且将遇到的这些问题发表在了题为Microservices - Not A Free Lunch!的文章中。文中提到的关于微服务的缺点有：

* 运维成本过高
* DevOps是必须的
* 接口不匹配
* 代码重复
* 分布式系统的复杂性

其中，“接口不匹配”是指服务依赖于彼此间的接口进行通信，改变一个服务的接口会对其他服务造成影响。Wootton谈到：

> 修改了契约一方的语法或是语义，那么所有其他服务都需要理解这个改变。在微服务环境下，这意味着简单的交叉改变会导致很多不同的组件发生变化，这些组件需要以一种协调一致的方式进行发布。
> 当然了，我们可以通过向后兼容的方式避免这些变化的发生，不过实际情况却是业务驱动的需求让我们没法实现分阶段的发布。
> 如果彼此协同的服务向前推进并且不再同步了，比如说使用canary发布方式，那么修改消息格式的效果就难以可视化了。

而契约测试能够帮助我们有效地降低微服务架构带来的这种风险，契约测试是一种针对外部服务的接口进行的测试，它能够验证服务是否满足消费方期待的契约。当一些消费方通过接口使用某个组件的提供的行为时，它们之间就产生了契约。这个契约包含了对输入和输出的数据结构的期望，性能以及并发性。

一个组件的每个消费者会根据需求的不同生成不同的契约。如果这个组件会被频繁修改，那么保证每个消费者的需求依旧能够被满足是十分重要的。契约测试能够提供一个机制去验证组件能否始终满足契约。当涉及的组件是微服务时，接口便是各服务暴露出来的公共API。每个消费方服务的维护者只需要编写独立的测试套件来验证服务提供方中他所使用的那一部分就可以了。这些测试不是组件测试，它不需要深入地去测试服务的行为，只需要去测服务的输入输出包含了必需的属性，响应的等待时间以及吞吐量。

理想的情况下，各个消费方的团队编写的契约测试套件是打包的并且在服务提供端的构建流水线上是可执行的。通过这种方式，服务提供方的维护者能够很容易发现他们的修改对消费方的影响。因此，契约测试不仅能够为服务的消费方提供信心，其实对服务提供方的维护者的价值更高。通过从各个消费者收到的契约测试套件，他们便能够安全的对服务进行修改而不会影响到服务的消费方。

契约测试对构建新服务也十分有价值。消费者可以通过构建一个测试套件来描述它对服务的需求，从而驱动服务API的设计。目前业界已经出现了很多用来编写契约测试的工具，譬如[Pact](https://github.com/realestate-com-au/pact)，[Pacto](https://github.com/thoughtworks/pacto) and [Janus](https://github.com/gga/janus).

### PACT

接下来，我们将介绍如何使用Pact编写契约测试。Pact为服务消费方提供了一个API来定义它们将要发送给服务提供方的HTTP请求以及它们期望的响应。这些期望值在消费方的测试中被用来提供一个Mock的服务提供方，这些交互会被记录下来，并在服务提供方的测试中重放来保证服务提供方确实提供了消费方期望的响应，这使得我们可以在集成点的任何一端通过快速的单元测试来进行契约测试。

还是回到之前文章中介绍过的例子：

假定某电商平台，作为消费方，它需要通过一个Product Service获取产品相关的信息。在消费方，我们需要定义一个model(Product类)来表示从Product Service返回回来的数据，以及一个客户端(Product Service Client)负责向Product Service发送HTTP请求。

<img src="/assets/article_images/2015-05-29-consumer-driven-contracts-test/architecture.jpg"/>

**在Online Shop(消费端)项目中**

1. 从创建model开始

    假设有一个Product类如下，Product的信息存放在远端的服务器上，我们需要通过向Product Service发送HTTP请求来获取Product数据。
    
        class Product
          include Virtus.model
        
          attribute :id, Integer
          attribute :name, String
          attribute :price, Float
          attribute :category, Integer
        end

2. 创建一个Product Service Client类的框架

    假设有一个像下面这样的Product Service Client类。
    
        require 'httparty'
        
        class ProductServiceClient
          include HTTParty
          base_uri 'http://product-service.com'
        
          def get_product(id)
            # Yet to be implemented because we're doing Test First Development...
          end
        end

3. 配置mock的Product Service

    下面的代码会创建一个mock的service运行在localhost:1234上, 它能够对你的应用发来的HTTP请求作出相应，就好像它是一个真的”Product Service”一样，你可以用它来构建你需要的期望值。你设置的service参数将作为访问这个mock service的方法名，在这个例子中，它是”product_service”。
    
        # In /spec/service_providers/pact_helper.rb
        
        require 'pact/consumer/rspec'
        # or require 'pact/consumer/minitest' if you are using Minitest
        
        Pact.service_consumer "Online Shop" do
          has_pact_with "Product Service" do
            mock_service :product_service do
              port 1234
            end
          end
        end

4. 为Product Service Client写一个失败的测试

        # In /spec/service_providers/product_service_client_spec.rb
        
        # When using RSpec, use the metadata `:pact => true` to include all the pact functionality in your spec.
        # When using Minitest, include Pact::Consumer::Minitest in your spec.
        
        describe ProductServiceClient, :pact => true do
        
          before do
            # Configure your client to point to the stub service on localhost using the port you have specified
            ProductServiceClient.base_uri 'localhost:1234'
          end
        
          subject { ProductServiceClient.new }
        
          describe 'get_product' do
        
            before do
              product_service.given('a product exists').
                  upon_receiving('a request for a product').
                  with(method: :get, path: '/products/1', query: '').
                  will_respond_with(
                      status: 200,
                      headers: {'Content-Type' => 'application/json'},
                      body: { id: 1, name: 'Giant', category: 'bicycle', price: 123.4} )
            end
        
            it 'returns a product' do
              expect(subject.get_product(1)).to eq(Product.new( id: 1, name: 'Giant', price: 123.4, category: 1))
            end
          end
        end

5. 运行测试

    运行ProductServiceClient的测试会在配置的pact目录下(默认是 spec/pacts)生成一个pact文件。日志会被输出到配置的log目录下(默认是log)。
    
    当然，上面的测试会挂掉，因为我们还没有实现ProductServiceClient的`get_product`方法。那么接下来我们就来实现ProductServiceClient。

6. 实现Product Service Client的消费者方法

          class ProductServiceClient
            include HTTParty
            base_uri 'http://product-service.com'
          
            def get_product(id)
              product_json = JSON.parse(self.class.get("/product/#{id}").body)
              Product.new(product_json)
            end
          end

    再跑一次测试，这次终于过了！我们现在就拥有了一个可以用来验证Product Service返回值是否满足期望的pact文件了。

**在Product Service中**

1. 创建API类的框架

    这里你可以选择你喜欢的框架，笔者比较倾向使用[Grape API](https://github.com/intridea/grape)和[Roar](https://github.com/apotonick/roar)。这里不要完成具体的实现，我们要用契约测试去驱动实现，你还记得吗？

2. 告诉你的服务提供方：你需要满足之前生成的pact文件。

    在Rakefile中`require “pact/tasks”`。
    
        # In Rakefile
        require 'pact/tasks'
    
    创建`pact_helper.rb`, 推荐的路径是`spec/service_consumers/pact_helper.rb`
    
        # In specs/service_consumers/pact_helper.rb
        
        require 'pact/provider/rspec'
        
        Pact.service_provider 'Product Service' do
        
          honours_pact_with 'Online Shop' do
        
            # This example points to a local file, however, on a real project with a continuous
            # integration box, you would use a [Pact Broker](https://github.com/bethesque/pact_broker) or publish your pacts as artifacts,
            # and point the pact_uri to the pact published by the last successful build.
        
            pact_uri '../online_shop/specs/pacts/online_shop-product_service.json'
          end
        end

3. 运行失败的测试

        $ rake pact:verify

    恭喜你！你现在可以基于这个失败的测试继续开发了。
    
    接下来，你就可以去编写实现代码满足你的第一个测试。就这样循环下去，不断编写测试去驱动你的实现，这就是所谓的消费者驱动契约测试。

### 总结

本文从微服务的优缺点入手，通过微服务可能导致的服务端和消费端接口不匹配这一问题，引出了消费者驱动的契约测试，并详细介绍了Pact测试框架的用法。由此可见，使用微服务不仅仅是采用一种新的架构这么简单，如果不对其开发技术做相应的改进的话，很难发挥出其真正的优势，也就是说，微服务其实是一把双刃剑，它有很多吸引人的地方，不过在拥抱微服务之前，你需要认清它所带来的挑战并将其克服。

### 参考资料

1. http://www.infoq.com/news/2014/05/microservices
2. http://martinfowler.com/articles/microservice-testing/#testing-contract-diagram
3. http://www.martinfowler.com/articles/consumerDrivenContracts.html
4. https://github.com/realestate-com-au/pact

