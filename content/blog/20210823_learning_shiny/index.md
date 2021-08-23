---
title: "Learning Shiny ( some tidymodels)"
categories:
- Shiny
- Learning
date: "2021-08-23"
tags:
- tidymodels
- Palmer Penguins
excerpt: Learning about Shiny while using tidymodels to make predictions on the Palmer
  Penguins dataset.
---



Let's start by taking a look at how many NAs are missing in the dataset:


```r
colSums(is.na(penguins))
```

```
##           species            island    bill_length_mm     bill_depth_mm 
##                 0                 0                 2                 2 
## flipper_length_mm       body_mass_g               sex              year 
##                 2                 2                11                 0
```

We see that the only missing values are in `sex`, and there is only 9. Let's see how those missing values are distributed across the species:


```r
penguins %>% 
  filter(is.na(sex)) %>% 
  count(species)
```

```
## # A tibble: 2 x 2
##   species     n
##   <fct>   <int>
## 1 Adelie      6
## 2 Gentoo      5
```

An even split between Adelie and Gentoo, the two species with the most representatives:


```r
penguins %>% 
  count(species)
```

```
## # A tibble: 3 x 2
##   species       n
##   <fct>     <int>
## 1 Adelie      152
## 2 Chinstrap    68
## 3 Gentoo      124
```

While we can probably throw away the data and not feel bad, let's see if we can impute the missing data.
Let's take a look at the other variables:


```r
penguins %>% 
  pivot_longer(cols = bill_length_mm:body_mass_g, names_to = "metric", values_to = "value") %>% 
  ggplot(aes(species, value, color = sex)) +
  geom_boxplot() +
  facet_wrap(~ metric, scales = "free")
```

```
## Warning: Removed 8 rows containing non-finite values (stat_boxplot).
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-5-1.png" width="672" />

There seems to be a significant difference between the male and female penguins, so it seems reasonable to impute the missing values.

## Parameter tuning

Let's start by splitting the data into a testing and training set:


```r
set.seed(489)

tidy_split <- initial_split(penguins %>% select(-year))
tidy_train <- training(tidy_split)
tidy_test <- testing(tidy_split)

tidy_kfolds <- vfold_cv(tidy_train)
```

Here, we're creating a recipe that will try to predict the species based on all the other columns after performing a few preprocessing steps:

* Impute the missing sex values via k-nearest neighbors
* Convert the factors to dummy variables via one hot encoding
* Normalize the predictors


```r
tidy_rec <- recipe(species ~ ., data = tidy_train) %>% 
  step_impute_knn("sex") %>% 
  step_dummy(all_nominal(), -all_outcomes(), one_hot = TRUE) %>% 
  step_impute_median(all_predictors()) %>% 
  step_normalize(all_predictors())
```

Before we actually get to prediction, we'll do some parameter tuning. We'll be using an xgboost model, so the parameters we'll be tuning are `trees`, `min_n`, and the `learn_rate`:



```r
tidy_boosted_model <- boost_tree(trees = tune(),
                                 min_n = tune(),
                                 learn_rate = tune()) %>% 
  set_mode("classification") %>% 
  set_engine("xgboost")
```

`grid_regular` creates a grid of potential parameter values 3 levels deep for the parameter tuning process. `tune_grid` then actually performs the tuning, after which we can retrieve the parameters that achieved the best ROC AUC on the test set. 

We're using ROC AUC here because the data set has some class imbalances.
While the imbalance isn't as bad as some other datasets, it's still enough to make me want to switch to ROC AUC.


```r
boosted_grid <- grid_regular(parameters(tidy_boosted_model), levels = 3)

boosted_tune <- tune_grid(tidy_boosted_model,
          tidy_rec, 
          resamples = tidy_kfolds,
          grid = boosted_grid)

boosted_params <- boosted_tune %>% 
  select_best("roc_auc")
