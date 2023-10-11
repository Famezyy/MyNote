# Java8日期API
## 1.Java8以前API存在的问题
`Date`、`DateFormat`、`Calendar`都是 Java8 以前的日期 API，存在以下问题：

1. 他们都是可变的，有线程安全问题。

  例如以下代码会产生`NumberFormatException`：

  ```java
  public class SImpleDateFormatTest {
      public static void main(String[] args) {
          DateFormat dateFormat = new SimpleDateFormat("yyyy/MM/dd");
          for (int i = 0; i < 30; i++) {
              Thread thread = new Thread(() -> {
                  try {
                  Date parseDate = dateFormat.parse("2022/06/15");
                  System.out.println(parseDate);
                  } catch (ParseException e) {
                      e.printStackTrace();
                  }
              });
              thread.start();
          }
      }
  }
  ```
2. `Date`既表示日期，也表示时间，指责混乱。
3. `Calendar`的月份是从 0 开始的，星期是从星期天开始的。

由于以上问题，JDK 开发人员开发了一套全新的日期时间 API，位于`java.time`包下。常用的 API 如下：

`LocalDate`、`LocalDateTime`、`Period`、`Duration`、`LocalTime`、`Instant`、`ZonedDateTime`、`OffsetDateTime`等。

Java8 日期 API 参考了`joda time`的实现，它是一个开源的日期 API。

## 2.LocalDate的使用
### 2.1 介绍
`java.time.LocalDate`表示没有时区的日期，是不可变的，也就是线程安全的。

