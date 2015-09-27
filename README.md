---
title: "Analysis of Personal Fitness Data"
author: "Michael Brown"
date: "September 2015"
output: html_document
---
```{r global_options, include=FALSE}
knitr::opts_chunk$set( warning=FALSE, message=FALSE)
```


### Executive Summary
This study predicts fitness outcomes for 20 different test cases using the boost gbm algorithm and machine learning techniques executed in an R environment. This study successfully predicted  outcomes in 20 out of 20 tests on the first run. Model accuracy and estimated error was calculated in two ways: a 15% validation data split and a k-fold cross validation where k = 10.  K-fold cross validation was .9621 while the validation set was predicted at a rate of .9609. This study is also posted on [rpubs](http://rpubs.com/mgbrown85/113387)

### Description of Experiment 
Experiment: Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. This purpose of this experiment is to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants and determine if they are performing perform barbell lifts correctly or incorrectly.  

Data: Data for this experiment is sourced from the Human Activity Recognition (HAR) project at http://groupware.les.inf.puc-rio.br/har.   The data was generated by six young health participants who were asked to perform one set of 10 repetitions of the Unilateral Dumbbell Biceps Curl while wearing a set of accelerometers.  The exercise was performed in one of five ways: 
Class A - exactly according to the specification 
Class B - throwing the elbows to the front 
Class C - lifting the dumbbell only halfway 
Class D - lowering the dumbbell only halfway 
Class E - throwing the hips to the front 


```{r Libraries and Data, echo=TRUE , results="hide"}

library(caret)
library(splines)
library(parallel)
library(plyr)
require(RCurl)

# Fetch data from Human Activity Recognition project
train.csv <- getURL("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv")
df.train.data <- read.csv(textConnection(train.csv))

test.csv <- getURL("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv")
df.test.data <- as.data.frame(read.csv(textConnection(test.csv)))

```


### Data Exploration
The data from the HAR project consists of a training set and a test set.  The training set is made up of 160 variables and 19,622 observations while the testing set consists of 160 variables and 20 observations or prediction instances.  The data presents no intuitive pathway to inform an initial model thus we look at the quality of the data and first focus on conditioning it for analysis. 

### Data Conditioning
Although the data suggests no intuitive model, the format of the data and the data types themselves suggest several areas for pre-processing:

NAs: looking first at the testing set, all columns with NAs were omitted as they bring no additional value to the analysis and potentially cause the model to fail if it is build on data that doesn't exist in the testing set.  This had the effect of reducing the test data from 160 variables to 53 variables.    

Match training and test data: Once the NAs were removed from the testing set, the same columns were removed from the training set in order to avoid modeling on data that was not available for predicting using the test set.  This has the added benefit of reducing the amount of data processed during the computationally intensive training process.

Remove non-relevant data: The training data consisted of a variety of metadata including the person conducting the exercise, time stamps and row identifiers. These data were removed as they had no effect on the outcome of the exercise and would otherwise have further complicated the training step.

```{r Data Preprocess, echo=FALSE}
# Reformed test df to remove columns with NAs and empty columns
names.test.not_na <- !is.na(df.test.data[1,]) 
df.test.not_na <- df.test.data[,names.test.not_na]  
names.df.test <- names(df.test.not_na[1,which(df.test.not_na[1,]!='')]) 

# reformed test data with no empty columns
df.test.meta <- df.test.not_na[,names.df.test] 

# remove metadata
df.test <- df.test.meta[,8:60]  

# Remove columns from training that are not found in pre-processed test data (except classe)
train.names.keep <- is.element(names(df.train.data) , names(df.test))
df.train.match <- df.train.data[,train.names.keep]
df.train.all <- cbind(df.train.match , df.train.data[,'classe'])

colnames(df.train.all)[53] <- 'classe'

in.train <- createDataPartition(y = df.train.all$classe , p = .85 , list = FALSE)

df.train <- df.train.all[in.train , ]
df.val <- df.train.all[-in.train , ]

```


### Boost GBM Model
Using training data, a model is created using GBM (generalized boosted regression model) package.  The results are summarized below.
```{r Model, echo=TRUE }
# Create model
control <- trainControl(method="cv", number=10)
model_gbm <- train(classe~. , data = df.train , method="gbm" , trControl=control ,  verbose = FALSE )
model_gbm$finalModel

```

### Visualization of Model Variables
The model variables and their relative importance are indicated below.  Based on the importance analysis, the model performance could have been further enhanced by deselecting the variables with no explanatory power.  Note also that out of the 53 input variables, ~10% have an importance >20 suggesting significant prospect for further enhancement.

```{r Visualization, echo=TRUE }
# Visualization
plot(varImp(model_gbm))
```


### Cross Validation and Predicted Error
The BOOST gbm algorithm was conducted with a 10-fold cross validation.  The data and plot below shows an achieved accuracy on the 3rd iteration of 96.09% rising from an initial accuracy of 75.15% in the first iteration.    

```{r Cross Validation , echo=TRUE }
plot(model_gbm)
model_gbm
```

### Validation Accuracy 
Model accuracy and error were further estimated by randomly splitting 15% of the training data prior to model construction.  The below confusion matrix shows the estimated accuracy to be 96.26%.
```{r Validation Data , echo=TRUE }
cross_validation_results_boost <- predict(model_gbm , newdata = df.val)
confusionMatrix(df.val$classe , cross_validation_results_boost)
```

### Prediction
```{r Prediction, echo=TRUE }
predict.test.gbm <- predict(model_gbm , newdata = df.test )

```

```{r Output, echo=TRUE }
# Produce result files
n = length(predict.test.gbm)
  for(i in 1:n){
    filename = paste0("problem_id_",i,".txt")
    write.table(predict.test.gbm[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
    }

```

### Results
The resulting data produced from the boost modeling of the HAR data presented below in order from 1 to 20.  On submission to the Coursera Project Page, the results were shown to be 100% correct.

```{r Results, echo=FALSE }

predict.test.gbm  

```

