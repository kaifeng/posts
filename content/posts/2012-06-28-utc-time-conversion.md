---
layout: post
title: UTC时间与年月日的互换应用
author: kaifeng
date: 2012-06-28
categories: lang
---

## 时间的表示和接口

关于时间的表示，有两种形式：
- time_t：从Epoch（1970年1月1日0时）至今经历的总秒数。
- struct tm：按年、月、日、时、分、秒方式提供的时间分离记录。

```
struct tm {
 int tm_sec;     /* seconds */
 int tm_min;     /* minutes */
 int tm_hour;     /* hours */
 int tm_mday;     /* day of the month */
 int tm_mon;     /* month */
 int tm_year;     /* year */
 int tm_wday;     /* day of the week */
 int tm_yday;     /* day in the year */
 int tm_isdst;     /* daylight saving time */
};
```

要注意的是，tm_year是指从1900年开始的年，举例，tm_year=70表示的是1970年。
与时间有关的主要函数有四个，由C库提供，在windows和linux上都可以使用。
```
    time_t time(time_t *t);
    time_t mktime(struct tm *tm);
    struct tm *gmtime(const time_t *timep);
    double difftime(time_t time1, time_t time0);
```

time返回当前时间，以time_t格式提供。mktime：将struct tm时间形式换算为time_t形式。gmtime：将time_t时间形式换算为struct tm时间形式。difftime：时间差。

## 常见转换

常见的转换有下面三种：

1. UTC秒计数 -> 年月日时分秒
   * 用time得到当前时间
   * 用gmtime得到年月日时分秒
2. 年月日时分秒 -> UTC秒计数
   * 填充struct tm
   * 调用mktime得到time_t
3. 时间差 -> 直观的年月日差
   * difftime得到时间差
   * gmtime得到年月日形式

1,2可用于异常日志时间与可读时间之间的转换，比如记录从1996年开始的秒数。相当于把Epoch的基准时间从1970年修改到1996年。

容易出错的地方在于：time是以1970为起点的，而struct tm中的tm_year是以1900为起点的，填值时需要注意。

## 举例

构造从1996年开始到当前的时间:

```
    #include <stdio.h>
    #include <time.h>
    #include <memory.h>
    int main()
    {
        time_t tm_now;
        time_t tm_compare;
        struct tm time1;
        memset(&time1, 0, sizeof(time1));
        time1.tm_year = 1996 - 1900;
        time1.tm_mday = 1;
        time1.tm_mon = 0;
        tm_compare = mktime(&time1);
        printf("tm_compare = %d\n", tm_compare);
        time(&tm_now);
        printf("tm_now = %d\n", tm_now);
        double sec = difftime(tm_now, tm_compare);
        printf("sec = %f\n", sec);
        return 0;
    }
```

这段代码是从1996年开始计时，而不是格林威治时间。

## 计时

```
#include <time.h>
time_t time(time_t *t);
```

入参可以为空，精度为秒，返回格林威治时间至今的秒数。
```
#include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tz);
```

精确到毫秒，tz已经废弃，可以设为NULL。
