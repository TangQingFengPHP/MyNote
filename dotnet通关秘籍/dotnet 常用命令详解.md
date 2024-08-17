## 命令预览

### 基础命令

* new：创建项目

* restore：恢复依赖

* build：编译项目

* publish：生成项目需要的文件准备发布项目

* run：运行项目

* test：测试项目

* vstest：从指定的程序集中运行测试

* pack：打包代码为nuget包

* clean：清理项目的输出文件，如：obj、out等

* sln：执行solution相关的操作

* help：打印帮助信息

* watch：运行项目在热重载模式下，即监听文件的变更会自动重启项目

* format：格式化代码匹配 `editorconfig` 文件中的设置格式

### 项目包和引用相关的命令

* add package：添加或更新项目的外部包引用

* add reference：添加项目到项目之间的引用关系

* remove package：移除项目的外部包引用

* remove reference：移除项目到项目之间的引用关系

* list package：列出项目或solution的外部包引用

* list reference：列出项目到项目之间的引用关系

### NuGet 相关的命令

* nuget delete：从服务器中删除或取消列出包

* nuget locals：命令清除或列出 http 请求缓存、临时缓存或计算机范围的全局包文件夹中的本地 NuGet 资源。

* nuget push：推送包到nuget服务器并进行发布

* nuget add source：添加nuget源

* nuget disable source：禁用nuget源

* nuget enable source：启用nuget源

* nuget list source：列出nuget源

* nuget remove source：移除nuget源

* nuget update source：更新nuget源

* nuget verify：验证已签名的 NuGet 包

* nuget trust：获取或设置 NuGet 配置的受信任签名者

* nuget sign：使用证书对与第一个参数匹配的所有 NuGet 包进行签名。

* package search：搜索nuget包

* nuget why：显示特定包的依赖关系图

### Workload相关的命令

* workload install：安装一个或多个工作负载

* workload list：列出已安装的工作负载

* workload update：更新已安装的工作负载

* workload restore：还原项目所需的工作负载

* workload repair：修复已安装的工作负载

* workload uninstall：卸载已安装的工作负载

* workload search：搜索可用的工作负载

* workload clean：删除可能已从之前的更新和卸载中保留的工作负载组件

### 高级命令

* sdk check：列出最新可用的 `.NET SDK` 和 `.NET Runtime` 版本

* dev-certs：生成一个自签名的证书来启用https以供开发环境下使用

### 工具管理命令

* tool install：安装指定的 `.NET 工具`

* tool list：列出所有已安装的工具

* tool update：更新已安装的工具

* tool restore：还原工具

* tool run：运行工具

* tool uninstall：卸载工具

* tool search：在 `nuget.org` 中搜索工具

## 命令用法示例

### 单dotnet命令通用选项

* 获取dotnet sdk版本

```shell
dotnet --version
```

* 获取dotnet相关的信息，包括：运行时环境、工作负载、主机信息、sdk信息、runtime信息等。

```shell
dotnet --info
```

![alt text](/images/dotnet常用命令详解/dotnet-常用命令.png)

* 列出安装的运行时信息

```shell
dotnet --list-runtimes
```

* 列出安装的sdk信息

```shell
dotnet --list-sdks
```

* 打印帮助信息

```shell
dotnet -h|--help
```

### 运行命令的通用选项

* 启用诊断输出

```shell
-d|--diagnostics

例如：dotnet build -d
```

* 设置命令调试信息的输出级别

```shell
-v|--verbosity <LEVEL>

级别有：[uiet], m[inimal], n[ormal], d[etailed], diag[nostic]

例如：dotnet build -v d
```

* 打印指定命令的帮助信息

```shell
-?|-h|--help

例如：dotnet build -h
```

### nuget包管理命令

添加nuget包后会在csproj配置文件中写入以下示例配置：

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="6.0.4" />
```

* 添加nuget包到项目中

```shell
dotnet add package Microsoft.EntityFrameworkCore
```

* 列出项目添加的包

```shell
dotnet list abc.csproj package
```

* 移除项目指定的包

```shell
dotnet remove package Microsoft.EntityFrameworkCore
```

### 项目到项目之间引用的命令

添加本地项目引用会在csproj配置文件中写入以下示例配置：

```xml
<ItemGroup>
  <ProjectReference Include="app.csproj" />
  <ProjectReference Include="..\lib2\lib2.csproj" />
  <ProjectReference Include="..\lib1\lib1.csproj" />
