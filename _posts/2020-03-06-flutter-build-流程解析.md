---
layout:     post
title:      flutter build 流程解析
subtitle:   如何切换Xcode命令行工具
date:       2020-03-06
author:     Cosin鱼弦
header-img: img/post-bg-flutter.png
catalog: true
tags:
    - Flutter
    - 源码
    - iOS
---
[TOC]

以编译一个aot模式的iOS版本的命令为例, 我们可以执行以下命令.

```shell
flutter build aot --target-platform=ios --release
```

那么这个命令内部做了哪些事情, 这篇文章主要跟踪这条命令来探索下Flutter的编译流程.

**NOTE:**
- 下文`${FLUTTER_ROOT}`是你的Flutter的安装目录.
- 系统信息

    > ProductName:	Mac OS X
    > ProductVersion:	10.15.1
    > BuildVersion:	19B88
    
- Flutter 版本信息

    > Flutter 1.12.13+hotfix.8 • channel stable • https://github.com/flutter/flutter.git
    > Framework • revision 0b8abb4724 (3 weeks ago) • 2020-02-11 11:44:36 -0800
    > Engine • revision e1e6ced81d
    > Tools • Dart 2.7.0


# 入口 flutter
**${FLUTTER_ROOT}/bin/flutter**脚本入口, 这是一个shell脚本, 核心代码就下面这一句.

```shell
"$DART" --packages="$FLUTTER_TOOLS_DIR/.packages" $FLUTTER_TOOL_ARGS "$SNAPSHOT_PATH" "$@"

# 输出结果如下, 其中${FLUTTER_ROOT}是你的flutter安装目录.
# ${FLUTTER_ROOT}/bin/cache/dart-sdk/bin/dart --packages=${FLUTTER_ROOT}/packages/flutter_tools/.packages  ${FLUTTER_ROOT}/bin/cache/flutter_tools.snapshot build aot --target-platform=ios --release --verbose
```

