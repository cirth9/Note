# Golang时间格式化

来自于知乎，懒得自己写了。



Golang时间类型通过自带的 Format 方法进行格式化。

需要注意的是Go语言中格式化时间模板不是常见的`Y-m-d H:M:S`而是使用Go语言的诞生时间 **2006-01-02 15:04:05 -0700 MST**。

为了记忆方便，按照美式时间格式 月日时分秒年 外加时区 排列起来依次是 **01/02 03:04:05PM ‘06 -0700**，刚开始使用时需要注意。

实际项目中，Format 函数中可以自定义时间格式，也可以使用time包中的预定义格式：

```go
const (
   ANSIC       = "Mon Jan _2 15:04:05 2006"
   UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
   RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
   RFC822      = "02 Jan 06 15:04 MST"
   RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
   RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
   RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
   RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
   RFC3339     = "2006-01-02T15:04:05Z07:00"
   RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
   Kitchen     = "3:04PM"
   // Handy time stamps.
   Stamp      = "Jan _2 15:04:05"
   StampMilli = "Jan _2 15:04:05.000"
   StampMicro = "Jan _2 15:04:05.000000"
   StampNano  = "Jan _2 15:04:05.000000000"
)
```

time包中，定义了年、月、日、时、分、秒、周、时区的多种表现形式：

- 年：　 06,2006
- 月份： 1,01,Jan,January
- 日：　 2,02,_2
- 时：　 3,03,15,PM,pm,AM,am
- 分：　 4,04
- 秒：　 5,05
- 周几： Mon,Monday
- 时区： -07,-0700,Z0700,Z07:00,-07:00,MST

根据以上提供的数据，我们可以组合成多种格式化模板，示例代码如下：

```go
func main() {

 currentTime := time.Now()

 fmt.Println("当前时间  : ", currentTime)

 fmt.Println("当前时间字符串: ", currentTime.String())

 fmt.Println("MM-DD-YYYY : ", currentTime.Format("01-02-2006"))

 fmt.Println("YYYY-MM-DD : ", currentTime.Format("2006-01-02"))

 fmt.Println("YYYY.MM.DD : ", currentTime.Format("2006.01.02 15:04:05"))

 fmt.Println("YYYY#MM#DD {Special Character} : ", currentTime.Format("2006#01#02"))

 fmt.Println("YYYY-MM-DD hh:mm:ss : ", currentTime.Format("2006-01-02 15:04:05"))

 fmt.Println("Time with MicroSeconds: ", currentTime.Format("2006-01-02 15:04:05.000000"))

 fmt.Println("Time with NanoSeconds: ", currentTime.Format("2006-01-02 15:04:05.000000000"))

 fmt.Println("ShortNum Month : ", currentTime.Format("2006-1-02"))

 fmt.Println("LongMonth : ", currentTime.Format("2006-January-02"))

 fmt.Println("ShortMonth : ", currentTime.Format("2006-Jan-02"))

 fmt.Println("ShortYear : ", currentTime.Format("06-Jan-02"))

 fmt.Println("LongWeekDay : ", currentTime.Format("2006-01-02 15:04:05 Monday"))

 fmt.Println("ShortWeek Day : ", currentTime.Format("2006-01-02 Mon"))

 fmt.Println("ShortDay : ", currentTime.Format("Mon 2006-01-2"))

 fmt.Println("Short Hour Minute Second: ", currentTime.Format("2006-01-02 3:4:5"))

 fmt.Println("Short Hour Minute Second: ", currentTime.Format("2006-01-02 3:4:5 PM"))

 fmt.Println("Short Hour Minute Second: ", currentTime.Format("2006-01-02 3:4:5 pm"))

}
```

输出结果:

```go
当前时间  :  2020-06-01 10:10:46.1551731 +0800 CST m=+0.002992001
当前时间字符串:  2020-06-01 10:10:46.1551731 +0800 CST m=+0.002992001
MM-DD-YYYY :  06-01-2020
YYYY-MM-DD :  2020-06-01
YYYY.MM.DD :  2020.06.01 10:10:46
YYYY#MM#DD {Special Character} :  2020#06#01
YYYY-MM-DD hh:mm:ss :  2020-06-01 10:10:46
Time with MicroSeconds:  2020-06-01 10:10:46.155173
Time with NanoSeconds:  2020-06-01 10:10:46.155173100
ShortNum Month :  2020-6-01
LongMonth :  2020-June-01
ShortMonth :  2020-Jun-01
ShortYear :  20-Jun-01
LongWeekDay :  2020-06-01 10:10:46 Monday
ShortWeek Day :  2020-06-01 Mon
ShortDay :  Mon 2020-06-1
Short Hour Minute Second:  2020-06-01 10:10:46
Short Hour Minute Second:  2020-06-01 10:10:46 AM
Short Hour Minute Second:  2020-06-01 10:10:46 am
```