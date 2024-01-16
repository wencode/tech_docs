# [翻译]使用Slog进行Go语言日志记录：最终指导手册

［[原文](https://betterstack.com/community/guides/logging/logging-in-go/)］: https://betterstack.com/community/guides/logging/logging-in-go/

本文将深入探讨Go语言中的结构化日志记录，特别是聚焦于最近引入的[log/slog](https://pkg.go.dev/log/slog)包。这个包的目标是为Go标准库引入高效能、结构化以及分层次的日志记录功能。

这个包的起源可以追溯到Jonathan Amsterdam在GitHub上发起的[讨论](https://github.com/golang/go/discussions/54763)，该讨论后来促成了一个提案，详细描述了这个包的设计。设计定稿后，该包在Go版本1.21中发布，现位于log/slog。

在本文后续部分，我会详细介绍Slog的各项功能及其示例。若想了解它与其他Go日志框架在性能上的对比，请参考这个[GitHub仓库](https://github.com/betterstack-community/go-logging-benchmarks)。

## 开始了解Slog
我们先从了解log/slog包的设计和架构开始探索。这个包提供了三种主要类型，您应该了解：

* Logger：作为日志记录的“前端”，提供了如Info()和Error()等级别方法，用于记录感兴趣的事件。
* Record：Logger创建的每个独立日志对象的表现形式。
* Handler：一旦实现，就决定了每个Record的格式和存储目的地。log/slog包内置了两种处理程序：TextHandler和JSONHandler，分别用于key=value格式和JSON格式的输出。

与大多数Go日志库类似，slog包提供了一个默认的Logger，可以通过顶级函数访问。这个Logger生成的输出几乎与旧版的log.Printf()方法相同，唯一不同的是增加了日志级别。


```go
package main

import (
    "log"
    "log/slog"
)

func main() {
    log.Print("Info message")
    slog.Info("Info message")
}
```

**输出：**
```plan/text
2024/01/03 10:24:22 Info message
2024/01/03 10:24:22 INFO Info message
```

这是一个有些奇怪的默认设置，因为Slog的主要目的是为标准库提供结构化日志记录功能，但是默认的Logger却没有这种功能。

通过使用slog.New()方法创建一个自定义的Logger实例，可以轻松改正这一默认设置。该方法接受一个Handler接口的实现，这个接口决定了日志的格式化方式和输出位置。

下面是一个例子，展示了如何使用内置的JSONHandler类型来输出JSON格式的日志到标准输出（stdout）：


```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    logger.Debug("Debug message")
    logger.Info("Info message")
    logger.Warn("Warning message")
    logger.Error("Error message")
}
```

**输出：**
```plan/text
{"time":"2023-03-15T12:59:22.227408691+01:00","level":"INFO","msg":"Info message"}
{"time":"2023-03-15T12:59:22.227468972+01:00","level":"WARN","msg":"Warning message"}
{"time":"2023-03-15T12:59:22.227472149+01:00","level":"ERROR","msg":"Error message"}
```

当使用TextHandler类型时，每条日志记录都会按照Logfmt标准进行格式化：

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
```

**输出：**
```plan/text
time=2023-03-15T13:00:11.333+01:00 level=INFO msg="Info message"
time=2023-03-15T13:00:11.333+01:00 level=WARN msg="Warning message"
time=2023-03-15T13:00:11.333+01:00 level=ERROR msg="Error message"
```

所有Logger实例默认设置为在INFO级别进行日志记录，这意味着DEBUG级别的条目会被屏蔽，但您可以根据实际需要轻松地进行[调整](https://betterstack.com/community/guides/logging/logging-in-go/#customizing-slog-log-levels)。

## 定制默认的Logger

定制默认Logger最直接的方法是使用slog.SetDefault()方法，这使您可以将默认的Logger替换为自定义的Logger。

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    slog.SetDefault(logger)

    slog.Info("Info message")
}
```

现在您将会发现，该包的顶级日志记录方法现在产生的是如下所示的JSON输出：

**输出：**
```plan/text
{"time":"2023-03-15T13:07:39.105777557+01:00","level":"INFO","msg":"Info message"}
```

使用SetDefault()方法也会改变log包中使用的默认log.Logger。这种变化使得那些原本使用旧log包的现有应用能够无缝过渡到[结构化日志](https://betterstack.com/community/guides/logging/log-formatting/)记录：

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    slog.SetDefault(logger)

    // elsewhere in the application
    log.Println("Hello from old logger")
}
```

**输出：**
```plan/text
{"time":"2023-03-16T15:20:33.783681176+01:00","level":"INFO","msg":"Hello from old logger"}
```

当您需要使用那些要求后者的API（例如http.Server.ErrorLog）时，slog.NewLogLogger()方法可以将slog.Logger转换为log.Logger：

```go
func main() {
    handler := slog.NewJSONHandler(os.Stdout, nil)

    logger := slog.NewLogLogger(handler, slog.LevelError)

    _ = http.Server{
        // this API only accepts `log.Logger`
        ErrorLog: logger,
    }
}
```

## 在日志记录中添加上下文属性

结构化日志记录相比非结构化格式的一大优点是能够在日志记录中添加任意属性作为键/值对。

这些属性为记录的事件提供了更多的上下文信息，对于诸如故障排除、生成度量指标、审计等任务非常有价值。

下面是一个例子，展示了Slog如何实现这一功能：

```go
logger.Info(
  "incoming request",
  "method", "GET",
  "time_taken_ms", 158,
  "path", "/hello/world?q=search",
  "status", 200,
  "user_agent", "Googlebot/2.1 (+http://www.google.com/bot.html)",
)
```

**输出：**
```plan/text
{
  "time":"2023-02-24T11:52:49.554074496+01:00",
  "level":"INFO",
  "msg":"incoming request",
  "method":"GET",
  "time_taken_ms":158,
  "path":"/hello/world?q=search",
  "status":200,
  "user_agent":"Googlebot/2.1 (+http://www.google.com/bot.html)"
}
```

所有级别方法（如Info()、Debug()等）首先接受一个日志消息作为它们的第一个参数，然后可以接收无限数量的松散类型键/值对。

这个API与Zap中的SugaredLogger API相似（特别是其以w结尾的级别方法），它在增加额外内存分配的情况下，优先考虑简洁性。

但需谨慎，因为这种方式可能带来意外的问题。具体来说，不成对的键/值可能会导致问题性的输出：

```go
logger.Info(
  "incoming request",
  "method", "GET",
  "time_taken_ms", // the value for this key is missing
)
```

因为time_taken_ms键没有对应的值，它会被当作一个键为!BADKEY的值。这种情况并不理想，因为属性不匹配可能导致错误的条目生成，并且您可能直到需要使用日志时才会发现这个问题。

**输出：**
```plan/text
{
  "time": "2023-03-15T13:15:29.956566795+01:00",
  "level": "INFO",
  "msg": "incoming request",
  "method": "GET",
  "!BADKEY": "time_taken_ms"
}
```

为了避免这类问题，您可以运行[vet](https://pkg.go.dev/cmd/vet)命令或使用静态代码分析工具[linter](https://github.com/go-simpler/sloglint)来自动报告这类问题。

![](https://imagedelivery.net/xZXo0QFi-1_4Zimer-T0XQ/db000eeb-f685-49ce-1541-92458f238100/lg1x)

另一种避免这类错误的方法是使用强类型的上下文属性，如下所示：


```go
logger.Info(
  "incoming request",
  slog.String("method", "GET"),
  slog.Int("time_taken_ms", 158),
  slog.String("path", "/hello/world?q=search"),
  slog.Int("status", 200),
  slog.String(
    "user_agent",
    "Googlebot/2.1 (+http://www.google.com/bot.html)",
  ),
)
```

虽然这是一种更好的上下文日志方法，但它也不是绝对可靠的，因为仍然无法阻止您混合使用强类型和松散类型的键/值对，如下所示：


```go
logger.Info(
  "incoming request",
  "method", "GET",
  slog.Int("time_taken_ms", 158),
  slog.String("path", "/hello/world?q=search"),
  "status", 200,
  slog.String(
    "user_agent",
    "Googlebot/2.1 (+http://www.google.com/bot.html)",
  ),
)
```

为了在向记录添加上下文属性时保证类型安全，您必须使用LogAttrs()方法，如下所示：

```go
logger.LogAttrs(
  context.Background(),
  slog.LevelInfo,
  "incoming request",
  slog.String("method", "GET"),
  slog.Int("time_taken_ms", 158),
  slog.String("path", "/hello/world?q=search"),
  slog.Int("status", 200),
  slog.String(
    "user_agent",
    "Googlebot/2.1 (+http://www.google.com/bot.html)",
  ),
)
```

这个方法只接受slog.Attr类型的自定义属性，因此不可能出现键/值对不匹配的情况。然而，它的API更为复杂，因为除了日志消息和自定义属性之外, 您需要总是传递一个上下文（或nil）和日志级别进行方法调用。

## 分组上下文属性

Slog也支持将多个属性分组到一个名称下，但具体的输出格式取决于所使用的Handler。例如，在使用JSONHandler时，每个分组都会被嵌套在JSON对象内：


```go
logger.LogAttrs(
  context.Background(),
  slog.LevelInfo,
  "image uploaded",
  slog.Int("id", 23123),
  slog.Group("properties",
    slog.Int("width", 4000),
    slog.Int("height", 3000),
    slog.String("format", "jpeg"),
  ),
)
```

**输出：**
```plan/text
{
  "time":"2023-02-24T12:03:12.175582603+01:00",
  "level":"INFO",
  "msg":"image uploaded",
  "id":23123,
  "properties":{
    "width":4000,
    "height":3000,
    "format":"jpeg"
  }
}
```

当使用TextHandler时，分组中的每个键都会以分组名作为前缀，如此展示：
**输出：**
```plan/text
time=2023-02-24T12:06:20.249+01:00 level=INFO msg="image uploaded" id=23123
  properties.width=4000 properties.height=3000 properties.format=jpeg
```

## 创建和使用子Logger
在特定范围内的所有日志记录中包含相同的属性可以避免重复的日志记录语句，同时确保这些属性的存在。

这就是子Logger发挥作用的场景，因为它们可以创建一个新的日志上下文，这个上下文继承自它们的父Logger，同时允许加入额外的字段。

在Slog中，可以通过使用Logger.With()方法来创建子Logger。这个方法接受一个或多个键值对，并返回一个包含了指定属性的新Logger。

考虑下面的代码片段，它在每个日志记录中添加了程序的进程ID和用于编译的Go版本信息，并将这些信息存储在program_info属性中：


```go
func main() {
    handler := slog.NewJSONHandler(os.Stdout, nil)
    buildInfo, _ := debug.ReadBuildInfo()

    logger := slog.New(handler)

    child := logger.With(
        slog.Group("program_info",
            slog.Int("pid", os.Getpid()),
            slog.String("go_version", buildInfo.GoVersion),
        ),
    )

    . . .
}
```

在这种配置下，只要在记录日志时没有被覆盖，由子Logger创建的所有记录都会在program_info属性下包含指定的属性：

```go
func main() {
    . . .

    child.Info("image upload successful", slog.String("image_id", "39ud88"))
    child.Warn(
        "storage is 90% full",
        slog.String("available_space", "900.1 mb"),
    )
}
```

**输出：**
```plan/text
{
  "time": "2023-02-26T19:26:46.046793623+01:00",
  "level": "INFO",
  "msg": "image upload successful",
  "program_info": {
    "pid": 229108,
    "go_version": "go1.20"
  },
  "image_id": "39ud88"
}
{
  "time": "2023-02-26T19:26:46.046847902+01:00",
  "level": "WARN",
  "msg": "storage is 90% full",
  "program_info": {
    "pid": 229108,
    "go_version": "go1.20"
  },
  "available_space": "900.1 MB"
}
```

您也可以使用WithGroup()方法创建一个子Logger，该子Logger开始一个分组，以便所有添加到Logger的属性（包括在记录日志时添加的属性）都被嵌套在该分组名下：

```go
handler := slog.NewJSONHandler(os.Stdout, nil)
buildInfo, _ := debug.ReadBuildInfo()
logger := slog.New(handler).WithGroup("program_info")

child := logger.With(
  slog.Int("pid", os.Getpid()),
  slog.String("go_version", buildInfo.GoVersion),
)

child.Warn(
  "storage is 90% full",
  slog.String("available_space", "900.1 MB"),
)
```

**输出：**
```plan/text
{
  "time": "2023-05-24T19:00:18.384136084+01:00",
  "level": "WARN",
  "msg": "storage is 90% full",
  "program_info": {
    "pid": 1971993,
    "go_version": "go1.20.2",
    "available_space": "900.1 mb"
  }
}
```

## 定制Slog的日志级别
log/slog包默认提供了四种日志级别，每种级别都对应一个整数值：DEBUG（-4）、INFO（0）、WARN（4）和ERROR（8）。

每个级别之间设置四个单位的间隔是一种故意的设计决定，目的是为了适应在默认级别之间设置自定义级别的日志方案。例如，您可以在INFO和WARN之间创建一个值为1、2或3的自定义级别。

我们之前观察到，默认情况下所有Logger都配置为在INFO级别进行记录，这会导致记录在更低严重级别（如DEBUG）的事件被忽视, 您可以通过[HandlerOptions](https://pkg.go.dev/log/slog#HandlerOptions)类型来自定义这种行为，如下所示：


```go
func main() {
    opts := &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }

    handler := slog.NewJSONHandler(os.Stdout, opts)

    logger := slog.New(handler)
    logger.Debug("Debug message")
    logger.Info("Info message")
    logger.Warn("Warning message")
    logger.Error("Error message")
}
```

**输出：**
```plan/text
{"time":"2023-05-24T19:03:10.70311982+01:00","level":"DEBUG","msg":"Debug message"}
{"time":"2023-05-24T19:03:10.703187713+01:00","level":"INFO","msg":"Info message"}
{"time":"2023-05-24T19:03:10.703190419+01:00","level":"WARN","msg":"Warning message"}
{"time":"2023-05-24T19:03:10.703192892+01:00","level":"ERROR","msg":"Error message"}
```

这种设定日志级别的方法会在处理器的整个生命周期内固定其级别。如果您需要动态调整最低日志级别，您应该使用LevelVar类型，如下所示：

```go
func main() {
    logLevel := &slog.LevelVar{} // INFO

    opts := &slog.HandlerOptions{
        Level: logLevel,
    }

    handler := slog.NewJSONHandler(os.Stdout, opts)

    . . .
}
```

接下来，您可以随时使用以下方法来更新日志级别：

```go
logLevel.Set(slog.LevelDebug)
```

## 创建自定义的日志级别

如果您需要的自定义日志级别超出了Slog默认提供的范围，您可以通过实现Leveler接口来创建这些级别，其接口签名如下所示：


```go
type Leveler interface {
    Level() Level
}
```

实现这个接口通过下面展示的Level类型非常简单（因为Level本身就实现了Leveler）：

```go
const (
    LevelTrace  = slog.Level(-8)
    LevelFatal  = slog.Level(12)
)
```

定义了自定义级别后，您只能通过Log()或LogAttrs()方法来使用它们：

```go
opts := &slog.HandlerOptions{
    Level: LevelTrace,
}

logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))

ctx := context.Background()
logger.Log(ctx, LevelTrace, "Trace message")
logger.Log(ctx, LevelFatal, "Fatal level")
```

**输出：**
```plan/text
{"time":"2023-02-24T09:26:41.666493901+01:00","level":"DEBUG-4","msg":"Trace level"}
{"time":"2023-02-24T09:26:41.666602404+01:00","level":"ERROR+4","msg":"Fatal level"}
```

注意到自定义级别是如何以默认级别的形式被标记的。这肯定不是您想要的效果，因此您应该通过HandlerOptions类型来自定义级别名称，如下所示：

```go
. . .

var LevelNames = map[slog.Leveler]string{
    LevelTrace:      "TRACE",
    LevelFatal:      "FATAL",
}

func main() {
    opts := slog.HandlerOptions{
        Level: LevelTrace,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            if a.Key == slog.LevelKey {
                level := a.Value.Any().(slog.Level)
                levelLabel, exists := LevelNames[level]
                if !exists {
                    levelLabel = level.String()
                }

                a.Value = slog.StringValue(levelLabel)
            }

            return a
        },
    }

    . . .
}
```

ReplaceAttr()函数被用来定制Handler处理Record中每个键值对的方式。它可以用于定制键名或以某种方式处理值。

在上述例子中，它将自定义的日志级别映射到对应的标签上，分别产生了TRACE和FATAL级别。

**输出：**
```plan/text
{"time":"2023-02-24T09:27:51.747625912+01:00","level":"TRACE","msg":"Trace level"}
{"time":"2023-02-24T09:27:51.747737319+01:00","level":"FATAL","msg":"Fatal level"}
```

## 定制Slog处理器

正如之前提到的，TextHandler和JSONHandler都可以通过HandlerOptions类型进行自定义。您已经了解了如何调整最小日志级别以及如何在记录日志之前修改属性。

通过HandlerOptions还可以完成的另一项定制是，如有必要则包括日志输出的源文件信息：

```go
opts := &slog.HandlerOptions{
    AddSource: true,
    Level:     slog.LevelDebug,
}
```

**输出：**
```plan/text
{
  "time": "2024-01-03T11:06:50.971029852+01:00",
  "level": "DEBUG",
  "source": {
    "function": "main.main",
    "file": "/home/ayo/dev/betterstack/demo/slog/main.go",
    "line": 17
  },
  "msg": "Debug message"
}
```

根据应用环境轻松切换不同的处理器也很简单。例如，您可能更倾向于在开发环境中使用TextHandler，因为它的可读性更好，然后在生产环境中切换到JSONHandler，以便获得更大的灵活性和与多种日志工具的兼容性。

通过环境变量，可以轻松地实现这种行为：

```go
var appEnv = os.Getenv("APP_ENV")

