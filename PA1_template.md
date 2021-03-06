# Reproducible Research: Peer Assessment 1


## Loading and preprocessing the data

Unzip the file

```r
unzip(zipfile="activity.zip")
actmoni_data <- read.csv("activity.csv")
```
Load the data using read.csv()

```r
actmoni_data <- read.csv('activity.csv', header = TRUE, sep = ",",
                         colClasses=c("numeric", "character", "numeric"))
```
Convert the interval and date field to factor and Date class respectively.

```r
actmoni_data$date <- as.Date(actmoni_data$date, format = "%Y-%m-%d")
actmoni_data$interval <- as.factor(actmoni_data$interval)
```

## What is mean total number of steps taken per day?

Ignore the missing data NA

```r
actmoni_data.ignore.na <- na.omit(actmoni_data) 
```
Calculate the sum of steps by date

```r
daily.steps <- rowsum(actmoni_data.ignore.na$steps, format(actmoni_data.ignore.na$date, '%Y-%m-%d')) 
daily.steps <- data.frame(daily.steps) 
names(daily.steps) <- ("steps")
```
Plot the histogram of total number of steps

```r
hist(daily.steps$steps, 
     main="Histogram of steps taken / day",
     breaks=15, col="red",
     xlab="Total number of steps taken daily",
     ylab="Number of times / day")
```

![](PA1_template_files/figure-html/unnamed-chunk-6-1.png) 


Calculate the mean and median of the number of steps/day

```r
mean(daily.steps$steps); 
```

```
## [1] 10766.19
```

```r
median(daily.steps$steps) 
```

```
## [1] 10765
```

## What is the average daily activity pattern?


Calculate the aggregation of steps

```r
steps_per_interval <- aggregate(actmoni_data$steps, 
                                by = list(interval = actmoni_data$interval),
                                FUN=mean, na.rm=TRUE)
```
Convert the intervals to integers for plotting

```r
steps_per_interval$interval <- 
  as.integer(levels(steps_per_interval$interval)[steps_per_interval$interval])
colnames(steps_per_interval) <- c("interval", "steps")
```
Plot with number of steps Vs 5-minute intervals

```r
ggplot(steps_per_interval, aes(x=interval, y=steps)) +   
  geom_line(color="green", size=1) +  
  labs(title="Average Daily Activity Pattern", x="5-minute Interval", y="Number of steps") +  
  theme_bw()
```

![](PA1_template_files/figure-html/unnamed-chunk-11-1.png) 

Find the maximum number of steps in 5-minute interval

```r
interval.mean.steps[which.max(interval.mean.steps$mean), ]
```

```
##     interval     mean
## 104      835 206.1698
```

## Imputing missing values

Calculate and report the total number of misssing values

Calculate the total number of missing values.


```r
tNA <- sqldf(' 
             SELECT d.*            
             FROM "actmoni_data" as d
             WHERE d.steps IS NULL 
             ORDER BY d.date, d.interval ')
```

```
## Loading required package: tcltk
```


```r
NROW(tNA) 
```

```
## [1] 2304
```
Fill all the missing values in dataset

```r
na_fill <- function(actmoni_data, pervalue) {
  na_index <- which(is.na(actmoni_data$steps))
  na_replace <- unlist(lapply(na_index, FUN=function(idx){
    interval = actmoni_data[idx,]$interval
    pervalue[pervalue$interval == interval,]$steps
  }))
  fill_steps <- actmoni_data$steps
  fill_steps[na_index] <- na_replace
  fill_steps
}

actmoni_data_fill <- data.frame(  
  steps = na_fill(actmoni_data, steps_per_interval),  
  date = actmoni_data$date,  
  interval = actmoni_data$interval)
str(actmoni_data_fill)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : num  1.717 0.3396 0.1321 0.1509 0.0755 ...
##  $ date    : Date, format: "2012-10-01" "2012-10-01" ...
##  $ interval: Factor w/ 288 levels "0","5","10","15",..: 1 2 3 4 5 6 7 8 9 10 ...
```
Check whether there is any missing values or not

```r
sum(is.na(actmoni_data_fill$steps))
```

```
## [1] 0
```
Plot histogram of total number of steps

```r
fill_steps_per_day <- aggregate(steps ~ date, actmoni_data_fill, sum)
colnames(fill_steps_per_day) <- c("date","steps")

ggplot(fill_steps_per_day, aes(x = steps)) + 
  geom_histogram(fill = "purple", binwidth = 1000) + 
  labs(title="Histogram of steps taken / day", 
       x = "Number of steps / Day", y = "Number of times / day") + theme_bw()
```

![](PA1_template_files/figure-html/unnamed-chunk-18-1.png) 

Calculate the mean and median of total number of steps taken per day

```r
t1.mean.steps.per.day <- as.integer(t1.total.steps / NROW(t1.total.steps.by.date) )
t1.mean.steps.per.day
```

```
## [1] 10766
```

```r
t1.median.steps.per.day <- median(t1.total.steps.by.date$t1.total.steps.by.date)
t1.median.steps.per.day
```

```
## [1] 10766.19
```
Do these values differ from the estimates from the first part of the assignment?

Yes, values differ slightly. Mean is 10766.19 and Median is 10765 in the first part of the assignment but after filling the data, Mean and median is 10766 and 10766.19 respectively.

What is the impact of imputing missing data on the estimates of the total daily number of steps?
 
 Imputing the missing values altered the peak slightly by increasing.

## Are there differences in activity patterns between weekdays and weekends?

Create a factor variable weektime with two levels-weekend and weekday

```r
weekdays_steps <- function(actmoni_data) {
  weekdays_steps <- aggregate(actmoni_data$steps, by=list(interval = actmoni_data$interval),
                              FUN=mean, na.rm=T)
  weekdays_steps$interval <- 
    as.integer(levels(weekdays_steps$interval)[weekdays_steps$interval])
  colnames(weekdays_steps) <- c("interval", "steps")
  weekdays_steps
}

actmoni_data_by_weekdays <- function(actmoni_data) {
  actmoni_data$weekday <- 
    as.factor(weekdays(actmoni_data$date))
  weekend_actmoni_data <- subset(actmoni_data, weekday %in% c("Saturday","Sunday"))
  weekday_actmoni_data <- subset(actmoni_data, !weekday %in% c("Saturday","Sunday"))
  
  weekend_steps <- weekdays_steps(weekend_actmoni_data)
  weekday_steps <- weekdays_steps(weekday_actmoni_data)
  
  weekend_steps$dayofweek <- rep("weekend", nrow(weekend_steps))
  weekday_steps$dayofweek <- rep("weekday", nrow(weekday_steps))
  
  actmoni_data_by_weekdays <- rbind(weekend_steps, weekday_steps)
  actmoni_data_by_weekdays$dayofweek <- as.factor(actmoni_data_by_weekdays$dayofweek)
  actmoni_data_by_weekdays
}

actmoni_data_weekdays <- actmoni_data_by_weekdays(actmoni_data_fill)
```
Plot with average number of steps taken and 5-minute interval

```r
ggplot(actmoni_data_weekdays, aes(x=interval, y=steps)) + geom_line(color="blue") + 
  facet_wrap(~ dayofweek, nrow=2, ncol=1) +
  labs(x="5-minute Interval", y="Number of steps") + theme_bw()
```

![](PA1_template_files/figure-html/unnamed-chunk-21-1.png) 

