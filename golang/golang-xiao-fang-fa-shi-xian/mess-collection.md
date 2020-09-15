# mess collection

### golang convert "type \[\]string" to string

`strings.Join(arr []string, seperator string) string`

```text
ips := []string
ipString := strings.Join(ips, ip)

fmt.Println(ipString)
```

### **golang 判断 IP 格式是否正确?**

```text
package main

import (
     "net"
     "fmt"
)

func main() {
   
    hostIP := "10.19.25.10"
    // ParseIP 可以用来检查 ip 地址是否正确，如果不正确，该方法返回 nil
    address := net.ParseIP(ipv4)  
    if address == nil {
         fmt.Println("ip 地址格式不正确")
    }else {
         fmt.Println("正确的 ip 地址", address.String()) 
    }
     
}
```

