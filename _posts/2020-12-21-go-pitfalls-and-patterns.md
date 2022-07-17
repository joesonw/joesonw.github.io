---
layout: post
title: Go Pitfalls and Patterns
---

## 1. 如何获取interface的reflect.Type

```go
type Duck interface {
	Quack()
}

var reflectTypeOfDuck = reflect.TypeOf(struct { Duck }).Field(0).Type()

func IsDuck(val reflect.Value) bool {
	return val.Type().Implements(reflectTypeOfDuck)
}
```

## 2. 不要用map[string]interface{}来Unmarshal
避免下面的用法
```go
m := map[string]interface{}{}
json.Unmarshal(data, &m)
println(m["message"].(string)) // 这里转换失败会panic, 除非 message, ok := m["message"].(string)
```
尝试使用下面的用法
```go
m := struct{
	Message string `json:"message"`
}{}
json.Unmarshal(data, &m)
println(m.Message)
```

## 3. defer时机

```go
func f() {
	defer println("a")
	{
		defer println("b")
	}
	if true {
		defer println("c")
	}
	for i := 0; i < 3; i++ {
		defer println(i)
	}
	defer println("end")
}
/*
end
2
1
0
c
b
a
*/
```
[# Go Playgound](https://play.golang.org/p/sJgplht4L6h)

注意 `{}` 中的`defer` (包括 `if`, `for` 中) 均会在函数返回时执行, 而不是出当前`作用域`