func main() {
    opts := &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }

    var handler slog.Handler = slog.NewTextHandler(os.Stdout, opts)
    if appEnv == "production" {
        handler = slog.NewJSONHandler(os.Stdout, opts)
    }

    logger := slog.New(handler)

    logger.Info("Info message")
}
```

```bash
$ go run main.go
```

**输出：**
```plan/text
time=2023-02-24T10:36:39.697+01:00 level=INFO msg="Info message"
```

```bash
$ APP_ENV=production go run main.go
```

**输出：**
```plan/text
{"time":"2023-02-24T10:35:16.964821548+01:00","level":"INFO","msg":"Info message"}
```

## 创建自定义的处理器
由于Handler是一个接口，因此有可能创建自定义处理器来以不同的方式格式化日志或者将它们写入到其他地方。

这个接口的签名如下所示：

```go
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, r Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

以下是每个方法的作用：

* Enabled()根据日志记录的级别决定是否处理或丢弃该记录。此方法还可以使用上下文来做出决策。
* Handle()处理发送到处理器的每条日志记录。只有在Enabled()返回true时，才会调用此方法。
* WithAttrs()基于现有的处理器创建一个新处理器，并添加指定的属性。
* WithGroup()基于现有的处理器创建一个新处理器，并添加指定的分组名称，以便该名称作为后续属性的限定符。