```

We now finalize our model with those parameters we found and wrap that finalized model up into a workflow with our preprocessing recipe. Finally, we fit it on the training set and make predictions on the test set.

## Predictions and metrics


```r
tidy_boosted_model <- finalize_model(tidy_boosted_model, boosted_params)

boosted_workflow <- workflow() %>% 
  add_model(tidy_boosted_model) %>% 
  add_recipe(tidy_rec)

boosted_results <- last_fit(boosted_workflow, tidy_split)
```


Let's see how it did!


```r
boosted_results %>% 
  unnest(.predictions) %>% 
  conf_mat(truth = species, estimate = .pred_class) %>% 
  pluck("table") %>% 
  reshape2::melt() %>% 
  ggplot(aes(Truth, Prediction, fill = value)) +
  geom_tile(color = "black", size = 1) +
  theme(
    panel.grid = element_blank(),
    panel.border = element_blank(),
    axis.ticks = element_blank(),
    legend.position = "none"
  ) + 
  geom_text(aes(label = value)) + 
  scale_fill_gradient(low = "white", high = "blue") + 
  scale_y_discrete(limits = rev) +
  labs(
    title = "Palmer Penguins Confusion Matrix"
  )
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-11-1.png" width="672" />

Looks like it did really well, having only missed two 2 predictions out of 86. The actual metrics are:


```r
boosted_results %>% 
  unnest(.metrics) %>% 
  select(.metric, .estimate)
```

```
## # A tibble: 2 x 2
##   .metric  .estimate
##   <chr>        <dbl>
## 1 accuracy     0.988
## 2 roc_auc      0.999
```

The metrics were very high, but this makes sense given the clustering of the dataset. We can visualize this with an NMDS:



```r
library(vegan)

penguins <- penguins %>% 
  mutate(row_num = row_number())

penguin.NMDS <- metaMDS(
  penguins %>% 
    select(-c(year, species, island, sex, row_num)) %>% 
    filter(!is.na(.)),
  k = 3)

# The stress of the NMDS was below 0.05, indicating that reducing the dimensions of the data does a fair job.
penguin.NMDS$stress


penguin.NMDS <- as.data.frame(vegan::scores(penguin.NMDS, display = "sites")) %>% 
  bind_cols(
    penguins %>% 
      select(-c(year, species, island, sex)) %>% 
      filter(!is.na(.)) %>% 
      left_join(penguins, by = "row_num") %>% 
      select(-row_num)
      
  )

penguin.NMDS %>% 
  ggplot(aes(NMDS1, NMDS2, color = species)) +
  geom_point(size = 2) + 
  labs(
    color = "Species",
    title = "Palmer Penguins NMDS"
  )
```


We see that the species of penguins cluster quite strongly, with Gentoo separating cleanly from the other two species. While Adelie and Chinstrap are relatively close, there are is still a clear distinction between the two clusters. Models doing this well make me nervous, but seeing how well-separated the plot is puts my mind to rest a little bit.

## Using the model in a Shiny app

Like I mentioned up top, I was mostly doing this to learn how to use a Shiny App. 
To start off with, let's get a final model and save it to an `RDS` for use in our app.


```r
final_boosted_model <- fit(boosted_workflow, penguins)
```

```
## [05:14:04] WARNING: amalgamation/../src/learner.cc:1095: Starting in XGBoost 1.3.0, the default evaluation metric used with the objective 'multi:softprob' was changed from 'merror' to 'mlogloss'. Explicitly set eval_metric if you'd like to restore the old behavior.
```

```r
saveRDS(final_boosted_model, "penguin_model.RDS")
```

I then loaded that model into a simple Shiny app that takes The different measurements and tries to predict the species:

![](shiny-db.png)

You can play with the app [https://pommevilla.shinyapps.io/shiny_penguins/](here) and check out the code [here](https://github.com/pommevilla/shiny_penguins/blob/main/R/app.R). It's a super simple app, but I wanted to get the hang of the Shiny workflow before I start doing some more intense ones.
