## 以太坊

[学习自](https://paper.seebug.org/642/)

此前，已经稍微的讨论过节点的发现、广播功能模块。现在分析**数据链路的建立和交互**。

### Ethereum源码结构

```shell
go-ethereum-master tree -d -L 1     
.
├── accounts				账号相关
├── build						编译生成的程序
├── cmd							geth程序主体
├── common					工具函数库
├── consensus				共识算法
├── console					交互式命令
├── contracts				合约相关
├── core						以太坊核心部分
├── crypto					加密函数库
├── docs						文档
├── eth							以太坊协议
├── ethclient				以太坊RPC客户端
├── ethdb						底层存储
├── ethstats				统计报告
├── event						事件处理
├── graphql					
├── internal				RPC调用
├── les							轻量级子协议
├── light						轻客户端部分功能
├── log							日志模块
├── metrics					服务监控相关
├── miner						挖矿相关
├── mobile					geth的移动API
├── node						接口节点
├── p2p							p2p网络协议
├── params					一些预设参数数值
├── rlp							RLP系列化格式
├── rpc							RPC接口
├── signer					签名相关
├── swarm						分布式存储
├── tests						以太坊JSON测试
├── trie						Merkle Patriia实现
└── whisper					分布式消息
```



#### 分析geth的代码

何为geth？geth是用于运行Go中实现的完整以太坊节点的命令行界面。

所以说，分析geth的代码，观察以太坊节点的构建过程是有益的。

```go
/cmd/geth/main.go
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

```go
/cmd/geth/main.go
	// The app that holds all commands and flags.
	app = utils.NewApp(gitCommit, gitDate, "the go-ethereum command line interface")
```

```go
/cmd/utils/flags.go
// NewApp creates an app with sane defaults.
func NewApp(gitCommit, gitDate, usage string) *cli.App {
	app := cli.NewApp()
	app.Name = filepath.Base(os.Args[0])
	app.Author = ""
	app.Email = ""
	app.Version = params.VersionWithCommit(gitCommit, gitDate)
	app.Usage = usage
	return app
}
```

Geth 使用了` gopkg.in/urfave/cli.v1 `扩展包，该扩展包用于管理程序的启动，以及命令行解析。在flags.go中的NewApp使用了cli。

然后，关键点在于`Run(os.Args)`，运行时带上命令行上的参数。

在 Go 语言中，在有 `init()` 函数的情况下，会默认先调用 `init()` 函数，然后再调用 `main()` 函数；Geth 几乎在 `./cmd/geth/main.go#init()` 中完成了所有的初始化操作：设置程序的子命令集，设置程序入口函数等，下面看下 `init()` 函数片段：

```go
/cmd/geth/main.go
func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2019 The go-ethereum Authors"
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,
		importCommand,
		exportCommand,
		importPreimagesCommand,
		exportPreimagesCommand,
		copydbCommand,
		removedbCommand,
		dumpCommand,
		inspectCommand,
		// See accountcmd.go:
		accountCommand,
		walletCommand,
		// See consolecmd.go:
		consoleCommand,
		attachCommand,
		javascriptCommand,
		// See misccmd.go:
		makecacheCommand,
		makedagCommand,
		versionCommand,
		licenseCommand,
		// See config.go
		dumpConfigCommand,
		// See retesteth.go
		retestethCommand,
	}
	sort.Sort(cli.CommandsByName(app.Commands))

	app.Flags = append(app.Flags, nodeFlags...)
	app.Flags = append(app.Flags, rpcFlags...)
	app.Flags = append(app.Flags, consoleFlags...)
	app.Flags = append(app.Flags, debug.Flags...)
	app.Flags = append(app.Flags, whisperFlags...)
	app.Flags = append(app.Flags, metricsFlags...)

	app.Before = func(ctx *cli.Context) error {
		return debug.Setup(ctx, "")
	}
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		console.Stdin.Close() // Resets terminal mode.
		return nil
	}
}
```

初始化app运行的时Action=geth，即运行geth()这个函数，如果没有子命令的话，那么如果有子命令的话，则会运行`app.Commands = []cli.Command{`这里面的命令函数，而不会运行默认的geth函数。

```go
/cmd/geth/main.go
// geth is the main entry point into the system if no special subcommand is ran.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.
func geth(ctx *cli.Context) error {
	if args := ctx.Args(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}
	prepare(ctx)
	node := makeFullNode(ctx)
	defer node.Close()
	startNode(ctx, node)
	node.Wait()
	return nil
}
```

其中 `makeFullNode()` 函数将返回一个节点实例，然后通过 `startNode()` 启动。在 Geth 中，每一个功能模块都被视为一个服务，每一个服务的正常运行驱动着 Geth 的各项功能；`makeFullNode()` 通过解析命令行参数，注册指定的服务。

```go
/cmd/geth/config.go
func makeFullNode(ctx *cli.Context) *node.Node {
	stack, cfg := makeConfigNode(ctx)
	if ctx.GlobalIsSet(utils.OverrideIstanbulFlag.Name) {
		cfg.Eth.OverrideIstanbul = new(big.Int).SetUint64(ctx.GlobalUint64(utils.OverrideIstanbulFlag.Name))
	}
	utils.RegisterEthService(stack, &cfg.Eth)

	// Whisper must be explicitly enabled by specifying at least 1 whisper flag or in dev mode
	shhEnabled := enableWhisper(ctx)
	shhAutoEnabled := !ctx.GlobalIsSet(utils.WhisperEnabledFlag.Name) && ctx.GlobalIsSet(utils.DeveloperFlag.Name)
	if shhEnabled || shhAutoEnabled {
		if ctx.GlobalIsSet(utils.WhisperMaxMessageSizeFlag.Name) {
			cfg.Shh.MaxMessageSize = uint32(ctx.Int(utils.WhisperMaxMessageSizeFlag.Name))
		}
		if ctx.GlobalIsSet(utils.WhisperMinPOWFlag.Name) {
			cfg.Shh.MinimumAcceptedPOW = ctx.Float64(utils.WhisperMinPOWFlag.Name)
		}
		if ctx.GlobalIsSet(utils.WhisperRestrictConnectionBetweenLightClientsFlag.Name) {
			cfg.Shh.RestrictConnectionBetweenLightClients = true
		}
		utils.RegisterShhService(stack, &cfg.Shh)
	}
	// Configure GraphQL if requested
	if ctx.GlobalIsSet(utils.GraphQLEnabledFlag.Name) {
		utils.RegisterGraphQLService(stack, cfg.Node.GraphQLEndpoint(), cfg.Node.GraphQLCors, cfg.Node.GraphQLVirtualHosts, cfg.Node.HTTPTimeouts)
	}
	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
		utils.RegisterEthStatsService(stack, cfg.Ethstats.URL)
	}
	return stack
}
```

也就是说，如果带上命令行参数，那么将会转到相应的命令函数，运行相应的服务。如果不带上命令的话，那么将会注册所有的服务。