这里有一个例子，演示了如何使用log、json和color包来实现日志记录的美化输出，适用于开发环境：


```go
// NOTE: Not well tested, just an illustration of what's possible
package main

import (
    "context"
    "encoding/json"
    "io"
    "log"
    "log/slog"

    "github.com/fatih/color"
)

type PrettyHandlerOptions struct {
    SlogOpts slog.HandlerOptions
}

type PrettyHandler struct {
    slog.Handler
    l *log.Logger
}

func (h *PrettyHandler) Handle(ctx context.Context, r slog.Record) error {
    level := r.Level.String() + ":"

    switch r.Level {
    case slog.LevelDebug:
        level = color.MagentaString(level)
    case slog.LevelInfo:
        level = color.BlueString(level)
    case slog.LevelWarn:
        level = color.YellowString(level)
    case slog.LevelError:
        level = color.RedString(level)
    }

    fields := make(map[string]interface{}, r.NumAttrs())
    r.Attrs(func(a slog.Attr) bool {
        fields[a.Key] = a.Value.Any()

        return true
    })

    b, err := json.MarshalIndent(fields, "", "  ")
    if err != nil {
        return err
    }

    timeStr := r.Time.Format("[15:05:05.000]")
    msg := color.CyanString(r.Message)

    h.l.Println(timeStr, level, msg, color.WhiteString(string(b)))

    return nil
}

func NewPrettyHandler(
    out io.Writer,
    opts PrettyHandlerOptions,
) *PrettyHandler {
    h := &PrettyHandler{
        Handler: slog.NewJSONHandler(out, &opts.SlogOpts),
        l:       log.New(out, "", 0),
    }

    return h
}

```