### 2.2 创建LocalDate
```java
public class LocalDateCreate {
    public static void main(String[] args) {
        // 当前的日期信息：2022-6-15
        LocalDate now = LocalDate.now();
        System.out.println(now);
        
        // 传入年、月、日
        LocalDate anyDate = LocalDate.of(2022, 6, 15);
        System.out.println(anyDate);
    }
}
```
### 2.3 常用方法
#### 2.3.1 获取日期相关信息
```java
public class LocalDateGetDateInfo {
    public static void main(Stirng[] args) {
        LocalDate now = LocalDate.now();
        
        // 获取年、月、日
        int year = now.getYear();
        Month month = now.getMonth(); // 枚举类型
        int dayOfMonth = now.getDayOfMonth();
        System.out.println(year +"/" + month.getValue() + "/" + dayOfMonth);
        
        // 判断是否是闰年
        boolean leapyear = now.isLeapYear();
        
        // 获取一年的天数
        int lengthOfYear = now.lengthOfYear();
        // 获取一月的天数
        int lengthOfMonth = now.lengthOfMonth();
        // 获取星期
        DayOfWeek dayOfWeek = now.getDayOfWeek(); // 枚举类型
        // 获取今天是一年中的第几天
        int dayOfYear = now.getDayOfYear();
    }
}
```
#### 2.3.2 修改日期
```java
public class LocalDateModifyDate {
    public static void main(Stirng[] args) {
        LocalDate now = LocalDate.now();
        // 修改年份为 2018 年
        LocalDate withYear = now.withYear(2018);
        // 修改月份为 10 月
        LocalDate withMonth = now.withMonth(10);
        // 修改日期为 10 号
        LocalDate withDayOfMonth = now.withDayOfMonth(10);
        // 链式编程
        LocalDate customDate = now.withYear(2018).withMonth(10).withDayOfMonth(10);
        // 使用 ChronoField 修改
        LocalDate withChronoField = now.with(ChronoField.YEAR, 2019);
    }
}
```
#### 2.3.3 日期的算数运算
```java
public class LocalDateOperateDate {
    public static void main(Stirng[] args) {
        LocalDate now = LocalDate.now();
        // 年的运算
        LocalDate oneYearAfterDate = now.plusYears(1);
        LocalDate fiveYearBeforeDate = now.minusYears(5);
        // 月的运算
        LocalDate threeMonthAfterDate = now.plusMonths(3);
        LocalDate fiveMonthsBeforeDate = now.minus(5, ChronoUnit.MONTHS);
        // 日期的运算
        LocalDate tenDaysAfterDate = now.plusDays(10);
    }
}
```
#### 2.3.4 日期的比较
```java
public class LocalDateCompareDate {
    public static void main(Stirng[] args) {
        LocalDate now = LocalDate.now();
        LocalDate fiveYearBeforeDate = now.minusYears(5);
        // 当前日期是否比 5 年之前的日期早——false
        boolean isBefore = now.isBefore(fiveYearBeforeDate);
        // 当前日期是否比 5 年之前的日期晚——true
        boolean isAfter = now.isAfter(fiveYearBeforeDate);
    }
}
```
## 3.Period的使用
### 3.1 介绍
`Period`主要用于计算两个日期的时间差。
### 3.2 常用方法
```java
public class PeriodTest {
    public static void main(Stirng[] args) {
        LocalDate startDate = LocalDate.of(2022, 6, 15);
        LocalDate endDate = LocalDate.of(2022, 10, 21);
        // 获得 Period 对象
        Period period = Period.between(startDate, endDate);
        System.out.println("两个日期间隔的年份：" + period.getYears()); // 0
        System.out.println("两个日期间隔的月份：" + period.getMonths()); // 4
        System.out.println("两个日期间隔的天数：" + period.getDays()); // 6
    }
}
```
## 4.LocalTime的使用
### 4.1 介绍
`java.time.LocalTime`用于表示时间（时分秒），时间的精度是**纳秒**，不可变且线程安全。
### 4.2 创建LocalTime
```java
public class LocalTimeCreate {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        System.out.println("当前时间：" + now); // 01:53:08.863256974
        LocalTime anyLocalTime = LocalTime.of(15, 15, 15);
        System.out.println("自定义时间：" + anyLocalTime); // 15:15:15
    }
}
```
### 4.2 常用方法
#### 4.2.1 获取时间相关信息
```java
public class LocalTimeGetTimeInfo {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        System.out.println("当前时间：" + now.getHour() + "时" + now.getMinute() + "分" + now.getSecond() + "秒"); // (0-23)时(0-59)分(0-59)秒
    }
}
```
#### 4.2.2 修改时间
```java
public class LocalDateModifyDate {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        
        LocalTime withOneHour = now.withHour(11);
        System.out.println("修改小时为11：" + withOneHour);
        LocalTime withTenMinute = now.withMinute(10);
        System.out.println("修改分钟为10：" + withTenMinute);
        LocalTime withTenSecond = now.withSecond(10);
        System.out.println("修改秒钟为10：" + withTenSecond);
    }
}
```
#### 4.2.3 时间的算术运算
```java
public class LocalTimeOperateTime {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        // 小时的运算
        LocalTime oneHourAfterTime = now.plusHours(1);
        LocalTime fiveHourBeforeTime = now.minusHours(5);
        // 分钟的运算
        LocalTime threeMinutesAfterTime = now.plusSeconds(3);
        LocalTime fiveMinutesBeforeTime = now.minus(5, ChronoUnit.MINUTES);
        // 秒钟的运算
        LocalTime tenSecondsAfterTime = now.plusSeconds(10);
    }
}
```
#### 4.2.4 时间的比较
```java
public class LocalTimeCompareTime {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        LocalTime fiveHourBeforeTime = now.withHour(1);
        System.out.println("当前日期是否在 fiveHourBeforeTime 之前:" + now.isBefore(fiveHourBeforeTime));
        System.out.println("当前日期是否在 fiveHourBeforeTime 之后:" +now.isAfter(fiveHourBeforeTime));
    }
}
```
## 5.LocalDateTime的使用
### 5.1 介绍
`java.time.LocalDateTime`用于表示日期时间，不可变且线程安全。例：`2007-12-03T10:15:35`。
`LocalDateTime`是由`LocalDate`和`LocalTime`组合而成。
### 5.2 创建LocalDateTime
```java
public class LocalDateTimeCreate {
    public static void main(Stirng[] args) {
        LocalDateTime now = LocalDateTime.of(LocalDate.now(), LocalTime.now());
        System.out.println("当前日期时间：" + now);
        LocalDateTime customDateTime = LocalDateTime.of(2022, 6, 15, 11, 22, 00);
        System.out.println("自定义日期时间：" + customDateTime);
    }
}
```
### 5.3 常用方法
由于`LocalDateTime`是由`LocalDate`和`LocalTime`组合而成，因此`LoalDate`和`LocalTime`方法在`LocalDateTime`中都可以找到。
#### 获取LocalDate和LocalTime
```java
public class LocalDateTimeGetLocalDateAndLocalTime {
    public static void main(Stirng[] args) {
        LocalDateTime now = LocalDateTime.of(LocalDate.now(), LocalTime.now());
        LocalDate localDate = now.toLocalDate();
        LocalTime localTime = now.toLocalTime();
    }
}
```
## 6.Duration的使用
### 6.1 介绍
`java.time.Duration`用来计算两个时间的时间差。不可变且线程安全。
### 6.2 常用方法
```java
public class DurationTest {
    public static void main(Stirng[] args) {
        LocalTime startTime = LocalTime.of(0, 1, 3);
        LocalTime endTime = LocalTime.now();
        Duration duration = Duration.between(startTime, endTime);
        System.out.println("两个时间相隔了多少秒：" + duration.getSeconds());
    }
}
```
## 7.DateTimeFormatter的使用
### 7.1 介绍
`java.time.format.DateTimeFormatter`用于格式化日期，`LocalDate`、`LocalTime`、`LocalDateTime`都有对应的`format()`和`parse()`方法来转换日期字符串和日期时间对象。
### 7.2 常用方法
#### 7.2.1 LocalDate中字符串与对象之间转换
```java
public class DateTimeFormatterLocalDateTest {
    public static void main(Stirng[] args) {
        LocalDate now = LocalDate.now();
        
        System.out.println("标准日期格式：" + now.format(DateTimeFormatter.ISO_LOCAL_DATE)); // 标准日期格式：2022-06-15
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd");
        System.out.println("自定义日期格式：" + now.format(dtf)); // 自定义日期格式：2022/06/15
        
        LocalDate parsedDate1 = LocalDate.parse("2022-06-15");
        // LocalDate parsedDate2 = LocalDate.parse("2022/06/15"); // 不能被解析
        LocalDate parsedDate2 = LocalDate.parse("2022/06/15", dtf);
    }
}
```
#### 7.2.2 LocalTime中字符串与对象之间转换
```java
public class DateTimeFormatterLocalTimeTest {
    public static void main(Stirng[] args) {
        LocalTime now = LocalTime.now();
        
        System.out.println("标准时间格式：" + now.format(DateTimeFormatter.ISO_LOCAL_TIME));
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("HH时mm分ss秒");
        System.out.println("自定义时间格式：" + now.format(dtf));
        
        LocalTime parsedTime1 = LocalTime.parse("11:53:00");    
        // LocalDate parsedDate2 = LocalDate.parse("11时53分15秒"); // 不能被解析
        LocalTime parsedTime2 = LocalTime.parse("11时53分15秒", dtf);
    }
}
```
#### 7.2.3 LocalDateTime中字符串与对象之间转换
```java
public class DateTimeFormatterLocalTimeTest {
    public static void main(Stirng[] args) {
        LocalDateTime now = LocalDateTime.now();
        
        System.out.println("标准日期时间格式：" + now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy年MM月dd日'T'HH时mm分ss秒");
        System.out.println("自定义日期时间格式：" + now.format(dtf));
        
        LocalDateTime parsedDateTime1 = LocalDateTime.parse("2022-06-07T11:53:00");    
        // LocalDateTime parsedDateTime2 = LocalDateTime.parse("2022年06月07日T11时53分00秒"); // 不能被解析
        LocalDateTime parsedDateTime2 = LocalDateTime.parse("2022年06月07日T11时53分00秒", dtf);
    }
}
```
### 7.3 解析模式

