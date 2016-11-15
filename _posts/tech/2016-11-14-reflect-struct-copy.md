---
layout: post
title: 通过反射实现结构体拷贝
category: 技术
tags: Go
description: 
---

下面这个函数实现的是一个通过反射，完成两个结构体之间的拷贝。只要两个结构体之间的成员名称一致，那么就会自动根据其类型进行相应的转换判断。例如，如果类型为time.Time，目的类型为string，则自动转换为"2006-01-02 15:04:05"
值得注意的是，此函数我们必须传入结构体的引用，如果直接传入结构体，则不会实现结构体的拷贝，也就是通常意义上的传值调用。如果我们传入的不是指针，下面的Set操作会panic。
当然下面的函数效率是非常低的，相比我们直接赋值来说，效率大概降低了上千倍。

```go
func StructClone(src, dst interface{}) {
    t1 := reflect.TypeOf(src).Elem()
    t2 := reflect.TypeOf(dst).Elem()
    v1 := reflect.ValueOf(src).Elem()
    v2 := reflect.ValueOf(dst).Elem()

    for i := 0; i < t1.NumField(); i++ {
        t1_temp := t1.Field(i)
        name := t1_temp.Name
        t2_temp, ok := t2.FieldByName(name)
        if !ok {
            continue
        }

        v1_temp := v1.FieldByName(name)
        v2_temp := v2.FieldByName(name)

        if t2_temp.Type == t1_temp.Type {
            v2_temp.Set(v1_temp)
        }else {

            switch t2_temp.Type.Kind() {
            case reflect.String:
                switch t1_temp.Type.Kind() {
                case reflect.Int:
                    v2_temp.SetString(v1_temp.String())
                case reflect.TypeOf(time.Time{}).Kind():
                    temp := v1_temp.Interface().(time.Time)
                    v2_temp.SetString(temp.Format("2006-01-02 15:04:05"))
                }
            case reflect.Int :
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    temp, err := strconv.ParseInt(v1_temp.String(), 10, 64)
                    if err == nil {
                        v2_temp.SetInt(temp)
                    }
                }
            case reflect.Int64 :
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    temp, err := strconv.ParseInt(v1_temp.String(), 10, 64)
                    if err == nil {
                        v2_temp.SetInt(temp)
                    }
                case reflect.Int:
                    v2_temp.SetInt(v1_temp.Int())
                case reflect.TypeOf(time.Time{}).Kind():
                    v2_temp.SetInt(v1_temp.Interface().(time.Time).Unix())
                }
            case reflect.TypeOf(time.Time{}).Kind():
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    temp, err := time.ParseInLocation("2006-01-02 15:04:05", v1_temp.String(), time.Local)
                    if err == nil {
                        v2_temp.Set(reflect.ValueOf(temp))
                    }
                case reflect.Int:
                    v2_temp.Set(reflect.ValueOf(time.Unix(v1_temp.Int(), 0)))
                case reflect.Int64:
                    v2_temp.Set(reflect.ValueOf(time.Unix(v1_temp.Int(), 0)))
                }
            }
        }
    }
}
```


实际上之前写了这个函数，主要原因是因为懒。。。那么我们可以想一下，如果做一个自动生成golang的结构体拷贝代码的函数，则可以避免效率的降低以及拷贝结构体时的一些低级错误。
下面这个函数可以用于生成结构体拷贝的相关go代码，值得注意的是下面这个接口必须要手动传入目的参数和源参数的名字，目前golang还不支持通过interface获取原入参的名字。

```go
func StructClone2(src, dst interface{},src_name,dst_name string) string {
    t1 := reflect.TypeOf(src).Elem()
    t2 := reflect.TypeOf(dst).Elem()

    var result string


    for i := 0; i < t1.NumField(); i++ {
        t1_temp := t1.Field(i)
        name := t1_temp.Name
        t2_temp, ok := t2.FieldByName(name)
        if !ok {
            continue
        }

        if t2_temp.Type == t1_temp.Type {
            result = fmt.Sprintf("%s    %s.%s = %s.%s\n", result, dst_name, name, src_name, name)
        }else {
            switch t2_temp.Type.Kind() {
            case reflect.String:
                switch t1_temp.Type.Kind() {
                case reflect.Int:
                    result = fmt.Sprintf("%s    %s.%s = strconv.Itoa(%s.%s)\n", result, dst_name, name, src_name, name)
                case reflect.TypeOf(time.Time{}).Kind():
                    result = fmt.Sprintf("%s    %s.%s = %s.%s.Format(\"2006-01-02 15:04:05\")\n", result, dst_name, name, src_name, name)
                default:
                    result = fmt.Sprintf("%s    //TODO\n    %s.%s With %s.%s\n", result, dst_name, name, src_name, name)
                }
            case reflect.Int :
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    result = fmt.Sprintf("%s    temp, err := strconv.ParseInt(%s.%s, 10, 64)\n    %s.%s = temp\n", result, src_name, name, dst_name, name)
                case reflect.TypeOf(time.Time{}).Kind():
                    result = fmt.Sprintf("%s    %s.%s = int(%s.%s.Unix())\n", result, dst_name, name, src_name, name)
                default:
                    result = fmt.Sprintf("%s    //TODO\n    %s.%s With %s.%s\n", result, dst_name, name, src_name, name)
                }
            case reflect.Int64 :
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    result = fmt.Sprintf("%s    temp, err := strconv.ParseInt(%s.%s, 10, 64)\n    %s.%s = temp\n", result, src_name, name, dst_name, name)
                case reflect.Int:
                    result = fmt.Sprintf("%s    %s.%s = int64(%s.%s)\n", result, dst_name, name, src_name, name)
                case reflect.TypeOf(time.Time{}).Kind():
                    result = fmt.Sprintf("%s    %s.%s = %s.%s.Unix()\n", result, dst_name, name, src_name, name)
                default:
                    result = fmt.Sprintf("%s    //TODO\n    %s.%s With %s.%s\n", result, dst_name, name, src_name, name)
                }
            case reflect.TypeOf(time.Time{}).Kind():
                switch t1_temp.Type.Kind() {
                case reflect.String:
                    result = fmt.Sprintf("%s    temp2, err := time.ParseInLocation(\"2006-01-02 15:04:05\", %s.%s, time.Local)\n    %s.%s = temp2\n", result, src_name, name, dst_name, name)
                case reflect.Int:
                    result = fmt.Sprintf("%s    %s.%s = time.Unix(%s.%s,0)\n", result, dst_name, name, src_name, name)
                case reflect.Int64:
                    result = fmt.Sprintf("%s    %s.%s = time.Unix(%s.%s,0)\n", result, dst_name, name, src_name, name)
                default:
                    result = fmt.Sprintf("%s    //TODO\n    %s.%s With %s.%s\n", result, dst_name, name, src_name, name)
                }
            }
        }
    }

    return result
}
```