可以在代码中这样使用PrettyHandler：

```go
func main() {
    opts := PrettyHandlerOptions{
        SlogOpts: slog.HandlerOptions{
            Level: slog.LevelDebug,
        },
    }
    handler := NewPrettyHandler(os.Stdout, opts)
    logger := slog.New(handler)
    logger.Debug(
        "executing database query",
        slog.String("query", "SELECT * FROM users"),
    )
    logger.Info("image upload successful", slog.String("image_id", "39ud88"))
    logger.Warn(
        "storage is 90% full",
        slog.String("available_space", "900.1 MB"),
    )
    logger.Error(
        "An error occurred while processing the request",
        slog.String("url", "https://example.com"),
    )
}

```

运行程序时，将看到如下的彩色输出：
![](https://imagedelivery.net/xZXo0QFi-1_4Zimer-T0XQ/c62a13a4-afea-4e53-2ac3-c5c8c790b200/lg1x)

您可以在[GitHub](https://github.com/search?q=slog-+language%3AGo&type=repositories&l=Go)和这个[Go Wiki](https://tip.golang.org/wiki/Resources-for-slog)页面上找到社区创建的多个自定义处理器。其中一些值得关注的例子包括：

* tint - 用于生成彩色化的日志。
* slog-sampling - 通过丢弃重复的日志记录来提高日志处理的吞吐量。
* slog-multi - 实现了中间件、分流、路由、故障转移、负载均衡等多种工作流程。
* slog-formatter - 提供了更灵活的属性格式化功能。

## slog结合context包使用

到目前为止，我们主要使用了如Info()、Debug()等级别方法的标准版本，但Slog也提供了感知上下文的变体，这些变体接受context.Context值作为它们的第一个参数。以下是每个方法的签名：


```go
func (ctx context.Context, msg string, args ...any)
```

使用这些方法，您可以通过在Context中存储上下文属性，从而在不同函数间传播这些属性。这样做的好处是，一旦这些值被检测到，它们就会被添加到生成的任何日志记录中。

考虑以下程序：

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    ctx := context.WithValue(context.Background(), "request_id", "req-123")

    logger.InfoContext(ctx, "image uploaded", slog.String("image_id", "img-998"))
}
```

一个request_id被添加到ctx变量中，并传递给了InfoContext方法。但是，当程序运行时，日志中并没有显示request_id字段：

**输出：**
```plan/text
{
  "time": "2024-01-02T11:04:28.590527494+01:00",
  "level": "INFO",
  "msg": "image uploaded",
  "image_id": "img-998"
}
```

为了让它正常工作，您需要创建一个自定义处理器，并按照下面的方式重新实现Handle方法：

```go
type ctxKey string

const (
    slogFields ctxKey = "slog_fields"
)

type ContextHandler struct {
    slog.Handler
}

// Handle adds contextual attributes to the Record before calling the underlying
// handler
func (h ContextHandler) Handle(ctx context.Context, r slog.Record) error {
    if attrs, ok := ctx.Value(slogFields).([]slog.Attr); ok {
        for _, v := range attrs {
            r.AddAttrs(v)
        }
    }

    return h.Handler.Handle(ctx, r)
}

// AppendCtx adds an slog attribute to the provided context so that it will be
// included in any Record created with such context
func AppendCtx(parent context.Context, attr slog.Attr) context.Context {
    if parent == nil {
        parent = context.Background()
    }

    if v, ok := parent.Value(slogFields).([]slog.Attr); ok {
        v = append(v, attr)
        return context.WithValue(parent, slogFields, v)
    }

    v := []slog.Attr{}
    v = append(v, attr)
    return context.WithValue(parent, slogFields, v)
}
```

ContextHandler结构体内嵌了slog.Handler接口，并实现了Handle方法，用于提取存储在提供的上下文中的Slog属性。如果这些属性被找到，它们将在调用底层Handler进行格式化和输出记录之前被添加到Record中。

而AppendCtx函数则使用slogFields键将Slog属性添加到context.Context中，这样ContextHandler就可以访问这些属性了。

下面是如何使用它们的示例：

```go
func main() {
    h := &ContextHandler{slog.NewJSONHandler(os.Stdout, nil)}

    logger := slog.New(h)

    ctx := AppendCtx(context.Background(), slog.String("request_id", "req-123"))

    logger.InfoContext(ctx, "image uploaded", slog.String("image_id", "img-998"))
}
```

现在您将会发现，所有使用ctx参数创建的日志记录中都包含了request_id：

**输出：**
```plan/text
{
  "time": "2024-01-02T11:29:15.229984723+01:00",
  "level": "INFO",
  "msg": "image uploaded",
  "image_id": "img-998",
  "request_id": "req-123"
}
```

## 使用Slog进行错误日志记录
在记录错误时，不像大多数框架那样为error类型提供专用助手函数，所以您需要使用slog.Any()来记录错误，如下所示：


```go
err := errors.New("something happened")

logger.ErrorContext(ctx, "upload failed", slog.Any("error", err))
```

**输出：**
```plan/text
{
  "time": "2024-01-02T14:13:44.41886393+01:00",
  "level": "ERROR",
  "msg": "upload failed",
  "error": "something happened"
}
```

为了获取并记录错误的堆栈跟踪信息，您可以使用像xerrors这样的库来创建带有堆栈跟踪的错误：

```go
err := xerrors.New("something happened")

logger.ErrorContext(ctx, "upload failed", slog.Any("error", err))
```

在错误日志中看到堆栈跟踪之前，您还需要通过之前展示的ReplaceAttr()函数来提取、格式化，并将其添加到相应的Record中。

以下是一个示例：

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "path/filepath"

    "github.com/mdobak/go-xerrors"
)

type stackFrame struct {
    Func   string `json:"func"`
    Source string `json:"source"`
    Line   int    `json:"line"`
}

func replaceAttr(_ []string, a slog.Attr) slog.Attr {
    switch a.Value.Kind() {
    case slog.KindAny:
        switch v := a.Value.Any().(type) {
        case error:
            a.Value = fmtErr(v)
        }
    }

    return a
}

// marshalStack extracts stack frames from the error
func marshalStack(err error) []stackFrame {
    trace := xerrors.StackTrace(err)

    if len(trace) == 0 {
        return nil
    }

    frames := trace.Frames()

    s := make([]stackFrame, len(frames))

    for i, v := range frames {
        f := stackFrame{
            Source: filepath.Join(
                filepath.Base(filepath.Dir(v.File)),
                filepath.Base(v.File),
            ),
            Func: filepath.Base(v.Function),
            Line: v.Line,
        }

        s[i] = f
    }

    return s
}

// fmtErr returns a slog.Value with keys `msg` and `trace`. If the error
// does not implement interface { StackTrace() errors.StackTrace }, the `trace`
// key is omitted.
func fmtErr(err error) slog.Value {
    var groupValues []slog.Attr

    groupValues = append(groupValues, slog.String("msg", err.Error()))

    frames := marshalStack(err)

    if frames != nil {
        groupValues = append(groupValues,
            slog.Any("trace", frames),
        )
    }

    return slog.GroupValue(groupValues...)
}

func main() {
    h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        ReplaceAttr: replaceAttr,
    })

    logger := slog.New(h)

    ctx := context.Background()

    err := xerrors.New("something happened")

    logger.ErrorContext(ctx, "image uploaded", slog.Any("error", err))
}
```

通过这种方式，使用xerrors.New()创建的所有错误都将被记录，并带有以下格式的堆栈跟踪信息：

**输出：**
```plan/text
{
  "time": "2024-01-03T07:09:31.013954119+01:00",
  "level": "ERROR",
  "msg": "image uploaded",
  "error": {
    "msg": "something happened",
    "trace": [
      {
        "func": "main.main",
        "source": "slog/main.go",
        "line": 82
      },
      {
        "func": "runtime.main",
        "source": "runtime/proc.go",
        "line": 267
      },
      {
        "func": "runtime.goexit",
        "source": "runtime/asm_amd64.s",
        "line": 1650
      }
    ]
  }
}
```

现在，您可以轻松地追溯到导致应用程序中任何意外错误的执行路径。

## 使用LogValuer接口隐藏敏感字段
LogValuer接口允许您通过指定自定义类型的日志记录方式来标准化您的日志输出。这是它的接口签名：


```go
type LogValuer interface {
    LogValue() Value
}
```

这个接口的一个主要应用场景是在您的自定义类型中隐藏敏感字段。例如，这里是一个没有实现LogValuer接口的User类型。注意，当这个实例被记录时，敏感信息是如何暴露的：

```go
// User does not implement `LogValuer` here
type User struct {
    ID        string `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Email     string `json:"email"`
    Password  string `json:"password"`
}