</ItemGroup>
```

* 指定项目添加项目引用

```shell
dotnet add app/app.csproj reference lib/lib.csproj
```

* 默认当前的项目添加项目引用

```shell
dotnet add reference lib1/lib1.csproj lib2/lib2.csproj
```

* 使用 `**/*` 匹配添加指定目录下的所有项目

** 表示递归匹配当前目录下的所有目录及其子目录

* 表示当前匹配目录下的所有文件

```shell
dotnet add app/app.csproj reference **/*.csproj
```

* 直接引用 `dll` 文件的方式

直接在csproj文件中编写如下示例配置

```shell
<ItemGroup>
  <Reference Include="MyAssembly">
    <HintPath>.\MyDLLFolder\MyAssembly.dll</HintPath>
  </Reference>
</ItemGroup>
```

* 列出指定项目添加的引用

```shell
dotnet list app/app.csproj reference
```

* 列出当前项目添加的引用

```shell
dotnet list reference
```

* 移除指定项目添加的引用

```shell
dotnet remove app/app.csproj reference lib/lib.csproj
```

* 移除指定项目添加的多个引用

```shell
dotnet remove app/app.csproj reference lib1/lib1.csproj lib2/lib2.csproj
```

* 使用 `**/*` 移除多个引用

```shell
dotnet remove app/app.csproj reference **/*.csproj
```

### 构建项目

`dotnet build` 子命令构建项目和依赖到一组二进制文件，二进制文件包含带有 `.dll` 扩展名的中间语言 (IL) 文件中的项目代码，构建后的文件有：

1. 如果当前项目是可运行的，则会输出一个可执行的文件

2. 如果使用的Debug模式构建的，则会输出一个 `.pdb` 后缀的文件

3. 输出一个 `.deps.json` 文件，作用是列出项目的依赖项

4. 输出一个 `.runtimeconfig.json` 文件，作用是指定项目的运行时和版本。
* 指定项目或solution进行构建

```shell
dotnet build [project|solution]
```

* 默认构建当前目录的项目或solution

```shell
dotnet build
```

* 指定构建后的目标架构

```shell
dotnet build -a|--arch<ARCHITECTURE>

示例：
在win-x86机器上构建，使用此选项，则默认给出了win-x86运行时。
dotnet build -a x86

可用的RID(运行时标识符)值有：

Windows：
win-x64、win-x86、win-arm64

Linux：
linux-x64、linux-musl-x64、linux-musl-arm64、linux-arm、linux-arm64、linux-bionic-arm64

macOS：
osx-x64
osx-arm64

IOS：
ios-arm64

Android：
android-arm64
```

* 指定构建的配置模式

默认为 `Debug`，可选 `Release`，构建为 `Debug` 则输出的二进制尺寸更大，有完整的debug信息，有利于调试。

```shell
dotnet build -c|--configuration <CONFIGURATION>

示例：
dotnet build -c Release
```

* 指定构建后的目标框架

框架代号必须预先在项目配置文件(.csproj)中存在

```shell
dotnet build -f|--framework <FRAMEWORK>

示例：
dotnet build -f net7.0,net6
```

* 忽略本地项目之间的引用，仅构建当前指定的项目

```shell
dotnet build --no-dependencies
```

* 在构建过程中不执行隐式的 `restore`

```shell
dotnet build --no-restore
```

* 不显示启动banner和版权信息

```shell
dotnet build --nologo
```

* 构建一个依赖框架的应用，如果要部署到服务器，则必须预先安装运行时

```shell
dotnet build --no-self-contained
```

* 指定构建输出的目录

如果不指定，则默认输出位置为：`./bin/<configuration>/<framework>/`，如 `.bin/Debug/net7.0`

```shell
dotnet build -o OutPutDir
```

* 指定构建的目标系统

如果当前在win-x64机器上，指定了 `--os linux`，则会自动推断 `RID` 为：`linux-x64` 

```shell
dotnet build --os <OS>
```

* 指定 `MSBuild` 构建属性

```shell
dotnet build --property:<NAME1>=<VALUE1>;<NAME2>=<VALUE2>

或

dotnet build --property:<NAME1>=<VALUE1> --property:<NAME2>=<VALUE2>

多个属性可以用分号隔开，也可以指定 --property 多次
```

* 指定构建目标的运行时

如果没有指定，则默认使用当前机器的系统和架构进行构建

```shell
dotnet build -r|--runtime <RUNTIME_IDENTIFIER>

