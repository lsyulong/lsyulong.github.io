---
layout: mypost
title: gravitino window开发环境搭建
categories: [gravitino]
---

### 目的
关于为什么要写这么一篇文章，因为目前Gravitino使用gradle进行编译构建的，所以不支持使用window系统进行编译，故而自己萌生了要写一篇文章来记录一下自己
搭建开发环境过程中遇到的一些问题，期间自己也尝试使用了wsl2+vscode组合进行开发环境搭建，但是由于个人不太习惯使用vscode进行java开发，所以暂时放弃，使用其他方式，同时希望也可以帮助到有需要的人。

### Apache Gravitino介绍
Gravitino是Datastrato自研的下一代多云原生的高性能联邦式元数据管理服务，可以管理不同来源、类型和区域的元数据，支持 Hive，Iceberg，MySQL，Fileset，Messaging 等类型的数据目录（不断增加中），为用户提供 Data 和 AI 资产的统一元数据访问，后将其开源并捐赠给Apache开源基金会。

### 安装依赖环境
##### 1、NodeJS安装
Gravitino前端使用NodeJS的进行进行构建，需要安装NodeJS环境

windows中node安装参照官网安装即可，目前笔者使用``V22.11.0LTS``，官网下载地址``https://nodejs.org/en/download/package-manager``
具体请参照官网中命令示意，同时附个人安装截图。
![官网示例](..\posts\2024\11\23\2.png)
![笔者安装截图](..\posts\2024\11\23\1.png)

##### 2、python环境安装
Gravitino支持使用python连接，需要配置python环境，python3.X均可，window的python环境安装可自行上网查找

##### 3、JAVA环境安装
目前gravitino会使用不同的JDK版本进行编译构建发布，所以我们需要本地安装多版本JDK，具体安装方式可自行上网查找，
Gravitino支持``JDK8、11、17``
java安装成功后可以使用``java -version``查看
![java安装验证](..\posts\2024\11\23\4.png)

##### 4、gradle安装
笔者本地环境配置的``gradle8.9``，本地安装成功后使用``gradle -v`` 命令查看是否安装成功
![gradle安装验证](..\posts\2024\11\23\3.png)

##### 5、docker安装
gravitino的集成测试使用docker镜像来测试的，window可以安装docker desktop，下载地址``https://www.docker.com/products/docker-desktop/``

