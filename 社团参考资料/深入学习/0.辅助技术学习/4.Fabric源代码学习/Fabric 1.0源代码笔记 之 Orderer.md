## Fabric 1.0源代码笔记 之 Orderer

## 1、Orderer概述

Orderer，为排序节点，对所有发往网络中的交易进行排序，将排序后的交易安排配置中的约定整理为块，之后提交给Committer进行处理。

Orderer代码分布在orderer目录，目录结构如下：

* orderer目录
    * main.go，main入口。
    * util.go，orderer工具函数。
    * metadata目录，metadata.go实现获取版本信息。
    * localconfig目录，config.go，本地配置相关实现。
    * ledger目录，账本区块存储。
        * file目录，账本区块文件存储。
        * json目录，账本区块json文件存储。
        * ram目录，账本区块内存存储。
    * common目录，通用代码。
        * bootstrap目录，初始区块的提供方式。
## Fabric 1.0源代码笔记 之 Orderer #orderer start命令实现

![x](E:\大三上\导师任务\Fabric学习\x.jpg)

## 1、加载命令行工具并解析命令行参数

orderer的命令行工具，基于gopkg.in/alecthomas/kingpin.v2实现，地址：http://gopkg.in/alecthomas/kingpin.v2。
相关代码如下

```go
var (
    //创建kingpin.Application
    app = kingpin.New("orderer", "Hyperledger Fabric orderer node")
    //创建子命令start和version
    start   = app.Command("start", "Start the orderer node").Default()
    version = app.Command("version", "Show version information")
)

kingpin.Version("0.0.1")
//解析命令行参数
switch kingpin.MustParse(app.Parse(os.Args[1:])) {
case start.FullCommand():
    //orderer start的命令实现，下文展开讲解
case version.FullCommand():
    //输出版本信息
    fmt.Println(metadata.GetVersionInfo())
}
//代码在orderer/main.go
```

metadata.GetVersionInfo()代码如下：

```go
func GetVersionInfo() string {
    Version = common.Version //var Version string，全局变量
    if Version == "" {
        Version = "development build"
    }

    return fmt.Sprintf("%s:\n Version: %s\n Go version: %s\n OS/Arch: %s",
        ProgramName, Version, runtime.Version(),
        fmt.Sprintf("%s/%s", runtime.GOOS, runtime.GOARCH))
}
//代码在orderer/metadata/metadata.go
```

## 2、加载配置文件

配置文件的加载，基于viper实现，即https://github.com/spf13/viper。

```go
conf := config.Load()
//代码在orderer/main.go
```

conf := config.Load()代码如下：

```go
func Load() *TopLevel {
    config := viper.New()
    //cf.InitViper作用为加载配置文件路径及设置配置文件名称
    cf.InitViper(config, configName) //configName = strings.ToLower(Prefix)，其中Prefix = "ORDERER"

    config.SetEnvPrefix(Prefix) //Prefix = "ORDERER"
    config.AutomaticEnv()
    replacer := strings.NewReplacer(".", "_")
    config.SetEnvKeyReplacer(replacer)

    err := config.ReadInConfig() //加载配置文件内容

    var uconf TopLevel
    //将配置文件内容输出到结构体中
    err = viperutil.EnhancedExactUnmarshal(config, &uconf)
    //完成初始化，即检查空项，并赋默认值
    uconf.completeInitialization(filepath.Dir(config.ConfigFileUsed()))

    return &uconf
}

//代码在orderer/localconfig/config.go
```

