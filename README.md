# [wsp](http://github.com/simplejia/wsp) (go http webserver)
## 实现初衷
* 简单可依赖，充分利用go已有的东西，不另外增加复杂、难以理解的东西，这样做的好处包括：更容易跟随go的升级而升级，降低使用者学习成本
* yii提供的controller/action的路由方式比较常用，在wsp里实现一套
* java annotation的功能挺方便，在wsp里，通过注释来实现过滤器方法的调用定义
* 不能因为wsp的引入而降低原生go http webserver的性能

## 使用场景
* 以http webserver方式对外提供服务
* 后台接口服务

## 使用案例
* 大型互联网社交业务

## 实现方式
* 路由自动生成，按要求提供controller/action的实现代码，wsp执行后会分析项目代码，自动生成路由表并记录在文件demo/WSP.go里，controller/action定义代码必须符合函数定义：func(http.ResponseWriter, *http.Request)，并且是带receiver的method
[demo_set.go](http://github.com/simplejia/wsp/tree/master/demo/controller/demo_set.go)
```
package controller

import (
	"net/http"

	"github.com/simplejia/wsp/demo/service"
)

// @prefilter("Login", {"Method":{"type":"get"}})
// @postfilter("Boss")
func (demo *Demo) Set(w http.ResponseWriter, r *http.Request) {
	key := r.FormValue("key")
	value := r.FormValue("value")
	demoService := service.NewDemo()
	demoService.Set(key, value)

	json.NewEncoder(w).Encode(map[string]interface{}{
		"code": 0,
	})
}
```

[WSP.go](http://github.com/simplejia/wsp/tree/master/demo/WSP.go)
```
// generated by wsp, DO NOT EDIT.

package main

import "net/http"
import "time"
import "github.com/simplejia/wsp/demo/controller"
import "github.com/simplejia/wsp/demo/filter"

func init() {
	http.HandleFunc("/Demo/Get", func(w http.ResponseWriter, r *http.Request) {
		t := time.Now()
		_ = t
		var e interface{}
		c := new(controller.Demo)
		defer func() {
			e = recover()
			if ok := filter.Boss(w, r, map[string]interface{}{"__T__": t, "__C__": c, "__E__": e}); !ok {
				return
			}
		}()
		c.Get(w, r)
	})

	http.HandleFunc("/Demo/Set", func(w http.ResponseWriter, r *http.Request) {
		t := time.Now()
		_ = t
		var e interface{}
		c := new(controller.Demo)
		defer func() {
			e = recover()
			if ok := filter.Boss(w, r, map[string]interface{}{"__T__": t, "__C__": c, "__E__": e}); !ok {
				return
			}
		}()
		if ok := filter.Login(w, r, map[string]interface{}{"__T__": t, "__C__": c, "__E__": e}); !ok {
			return
		}
		if ok := filter.Method(w, r, map[string]interface{}{"type": "get", "__T__": t, "__C__": c, "__E__": e}); !ok {
			return
		}
		c.Set(w, r)
	})

}
```

* wsp分析项目代码，寻找符合要求的注释（见demo/controller/demo_set.go），自动生成过滤器调用代码在文件demo/WSP.go里，filter注解分为前置过滤器(prefilter）和后置过滤器（postfilter），格式如：@prefilter({json body})，{json body}代表传入参数，符合json array定义格式（去掉前后的中括号），可以包含string值或者object值，filter函数定义满足：func (http.ResponseWriter, *http.Request, map[string]interface{}) bool，过滤器函数如下：
[method.go](http://github.com/simplejia/wsp/tree/master/demo/filter/method.go)
```
package filter

import (
	"net/http"
	"strings"
)

func Method(w http.ResponseWriter, r *http.Request, p map[string]interface{}) bool {
	method, ok := p["type"].(string)
	if ok && strings.ToLower(r.Method) != strings.ToLower(method) {
		http.Error(w, "405 Method Not Allowed", http.StatusMethodNotAllowed)
		return false
	}
	return true
}
```

> filter输入参数map[string]interface{}，会自动设置"__T__"，time.Time类型，值为执行起始时间，可用于耗时统计，"__C__"，*{Controller}类型，值为*{Controller}实例，可通过接口方式存取相关数据（这种方式存取数据较context方式更简单实用），"__E__"，值为recover()返回值，用于检测错误并处理（后置过滤器必须recover()）

* 项目main.go代码示例
[main.go](http://github.com/simplejia/wsp/tree/master/demo/main.go)
```
package main

import (
	"log"

	"github.com/simplejia/clog"
	"github.com/simplejia/lc"

	"net/http"

	_ "github.com/simplejia/wsp/demo/clog"
	_ "github.com/simplejia/wsp/demo/conf"
	_ "github.com/simplejia/wsp/demo/mysql"
	_ "github.com/simplejia/wsp/demo/redis"
)

func init() {
	lc.Init(1e5)

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		http.NotFound(w, r)
	})
}

func main() {
	clog.Info("main()")

	log.Panic(http.ListenAndServe(":8080", nil))
}
```

## miscellaneous
* 通过wrk压测工具在同样环境下（8核，8g），wsp空跑qps：9万，beego1.7.1空跑qps：5.5万
* 更方便加入middleware（func(http.Handler) http.Handler），其实更推荐通过定义过滤器的方式支持类似功能
* 更方便编写如下的测试用例：
  * [test](http://github.com/simplejia/wsp/tree/master/demo/test) (测试用例运行时需要用到项目配置文件，所以请在test目录生成../clog,../conf,../mysql,../redis的软链接)

---

# [demo](http://github.com/simplejia/wsp/tree/master/demo)
* 提供一个简单易扩展的项目stub

## 实现初衷
* 简单可依赖，充分利用go已有的东西，不另外增加复杂、难以理解的东西，这样做的好处包括：更容易跟随go的升级而升级，降低使用者学习成本
* 提供常用组件的简单包装，如下：
  * config，提供项目主配置文件自动解析，见[conf](http://github.com/simplejia/wsp/tree/master/demo/conf)，可以通过传入自定义的env及conf参数来重定义配置文件里的参数，如：./demo -env dev -conf='app.port=8080::clog.mode=1'，多个参数用`::`分隔
  * redis，使用(github.com/garyburd/redigo)，提供配置文件自动解析，见[redis](http://github.com/simplejia/wsp/tree/master/demo/redis)
  * mysql，使用(database/sql)，提供配置文件自动解析，见[mysql](http://github.com/simplejia/wsp/tree/master/demo/mysql)，同时为了方便对象映射，提供了最常用的orm组件供选择使用，见[orm](http://github.com/simplejia/orm)

## 项目编写指导意见
* 目录结构：
```
├── WSP.go
├── clog
│   └── clog.go
├── conf
│   ├── conf.go
│   └── conf.json
├── controller
│   ├── base.go
│   ├── demo.go
│   ├── demo_get.go
│   └── demo_set.go
├── demo
├── filter
│   ├── boss.go
│   ├── login.go
│   └── method.go
├── main.go
├── model
│   ├── demo.go
│   ├── demo_get.go
│   └── demo_set.go
├── mysql
│   ├── demo_db.json
│   └── mysql.go
├── redis
│   ├── demo.json
│   └── redis.go
├── service
│   ├── demo.go
│   ├── demo_get.go
│   └── demo_set.go
└── test
    ├── clog -> ../clog
    ├── conf -> ../conf
    ├── demo_get_test.go
    ├── demo_set_test.go
    ├── init_test.go
    ├── mysql -> ../mysql
    └── redis -> ../redis
```
  * controller目录：负责request参数解析，service调用
  * service目录：负责逻辑处理，model调用
  * model目录：负责数据处理
* 接口实现上，建议一个接口对应一个文件，如controller/demo_get.go, service/demo_get.go, model/demo_get.go


## LICENSE
wsp is licensed under the Apache Licence, Version 2.0
(http://www.apache.org/licenses/LICENSE-2.0.html)
