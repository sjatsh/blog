---
title: "Go项目实战总结"
description: ""
tags: "golang"
date: "2017-09-25"
categories: "golang"
---

### GOPATH
go get、go install等命令都会用到gopath，gopath告诉go命令和和其他相关工具到哪找安装在你系统上的go包。  

<!--more-->

windows设置gopath
```
set GOPATH=c:/Users/user/go;d:/workspace/mygo
```
linux设置gopath
```
export GOPATH=/home/user/go:/usr/local/mygo
```

gopath也是工作目录，用户源码文件放在gopath/src下,go install 生成的可执行文件在gopath/bin下，go install产生的
中间文件.a文件在gopath/pkg下，go install编译器会判断源码是否有改动，如果没有改动就不会重新生成.a文件，可以加快编译速度。

### vendor
golang版本依赖是个很头疼的问题，如果我们的项目依赖某一个特定的版本，或者第三方库不满足我们需要但是又很难让作者做出改动的时候，
依赖问题都是个痛点。通过go get 获取的依赖在不同的电脑上可能出现编译不通过的情况。  
go1.5后提供了GO15VENDOREXPERIMENT环境变量，用于将go build、go install时的搜索路径修改为```当前目录/vendor```下，
通过依赖项目vendor目录可以依赖一些特定版本的第三方依赖，或者是自己做出修改的依赖包但是又不属于项目本身业务代码，就可以放在
vendor目录下。

