---
title: golang的一个自重写程序
date: 2020-06-01 23:49:00
categories:
  - tutorial
tags: 
  - golang
  - self-reproducing programs
---

## golang的一个自重写程序

今日在看书时，突然想用golang尝试一下**自重写**. 

所谓自重写，就是程序打印的结果为程序源代码本身. 听起来似乎不难, 只要用文件系统读取源代码文件并打印即可， 不过这样就完全没有挑战了不是吗😂

我自己实现的版本如下, 虽然不优雅，但至少起到了作用.
```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	code := "package main\n\nimport (\n    \"fmt\"\n    \"strings\"\n)\n\nfunc main() {\n    code := %s\n    tmp := strings.Replace(code, \"\\\\\", \"\\\\\\\\\", -1)\n    tmp = strings.Replace(tmp, \"\\n\", \"\\\\n\", -1)\n    tmp = strings.Replace(tmp, \"\\\"\", \"\\\\\\\"\", -1)\n    code = fmt.Sprintf(code, fmt.Sprintf(\"\\\"%%s\\\"\", tmp))\n    println(code)\n}"
	tmp := strings.Replace(code, "\\", "\\\\", -1)
	tmp = strings.Replace(tmp, "\n", "\\n", -1)
	tmp = strings.Replace(tmp, "\"", "\\\"", -1)
	code = fmt.Sprintf(code, fmt.Sprintf("\"%s\"", tmp))
	println(code)
}
```



广为流传的有[rsc](https://research.swtch.com/zip)提供的实现, 确实更为精简

```go
/* Go quine */
package main
import "fmt"
func main() {
    fmt.Printf("%s%c%s%c\n", q, 0x60, q, 0x60)
}
var q = `/* Go quine */
package main
import "fmt"
func main() {
    fmt.Printf("%s%c%s%c\n", q, 0x60, q, 0x60)
}
var q = `
```



而[golang-nuts](http://groups.google.com/group/golang-nuts)提供了更短的代码, 思路和rsc的类似，不过没有经过`go fmt`，写成一行的话看起来也太奇怪了.

```go
package main;func main(){print(c+"\x60"+c+"\x60")};var c=`package main;func main(){print(c+"\x60"+c+"\x60")};var c=`
```