- **$DART:** dart可执行文件, 用于启动dart VM.
- **$FLUTTER_TOOL_ARGS:**  工具debug参数,可以设置 FLUTTER_TOOL_ARGS="--enable-asserts --observe=${port}".
- **$SNAPSHOT_PATH:** 运行的snapshot文件([什么是snapshot](https://github.com/dart-lang/sdk/wiki/Snapshots)), 这里是 `${FLUTTER_ROOT}/bin/cache/flutter_tools.snapshot`
- **$@:** 透传用户参数

# flutter_tools
由上面的分析可以看出接下来的关键代码在`flutter_tools`这里.

## flutter_tools.dart

我们可以在 `${FLUTTER_ROOT}/packages/flutter_tools/bin/flutter_tools.dart` 找到源码
```dart
import 'package:flutter_tools/executable.dart' as executable;
void main(List<String> args) {
  executable.main(args);
}
```
## executable.dart
这里调用了 `executable` 的main方法, 该文件在 `${FLUTTER_ROOT}/packages/flutter_tools/lib/executable.dart` 可以找到.
```dart
import 'runner.dart' as runner;
import ...;
Future<void> main(List<String> args) async {
 ... 忽略非关键代码 ...
  await runner.run(args, <FlutterCommand>[...忽略具体参数...]);
}
```
## runner.dart
接着这个方法包装了下参数执行`runner`的run方法, 该方法是 Flutter 命令行运行解析的通用方法, 在 `${FLUTTER_ROOT}/packages/flutter_tools/lib/runner.dart` 可以查看源码.
```dart
Future<int> run(
  List<String> args,
  List<FlutterCommand> commands, {
  bool muteCommandLogging = false,
  bool verbose = false,
  bool verboseHelp = false,
  bool reportCrashes,
  String flutterVersion,
  Map<Type, Generator> overrides,
}) {
    ....
    final FlutterCommandRunner runner = FlutterCommandRunner(verboseHelp: verboseHelp);
  commands.forEach(runner.addCommand);
    ...
    return runInContext<int>(() async {
    ...
    return await runZoned<Future<int>>(() async {
      try {
        await runner.run(args);
        return await _exit(0);
      } catch (error, stackTrace) {
        ...
      }
    }, onError: (Object error, StackTrace stackTrace) async {
      ...
    });
  }, overrides: overrides);
}
```
上面省略了部分干扰代码, 其中 `args` 包含了你输入的命令, 拿文章开始的命令举例这里`args`值为 
```dart
["build", "aot", "--target-platform=ios", "--release", "--verbose"]
```
## flutter_command_runner.dart & command_runner.dart

```dart
class FlutterCommandRunner extend CommandRunner<void> {
  ...
  @override
  ArgResults parse(Iterable<String> args) {
    try {
      ...
      return tryArgsCompletion(args.toList(), argParser);
    } on ArgParserException catch (error) {
      ...
      return null;
    }
  }
  @override
  Future<void> run(Iterable<String> args) {
    if (args.length == 1 && args.first == 'build') {
      args = <String>['build', '-h'];
    }
    return super.run(args);
  }
  ...
}
class CommandRunner<T> {
  ...
  Future<T> runCommand(ArgResults topLevelResults) async {
    ...
    Command command;
    var commandString = executableName;
    ...
    while (commands.isNotEmpty) {
      ...
      // Step into the command.
      argResults = argResults.command;
      command = commands[argResults.name];
      command._globalResults = topLevelResults;
      command._argResults = argResults;
      commands = command._subcommands;
      commandString += " ${argResults.name}";
    }
    ...
    return (await command.run()) as T;
  }
}
```
通过上下文我们可以知道这里的`command`是 `runner.run` 中的commands参数是`FlutterCommand`类型, 比如文章开头 `flutter build` 命令的实现在 `${FLUTTER_ROOT}/packages/flutter_tools/lib/src/commands/build.dart` 的 `BuildCommand` 类中.

## build.dart & flutter_command.dart

查看`BuildCommand`类, 并没有 @override `run` 函数, 通过继承链可以发现该方法在 `FlutterCommand`中, 具体的细节我在代码里备注了.
```dart
abstract class FlutterCommand extends Command<void> {
  @override
  Future<void> run() {
    ...
    return context.run<void>(
      name: 'command',
      overrides: <Type, Generator>{FlutterCommand: () => this},
      body: () async {
        ...
        try {
          commandResult = await verifyThenRunCommand(commandPath);
        } ...
        ....
      },
    );
  }
  @mustCallSuper
  Future<FlutterCommandResult> verifyThenRunCommand(String commandPath) async {
    // 1. 校验
    await validateCommand();
    // 2. 更新flutter cache
    if (shouldUpdateCache) {
      await cache.updateAll(await requiredArtifacts);
    }
    
    if (shouldRunPub) {
      // 3. 验证&下载pubspec.yaml里配置的依赖
      await pub.get(context: PubContext.getVerifyContext(name));
      final FlutterProject project = FlutterProject.current();

      // 4. 生成项目文件, 比如Android下执行 Gradle builds, iOS下执行 Cocoapods以及一些Xcode配置覆盖.
      await project.ensureReadyForPlatformSpecificTooling(checkProjects: true);
    }

    setupApplicationPackages();
    ...
    // 5. 执行 runCommand, 比如`build aot`命令对应的子类BuildAotCommand中的方法
    return await runCommand();
  }

  void setupApplicationPackages() {
    applicationPackages ??= ApplicationPackageStore();
  }
}
```
## build_aot_command.dart & aot.dart

`build aot`命令对应的子类BuildAotCommand中的方法.
```dart
/// Builds AOT snapshots into platform specific library containers.
class BuildAotCommand extends BuildSubCommand with TargetPlatformBasedDevelopmentArtifacts {
  ....
  AotBuilder aotBuilder;
  @override
  Future<FlutterCommandResult> runCommand() async {
    ...
    aotBuilder ??= AotBuilder();
    await aotBuilder.build(
      platform: platform,
      ....
    );
    return null;
  }
}
```

最终会调用 `AotBuilder.build`方法.

```dart
const List<DarwinArch> defaultIOSArchs = <DarwinArch>[
  DarwinArch.arm64,
];
/// Builds AOT snapshots given a platform, build mode and a path to a Dart
/// library.
class AotBuilder {
  Future<void> build({....
  }) async {
    
    if (Android打包是否支持assemble) {
      await _buildWithAssemble(....);
      return;
    }
    ...
    try {
      final AOTSnapshotter snapshotter = AOTSnapshotter(reportTimings: reportTimings);

      // 1. Compile to kernel.
      final String kernelOut = await snapshotter.compileKernel(
        platform: platform,
        buildMode: buildMode,
        mainPath: mainDartFile,
        packagesPath: PackageMap.globalPackagesPath,
        trackWidgetCreation: false,
        outputPath: outputPath,
        extraFrontEndOptions: extraFrontEndOptions,
        dartDefines: dartDefines,
      );
      if (kernelOut == null) {
        throwToolExit('Compiler terminated unexpectedly.');
        return;
      }

      // 2. Build AOT snapshot.
      if (platform == TargetPlatform.ios) { // iOS
        // 2-1. Determine which iOS architectures to build for.
        final Map<DarwinArch, String> iosBuilds = <DarwinArch, String>{};
        for (DarwinArch arch in iosBuildArchs) {
          iosBuilds[arch] = fs.path.join(outputPath, getNameForDarwinArch(arch));
        }

        // 2-2. Generate AOT snapshot and compile to arch-specific App.framework.
        final Map<DarwinArch, Future<int>> exitCodes = <DarwinArch, Future<int>>{};
        iosBuilds.forEach((DarwinArch iosArch, String outputPath) {
          exitCodes[iosArch] = snapshotter.build(
            platform: platform,
            darwinArch: iosArch,
            buildMode: buildMode,
            mainPath: kernelOut,
            packagesPath: PackageMap.globalPackagesPath,
            outputPath: outputPath,
            extraGenSnapshotOptions: extraGenSnapshotOptions,
            bitcode: bitcode,
            quiet: quiet,
          ).then<int>((int buildExitCode) {
            return buildExitCode;
          });
        });

        // 2-3. Merge arch-specific App.frameworks into a multi-arch App.framework.
        if ((await Future.wait<int>(exitCodes.values)).every((int buildExitCode) => buildExitCode == 0)) {
          final Iterable<String> dylibs = iosBuilds.values.map<String>(
              (String outputDir) => fs.path.join(outputDir, 'App.framework', 'App'));
          fs.directory(fs.path.join(outputPath, 'App.framework'))..createSync();
          await processUtils.run(
            <String>[
              'lipo',
              ...dylibs,
              '-create',
              '-output', fs.path.join(outputPath, 'App.framework', 'App'),
            ],
            throwOnError: true,
          );
        } else {
          ...
        }
      } else { // Android
        // Android AOT snapshot.
        final int snapshotExitCode = await snapshotter.build(
          platform: platform,
          buildMode: buildMode,
          mainPath: kernelOut,
          packagesPath: PackageMap.globalPackagesPath,
          outputPath: outputPath,
          extraGenSnapshotOptions: extraGenSnapshotOptions,
          bitcode: false,
        );
        ....
      }
    } on ProcessException catch (error) {
      ...
      return;
    }
    status?.stop();
    .....
    return;
  }
}
```

- 针对Android机器判断是否支持 assembly, 并根据结果是否执行assemble操作.
- 编译成kernel中间文件(具体见下面[Compile To Kernel](#toc_9)),这里的 kernel 文件相当于 LLVM IR.
    
    ```shell
    ${FLUTTER_ROOT}/bin/cache/artifacts/engine/ios-release/gen_snapshot_arm64 --causal_async_stacks --deterministic --snapshot_kind=app-aot-assembly --assembly=build/aot/arm64/snapshot_assembly.S build/aot/app.dill
    
    ```
- 根据不同平台和内核执行 `snapshotter.build` 编译, iOS目前默认只支持arm64.

    ```shell
    xcrun cc -arch arm64 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS13.2.sdk -c build/aot/arm64/snapshot_assembly.S -o build/aot/arm64/snapshot_assembly.o
    
    xcrun clang -arch arm64 -miphoneos-version-min=8.0 -dynamiclib -Xlinker -rpath -Xlinker @executable_path/Frameworks -Xlinker -rpath -Xlinker @loader_path/Frameworks -install_name @rpath/App.framework/App -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS13.2.sdk -o build/aot/arm64/App.framework/App build/aot/arm64/snapshot_assembly.o
    ```
- 如果是iOS还会执行`lipo`命令将不同架构的App合成一个到App.framework中(不过目前就arm64一种).

    ```shell
    lipo build/aot/arm64/App.framework/App -create -output build/aot/App.framework/App
    ```

# Compile To Kernel
上面还有一个地方没具体展开的是编译到Kernel的过程. 还是以文章开头命令为例, aot编译模式对应调用的是 `AOTSnapshotter.compileKernel`. 
```dart
class AOTSnapshotter {
    Future<String> compileKernel({
    @required TargetPlatform platform,
    @required BuildMode buildMode,
    @required String mainPath,
    @required String packagesPath,
    @required String outputPath,
    @required bool trackWidgetCreation,
    @required List<String> dartDefines,
    List<String> extraFrontEndOptions = const <String>[],
  }) async {
    ...
    final KernelCompiler kernelCompiler = await kernelCompilerFactory.create(flutterProject);
    final CompilerOutput compilerOutput =
      await _timedStep('frontend(CompileTime)', 'aot-kernel',
        () => kernelCompiler.compile(...));

    // Write path to frontend_server, since things need to be re-generated when that changes.
    final String frontendPath = artifacts.getArtifactPath(Artifact.frontendServerSnapshotForEngineDartSdk);
    fs.directory(outputPath).childFile('frontend_server.d').writeAsStringSync('frontend_server.d: $frontendPath\n');

    return compilerOutput?.outputFilename;
  }
}
```
该方法内部调用了 `KernelCompiler.compile`, 并将 `frontend_server.dart.snapshot` 的路径写入到 build/aot/frontend_server.d 文件中.
```dart
class KernelCompiler {
  const KernelCompiler();

  Future<CompilerOutput> compile({....}) async {
    ....
    final List<String> command = <String>[
      engineDartPath,
      frontendServer,
      '--sdk-root',
      sdkRoot,
      '--target=$targetModel',
      '-Ddart.developer.causal_async_stacks=$causalAsyncStacks',
      for (Object dartDefine in dartDefines)
        '-D$dartDefine',
      ..._buildModeOptions(buildMode),
      if (trackWidgetCreation) '--track-widget-creation',
      if (!linkPlatformKernelIn) '--no-link-platform',
      if (aot) ...<String>[
        '--aot',
        '--tfa',
      ],
      if (packagesPath != null) ...<String>[
        '--packages',
        packagesPath,
      ],
      if (outputFilePath != null) ...<String>[
        '--output-dill',
        outputFilePath,
      ],
      if (depFilePath != null && (fileSystemRoots == null || fileSystemRoots.isEmpty)) ...<String>[
        '--depfile',
        depFilePath,
      ],
      if (fileSystemRoots != null)
        for (String root in fileSystemRoots) ...<String>[
          '--filesystem-root',
          root,
        ],
      if (fileSystemScheme != null) ...<String>[
        '--filesystem-scheme',
        fileSystemScheme,
      ],
      if (initializeFromDill != null) ...<String>[
        '--initialize-from-dill',
        initializeFromDill,
      ],
      if (platformDill != null) ...<String>[
        '--platform',
        platformDill,
      ],
      ...?extraFrontEndOptions,
      mainUri?.toString() ?? mainPath,
    ];

    final Process server = await processManager
      .start(command)
      .catchError((dynamic error, StackTrace stack) {
        printError('Failed to start frontend server $error, $stack');
      });

    .....
    final int exitCode = await server.exitCode;
    if (exitCode == 0) {
      return _stdoutHandler.compilerOutput.future;
    }
    return null;
  }
}
```
可以看到这里拼装了一条如下的命令并执行, 通过dart虚拟机启动frontend_server.dart.snapshot，将dart代码编译输出 `app.dill` 和 `kernel_compile.d` 文件.(TODO: .dill 文件格式)

```shell
${FLUTTER_ROOT}/bin/cache/dart-sdk/bin/dart \
${FLUTTER_ROOT}/bin/cache/artifacts/engine/darwin-x64/frontend_server.dart.snapshot \
--sdk-root ${FLUTTER_ROOT}/bin/cache/artifacts/engine/common/flutter_patched_sdk_product/ \
--target=flutter -Ddart.developer.causal_async_stacks=true -Ddart.vm.profile=false -Ddart.vm.product=true \
--bytecode-options=source-positions \
--aot \
--tfa \
--packages .packages \
--output-dill build/aot/app.dill \
--depfile build/aot/kernel_compile.d \
path/to/testproject/main.dart 
```

# frontend_server
介于frontend_server.dart.snapshot是二进制文件, 没法查看具体代码, 通过路径分析找到了源码位置在 [flutter/engine](https://github.com/flutter/engine) 项目的 flutter_frontend_server目录下. 为了验证猜想我编译了iOS平台下的引擎, 将上面命令行snapshot替换成源码调用, 也方便后面阅读调试.

```shell
${FLUTTER_ROOT}/bin/cache/dart-sdk/bin/dart \
path/to/engine/src/third_party/dart/pkg/frontend_server/bin/frontend_server_starter.dart \
##⚠️注意这里的路径不能使用 $FLUTTER_HOME 下的
--sdk-root path/to/engine/src/out/ios_debug_unopt/flutter_patched_sdk \
--target=flutter -Ddart.developer.causal_async_stacks=true -Ddart.vm.profile=false -Ddart.vm.product=true \
--bytecode-options=source-positions \
--aot \
--tfa \
--packages .packages \
--output-dill build/aot/app.dill \
--depfile build/aot/kernel_compile.d \
path/to/testproject/main.dart
```
查看入口函数直接调用了 `frontend_server.dart` 中的 `starter` 函数.
## frontend_server.dart

```dart
Future<int> starter(...) async {
  ...
  compiler ??= FrontendCompiler(output,
      printerFactory: binaryPrinterFactory,
      unsafePackageSerialization: options["unsafe-package-serialization"],
      incrementalSerialization: options["incremental-serialization"]);
  print("${options.rest}");
  if (options.rest.isNotEmpty) {
    return await compiler.compile(options.rest[0], options,
            generator: generator)
        ? 0
        : 254;
  }
  ...
}
```
这个函数主要解析了下命令参数, 并创建 `FrontendCompiler` 实例执行 compile 操作.
## FrontendCompiler
这个方法主要做了:
1. 转发到`compileToKernel`方法编译出将 dart 源码转化成 Component 对象;
2. 对 components 执行 transform.
3. 输出 .dill 和 .d 文件(如果入需要生成bytecode是在write里不是第1步);

```dart
class FrontendCompiler implements CompilerInterface {
  @override
  Future<bool> compile(
    String entryPoint,
    ArgResults options, {
    IncrementalCompiler generator,
  }) async {
    ...
    if (options['aot']) {
      // 检查 aot 模式下不适用的option
      ...
    }

    // Initialize additional supported kernel targets.
    ...
    if (options['incremental']) {
      ...
    } else {
      ....
      // No bytecode at this step. Bytecode is generated later in _writePackage.
      results = await _runWithPrintRedirection(() => compileToKernel(
          _mainSource, compilerOptions,
          includePlatform: options['link-platform'],
          aot: options['aot'],
          useGlobalTypeFlowAnalysis: options['tfa'],
          environmentDefines: environmentDefines,
          enableAsserts: options['enable-asserts'],
          useProtobufTreeShaker: options['protobuf-tree-shaker']));
    }
    if (results.component != null) {
      // 
      transformer?.transform(results.component);

      if (_compilerOptions.target.name == 'dartdevc') {
        await writeJavascriptBundle(
            results, _kernelBinaryFilename, options['filesystem-scheme']);
      } else {
        // 输出 .dill 文件
        await writeDillFile(results, _kernelBinaryFilename,
            filterExternal: importDill != null,
            incrementalSerializer: incrementalSerializer);
      }

      _outputStream.writeln(boundaryKey);
      await _outputDependenciesDelta(results.compiledSources);
      _outputStream
          .writeln('$boundaryKey $_kernelBinaryFilename ${errors.length}');
      final String depfile = options['depfile'];
      if (depfile != null) {
        // 输出 .d 文件
        await writeDepfile(compilerOptions.fileSystem, results.compiledSources,
            _kernelBinaryFilename, depfile);
      }

      _kernelBinaryFilename = _kernelBinaryFilenameIncremental;
    } else
      _outputStream.writeln(boundaryKey);
    return errors.isEmpty;
  }
}
```
### compileToKernel
这一步可以拆分成一下几个主要操作:
1. 调用 `kernelForProgram` 函数将dart源码解析成 `Component` 对象;
2. 运行全局转化器;
3. 生成字节码, 不过这里genBytecode默认是false, 并不会执行;

```dart
Future<KernelCompilationResults> compileToKernel(
    Uri source, CompilerOptions options,
    {bool includePlatform: false,
    bool aot: false,
    bool useGlobalTypeFlowAnalysis: false,
    Map<String, String> environmentDefines,
    bool enableAsserts: true,
    bool genBytecode: false,
    BytecodeOptions bytecodeOptions,
    bool dropAST: false,
    bool useProtobufTreeShaker: false}) async {
  // Replace error handler to detect if there are compilation errors.
  final errorDetector =
      new ErrorDetector(previousErrorHandler: options.onDiagnostic);
  options.onDiagnostic = errorDetector;

  options.environmentDefines =
      options.target.updateEnvironmentDefines(environmentDefines);
  // 1. 将dart源码解析成 component 对象.
  CompilerResult compilerResult = await kernelForProgram(source, options);
  Component component = compilerResult?.component;
  final compiledSources = component?.uriToSource?.keys;

  Set<Library> loadedLibraries = createLoadedLibrariesSet(
      compilerResult?.loadedComponents, compilerResult?.sdkComponent,
      includePlatform: includePlatform);


  // Run global transformations only if component is correct. 2.运行全局转化器
  if (aot && component != null) {
    await runGlobalTransformations(
        options.target,
        component,
        useGlobalTypeFlowAnalysis,
        enableAsserts,
        useProtobufTreeShaker,
        errorDetector);
  }
  // 3. 生成字节码, 这里genBytecode默认是false
  if (genBytecode && !errorDetector.hasCompilationErrors && component != null) {
    List<Library> libraries = new List<Library>();
    for (Library library in component.libraries) {
      if (loadedLibraries.contains(library)) continue;
      libraries.add(library);
    }
    // 运行编译后端
    await runWithFrontEndCompilerContext(source, options, component, () {
      generateBytecode(component,
          libraries: libraries,
          hierarchy: compilerResult.classHierarchy,
          coreTypes: compilerResult.coreTypes,
          options: bytecodeOptions);
    });

    if (dropAST) {
      component = createFreshComponentWithBytecode(component);
    }
  }

  // Restore error handler (in case 'options' are reused).
  options.onDiagnostic = errorDetector.previousErrorHandler;

  return new KernelCompilationResults(...)
}

void _writePackage(KernelCompilationResults result, String package,
      List<Library> libraries, IOSink sink) {
    final canCache = libraries.isNotEmpty &&
        _compilerOptions.bytecode &&
        errors.isEmpty &&
        package != "main";

    if (canCache) {
      var cachedBytes = BinaryCacheMetadataRepository.lookup(libraries.first);
      if (cachedBytes != null) {
        sink.add(cachedBytes);
        return;
      }
    }

    Component partComponent = result.component;
    if (_compilerOptions.bytecode && errors.isEmpty) {
      final List<Library> librariesFiltered = new List<Library>();
      final Set<Library> loadedLibraries = result.loadedLibraries;
      for (Library library in libraries) {
        if (loadedLibraries.contains(library)) continue;
        librariesFiltered.add(library);
      }

      generateBytecode(partComponent,
          options: _bytecodeOptions,
          libraries: librariesFiltered,
          coreTypes: _generator?.getCoreTypes(),
          hierarchy: _generator?.getClassHierarchy());

      if (_options['drop-ast']) {
        partComponent = createFreshComponentWithBytecode(partComponent);
      }
    }

    final byteSink = ByteSink();
    final BinaryPrinter printer = BinaryPrinter(byteSink,
        libraryFilter: (lib) =>
            packageFor(lib, result.loadedLibraries) == package);
    printer.writeComponentFile(partComponent);

    final bytes = byteSink.builder.takeBytes();
    sink.add(bytes);
    if (canCache) {
      BinaryCacheMetadataRepository.insert(libraries.first, bytes);
    }
  }
```

这里最主要的也就只是通过 `kernelForProgram` 将dart源码转化成 Component 的过程, 
该方法经过一系列调用最终会进入 `generateKernelInternal` 函数中.
下图是具体的调用栈.
![调用栈](https://ftp.bmp.ovh/imgs/2020/03/b362c1825045ac1a.png)

#### kernelForProgram && generateKernelInternal
```dart

Future<CompilerResult> generateKernelInternal({...}) async {
  ...
  return withCrashReporting<CompilerResult>(() async {
    ...
    DillTarget dillTarget =
        new DillTarget(options.ticker, uriTranslator, options.target);
    ...
    KernelTarget kernelTarget =
        new KernelTarget(fs, false, dillTarget, uriTranslator);
    ......
    Component component;
    if (buildComponent) {
      component = await kernelTarget.buildComponent(verify: options.verify);
      ...
    }
    return new InternalCompilerResult(...);
}

```

#### KernelTarget.buildComponent
用来构建 component 的 kernel表示.
```dart
@override
  Future<Component> buildComponent({bool verify: false}) async {
    if (loader.first == null) return null;
    return withCrashReporting<Component>(() async {
      // 对源码进行词法分析&语法分析, 并生成 AST.
      await loader.buildBodies();
      finishClonedParameters();
      loader.finishDeferredLoadTearoffs();
      loader.finishNoSuchMethodForwarders();
      List<SourceClassBuilder> myClasses = collectMyClasses();
      loader.finishNativeMethods();
      loader.finishPatchMethods();
      finishAllConstructors(myClasses);
      runBuildTransformations();

      if (verify) this.verify();
      installAllComponentProblems(loader.allComponentProblems);
      return component;
    }, () => loader?.currentUriForCrashReporting);
  }
```
#####  Loader.buildBodies (loader.dart)

```dart
Future<Null> buildBodies() async {
    assert(coreLibrary != null);
    for (LibraryBuilder library in builders.values) {
      if (library.loader == this) {
        currentUriForCrashReporting = library.importUri;
        await buildBody(library);
      }
    }
    ...
  }
```
此处的loader为SourceLoader，是在KernelTarget对象创建过程初始化的.

##### SourceLoader.buildBody(source_loader.dart)

```dart
Future<Null> buildBody(LibraryBuilder library) async {
    if (library is SourceLibraryBuilder) {
      // 词法分析
      Token tokens = await tokenize(library, suppressLexicalErrors: true);
      if (tokens == null) return;
      DietListener listener = createDietListener(library);
      DietParser parser = new DietParser(listener);
      // 语法分析
      parser.parseUnit(tokens);
      // 针对part部分进行词法分析&语法分析
      for (SourceLibraryBuilder part in library.parts) {
        if (part.partOfLibrary != library) {
          // Part was included in multiple libraries. Skip it here.
          continue;
        }
        Token tokens = await tokenize(part);
        if (tokens != null) {
          listener.uri = part.fileUri;
          parser.parseUnit(tokens);
        }
      }
    }
  }
```
> 这里对源文件做了两次 tokenize, 以保持较低的内存使用.这里是第二次, 第一次在[buildOutline]中. 这次为了抑制词法错误.

#### runGlobalTransformations

```dart
Future runGlobalTransformations(...) async {
  if (errorDetector.hasCompilationErrors) return;

  final coreTypes = new CoreTypes(component);

  // 混淆, 得益于mixin去重功能,对于后端(以及一开始就执行的转换)都有好处, 建议除AOT外都开启.
  mixin_deduplication.transformComponent(component);
  
  // 不可达代码剔除
  unreachable_code_elimination.transformComponent(component, enableAsserts);

  if (useGlobalTypeFlowAnalysis) {
    globalTypeFlow.transformComponent(target, coreTypes, component);
  } else {
    devirtualization.transformComponent(coreTypes, component);
    no_dynamic_invocations_annotator.transformComponent(component);
  }

  if (useProtobufTreeShaker) {
    if (!useGlobalTypeFlowAnalysis) {
      throw 'Protobuf tree shaker requires type flow analysis (--tfa)';
    }

    protobuf_tree_shaker.removeUnusedProtoReferences(
        component, coreTypes, null);

    globalTypeFlow.transformComponent(target, coreTypes, component);
  }

  // TODO(35069): avoid recomputing CSA by reading it from the platform files.
  void ignoreAmbiguousSupertypes(cls, a, b) {}
  final hierarchy = new ClassHierarchy(component, coreTypes,
      onAmbiguousSupertypes: ignoreAmbiguousSupertypes);
  call_site_annotator.transformLibraries(
      component, component.libraries, coreTypes, hierarchy);

  // 不确定gen_snapshot是否要进行混淆处理，但是如果这样做，则需要混淆处理禁止
  obfuscationProhibitions.transformComponent(component, coreTypes);
}
```

## writeDillFile
```dart
writeDillFile(....) async {
    final Component component = results.component;
    // Remove the cache that came either from this function or from
    // initializing from a kernel file.
    component.metadata.remove(BinaryCacheMetadataRepository.repositoryTag);

    if (_compilerOptions.bytecode) {
        ....
        async {
            ...
            _writePackage(results, 'main', component.libraries, sink);
        });
      }
       ......
    } else {
      // Generate AST as the output proper.
      .....
      printer.writeComponentFile(component);
    }
    .....
  }
```
### BinaryPrinter.writeComponentFile
```dart
void writeComponentFile(Component component) {
    Timeline.timeSync("BinaryPrinter.writeComponentFile", () {
      computeCanonicalNames(component);
      final componentOffset = getBufferOffset();
      writeUInt32(Tag.ComponentFile);
      writeUInt32(Tag.BinaryFormatVersion);
      writeListOfStrings(component.problemsAsJson);
      indexLinkTable(component);
      _collectMetadata(component);
      if (_metadataSubsections != null) {
         // 将asm.bytecode写入文件
        _writeNodeMetadataImpl(component, componentOffset);
      }
      libraryOffsets = <int>[];
      CanonicalName main = getCanonicalNameOfMember(component.mainMethod);
      if (main != null) {
        checkCanonicalName(main);
      }
      writeLibraries(component);
      writeUriToSource(component.uriToSource);
      writeLinkTable(component);
      _writeMetadataSection(component);
      writeStringTable(stringIndexer);
      writeConstantTable(_constantIndexer);
      List<Library> libraries = component.libraries;
      if (libraryFilter != null) {
        List<Library> librariesNew = new List<Library>();
        for (int i = 0; i < libraries.length; i++) {
          Library library = libraries[i];
          if (libraryFilter(library)) librariesNew.add(library);
        }
        libraries = librariesNew;
      }
      writeComponentIndex(component, libraries);

      _flush();
    });
  }
```
到此便完成的kernel编译, 以及生成.dill文件。


# 参考
- [AOT Compilation, Kernel, and other Dart Hackery](https://thosakwe.com/aot-compilation-and-other-dart-hackery/)
- [Compiling the engine
](https://github.com/flutter/flutter/wiki/Compiling-the-engine)

<div  align="center"> 
<img src="https://ftp.bmp.ovh/imgs/2020/03/1b3d48e848798776.png"/>
</div>