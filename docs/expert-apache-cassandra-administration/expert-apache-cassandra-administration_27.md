# docker-compose
Define and run multi-container applications with Docker.
Usage:
docker-compose [-f ...] [options] [COMMAND] [ARGS...]
docker-compose -h|--help
#
```

#### 安装 Cucumber

我使用的是 Ubuntu 系统，因此我通过以下方式安装 `Cucumber`：

```
$ gem install cucumber
Fetching: gherkin-4.1.3.gem (100%)
...
Done installing documentation for gherkin, cucumber-core, cucumber-wire, cucumber after 4 seconds
4 gems installed
$
```

建议安装一个名为 `bundler` 的额外工具。`Bundler` 为基于 Ruby 的项目（例如我们将要使用的 `Cucumber`）提供了一致的环境。它通过跟踪并安装你需要的所有 gems 及其正确版本来实现这一点。换句话说，`bundler` 提供了一种摆脱依赖地狱的方法。

```
$ gem install bundler
Successfully installed bundler-1.15.1
1 gem installed
#
```

现在你已经安装了 Docker、`Docker-Compose` 和 `Cucumber`，准备好使用 `BDD` 方法启动一个 Cassandra 集群了。但首先，简单介绍一下用户故事和 `BDD`。

### 用户故事

`BDD` 使用故事来表达你想要完成的目标。它使用基于自然语言的 `Gherkin 语法`，这种语法与我们自然说话的方式非常相似，用来描述你想要做的事情。

你将你的主要目标描述为特性（features），如下所示，这里你想要部署一个新的 Cassandra 集群：

> 特性：Cassandra 集群
>   作为一名大数据管理员
>   我想要部署一个 Cassandra 集群
>   这样我就能存储和处理海量数据

一旦你描述了你的高层目标，你就描述使目标得以实现的具体场景。在这个例子中，你想要启动一个 Cassandra 数据库并测试其版本。

> 场景：启动 CQL Shell
>   假设相关服务正在运行
>    并且我在 `cassandra` 上执行 `cqlsh -e 'show version'`
>    那么我应该看到 "CQL spec 3.4.2"
>    并且我应该看到 "Cassandra 3.7"



### 准备运行 BDD 测试

此时，你无需从头开始，可以直接从 GitHub 下载测试和工作所需步骤。在 GitHub 上，coshxlabs（[`www.coshx.com`](http://www.coshx.com)）有一个仓库（[`https://github.com/coshx/docker-bdd`](https://github.com/coshx/docker-bdd)），演示了如何使用 BDD 和 Cucumber 来编排 Docker 容器，帮助你快速启动开发环境。

1.  你可以从 GitHub 克隆 Coshxlabs 的代码。
    ```
    $ git clone https://github.com/coshx/docker-bdd
    $ git reset --hard e9f807c
    ```

2.  移动到你从 GitHub 克隆的目录，该目录名为 `docker-bdd`。这个目录包含了所有生成用于运行 Cassandra 的 Docker 容器所需的配置文件。
    ```
    # cd docker-bdd
    # ls
    circle.yml  docker-compose.yml  example   Gemfile       README.md
    deploy.sh   dockerfiles         features  Gemfile.lock
    #
    ```

3.  进入 `features` 目录。
    ```
    # cd features
    # ls
    android_app.feature  app.feature  db.feature  step_definitions    support  web.feature
    ```

4.  创建一个名为 `cassandra.feature` 的文件，并添加上一节描述的与 Cassandra 相关的功能（feature）和场景（scenario）。保存文件。`cassandra.feature` 文件的内容应如下所示：
    ```
    # cat cassandnra.feature
    Feature: Cassandra Cluster
    As a Big Data Administrator
    I want to deploy a cassandra cluster
    So I can store and query lots of data
    Scenario: Launch a CQL Shell
    Given the services are running
    And I run "cqlsh cassandra -e 'show version'" on "cassandra-dev"
    Then I should see "CQL spec"
    And I should see "Cassandra 3.7"
    ```

5.  将工作目录切换到 `docker-bdd` 目录，并编辑 `docker-compose.yml` 文件，使其包含以下内容（`docker-compose.yml` 文件包含了 Docker 应该下载和安装的镜像名称）。
    ```
    cassandra:
    image: cassandra
    cassandra-dev:
    image: cassandra
    links:
    - cassandra
    ```

这段代码创建了两个独立的 Docker 容器，都使用相同的 Docker 镜像（“cassandra”）。你需要两个容器：一个在后台运行 Cassandra 实例，另一个用于运行 `cqlsh`。记住，你的场景中包含了以下内容：

```
在 "cassandra" 上运行 "cqlsh -e 'show version'"
```

### 运行 BDD 测试

现在你已准备好使用 Cucumber 运行 BDD 测试。操作方法如下：

```
