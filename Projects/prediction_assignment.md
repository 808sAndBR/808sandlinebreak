---
title: "Prediction Assignment"
author: "Scott Brenstuhl"
date: "November 15, 2015"
output: html_document
---

```r
library(caret)
library(doMC)
library(ggplot2)
library(printr)
library(plotly)
library(reshape2)

registerDoMC(cores = 4)
set.seed(1809)
```


```r
testing <- read.csv(file = 'pml-testing.csv')
```

```
## Warning in file(file, "rt"): cannot open file 'pml-testing.csv': No such
## file or directory
```

```
## Error in file(file, "rt"): cannot open the connection
```

```r
# Set blanks as NA for cleaner explotation
training <- read.csv(file = 'pml-training.csv', na.strings = c('','NA'))
```

```
## Warning in file(file, "rt"): cannot open file 'pml-training.csv': No such
## file or directory
```

```
## Error in file(file, "rt"): cannot open the connection
```

### Data Exploration / Feature Selection

```r
nacols <- function(df) {
    colnames(df)[!unlist(lapply(df, function(x) any(is.na(x))))]
}

complete_cols <- nacols(training)
```

```
## Error in is.data.frame(x): object 'training' not found
```

```r
incomplete_cols <- setdiff(names(training),complete_cols)
```

```
## Error in as.vector(x): object 'training' not found
```

```r
# check that none of the columns are being dropped without
# a significant amount of missing data
incomp_check <- lapply(lapply(training[incomplete_cols],is.na),table)
```

```
## Error in lapply(training[incomplete_cols], is.na): object 'training' not found
```

```r
incomp_df <- as.data.frame(t(sapply(incomp_check, rbind)))
```

```
## Error in lapply(X = X, FUN = FUN, ...): object 'incomp_check' not found
```

```r
names(incomp_df) <- c('True','False')
```

```
## Error in names(incomp_df) <- c("True", "False"): object 'incomp_df' not found
```

```r
incomp_df$NA_percent <- round(incomp_df$True/(incomp_df$True+incomp_df$False),4)*100
```

```
## Error in eval(expr, envir, enclos): object 'incomp_df' not found
```

```r
print(table(incomp_df$NA_percent >2.07))
```

```
## Error in table(incomp_df$NA_percent > 2.07): object 'incomp_df' not found
```

Since I loaded blanks as NA we can quickly grab a list of all the columns without any NAs. With how I implemented this it's important to make sure none got dropped becuause of just a few missing values, which is confirmed above since all variables only have values 2.07% of the time.


```r
#summary(training[complete_cols])
suspect <- training[,c('magnet_dumbbell_y', 'gyros_forearm_y', 'gyros_forearm_z', 'gyros_dumbbell_z', 'gyros_dumbbell_x', 'magnet_belt_x')]
```

```
## Error in eval(expr, envir, enclos): object 'training' not found
```

```r
summary(suspect)
```

```
## Error in summary(suspect): object 'suspect' not found
```

```r
# Dont predict on 'new_window', 'user_name', 'raw_timestamp_part_1', 
# 'raw_timestamp_part_2', 'cvtd_timestamp', 'new_window', 'num_window', 'classe'
predictors <- complete_cols[8:(length(complete_cols)-1)]
```

```
## Error in eval(expr, envir, enclos): object 'complete_cols' not found
```

Looking at the Summary of all the values that are left, the first 6 columns have labeling information, as well as timestamps. I’m dropping all of these since they aren’t raw measurements and the test data doesn’t have seem to have enough occurrences to use time windows for prediction. It also looks like magnet_dumbbell_y,gyros_forearm_y, gyros_forearm_z, gyros_dumbbell_z, gyros_dumbbell_x, magnet_belt_x all have some suspicious outliers, which I’m saving as “suspect” to explore later (summary of suspicious values above).


```r
# check variability
O_Var_Predictors <- table(nearZeroVar(training[predictors], saveMetrics = TRUE)$zeroVar)
```