示例：
dotnet build -r linux-x64
```

* 构建一个 `.NET` 运行时自包含的应用

例如部署到服务器上，服务器不需要安装运行时也能直接运行应用

注意：如果使用了 `-r` 选项指定了 `RID`（RUNTIME_IDENTIFIER） ，则此选项默认为true。

```shell
dotnet build --self-contained [true|false]
```

### 清理构建输出

仅清理构建期间创建的输出，中间文件目录：`obj` 和 最终输出目录 `bin`

* 指定项目或solution

```shell
dotnet clean <PROJECT|SOLUTION>
```

* 清理指定配置构建的输出目录

```shell
dotnet clean -c|--configuration <CONFIGURATION>

示例：
dotnet clean -c Release
```

* 清理指定框架构建的输出目录

```shell
dotnet clean -f|--framework <FRAMEWORK>

示例：
dotnet clean -f net7.0
```

* 清理指定输出目录构建的输出

```shell
dotnet clean -o|--output <OUTPUT_DIRECTORY>

示例：
dotnet clean -o abc
```

* 清理指定运行时构建的输出目录

```shell
dotnet clean -r|--runtime <RUNTIME_IDENTIFIER>

示例：
dotnet clean -r win-x64
```

### 开发证书命令

* 使用 `https` 子命令查看已创建的证书或者创建新证书

首先会查找当前机器上是否已生成后有效的开发证书，没有的话会创建一个开发证书

```shell
dotnet dev-certs https
```

* 显式检测证书的有效性

```shell
dotnet dev-certs https -c|--check
```

* 显式检测证书的有效性，并信任此证书

```shell
dotnet dev-certs https -c --trust
```

* 清理生成的开发证书

```shell
dotnet dev-certs https --clean
```

* 使用 `-ep|--export-path <PATH>` 导出证书文件

* 导出证书的公共部分为一个 `PFX` 文件

```shell
dotnet dev-certs https -ep ./certificate.pfx
```

* 导出证书的公共部分为一个 `PEM` 格式的文件

```shell
dotnet dev-certs https -ep ./certificate.pfx --format PEM
```

* 导入一个证书文件

```shell
dotnet dev-certs https --import ./certificate.pfx
```

### 代码格式化命令

* 指定solution进行格式化

```shell
dotnet format ./solution.sln
```

* 指定项目进行格式化

```shell
dotnet format ./application.csproj
```

* 验证所有代码文件是否正确格式化

```shell
dotnet format --verify-no-changes
```

* 显式包含指定目录进行格式化

```shell
dotnet format --include ./src/ ./tests/
```

* 显式排除指定目录进行格式化

```shell
dotnet format --exclude ./src/a/
```

### NuGet相关的操作命令

* 添加一个包源，并设置源的名称

```shell
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
```

* 添加当前机器的目录为包源

```shell
dotnet nuget add source ./NuGetDir
```

* 添加一个需要授权的包源，例如需要输入用户名，密码

-n 指定包源的名称
-u 指定需要授权的用户名
-p 指定需要授权的密码

```shell
dotnet nuget add source https://abc/devTeam -n devTeam -u root -p root
```

* 获取当前目录 `NuGet` 所有配置项

```shell
dotnet nuget config get all
```

* 获取 `NuGet` 指定配置项的值

```shell
dotnet nuget config get repositoryPath
```

* 设置 `NuGet` 指定配置项的值

```shell
dotnet nuget config set repositoryPath ./abc
```

* 设置指定配置文件中的配置项的值

```shell
dotnet nuget config set repositoryPath ./abc --configfile ./nuget.config
```

* 移除指定配置项的值

```shell
dotnet nuget config unset repositoryPath
```

* 移除指定配置文件中的配置项的值

```shell
dotnet nuget config unset repositoryPath --configfile ./nuget.config
```

### 打包代码为NuGet包

* 默认在当前项目为NuGet包

```shell
dotnet pack
```

* 指定项目打包为NuGet包

```shell
dotnet pack ./project1.csproj
```

* 指定打包后的输出目录

```shell
dotnet pack -o|--output