|     ResolverStyle     |                             作用                             |
| :-------------------: | :----------------------------------------------------------: |
| ResolverStyle.STRICT  |                 严格遵守有效日期，无效则报错                 |
|  ResolverStyle.SMART  | 智能遵守日期规范，例如日只能从1-31，超过实际的天数不会报错，会被自动替换为当月最后一天 |
| ResolverStyle.LENIENT |          更宽松的规则，超过的会累计到下个年、月、日          |

```java
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").withResolverStyle(ResolverStyle.STRICT);
LocalDateTime parse = LocalDateTime.parse("2012-11-31 12:30:20", dateTimeFormatter); // 报错
```

```java
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").withResolverStyle(ResolverStyle.LENIENT);
LocalDateTime parse = LocalDateTime.parse("2012-12-32 25:61:61", dateTimeFormatter);
System.out.println(parse); // 2013-01-02T02:02:01
```

### 7.4 自定义解析模式

默认解析两位数年份时会自动解析为 2000 年后的年份，如果需要自定义解析，则可以使用`DateTimeFormatterBuilder`。

```java
 DateTimeFormatter fmt = new DateTimeFormatterBuilder()
                .appendPattern("d/M/")
                .appendValueReduced(ChronoField.YEAR_OF_ERA, 2, 2, LocalDate.now().minusYears(80))
                .toFormatter();
        System.out.println(LocalDate.parse("13/12/93", fmt)); // 1993-12-13
```