func main() {
    handler := slog.NewJSONHandler(os.Stdout, nil)
    logger := slog.New(handler)

    u := &User{
        ID:        "user-12234",
        FirstName: "Jan",
        LastName:  "Doe",
        Email:     "jan@example.com",
        Password:  "pass-12334",
    }

    logger.Info("info", "user", u)
}
```

**输出：**
```plan/text
{
  "time": "2023-02-26T22:11:30.080656774+01:00",
  "level": "INFO",
  "msg": "info",
  "user": {
    "id": "user-12234",
    "first_name": "Jan",
    "last_name": "Doe",
    "email": "jan@example.com",
    "password": "pass-12334"
  }
}
```

这种情况存在问题，因为这种类型包含了不应该出现在日志中的敏感字段（例如电子邮件和密码），同时它也可能使您的日志过于冗长。

您可以通过指定该类型在日志中应如何表示来解决这个问题。例如，您可以指定只记录ID字段，如下所示：

```go
// implement the `LogValuer` interface
func (u *User) LogValue() slog.Value {
    return slog.StringValue(u.ID)
}
```

现在您将看到如下输出：

**输出：**
```plan/text
{
  "time": "2023-02-26T22:43:28.184363059+01:00",
  "level": "INFO",
  "msg": "info",
  "user": "user-12234"
}
```

您还可以像这样对多个属性进行分组：

```go
func (u *User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.String("id", u.ID),
        slog.String("name", u.FirstName+" "+u.LastName),
    )
}
```

**输出：**
```plan/text
{
  "time": "2023-03-15T14:44:24.223381036+01:00",
  "level": "INFO",
  "msg": "info",
  "user": {
    "id": "user-12234",
    "name": "Jan Doe"
  }
}
```

## 使用第三方日志后端结合Slog
Slog的一个主要设计目标是为Go应用程序提供一个统一的日志前端（slog.Logger），而其后端（slog.Handler）则保持在不同程序中可自定义。

这种方式确保了即使后端不同，日志API在所有依赖项中也保持一致。它还避免了将日志实现与特定的包耦合，使在项目需求变化时轻松切换到不同的后端变得简单。

以下是一个示例，展示了如何将Slog前端与Zap后端结合使用，可能实现了两者的最佳结合：

```bash
$ go get go.uber.org/zap
```

```bash
$ go get go.uber.org/zap/exp/zapslog
```

```go
package main