示例：
dotnet pack --output nupkgs
```

* 打包时跳过构建步骤

```shell
dotnet pack --no-build
```

* 打包文件带上在 `.csproj` 文件指定的版本后缀

`<VersionSuffix>$(VersionSuffix)</VersionSuffix>`

```shell
dotnet pack --version-suffix "ci-1234"
```

* 指定包版本进行打包

```shell
dotnet pack -p:PackageVersion=2.1.0
```

* 指定目标框架进行打包

```shell
dotnet pack -p:TargetFrameworks=net7.0
```

* 指定目标运行时进行打包

```shell
dotnet pack --runtime linux-x64
```

* 在 `NuGet.org` 上搜索指定的包

```shell
dotnet package search Newtonsoft.Json --source https://api.nuget.org/v3/index.json
```

### 项目发布命令

发布应用和它的依赖到指定文件夹准备部署服务器，输出的目录包括以下文件：

1. 中间码（IL）dll文件

2. `.deps.json` 文件包含项目的依赖配置

3. `.runtimeconfig.json` 项目运行时配置

4. 项目的外部依赖，包含远程依赖和本地项目依赖。

基本选项与 `build` 命令大部分相同，不在赘述

* 发布成依赖框架的跨平台的二进制可执行文件

```shell
dotnet publish
```

* 发布成指定运行时的且自包含运行时的可执行文件

```shell
dotnet publish --runtime linux-x64
```

* 发布成指定运行时的且不包含运行时的可执行文件

```shell
dotnet publish --runtime linux-x64 --self-contained false
```

* 发布当前的应用不带本地引用的项目

```shell
dotnet publish --no-dependencies
```

### 依赖还原命令

.NET CLI 使用 NuGet 来查找项目需要的依赖，如果本地缓存中没找到，则下载它们放到本地缓存中，还原过程中会检测相关依赖的兼容性。

一些命令会在运行前隐式执行 `restore`，例如：

1. dotnet new 

2. dotnet build

3. dotnet run

4. dotnet test

5. dotnet publish

6. dotnet pack

所以此命令的场景一般用于在自动化部署的流程中控制执行流程中使用，先在其他命令中使用 `--no-restore` 选项禁止隐式执行 `restore`

如果项目中有 `nuget.config` 文件，则优先取此文件中的配置

* 还原当前项目的依赖和工具

```shell
dotnet restore
```

* 指定项目进行还原

```shell
dotnet restore ./app1.csproj
```

* 指定依赖源进行还原

```shell
dotnet restore -s ../MyPackages
```

### 使用run在本地开发环境运行应用

`dotnet run` 无需任何显式编译或启动命令即可运行源代码。

即不用预先编译成dll文件即可直接运行源码，适用于本地开发环境快速开发迭代

run 命令会使用依赖的缓存，所以不建议在生产环境使用此命令运行应用

如果运行编译后的dll文件，必须直接运行dotnet 某某dll文件

```shell
dotnet myapp.dll
```

* 分割 `run` 命令的参数和应用使用的参数

```shell
dotnet run -- --property name=value
```

* 指定目标系统架构

```shell
dotnet run -a|--arch <ARCHITECTURE>

示例：
dotnet run -a x64
```

* 指定配置模式

```shell
dotnet run -c Release
```

* 指定目标框架

```shell
dotnet run -f net7.0
```

* 在运行之前不构建项目

```shell
dotnet run --no-build
```

* 指定目标系统

```shell
dotnet run --os <OS>

示例：
dotnet run --os linux
```

* 指定运行的项目

```shell
dotnet run --project <PATH>

示例：
dotnet run --project ./project1.csproj
```

* 指定构建属性

```shell
dotnet run --property:<NAME>=<VALUE>

示例：
dotnet run --property:<NAME1>=<VALUE1>;<NAME2>=<VALUE2>
或
dotnet run --property:<NAME1>=<VALUE1> --property:<NAME2>=<VALUE2>
```

* 指定目标运行时

```shell
dotnet run -r|--runtime <RUNTIME_IDENTIFIER>

示例：
dotnet run -r linux-x64
```

### 管理项目solution的命令

* 在当前目录创建与目录同名的solution文件

```shell
dotnet new sln
```

* 在当前目录创建指定名称的solution文件

```shell
dotnet new sln --name MySolution
```

* 在指定目录下创建于目录同名的solution文件

```shell
dotnet new sln --output MySolution
```

* 列出solution中的项目

```shell
dotnet sln <SOLUTION_FILE> list

示例：
dotnet sln todo.sln list
```

* 添加项目到solution

```shell
dotnet sln add <PROJECT_NAME>

示例：
dotnet sln add ./todo.csproj
```

* 从solution中移除指定的项目

```shell
dotnet sln remove <PROJECT_NAME>

示例：
dotnet sln remove ./todo.csproj
```

* 把项目与solution放置在同一个目录

```shell
dotnet sln <SOLUTION_NAME> add <PROJECT_NAME> --in-root

示例：
dotnet sln todo.sln add ./todo.csproj --in-root
```

* 添加多个项目到solution

```shell
dotnet sln <SOLUTION_NAME> add <PROJECT_NAME> ...

示例：
dotnet sln todo.sln add ./todo.csproj ./todo2.csproj
```

* 从solution中移除多个项目

```shell
dotnet sln <SOLUTION_NAME> remove <PROJECT_NAME> ...