## 8.ZonedDateTime的使用

### 8.1 介绍
`java.time.ZonedDateTime`表示带时区的日期时间，例如`2007-12-03T10:15:30+01:00[Europe/Paris]`。表示巴黎当前是 10 点，比标准时间早 1 个小时。

### 8.2 创建ZonedDateTime
#### 8.2.1 获取时区信息
```java
public class ZonedDateTimeGetZoneInfo {
    public static void main(Stirng[] args) {
        // 默认无序
        Set<String> availableZoneIds = ZoneId.getAvailableZoneIds();
        TreeSet<String> sortedAvailableZonedIds = new TreeSet<>(availableZoneIds);
        
        ZoneId zoneId = ZoneId.of("Asia/Tokyo");
        XoneId defaultZoneId = ZoneId.systemDefault();
        
    }
}
```
#### 8.2.2 创建ZoneDateTime实例
```java
public class ZonedDateTimeCreate {
    public static void main(Stirng[] args) {
        ZonedDateTime now = ZonedDateTime.now();
        System.out.println("系统默认时区的时间信息：" + now); // 2022-06-15T04:06:21.176808969Z[Etc/UTC]
        
        ZoneId tokyoZoneId = ZoneId.of("Asia/Tokyo");
        ZonedDateTime tokyoZoneDateTime = ZonedDateTime.now(tokyoZoneId);
        System.out.println("东京的时间为：" + tokyoZoneDateTime); // 2022-06-15T13:06:21.185318019+09:00[Asia/Tokyo]
        
        // 只修改时区，时间还是本地时间，不会受到影响
        LocalDateTime currentLocalDateTime = LocalDateTime.now();
        ZonedDateTime shanghaiZonedDateTime = ZonedDateTime.of(currentLocalDateTime, ZoneId.of("Asia/Shanghai"));
        System.out.println("上海的时间为：" + shanghaiZonedDateTime); // 2022-06-15T04:06:21.185612081+08:00[Asia/Shanghai]
        // 同上，只修改时区，时间不变
        System.out.println(shanghaiZonedDateTime.withZoneSameLocal(tokyoZoneId)); // 2022-06-15T04:06:21.185612081+09:00[Asia/Tokyo]
        // 根据时区修改时间
        System.out.println(shanghaiZonedDateTime.withZoneSameInstant(tokyoZoneId)); // 2022-06-15T05:06:21.185612081+09:00[Asia/Tokyo]
    }
}
```
### 8.3 常用方法
参考<a href="#5.3 常用方法">LocalDateTime</a>。
## 9.Instant的使用
### 9.1 介绍
`Instant`用于表示瞬间，即时间戳。从`1970-01-01T00:00:00Z`的标准 Java 纪元开始测量。之前的时刻为负值，之后的时刻为正值。

