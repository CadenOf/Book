# golang 字符串大小写转换

1. **func ToLower\(s string\) string**  将字符串s转换成小写返回 
2. **func ToUpper\(s string\) string** 将字符串s转换成大写返回 
3. **func Title\(s string\) string** 将字符串s每个单词首字母大写返回 
4. **func ToTitle\(s string\) string** 将字符串s转换成大写返回 
5. **func ToLowerSpecial\(\_case unicode.SpecialCase, s string\) string** 将字符串s中所有字符按\_case指定的映射转换成小写返回 
6. **func ToUpperSpecial\(\_case unicode.SpecialCase, s string\) string** 将字符串s中所有字符按\_case指定的映射转换成大写返回 
7. **func ToTitleSpecial\(\_case unicode.SpecialCase, s string\) string** 将字符串s中所有字符按\_case指定的映射转换成大写返回

```text
package main

import (
	"fmt"
	"strings"
	"unicode"
)


func main() {

    // 将字符串s转换成小写返回
    fmt.Println(strings.ToLower("string to lower"))

    //将字符串s转换成大写返回
    fmt.Println(strings.ToUpper("string to upper"))

    //将字符串s每个单词首字母大写返回
    fmt.Println(strings.Title("string title"))

    //将字符串s每个字母都转换成大写返回
    fmt.Println(strings.ToTitle("string to titile"))

	//将字符串s中所有字符按_case指定的映射转换成小写返回
	var _MyCase = unicode.SpecialCase{
		// 将半角逗号替换为全角逗号，ToTitle 不处理
		unicode.CaseRange{',', ',',
			[unicode.MaxCase]rune{'，' - ',', '，' - ',', 0}},
		// 将半角句号替换为全角句号，ToTitle 不处理
		unicode.CaseRange{'.', '.',
			[unicode.MaxCase]rune{'。' - '.', '。' - '.', 0}},
		// 将 ABC 分别替换为全角的 ＡＢＣ、ａｂｃ，ToTitle 不处理
		unicode.CaseRange{'A', 'C',
			[unicode.MaxCase]rune{'Ａ' - 'A', 'ａ' - 'A', 0}},
	}

	fmt.Println(strings.ToLowerSpecial(_MyCase, "string ,.- ABC tolowerspecial"))
	fmt.Println(strings.ToUpperSpecial(_MyCase, "string ,.- ABC toupperspecial"))
	fmt.Println(strings.ToTitleSpecial(_MyCase, "string ,.- ABC totitlespecial"))

}

Result:

string to lower
STRING TO UPPER
String Title
STRING TO TITILE
string ，。- ａｂｃ tolowerspecial
STRING ，。- ＡＢＣ TOUPPERSPECIAL
STRING ,.- ABC TOTITLESPECIAL

```