import (
    "log/slog"

    "go.uber.org/zap"
    "go.uber.org/zap/exp/zapslog"
)

func main() {
    zapL := zap.Must(zap.NewProduction())

    defer zapL.Sync()

    logger := slog.New(zapslog.NewHandler(zapL.Core(), nil))

    logger.Info(
        "incoming request",
        slog.String("method", "GET"),
        slog.String("path", "/api/user"),
        slog.Int("status", 200),
    )
}
```

这段代码片段创建了一个新的Zap生产日志器，然后通过zapslog.NewHandler()将其用作Slog包的处理器。这样一来，您只需要使用slog.Logger上提供的方法来写日志，而生成的记录将按照所提供的zapL配置进行处理。

**输出：**
```plan/text
{"level":"info","ts":1697453912.4535635,"msg":"incoming request","method":"GET","path":"/api/user","status":200}
```

切换到不同的日志处理器非常简单，因为日志记录是通过slog.Logger来完成的。例如，您可以这样从Zap切换到Zerolog：

```bash
$ go get github.com/rs/zerolog
```

```bash
$ go get github.com/samber/slog-zerolog
```

```go
package main

import (
    "log/slog"
    "os"

    "github.com/rs/zerolog"
    slogzerolog "github.com/samber/slog-zerolog"
)