### 9.2 创建Instant
```java
public class InstantCreate {
    public static void main(String[] args) {
        Instant now = Instant.now();
        System.out.println("当前的时间戳：" + now); // UTC时间，2022-06-15T03:40:44.879557744Z
        
        Instant plusThreeSecondsInstant = Instant.ofEpochSecond(3);
        System.out.println("UTC标准纪元时间加上3秒的时间戳：" + plusThreeSecondsInstant); // 1970-01-01T00:00:03Z
        
        // 1s = 1_000_000_000s
        Instant plusThreeSecondsInstant2 = Instant.ofEpochSecond(3, 1_000_000_000);
        System.out.println("UTC标准纪元时间加上3秒的时间戳：" + plusThreeSecondsInstant2); // 1970-01-01T00:00:04Z
    }
}
```
### 9.3 Instant转为ZonedDateTime
```java
public class InstantToZonedDateTime {
    public static void main(String[] args) {
        Instant now = Instant.now();
        ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");
        ZonedDateTime shanghaiZonedDateTime = now.atZone(shanghaiZoneId);
        System.out.println("上海的当前时间为：" + shanghaiZonedDateTime);
    }
}
```
## 10.TemporalAdjusters的使用
### 10.1 介绍
`java.time.TemporalAdjusters`用于查找特殊的日期，例如一个月的第一天，一个月的最后一天，一年的第一天，一年的最后一天等。
### 10.2 常用方法
```java
public class TemporalAdjustersTest {
    public static void main(String[] args) {
        LocalDate now = LocalDate.now();
        TemporalAdjuster temporalAdjuster = TemporalAdjusters.firstDayOfMonth();
        System.out.println("当前月的第一天：" + now.with(temporalAdjuster)); // 2022-06-01
        System.out.println("当前月的最后一天：" + now.with(TemporalAdjusters.lastDayOfMonth()));
        System.out.println("当前月的第一个星期天：" + now.with(TemporalAdjusters.firstInMonth(DayOfWeek.SUNDAY)));
    }
}
```
## 11.Java-Joda-Time的使用
### 11.1 介绍
`Joda-Time`是一个日期时间类库，如果你的开发环境是 JDK7 或者更低的版本，又不想使用 JDK 提供的日期时间 API，那么就可以使用`Joda-Time`。

**导入依赖**
```xml
<dependency>
    <groupId>joda-time</groupId>
    <artifactId>joda-time</artifactId>
    <version>2.18.10</version>
</dependency>
```
### 11.2 常用API

与 Java8 的 API 基本一致。

```java
DateTime now = DateTime.now();
System.out.println("现在的日期时间为：" + now.toString("yyyy/MM/dd HH:mm:ss"));
now.plusDays(1);

LocalDate now = LocalDate.now();
// 计算当前日期的三个月后的那个月的最后一天
LocalDate threeMonthAfterLastDay = now.plusMonths(3).dayOfMonth().withMaxmumValue();
// 计算当前日期3年前第5个月最后一天的日期
LocalDate result = now.minusYears(3).monthOfYear().setCopy(5).dayOfMonth().withMaxmumValue();
```

### 11.3 封装UTC时间和Date相互转换的工具方法

基于`Joda-Time`的 API 实现。

> **UTC 时间格式**：`yyyy-MM-dd'T'HH:mm:ss.SSSZ`

1. 将 UTC 时间转换为`java.util.date`

   ```java
   DateTime dateTime = DateTime.parse(utcTime, DateTimeFormat.forPattern("yyyy-MM-dd'T'HH:mm:ss.SSSZ"));
   Date date = dateTime.toDate();
   ```

2. 将`java.util.date`转换为 UTC 时间（字符串）

   ```java
   DateTime dateTime = new DateTime(date, DateTimeZone.UTC);
   String result = dateTime.toString();
   ```

   