##### 6、gravitino编译
参照官网教程，笔者自己在一开始编译的gravitino的时候总是不成功，编译到web前端报如下错误
```
Build gravitino FAILURE reason:
Execution failed for task ':web:web:prettierCheck':
org.gradle.process.internal.ExecException: Process 'command 'C:\github_pro\gravitino_new\web\web\.gradle\pnpm\pnpm-v9.x\pnpm.cmd'' finished with non-zero exit value 1
at org.gradle.process.internal.DefaultExecHandle$ExecResultImpl.assertNormalExitValue(DefaultExecHandle.java:415)
at org.gradle.process.internal.DefaultExecAction.execute(DefaultExecAction.java:38)
at org.gradle.process.internal.DefaultExecActionFactory.exec(DefaultExecActionFactory.java:202)
at org.gradle.process.internal.DefaultExecOperations.exec(DefaultExecOperations.java:37)
at com.github.gradle.node.util.DefaultProjectApiHelper.exec(ProjectApiHelper.kt:68)
at com.github.gradle.node.exec.ExecRunner.execute(ExecRunner.kt:46)
at com.github.gradle.node.pnpm.exec.PnpmExecRunner.executeCommand(PnpmExecRunner.kt:38)
at com.github.gradle.node.pnpm.exec.PnpmExecRunner.executePnpmCommand(PnpmExecRunner.kt:26)
at com.github.gradle.node.pnpm.task.PnpmTask.exec(PnpmTask.kt:76)
at org.gradle.internal.reflect.JavaMethod.invoke(JavaMethod.java:125)
at org.gradle.api.internal.project.taskfactory.StandardTaskAction.doExecute(StandardTaskAction.java:58)
at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:51)
at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:29)
at org.gradle.api.internal.tasks.execution.TaskExecution$3.run(TaskExecution.java:248)
at org.gradle.internal.operations.DefaultBuildOperationRunner$1.execute(DefaultBuildOperationRunner.java:29)
at org.gradle.internal.operations.DefaultBuildOperationRunner$1.execute(DefaultBuildOperationRunner.java:26)
at org.gradle.internal.operations.DefaultBuildOperationRunner$2.execute(DefaultBuildOperationRunner.java:66)
at org.gradle.internal.operations.DefaultBuildOperationRunner$2.execute(DefaultBuildOperationRunner.java:59)
at org.gradle.internal.operations.DefaultBuildOperationRunner.execute(DefaultBuildOperationRunner.java:157)
at org.gradle.internal.operations.DefaultBuildOperationRunner.execute(DefaultBuildOperationRunner.java:59)
at org.gradle.internal.operations.DefaultBuildOperationRunner.run(DefaultBuildOperationRunner.java:47)
at org.gradle.internal.operations.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:68)
at org.gradle.api.internal.tasks.execution.TaskExecution.executeAction(TaskExecution.java:233)
at org.gradle.api.internal.tasks.execution.TaskExecution.executeActions(TaskExecution.java:216)
at org.gradle.api.internal.tasks.execution.TaskExecution.executeWithPreviousOutputFiles(TaskExecution.java:199)
at org.gradle.api.internal.tasks.execution.TaskExecution.execute(TaskExecution.java:166)
at org.gradle.internal.execution.steps.ExecuteStep.executeInternal(ExecuteStep.java:105)
at org.gradle.internal.execution.steps.ExecuteStep.access$000(ExecuteStep.java:44)
at org.gradle.internal.execution.steps.ExecuteStep$1.call(ExecuteStep.java:59)
at org.gradle.internal.execution.steps.ExecuteStep$1.call(ExecuteStep.java:56)
at org.gradle.internal.operations.DefaultBuildOperationRunner$CallableBuildOperationWorker.execute(DefaultBuildOperationRunner.java:204)
at org.gradle.internal.operations.DefaultBuildOperationRunner$CallableBuildOperationWorker.execute(DefaultBuildOperationRunner.java:199)
at org.gradle.internal.operations.DefaultBuildOperationRunner$2.execute(DefaultBuildOperationRunner.java:66)
at org.gradle.internal.operations.DefaultBuildOperationRunner$2.execute(DefaultBuildOperationRunner.java:59)
at org.gradle.internal.operations.DefaultBuildOperationRunner.execute(DefaultBuildOperationRunner.java:157)
at org.gradle.internal.operations.DefaultBuildOperationRunner.execute(DefaultBuildOperationRunner.java:59)
at org.gradle.internal.operations.DefaultBuildOperationRunner.call(DefaultBuildOperationRunner.java:53)
at org.gradle.internal.operations.DefaultBuildOperationExecutor.call(DefaultBuildOperationExecutor.java:73)
at org.gradle.internal.execution.steps.ExecuteStep.execute(ExecuteStep.java:56)
at org.gradle.internal.execution.steps.ExecuteStep.execute(ExecuteStep.java:44)
at org.gradle.internal.execution.steps.RemovePreviousOutputsStep.execute(RemovePreviousOutputsStep.java:67)
at org.gradle.internal.execution.steps.RemovePreviousOutputsStep.execute(RemovePreviousOutputsStep.java:37)
at org.gradle.internal.execution.steps.CancelExecutionStep.execute(CancelExecutionStep.java:41)
at org.gradle.internal.execution.steps.TimeoutStep.executeWithoutTimeout(TimeoutStep.java:74)
at org.gradle.internal.execution.steps.TimeoutStep.execute(TimeoutStep.java:55)
at org.gradle.internal.execution.steps.CreateOutputsStep.execute(CreateOutputsStep.java:50)
at org.gradle.internal.execution.steps.CreateOutputsStep.execute(CreateOutputsStep.java:28)
at org.gradle.internal.execution.steps.CaptureStateAfterExecutionStep.executeDelegateBroadcastingChanges(CaptureStateAfterExecutionStep.java:100)
at org.gradle.internal.execution.steps.CaptureStateAfterExecutionStep.execute(CaptureStateAfterExecutionStep.java:72)
at org.gradle.internal.execution.steps.CaptureStateAfterExecutionStep.execute(CaptureStateAfterExecutionStep.java:50)
at org.gradle.internal.execution.steps.ResolveInputChangesStep.execute(ResolveInputChangesStep.java:40)
at org.gradle.internal.execution.steps.ResolveInputChangesStep.execute(ResolveInputChangesStep.java:29)
at org.gradle.internal.execution.steps.BuildCacheStep.executeWithoutCache(BuildCacheStep.java:179)
at org.gradle.internal.execution.steps.BuildCacheStep.lambda$execute$1(BuildCacheStep.java:70)
at org.gradle.internal.Either$Right.fold(Either.java:175)
at org.gradle.internal.execution.caching.CachingState.fold(CachingState.java:59)
at org.gradle.internal.execution.steps.BuildCacheStep.execute(BuildCacheStep.java:68)
at org.gradle.internal.execution.steps.BuildCacheStep.execute(BuildCacheStep.java:46)
at org.gradle.internal.execution.steps.StoreExecutionStateStep.execute(StoreExecutionStateStep.java:36)
at org.gradle.internal.execution.steps.StoreExecutionStateStep.execute(StoreExecutionStateStep.java:25)
at org.gradle.internal.execution.steps.RecordOutputsStep.execute(RecordOutputsStep.java:36)
at org.gradle.internal.execution.steps.RecordOutputsStep.execute(RecordOutputsStep.java:22)
at org.gradle.internal.execution.steps.SkipUpToDateStep.executeBecause(SkipUpToDateStep.java:91)
at org.gradle.internal.execution.steps.SkipUpToDateStep.lambda$execute$2(SkipUpToDateStep.java:55)
at org.gradle.internal.execution.steps.SkipUpToDateStep.execute(SkipUpToDateStep.java:55)
at org.gradle.internal.execution.steps.SkipUpToDateStep.execute(SkipUpToDateStep.java:37)
at org.gradle.internal.execution.steps.ResolveChangesStep.execute(ResolveChangesStep.java:65)
at org.gradle.internal.execution.steps.ResolveChangesStep.execute(ResolveChangesStep.java:36)
at org.gradle.internal.execution.steps.legacy.MarkSnapshottingInputsFinishedStep.execute(MarkSnapshottingInputsFinishedStep.java:37)
at org.gradle.internal.execution.steps.legacy.MarkSnapshottingInputsFinishedStep.execute(MarkSnapshottingInputsFinishedStep.java:27)
at org.gradle.internal.execution.steps.ResolveCachingStateStep.execute(ResolveCachingStateStep.java:77)
at org.gradle.internal.execution.steps.ResolveCachingStateStep.execute(ResolveCachingStateStep.java:38)
at org.gradle.internal.execution.steps.ValidateStep.execute(ValidateStep.java:94)
at org.gradle.internal.execution.steps.ValidateStep.execute(ValidateStep.java:49)
at org.gradle.internal.execution.steps.CaptureStateBeforeExecutionStep.execute(CaptureStateBeforeExecutionStep.java:71)
at org.gradle.internal.execution.steps.CaptureStateBeforeExecutionStep.execute(CaptureStateBeforeExecutionStep.java:45)
at org.gradle.internal.execution.steps.SkipEmptyWorkStep.executeWithNonEmptySources(SkipEmptyWorkStep.java:177)
at org.gradle.internal.execution.steps.SkipEmptyWorkStep.execute(SkipEmptyWorkStep.java:81)
at org.gradle.internal.execution.steps.SkipEmptyWorkStep.execute(SkipEmptyWorkStep.java:53)
at org.gradle.internal.execution.steps.RemoveUntrackedExecutionStateStep.execute(RemoveUntrackedExecutionStateStep.java:32)
at org.gradle.internal.execution.steps.RemoveUntrackedExecutionStateStep.execute(RemoveUntrackedExecutionStateStep.java:21)
at org.gradle.internal.execution.steps.legacy.MarkSnapshottingInputsStartedStep.execute(MarkSnapshottingInputsStartedStep.java:38)
at org.gradle.internal.execution.steps.LoadPreviousExecutionStateStep.execute(LoadPreviousExecutionStateStep.java:36)
at org.gradle.internal.execution.steps.LoadPreviousExecutionStateStep.execute(LoadPreviousExecutionStateStep.java:23)
at org.gradle.internal.execution.steps.CleanupStaleOutputsStep.execute(CleanupStaleOutputsStep.java:75)
at org.gradle.internal.execution.steps.CleanupStaleOutputsStep.execute(CleanupStaleOutputsStep.java:41)
at org.gradle.internal.execution.steps.AssignWorkspaceStep.lambda$e
```
由于个人能力水平有限，一直没找到对应的解决办法，后决定先编译web前端，再编译整体项目
地址链接参照 ``https://github.com/apache/gravitino/tree/main/web/web`` 当编译完成且格式化代码完成后，退出web目录，进入当前项目的根目录下
，开始编译整个项目
执行如下命令开始编译``./gradlew clean build -x test ``
![](..\posts\2024\11\23\5.png)
![](..\posts\2024\11\23\6.png)
![](..\posts\2024\11\23\7.png)

至此，编译成功，文章后续还会补充更新，同时欢迎各位大佬指教

最后，还是建议各位小伙伴还是要有个梯子，科学上网，个别时候如果国内镜像加速不给力，这时候梯子就排上用场了，懂得都懂！！！
