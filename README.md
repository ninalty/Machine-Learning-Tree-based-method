# Machine-Learning-Tree-based-method
This project practices tree-based methods on predicting the presence/absence of 'bigfoot' with climate data

This exercise applied CART and Random Forest to predict the presence of 'big foot" in area with known climate.

source: [Quantative Geogrpahy](https://rspatial.org/raster/analysis/5-global_regression.html#google_vignette)

<font size="3">Load packages</font>
```{r echo=FALSE, warning=FALSE, collapse=TRUE}
library(rspatial)
library(maptools)
library(raster)
library(rpart)
library(dismo)
library(randomForest)
library(maptools)
```

<font size="3">Observations -- Occurrence of "big foot" worldwide</font>\
```{r}
bf <- sp_data('bigfoot')

plot(bf[,1:2], cex=0.5, col='red')
data(wrld_simpl)

plot(wrld_simpl, add=TRUE)
```

<font size="3">Predictors</font>\
Climate is used as predictors.\
In world climate raster file, bio1 is annual mean temperature(C). bio12 is annual precipitation.
```{r}
wc <- raster::getData('worldclim', res=10, var='bio')

# plot two bio data
plot(wc[[c(1, 12)]], nr=2)

```

```{r}
bfc <- extract(wc, bf[,1:2])

# Any missing values?
i <- which(is.na(bfc[,1]))

plot(bf[,1:2], cex=0.5, col='blue')

plot(wrld_simpl, add=TRUE)

points(bf[i, ], pch=20, cex=3, col='red')
```

<font size="3">Background data</font>\
To create climate data without knowing big foot exists or not.
```{r}
# extent of all points
e <- extent(SpatialPoints(bf[, 1:2]))

# 5000 random samples (excluding NA cells) from extent e
set.seed(0)
bg <- sampleRandom(wc, 5000, ext=e)

#combine presence and background data, pa = 1 is presence; pa = 0 is absence.
d <- rbind(cbind(pa=1, bfc), cbind(pa=0, bg))
d <- data.frame(d)

#split the data into east and west
de <- d[bf[,1] > -102, ]
dw <- d[bf[,1] <= -102, ]
```

<font size="3">Fit a model</font>\
<font size="3">Classification and Regression Trees (CART) model</font>
```{r collapse=TRUE}
cart <- rpart(pa~., data=dw)

printcp(cart) 
# CP is complexity parameter->control the size of decision tree
# if CP increases as the nsplit increases. The tree will stop growing.

plotcp(cart)
```

Here is the tree
```{r}
plot(cart, uniform=TRUE)
# text(cart, use.n=TRUE, all=TRUE, cex=.8)
text(cart, cex=.7, digits=1)
```
Question 1\
Describe the conditions under which you have the highest probability of finding our beloved species?

The highest probability is 0.9. There are two highest probability.

One is in condition when bio4<8679, and bio10<218.5 and bio15>=40.5, and bio8<88.5.
The other is when bio4<8679, and bio10<218.5, and bio15<40.5, and bio14<44.5, and bio5<280.5, bio3<33.5, bio6>=124.5


<font size="3">Random Forest</font>
# Classification
The plot below shows that as the increase of tree numbers, error rate was significantly reduced at the beginning, then reached a steady error rate no matter how many trees are trained.
```{r warning=FALSE, collapse=TRUE}
# create a factor to indicated that we want classification
fpa <- as.factor(dw[, 'pa'])

#fit the random forest model
crf <- randomForest(dw[, 2:ncol(dw)], fpa)
crf

plot(crf)
```

The variable importance plot shows which variables are most important in fitting the model.\
Gini index indicates the degree of impurity of the split. High Gini index indicates high impurity.\
Therefore, variables that cause high mean decrease in Gini index rank top.
```{r}
varImpPlot(crf)
```

# Regression of Random Forest
Question 2: What did tuneRF help us find? What does the values of mt represent?

'tuneRF' helps to find the optimal value of variable number for randomForest.\
'mt' represents the number of variables that has the minimum error.\
In this case, it is 3.
```{r warning=FALSE}
#tune the parameter and get the best variable number
trf <- tuneRF(dw[, 2:ncol(dw)], dw[, 'pa'])

mt <- trf[which.min(trf[,2]), 1]

mt
```

Train regression tree
```{r}
rrf <- randomForest(dw[, 2:ncol(d)], dw[, 'pa'], mtry=mt)

plot(rrf)
```

x axis is increased node purity. 
```{r}
varImpPlot(rrf)
```

<font size="3">Predict</font>
```{r warning=FALSE}
# Extent of the western points
ew <- extent(SpatialPoints(bf[bf[,1] <= -102, 1:2]))
```

<font size="3">Regression</font>
Here is our prediction based on the trained regression model.
```{r}
rp <- predict(wc, rrf, ext=ew)
plot(rp)
```

To get the optimal threshold of deciding presence/absence, you would normally have a hold out data set, but here I used the training data for simplicity.\

The plot below is the plot of Receiver Operating Characteristic curve (ROC curve). AUC is area under curve. If ROC is close to the top left corner, the accuracy of that threshold value is the best. If ROC is close to the right bottom corner, the accuracy is low.
```{r warning=FALSE}
# Cross-validation of models with presence/absence data.
eva <- evaluate(dw[dw$pa==1, ], dw[dw$pa==0, ], rrf)

plot(eva, 'ROC')
```

Get a threshold value to determine presence/absence and plot the prediction.\
Green area shows the presence of 'bigfoot'.
```{r warning=FALSE}
tr <- threshold(eva)

plot(rp > tr[1, 'spec_sens'])
```

<font size="3">Classification</font>
```{r}
rc <- predict(wc, crf, ext=ew)
plot(rc)
```
can also get probabilities for the classes
```{r}
rc2 <- predict(wc, crf, ext=ew, type='prob', index=2)
plot(rc2)
```

<font size="3">Extrapolation</font>
Making prediction with regression random forest model.
The AUC plot show the model accuracy is low.
```{r}
de <- na.omit(de)
eva2 <- evaluate(de[de$pa==1, ], de[de$pa==0, ], rrf)

plot(eva2, 'ROC')
```

Take a look at it on a map.

Question 3: Why would it be that the model does not extrapolate well?

1. Training data is not comprehensive. The model is trained by the dw data. It didn't include any data in the eastern US. Because the topography and climate in western area are quite different from each other. So it is possible the training data is not inclusive enough for the general prediction.\

2. Over fitting. It is also possible that the number of variables are way more than needed which led the model fit the western region too well to be general.The tuning process also indicated that 3 is the optimal number of regression random forest.

```{r}
eus <- extent(SpatialPoints(bf[, 1:2]))
rcusa <- predict(wc, rrf, ext=eus)
plot(rcusa)
points(bf[,1:2], cex=.25)
```

Question 4: Make a map to show where conditions are improving for western bigfoot, and where they are not. Is the species headed toward extinction?

The species do not head toward extinction.\
Most of the area have variation smaller than zero as shown in the histogram which indicates that population of bigfoot increases in the future.
```{r warning=FALSE}
#get the new climate data
fut <- getData('CMIP5', res=10, var='bio', rcp=85, model='AC', year=70)

#keep the column nama the same
names(fut) <- names(wc)

##extrapolate the spatial distribution of bigfoot with the new climate data
futusa <- predict(fut, rrf, ext=ew, progress='window')

#difference between occurence predicted by regression random forest -future
bd_dift <- rp-futusa

plot(bd_dift, col = topo.colors(20), main="Bigfoot Shift")
```

```{r warning=FALSE}
##plot histogram on the variation data
hist(bd_dift, main="Bigfoot Shift")
```