TopLevel结构体及本地配置更详细内容，参考：[Fabric 1.0源代码笔记 之 Orderer #localconfig（Orderer配置文件定义）](localconfig.md)

## 3、初始化日志系统（日志输出、日志格式、日志级别、sarama日志）

```go
initializeLoggingLevel(conf)
//代码在orderer/main.go
```

initializeLoggingLevel(conf)代码如下：

```go
func initializeLoggingLevel(conf *config.TopLevel) {
    //初始化日志输出对象及输出格式
    flogging.InitBackend(flogging.SetFormat(conf.General.LogFormat), os.Stderr)
    //按初始化日志级别
    flogging.InitFromSpec(conf.General.LogLevel)
    if conf.Kafka.Verbose {
        //sarama为go语言版kafka客户端
        sarama.Logger = log.New(os.Stdout, "[sarama] ", log.Ldate|log.Lmicroseconds|log.Lshortfile)
    }
}
//代码在orderer/main.go
```



## 4、启动Go profiling服务（Go语言分析工具）

```go
initializeProfilingService(conf)
//代码在orderer/main.go
```

initializeProfilingService(conf)代码如下：

```go
func initializeProfilingService(conf *config.TopLevel) {
    if conf.General.Profile.Enabled { //是否启用Go profiling
        go func() {
            //Go profiling绑定的监听地址和端口
            logger.Info("Starting Go pprof profiling service on:", conf.General.Profile.Address)
            //启动Go profiling服务
            logger.Panic("Go pprof service failed:", http.ListenAndServe(conf.General.Profile.Address, nil))
        }()
    }
}
//代码在orderer/main.go
```

## 5、创建Grpc Server

```go
grpcServer := initializeGrpcServer(conf)
//代码在orderer/main.go
```

initializeGrpcServer(conf)代码如下：

```go
func initializeGrpcServer(conf *config.TopLevel) comm.GRPCServer {
    //按conf初始化安全服务器配置
    secureConfig := initializeSecureServerConfig(conf)
    //创建net.Listen
    lis, err := net.Listen("tcp", fmt.Sprintf("%s:%d", conf.General.ListenAddress, conf.General.ListenPort))
    //创建GRPC Server
    grpcServer, err := comm.NewGRPCServerFromListener(lis, secureConfig)
    return grpcServer
}
//代码在orderer/main.go
```

## 6、初始化本地MSP并获取签名

```go
initializeLocalMsp(conf)
signer := localmsp.NewSigner() //return &mspSigner{}
//代码在orderer/main.go
```

initializeLocalMsp(conf)代码如下：

```go
func initializeLocalMsp(conf *config.TopLevel) {
    //从指定目录加载本地MSP
    err := mspmgmt.LoadLocalMsp(conf.General.LocalMSPDir, conf.General.BCCSP, conf.General.LocalMSPID)
}
//代码在orderer/main.go
```



## 7、初始化MultiChain管理器（启动共识插件goroutine，接收和处理消息）

```go
manager := initializeMultiChainManager(conf, signer)
//代码在orderer/main.go
```

initializeMultiChainManager(conf, signer)代码如下：

```go
func initializeMultiChainManager(conf *config.TopLevel, signer crypto.LocalSigner) multichain.Manager {
    lf, _ := createLedgerFactory(conf) //创建LedgerFactory
    if len(lf.ChainIDs()) == 0 { //链不存在
        initializeBootstrapChannel(conf, lf) //初始化引导通道（获取初始区块、创建链、添加初始区块）
    } else {
        //链已存在
    }

    consenters := make(map[string]multichain.Consenter) //共识
    consenters["solo"] = solo.New()
    consenters["kafka"] = kafka.New(conf.Kafka.TLS, conf.Kafka.Retry, conf.Kafka.Version)
    return multichain.NewManagerImpl(lf, consenters, signer) //LedgerFactory、Consenter、签名
}
//代码在orderer/main.go
```

initializeBootstrapChannel(conf, lf)代码如下：

```go
func initializeBootstrapChannel(conf *config.TopLevel, lf ledger.Factory) {
    var genesisBlock *cb.Block
    switch conf.General.GenesisMethod { //初始区块的提供方式
    case "provisional": //根据GenesisProfile提供
        genesisBlock = provisional.New(genesisconfig.Load(conf.General.GenesisProfile)).GenesisBlock()
    case "file": //指定现成的初始区块文件
        genesisBlock = file.New(conf.General.GenesisFile).GenesisBlock()
    default:
        logger.Panic("Unknown genesis method:", conf.General.GenesisMethod)
    }
    chainID, err := utils.GetChainIDFromBlock(genesisBlock) //获取ChainID
    gl, err := lf.GetOrCreate(chainID) //获取或创建chain
    err = gl.Append(genesisBlock) //追加初始区块
}
//代码在orderer/main.go
```

## 8、注册orderer service并启动grpcServer

```go
server := NewServer(manager, signer) //构造server
ab.RegisterAtomicBroadcastServer(grpcServer.Server(), server) //service注册到grpcServer
grpcServer.Start()
//代码在orderer/main.go
```

server := NewServer(manager, signer)代码如下：

```go
func NewServer(ml multichain.Manager, signer crypto.LocalSigner) ab.AtomicBroadcastServer {
    s := &server{
        dh: deliver.NewHandlerImpl(deliverSupport{Manager: ml}),
        bh: broadcast.NewHandlerImpl(broadcastSupport{
            Manager:               ml,
            ConfigUpdateProcessor: configupdate.New(ml.SystemChannelID(), configUpdateSupport{Manager: ml}, signer),
        }),
    }
    return s
}
//代码在orderer/server.go
```