示例：
dotnet sln todo.sln remove ./todo.csproj ./todo2.csproj
```

* 使用通配符模式添加项目到solution

```shell
dotnet sln <SOLUTION_NAME> add **/*.csproj

示例：
dotnet sln todo.sln add **/*.csproj
```

* 使用通配符模式从solution中移除多个项目

```shell
dotnet sln <SOLUTION_NAME> remove **/*.csproj

示例：
dotnet sln todo.sln remove **/*.csproj
```

* 添加项目到solution时指定solution文件夹的路径

```shell
dotnet sln <SOLUTION_NAME> add <PROJECT_NAME> -s|--solution-folder <PATH>

示例：
dotnet sln todo.sln add ./todo.csproj -s MyProjects
```

### dotnet tool命令

* 全局安装工具在默认位置

```shell
dotnet tool install -g|--global <TOOL_NAME>

示例：
dotnet tool install -g|--global dotnetsay
```

* 局部安装工具在当前目录

```shell
dotnet tool install <TOOL_NAME>

示例：
dotnet tool install dotnetsay
```

* 全局安装工具且指定安装的目录

```shell
dotnet tool install <TOOL_NAME> --tool-path <PATH_NAME>

示例：
dotnet tool install dotnetsay --tool-path ./bin
```

* 列出全局安装的工具

```shell
dotnet tool list -g|--global
```

* 列出指定位置的全局安装的工具

```shell
dotnet tool list --tool-path ./bin
```

* 列出在当前目录安装的局部工具

```shell
dotnet tool list
```

* 指定工具的包id列出全局安装的工具

```shell
dotnet tool list -g dotnetsay
```

* 指定工具的包id列出当前目录安装的局部工具

```shell
dotnet tool list dotnetsay
```

* 运行当前目录的局部工具

```shell
dotnet tool run <COMMAND_NAME>

示例：
dotnet tool run dotnetsay
```

* 指定包名搜索在NuGet上的工具

```shell
dotnet tool search <TOOL_NAME>

示例：
dotnet tool search format
```

* 卸载全局安装的工具

```shell
dotnet tool uninstall -g <TOOL_NAME>

示例：
dotnet tool uninstall -g dotnetsay
```

* 卸载指定路径的全局安装的工具

```shell
dotnet tool uninstall <TOOL_NAME> --tool-path <PATH_NAME>

示例：
dotnet tool uninstall dotnetsay --tool-path ./bin
```

* 卸载在当前目录局部安装的工具

```shell
dotnet tool uninstall <TOOL_NAME>

示例：
dotnet tool uninstall dotnetsay
```

* 更新全局安装的工具

```shell
dotnet tool update -g <TOOL_NAME>

示例：
dotnet tool update -g dotnetsay
```

* 更新指定路径的全局安装的工具

```shell
dotnet tool update <TOOL_NAME> --tool-path <PATH_NAME>

示例：
dotnet tool update dotnetsay --tool-path ./bin
```

* 更新当前目录局部安装的工具

```shell
dotnet tool update <TOOL_NAME>

示例：
dotnet tool update dotnetsay
```

### 监听文件变更的命令

重启或热重载应用

默认情况下监听器会监听以下文件：

1. **/*.cs

2. *.csproj

3. **/*.resx

4. wwwroot/**

* 列出所有监听器发现的文件

```shell
dotnet watch --list
```

* 非交互式模式启动监听器

当文件被粗鲁的编辑后，监听器会自动重启应用

```shell
dotnet watch --non-interactive
```

* 指定监听的项目

```shell
dotnet watch --project <PATH>


示例：dotnet watch --project ./abc
```

* 监听时显示详细的输出信息

```shell
dotnet watch -v|--verbose
```

* 通过监听器来运行其他命令

```shell
dotnet watch <OTHER_COMMAND>

示例：
dotnet watch test
```

* 配置监听器包含指定的文件

```xml
<ItemGroup>
  <Watch Include="**\*.js" />
</ItemGroup>
```

* 配置监听器排除指定的文件

```xml
<ItemGroup>
  <Watch Exclude="node_modules\**\*;**\*.js.map;obj\**\*;bin\**\*" />
</ItemGroup>
```

* 使用 `Watch=false` 配置监听器忽略在 `Compile` 或 `EmbeddedResource` 组中的文件

```xml
<ItemGroup>
  <Compile Update="Generated.cs" Watch="false" />
  <EmbeddedResource Update="Strings.resx" Watch="false" />
</ItemGroup>
```

* 使用 `Watch=false` 配置监听器忽略引用的项目

```xml
<ItemGroup>
  <ProjectReference Include="..\ClassLibrary1\ClassLibrary1.csproj" Watch="false" />
</ItemGroup>
```


