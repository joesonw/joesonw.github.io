---
layout: post
title: Go Pitfalls and Patterns (持续更新中)
top: true
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
```go
m := map[string]interface{}{}
json.Unmarshal(data, &m)
println(m["message"].(string)) // 这里转换失败会panic, 除非 message, ok := m["message"].(string)
```

```go
m := struct{
	Message string `json:"message"`
}{}
json.Unmarshal(data, &m)
println(m.Message)
```