```
## Error in is.vector(x): object 'training' not found
```

```r
print(O_Var_Predictors)
```

```
## Error in print(O_Var_Predictors): object 'O_Var_Predictors' not found
```

All of remaining features have good variability so I'll keep them all.

### Outliers

```r
qp <- qplot(x=factor(0),y= magnet_dumbbell_y,data=suspect, geom = "boxplot")
```

```
## Error in ggplot(data, aesthetics, environment = env): object 'suspect' not found
```

```r
ggplotly(qp)
```

```
## Error in gg2list(p): object 'qp' not found
```

```r
p<- ggplot(data = melt(suspect[2:6]), aes(x=variable, y=value)) + geom_boxplot(aes(fill=variable))
```

```
## Error in melt(suspect[2:6]): object 'suspect' not found
```

```r
ggplotly(p)
```

```
## Error in gg2list(p): object 'p' not found
```

(Note: Plotly open sourced their full stack this week so I wanted to play with it, labels should appear as you hover over the boxes above. Check out their R client here:https://plot.ly/r/)

Taking a closer look at the suspect values from before it looks like something is definitely off with the magnet_dumbbell_y that scored -3600. It also turns out that the outlier value for gyros_forearm_y, gyros_forearm_z, gyros_dumbbell_z, gyros_dumbbell_x all came from the same observation. I removed both of these rows but didn't remove any because of the magnet_belt_x values since its max doesn't look out of place at all.


```r
training <- subset(training, magnet_dumbbell_y != -3600)
```

```
## Error in subset(training, magnet_dumbbell_y != -3600): object 'training' not found
```

```r
training <- subset(training, gyros_forearm_y != 311)
```

```
## Error in subset(training, gyros_forearm_y != 311): object 'training' not found
```

### Model

```r
# model_predictors <- paste(predictors,collapse = " + ")
modFit <- train(classe ~ roll_belt + pitch_belt + yaw_belt + total_accel_belt + gyros_belt_x + gyros_belt_y + gyros_belt_z + accel_belt_x + accel_belt_y + accel_belt_z + magnet_belt_x + magnet_belt_y + magnet_belt_z + roll_arm + pitch_arm + yaw_arm + total_accel_arm + gyros_arm_x + gyros_arm_y + gyros_arm_z + accel_arm_x + accel_arm_y + accel_arm_z + magnet_arm_x + magnet_arm_y + magnet_arm_z + roll_dumbbell + pitch_dumbbell + yaw_dumbbell + total_accel_dumbbell + gyros_dumbbell_x + gyros_dumbbell_y + gyros_dumbbell_z + accel_dumbbell_x + accel_dumbbell_y + accel_dumbbell_z + magnet_dumbbell_x + magnet_dumbbell_y + magnet_dumbbell_z + roll_forearm + pitch_forearm + yaw_forearm + total_accel_forearm + gyros_forearm_x + gyros_forearm_y + gyros_forearm_z + accel_forearm_x + accel_forearm_y + accel_forearm_z + magnet_forearm_x + magnet_forearm_y + magnet_forearm_z, 
                data = training, method = "rf", 
                trControl = trainControl(method = 'cv',
                                         number=5, 
                                         allowParallel = TRUE))
```

```
## Error in eval(expr, envir, enclos): object 'training' not found
```

I then build the model with all of the columns that haven't been eliminated, using cross validation with 5 folds. 


```r
subset(modFit$results, mtry== modFit$bestTune$mtry)
```

```
## Error in subset(modFit$results, mtry == modFit$bestTune$mtry): object 'modFit' not found
```

The cross validation from this model has an accuracy of 99.45% so we can expect the out of sample error to be .55%.

### Predictions


```r
predictions <- predict(modFit,testing)
```

```
## Error in predict(modFit, testing): object 'modFit' not found
```

```r
predictions
```

```
## Error in eval(expr, envir, enclos): object 'predictions' not found
```

Finally based on the model built above, predictions can be made for the testing dataset.