func main() {
    zerologL := zerolog.New(os.Stdout).Level(zerolog.InfoLevel)

    logger := slog.New(
        slogzerolog.Option{Logger: &zerologL}.NewZerologHandler(),
    )

    logger.Info(
        "incoming request",
        slog.String("method", "GET"),
        slog.String("path", "/api/user"),
        slog.Int("status", 200),
    )

```

**输出：**
```plan/text
{"level":"info","time":"2023-10-16T13:22:33+02:00","method":"GET","path":"/api/user","status":200,"message":"incoming request"}
```

在上述代码片段中，Zap处理器已经被一个自定义的[Zerolog处理器](https://github.com/samber/slog-zerolog)替换。由于这里的日志记录不是使用这两个库的特定API进行的，所以与在整个应用程序中更换一个日志API为另一个相比，这种迁移过程只需要几分钟。

## 书写与存储Go logs的最佳实践

一旦您配置了Slog或者您偏好的其他第三方Go日志框架，您需要遵循以下最佳实践，以确保充分利用应用程序日志：

1. **标准化日志接口**

通过实现LogValuer接口，您可以统一应用中各种类型的日志记录方式，确保它们在日志中的表现形式贯穿应用始终一致。正如文章前面所探讨的，这也是确保敏感信息不会出现在应用日志中的有效策略。

2. **为错误日志添加堆栈跟踪**

为了更好地在生产环境中调试意外问题，您应该为错误日志添加堆栈跟踪。这样做将有助于快速定位错误在代码库中的起源位置和导致问题的程序流。

目前Slog没有提供内置的方法来为错误添加堆栈跟踪，但如我们之前演示的，可以利用像[pkgerrors](https://github.com/pkg/errors)或[go-xerrors](https://github.com/MDobak/go-xerrors)这样的包以及一些辅助函数来实现这个功能。

3. **使用静态代码分析工具确保Slog语句的一致性**：

Slog API的一个主要缺点是它允许使用两种不同类型的参数，这可能导致代码库中出现不一致性。除此之外，您还可能想要确保键名的命名约定（如snake_case、camelCase等）保持一致，或者日志调用是否应始终包含上下文参数。

像sloglint这样的静态代码分析工具可以帮助您根据您偏好的代码风格来强制执行Slog的各种规则。以下是通过golangci-lint使用时的示例配置：

**.golongci.yml**
```yaml
linters-settings:
  sloglint:
    # Enforce not mixing key-value pairs and attributes.
    # Default: true
    no-mixed-args: false
    # Enforce using key-value pairs only (overrides no-mixed-args, incompatible with attr-only).
    # Default: false
    kv-only: true
    # Enforce using attributes only (overrides no-mixed-args, incompatible with kv-only).
    # Default: false
    attr-only: true
    # Enforce using methods that accept a context.
    # Default: false
    context-only: true
    # Enforce using static values for log messages.
    # Default: false
    static-msg: true
    # Enforce using constants instead of raw keys.
    # Default: false
    no-raw-keys: true
    # Enforce a single key naming convention.
    # Values: snake, kebab, camel, pascal
    # Default: ""
    key-naming-case: snake
    # Enforce putting arguments on separate lines.
    # Default: false
    args-on-sep-lines: true
```

4. **集中管理日志，但首先持久化到本地文件**

通常最好将写日志的任务与将日志发送到集中式日志管理系统的任务分离。首先将日志写入本地文件可以在日志管理系统或网络出现问题时提供备份，以防止关键数据的潜在丢失。

此外，将日志先存储在本地再发送出去可以帮助缓存日志，允许批量传输，从而有助于优化网络带宽使用并减少对应用性能的影响。

本地存储日志还提供了更大的灵活性，如果需要切换到不同的日志管理系统，只需更改日志传输方法，而不必修改整个应用的日志记录机制。有关使用如Vector或Fluentd等专业日志转发工具的更多细节，请参考我们的相关文章。

将日志记录到文件并不一定要求您配置所选择的框架直接写入文件，因为Systemd可以轻松重定向应用程序的标准输出和错误流到文件。Docker也默认会收集发送到这两个流的所有数据，并将它们路由到宿主机上的本地文件。


```go
package main

import (
    "fmt"
    "log/slog"
    "os"

    slogmulti "github.com/samber/slog-multi"
    slogsampling "github.com/samber/slog-sampling"
)

func main() {
    // Will print 20% of entries.
    option := slogsampling.UniformSamplingOption{
        Rate: 0.2,
    }

    logger := slog.New(
        slogmulti.
            Pipe(option.NewMiddleware()).
            Handler(slog.NewJSONHandler(os.Stdout, nil)),
    )

    for i := 1; i <= 10; i++ {
        logger.Info(fmt.Sprintf("a message from the gods: %d", i))
    }
}
```

**输出：**
```plan/text
{"time":"2023-10-18T19:14:09.820090798+02:00","level":"INFO","msg":"a message from the gods: 4"}
{"time":"2023-10-18T19:14:09.820117844+02:00","level":"INFO","msg":"a message from the gods: 5"}
``` 

5. **对日志进行采样**：

日志采样是一种仅记录日志条目的代表性子集而非每一个日志事件的做法。这种方法在高流量环境下非常有用，因为在这些环境中，系统会生成大量日志数据，而且处理每一条记录可能会非常昂贵。这是因为集中式日志解决方案的费用通常是基于数据摄取率或存储量来计算的。

```go
package main

import (
    "fmt"
    "log/slog"
    "os"

    slogmulti "github.com/samber/slog-multi"
    slogsampling "github.com/samber/slog-sampling"
)

func main() {
    // Will print 20% of entries.
    option := slogsampling.UniformSamplingOption{
        Rate: 0.2,
    }

    logger := slog.New(
        slogmulti.
            Pipe(option.NewMiddleware()).
            Handler(slog.NewJSONHandler(os.Stdout, nil)),
    )

    for i := 1; i <= 10; i++ {
        logger.Info(fmt.Sprintf("a message from the gods: %d", i))
    }
}
```

**输出：**
```plan/text
{"time":"2023-10-18T19:14:09.820090798+02:00","level":"INFO","msg":"a message from the gods: 4"}
{"time":"2023-10-18T19:14:09.820117844+02:00","level":"INFO","msg":"a message from the gods: 5"}
```

诸如Zerolog和Zap这样的第三方框架提供了内置的日志采样功能。在使用Slog时，您需要集成诸如[slog-sampling](https://github.com/samber/slog-sampling)之类的第三方处理器，或者开发自定义解决方案。您还可以选择使用专门的日志转发工具，如[Vector](https://vector.dev/docs/reference/configuration/transforms/sample/)，来进行日志采样。

6. **使用日志管理服务**：

将日志集中到日志管理系统中，可以方便地搜索、分析和监控应用在多个服务器和环境中的行为。当所有日志都汇聚在一个地方时，您识别和诊断问题的能力会大大提升，因为您无需在不同的服务器之间切换来搜集关于服务的信息了。

![](https://imagedelivery.net/xZXo0QFi-1_4Zimer-T0XQ/77b8e8bc-b89d-4a7a-1a89-3a2103d44600/lg1x)

虽然有很多日志管理解决方案可供选择，但[Better Stack](https://betterstack.com/logs)提供了一种快速设置集中日志管理的简单方法。它包含实时追踪日志、警报、仪表板、运行时间监控和事故管理等功能，这些都集成在一个现代直观的界面中。您可以在[这里](https://betterstack.com/logs)尝试它的完全免费计划。

结束语
我希望这篇文章能帮助您了解Go中新的结构化日志记录包，以及如何在您的项目中应用它。如果您想更深入地了解这个主题，我建议您查看完整的提案和相关的包文档。

感谢您的阅读，祝您日志记录愉快！