### bindata的使用
我们的项目中可能带有前端页面，虽然是可以前后端分离，但是前端页面比较简单就没有必要单独分出来。如果前端代码也能直接编译到go的可执行
程序中那不是一件很爽的事么，这就用到了go-bindata，他会把静态文件生成为.go文件。[go-bindata](https://github.com/jteeuwen/go-bindata)
[go-bindata-assetfs](https://github.com/elazarl/go-bindata-assetfs)  
go-bindata与go-bindata-assetfs的区别:  
- go-bindata仅仅用于生成bindata文件提供基本的方法  
- go-bindata-assetfs依赖go-bindata并提供方法让我们能够很轻松的向外提供FileServer

在项目中我们如何使用呢：
```
go-bindata -pkg=asserts -o src/isesol.com/agentServer/asserts/bindata.go asserts/...
```
 - pkg:指定生成的.go文件的package名
 - o:指定生成的.go文件的路径已经名称
 - asserts/...:asserts静态文件目录,...循环迭代asserts目录下的所有文件
 
 自己的代码中
 ```
 var fsh http.Handler
 if config.AgentConfig.LocalTest {
 	fsh = http.FileServer(http.Dir("asserts"))
 } else {
 	fsh = http.FileServer(&assetfs.AssetFS{Asset: asserts.Asset, AssetDir: asserts.AssetDir, Prefix: "asserts"})
 }
 r.PathPrefix("/").Handler(fsh)
 ```
我们可以通过配置项指定使用本地静态文件还是生成的bindata文件，需要注意的是其中: ```Prefix: "asserts"```asserts一定要与静态文件目录同名。
 
### xorm的使用
xorm是一个数据库框架，我们的项目使用sqlite数据库，但是有个很恶心的地方就是sqlite3的驱动部分代码确是c，这导致windows上编译不了其他平台的代码。
毕竟数据库发展这么多年，以前的数据库都是c写的，想要把几M的c代码全部替换成go确实也不是一件容易的事。  
xorm使用过程中的坑：
- 数据库主键自增:数据库设置了主键自增，我们insert的时候不设置主键值默认是0，struct中id字段tag必须加上autoincr
- 多表联合查询时:如果不同struct中有相同的字段
```
struct A {
 ID int64 `xorm:"id" json:"id"`
 Name string `xorm:"name" json:"name"`
}
struct B {
 ID id int64 `json:"id"`
}
struct C {
 A
 B
}
var c C
xorm.Find(&c)
```
A、B中都有id字段并且json的tag都是id，查询出来的结果会发现没有id字段，如果前端需要使用id字段那就会有问题,解决办法是在struct C中添加ID字段就可以了。     


- 表名与struct名不一致，查询时一定要指定表名
- 分页操作需要count获取总条数，有时候我们查询时可能需要指定哪些字段，Select()语句不要放在Count()之前，否则生成的sql会变成select Select()中的字段
而不是select count(*)
- 复杂的查询使用sql语句
- xorm查询时不要new一个变量，应为查询时xorm会new一个，相当于申请了两次内存，会造成内存内存浪费。

### go中cgi接口的设计
go中实现一个web服务非常的容易:
```
http.HandlerFunc("/test",func(w http.ResponseWriter, r *http.Request){

})
http.HandlerFunc("/cgi",func(w http.ResponseWriter, r *http.Request){

})
http.ListenAndServe(":8080", nil)
```
几行代码就可以实现一个http服务器，test、cgi就是对应的处理函数用于处理请求。但是如果我们想设计一个统一的cgi接口作为请求的唯的一入口而不是添加一个又一个HandlerFunc，
该怎么做呢，首先为什么要设计一个post请求的cgi接口，以及他的好处：
- 用户不能通过url探测接口功能
- 参数不会显式的传递，一定程度上保证了接口安全性
- 即使接口被探测到，修改接口也非常方便，只需要修改body内容而不需要url地址  
实现：  
我们可以使用cgi作为统一的入口，直接上代码：
```
func cgi(w http.ResponseWriter, r *http.Request) {
    if r.Body == nil {
    	util.JSON(w, map[string]string{"errorCode": "-1", "errorInfo": "systemError"})
    	return
    }
    defer r.Body.Close()
    
    var req map[string]interface{}
    err = jsoniter.NewDecoder(r.Body).Decode(&req)
    if err != nil {
    	util.JSON(w, map[string]string{"errorCode": "-1", "errorInfo": "systemError"})
    	return
    }
    cmd := strings.TrimPrefix(req["cmd"].(string), constant.ServiceName)
    if fn, ok := util.Cgi.Get(cmd); ok {
    	res = fn(req)
    } else {
    	util.JSON(w, model.RestError{ErrorCode: "-1", ErrorInfo: "cmd not found"})
    	return
    }
    
    switch res.(type) {
    case error:
    	util.JSON(w, model.RestError{ErrorCode: "-1", ErrorInfo: res.(error).Error()})
    case model.RestError:
    	util.JSON(w, res)
    default:
    	util.JSON(w, map[string]interface{}{"cmd": constant.ServiceName + cmd, "response": res})
    }
}
```
将request body中内容解析出来，cmd指向对应处理函数，我们通过一个map[string]interface,其中key是cmd的值value是
接口类型。service包中通过
```
func init() {
    util.Cgi.Register("test/test", service)
}
func service(req interface{}) (interface{}) {
    //业务代码
}
```
来实现处理方法的注册，这里用到的其实是接口值。  
  
结果通过res返回，这里一个很重要的地方就是接口的类型判断，通过switch res.(type)判断返回类型是系统异常、业务异常、正常返回
中的哪种。这样一个简单的cgi接口就完成了。

### 如何统一返回格式化时间   
```
package model

// JSONTime JSONTime
type JSONTime time.Time

// MarshalJSON MarshalJSON
func (t JSONTime) MarshalJSON() ([]byte, error) {
	stamp := fmt.Sprintf("\"%s\"", time.Time(t).Format("2006-01-02 15:04:05"))
	return []byte(stamp), nil
}

//model
type Subscription struct {
	ID          int64          `xorm:"id pk autoincr" json:"id"`
	MachineID   string         `xorm:"machine_id" json:"machineId"`
	MsgID       int64          `xorm:"msg_id" json:"msgId"`
	MessageID   string         `xorm:"message_id" json:"messageId"`
	UsingStatus string         `json:"usingStatus"`
	MachineType string         `json:"machineType"`
	Status      string         `json:"status"`
	RuleName    string         `json:"ruleName"`
	GmtCreate   model.JSONTime `xorm:"created" json:"-"`
	GmtModify   model.JSONTime `xorm:"created updated" json:"-"`
}
json.Marshal(sub)
```   
当我们调json.Marshal时，其实就是调了这里我们这里我们自己实现的MarshalJSON方法，我们一层一层进去会发现。  
```
// Marshaler is the interface implemented by types that
// can marshal themselves into valid JSON.
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}
var (
	marshalerType     = reflect.TypeOf(new(Marshaler)).Elem()
	textMarshalerType = reflect.TypeOf(new(encoding.TextMarshaler)).Elem()
)
// newTypeEncoder constructs an encoderFunc for a type.
// The returned encoder only checks CanAddr when allowAddr is true.
func newTypeEncoder(t reflect.Type, allowAddr bool) encoderFunc {
	if t.Implements(marshalerType) {
    		return marshalerEncoder
    	}
    	if t.Kind() != reflect.Ptr && allowAddr {
    		if reflect.PtrTo(t).Implements(marshalerType) {
    			return newCondAddrEncoder(addrMarshalerEncoder, newTypeEncoder(t, false))
    		}
    	}
    
    	if t.Implements(textMarshalerType) {
    		return textMarshalerEncoder
    	}
    	if t.Kind() != reflect.Ptr && allowAddr {
    		if reflect.PtrTo(t).Implements(textMarshalerType) {
    			return newCondAddrEncoder(addrTextMarshalerEncoder, newTypeEncoder(t, false))
    		}
    	}
    
    	switch t.Kind() {
    	case reflect.Bool:
    		return boolEncoder
    	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
    		return intEncoder
    	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
    		return uintEncoder
    	case reflect.Float32:
    		return float32Encoder
    	case reflect.Float64:
    		return float64Encoder
    	case reflect.String:
    		return stringEncoder
    	case reflect.Interface:
    		return interfaceEncoder
    	case reflect.Struct:
    		return newStructEncoder(t)
    	case reflect.Map:
    		return newMapEncoder(t)
    	case reflect.Slice:
    		return newSliceEncoder(t)
    	case reflect.Array:
    		return newArrayEncoder(t)
    	case reflect.Ptr:
    		return newPtrEncoder(t)
    	default:
    		return unsupportedTypeEncoder
    	}
}
```  
这里会判断我们是不是实现了Marshaler接口，如果实现了就返回这个encoderFunc，否则t.Kind()判断是哪种类型返回对应的Encoder。  

### mapstructure使用
[mapstructure](https://github.com/mitchellh/mapstructure)
```
m.(map[string]interface{})["parameters"]
```
parameters参数与struct映射存在一个问题就是，后台struct使用string字段去接收时，前端传一个int型的值，会发生类型转换错误，一起开始
没有发现解决方法。  
后来发现mapstructure有一个配置项
```
mapstructure.DecoderConfig.WeaklyTypedInput
```
默认是false，设置成true工具内部就会帮我们进行类型转换，前端传来的int或float类型会自动帮我们转成struct中对应的类型。  

爽了！不用再去修改前端代码了！^_^

### negroni中间件
Negroni 不是一个框架，它是为了方便使用 net/http 而设计的一个库而已。  
可以用 Use 函数把这些 http.Handler 处理器引进到处理器链上来：
```
mux := http.NewServeMux()

n := negroni.New()
n.Use(negroni.HandlerFunc(Test1))
n.Use(negroni.HandlerFunc(Test2))

n.UseHandler(mux)

n.Run(":8080")
```
这个功能非常的好用，如果我们要在所有的处理函数之前添加一个权限校验或者其他的全局处理的handler，name这个功能是一个很好的选择。

### 内嵌ngrok与源码修改
基于ngrok 1.7  
- 添加/api/tunnels接口，显示本地暴露端口
client/views/web/http.go: register方法添加  
```
http.HandleFunc("/api/tunnels", func(w http.ResponseWriter, r *http.Request) {
    payloadData := whv.ctl.State().GetTunnels()
	payload, err := json.Marshal(payloadData)
	if err != nil {
		panic(err)
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(payload)
})
```
- 去除ngrok启动页面  

```
166 //var termView *term.TermView
167 //if config.LogTo != "stdout" {
168 //  termView = term.NewTermView(ctl)
169 //  ctl.AddView(termView)
170 //}

175 //if termView != nil {
176 //	ctl.AddView(termView.NewHttpView(p))
177 //}
```  
- 修改启动配置项
client/main.go: 部分代码注释  

```
//func init() {
//	if runtime.GOOS == "windows" {
//		if mousetrap.StartedByExplorer() {
//			fmt.Println("Don't double-click ngrok!")
//			fmt.Println("You need to open cmd.exe and run it from the command line!")
//			time.Sleep(5 * time.Second)
//			os.Exit(1)
//		}
//	}
//}
```
- client/cli.go: ParseArgs方法代码修改为  

```
opts = &Options{}
	configS := strings.Fields(config.AgentConfig.NgrokArgs)
	confMap := make(map[string]string, 0)
	commandSlice := make([]string, 0)
	for _, v := range configS {

		if strings.HasPrefix(v, "-") {

			v = strings.TrimPrefix(v, "-")
			if strings.Contains(v, "=") {
				cfgVal := strings.Split(v, "=")
				confMap[cfgVal[0]] = cfgVal[1]
			} else {
				confMap[v] = ""
			}

		} else {
			commandSlice = append(commandSlice, v)
		}
	}
	if v, ok := confMap["config"]; ok {
		opts.config = v
	}
	if v, ok := confMap["log"]; ok {
		opts.logto = v
	}
	if v, ok := confMap["log-level"]; ok {
		opts.loglevel = v
	}
	if v, ok := confMap["authtoken"]; ok {
		opts.authtoken = v
	}
	if v, ok := confMap["httpauth"]; ok {
		opts.httpauth = v
	}
	if v, ok := confMap["subdomain"]; ok {
		opts.subdomain = v
	}
	if v, ok := confMap["hostname"]; ok {
		opts.hostname = v
	}

	opts.command = commandSlice[0]

	switch opts.command {
	case "list":
		opts.args = commandSlice[1:]
	case "start":
		opts.args = commandSlice[1:]
	case "start-all":
		opts.args = commandSlice[1:]
	case "version":
		fmt.Println(version.MajorMinor())
		os.Exit(0)
	case "help":
		flag.Usage()
		os.Exit(0)
	case "":
		fmt.Errorf("Error: Specify a local port to tunnel to, or " +
			"an ngrok command.\n\nExample: To expose port 80, run " +
			"'ngrok 80'")
		return

	default:
		if len(commandSlice) > 1 {
			fmt.Errorf("You may only specify one port to tunnel to on the command line, got %d: %v",
				len(commandSlice),
				commandSlice)
			return
		}

		opts.command = "default"
		opts.args = commandSlice
	}

	return
```  
- client/config.go：LoadConfiguration代码修改  

```
if matched, err = regexp.MatchString("^[0-9a-zA-Z_\\-!]+$", content); err != nil {}
之前代码替换成：

configPath := opts.config
if configPath == "" {
	configPath = defaultPath()
}

log.Info("Reading configuration file %s", configPath)
// deserialize/parse the config
config = &Configuration{}
// try to parse the old .ngrok format for backwards compatibility
matched := false
var content string
//config from file
if opts.config != "" {

	configBuf, err := ioutil.ReadFile(configPath)
	if err != nil {
		panic("Failed to read configuration file " + configPath + ": " + err.Error())
	}
	if err = yaml.Unmarshal(configBuf, &config); err != nil {
		panic("Error parsing configuration file " + configPath + ": " + err.Error())
	}
	content = strings.TrimSpace(string(configBuf))

} else {

	config = &Configuration{
		ServerAddr:         "tunnel.qydev.com:4443",
		TrustHostRootCerts: false,
		Tunnels: map[string]*TunnelConfiguration{
			"ssh": {
				Protocols: map[string]string{"tcp": "127.0.0.1:22"},
			},
			"web": {
				Subdomain: strings.TrimPrefix(agentConf.AgentConfig.BoxName, "LocalBox-"),
				Protocols: map[string]string{"http": "80"},
			},
		},
	}
}

for name, t := range config.Tunnels {
  //添加
  if "web" == name {
  	t.Subdomain = strings.TrimPrefix(agentConf.AgentConfig.BoxName, "LocalBox-")
  }
  for k, addr := range t.Protocols {
     //添加
     if "http" == k {
     	t.HttpAuth = agentConf.AgentConfig.BoxUserName + ":" + agentConf.AgentConfig.BoxPassword
     }
  }
}
```


### 代码扫描工具   

```
go get github.com/alecthomas/gometalinter 
gometalinter --install --update
gometalinter ./... --vendor -e asserts --fast
```   