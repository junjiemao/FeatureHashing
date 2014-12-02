FeatureHashing
==============

Implement feature hashing with R

## Introduction

[Feature hashing](http://en.wikipedia.org/wiki/Feature_hashing) is a popular trick to vectorize text features. 
The pakcage FeatureHashing implements the hashing trick introduced in 
Weinberger et. al. (2009) 
and provides an API similar to model.matrix and Matrix::sparse.model.matrix 
in R. 
The FeatureHashing also supports the splitting of concatenated data during 
the construction of model matrix to save the memory.

### When do I use Feature Hashing?

- I don't need to know the meaning of the cofficient.

    After feature hashing, it is hard to connect the trained coeifficents to the original data.

- I cannot convert all data including training data and testing data to a matrix in one time. 

    To make the matrix consistent, I need to synchronize the mapping between the position and the feature. Feature hashing makes my life easier.

- You get a concatenated data in one column.

    Feature Hashing supports the splitting of concatenated data and the user only need to specify the concatenated data and the delimiter in the formula.

## Demo

### Feature Hashing


```r
library(FeatureHashing)
# Construct the model matrix. The transposed matrix is returned by default.
m <- hashed.model.matrix(~ ., CO2, 2^6, keep.hashing_mapping = TRUE)
mapping <- as.list(attr(m, "mapping"))
# Print the matrix via dgCMatrix
as(m, "dgCMatrix")[1:20,1:17]
```

```
## 20 x 17 sparse Matrix of class "dgCMatrix"
```

```
##    [[ suppressing 17 column names '1', '2', '3' ... ]]
```

```
##                                                                        
## <NA>  1   1   1   1   1   1    1  1   1   1   1   1   1    1  1   1   1
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA> 95 175 250 350 500 675 1000 95 175 250 350 500 675 1000 95 175 250
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
## <NA>  .   .   .   .   .   .    .  .   .   .   .   .   .    .  .   .   .
```

```r
# Check the result of hashing
mapping <- unlist(as.list(attr(m, "mapping")))
mapping %% 2^6 # the 9-th row is conc
```

```
##            PlantQn1            PlantQn2            PlantQn3 
##                  25                  56                  26 
##              uptake     TypeMississippi    Treatmentchilled 
##                  27                  56                  18 
##            PlantMn1            PlantMn2            PlantMn3 
##                  54                  34                  54 
##            PlantQc1            PlantQc2            PlantQc3 
##                  10                   2                   1 
## Treatmentnonchilled            PlantMc1            PlantMc2 
##                  38                   6                  22 
##            PlantMc3                conc          TypeQuebec 
##                  26                   8                  38
```

```r
# Check the rate of collision
mean(duplicated(mapping %% 2^6))
```

```
## [1] 0.2222
```

```r
# The result is CSCMatrix which supports simple subsetting and matrix-vector
# multiplication
# rnorm(2^6) %*% m

# Detail of the hashing
## The main effect is hashed via `hash_h`
all(hash_h(names(mapping)) %% 2^6 == mapping %% 2^6)
```

```
## [1] TRUE
```

```r
## The sign is corrected by `hash_xi`
hash_xi(names(mapping))
```

```
##  [1]  1 -1  1 -1 -1 -1 -1  1  1 -1  1 -1  1  1  1  1  1 -1
```

```r
## The interaction term is implemented as follow:
m2 <- hashed.model.matrix(~ .^2, CO2, 2^6, keep.hashing_mapping = TRUE)
mapping2 <- unlist(as.list(attr(m2, "mapping")))
mapping2[2] # PlantQn2:uptake
```

```
## PlantQn2:uptake 
##        7.55e+08
```

```r
h1 <- mapping2["PlantQn2"]
h2 <- mapping2["uptake"]
library(pack)
hash_h(rawToChar(c(numToRaw(h1, 4), numToRaw(h2, 4)))) # should be mapping2[2]
```

```
## [1] 755024387
```

### Concatenated Data


```r
# The tag-like feature
data(test.tag)
df <- data.frame(a = test.tag, b = rnorm(length(test.tag)))
head(df)
```

```
##                                        a       b
## 1                       1,27,19,25,tp,tw  0.9208
## 2                       1,27,19,25,tp,tw  0.3930
## 3 25,1,19,6,29,17,16,21,26,23,27,4,ty,tw  0.6202
## 4                               19,tp,tw  0.8814
## 5                                  19,tw -2.3572
## 6                                 ,ch,tw -2.1393
```

```r
m <- hashed.model.matrix(~ tag(a, split = ",", type = "existence"):b, df, 2^6,
 keep.hashing_mapping = TRUE)
# The column `a` is splitted by "," and have an interaction with "b":
mapping <- unlist(as.list(attr(m, "mapping")))
names(mapping)
```

```
##  [1] "a20"    "a"      "a21"    "atn:b"  "ant:b"  "a19:b"  "akh:b" 
##  [8] "a25:b"  "a16:b"  "atw:b"  "b"      "a4:b"   "a10:b"  "a1:b"  
## [15] "ahc"    "atc"    "a23"    "a24"    "a25"    "a26"    "a27"   
## [22] "antw"   "akh"    "a29"    "atn"    "a30"    "atp"    "a1"    
## [29] "a11:b"  "a8:b"   "a23:b"  "ach:b"  "a20:b"  "ahc:b"  "atc:b" 
## [36] "a29:b"  "a26:b"  "a17:b"  "a3"     "ant"    "a4"     "a6"    
## [43] "atw"    "a8"     "ach"    "aty"    "a9"     "ail"    "a10"   
## [50] "a11"    "ail:b"  "a3:b"   "a9:b"   "atp:b"  "a12"    "a27:b" 
## [57] "a:b"    "antw:b" "aty:b"  "a15:b"  "a24:b"  "a21:b"  "a6:b"  
## [64] "a12:b"  "a30:b"  "a15"    "a16"    "a17"    "a19"
```

