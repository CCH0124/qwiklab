---
layout: post
title: BigQuery for Machine Learning-04
date: 2020-09-08
excerpt: "Bracketology with Google Machine Learning"
tags: [qwiklab, ML, google, BigQuery]
comments: true
---

本實驗中，使用 `BigQuery`、`Machine Learning` 和 `NCAA` 男子籃球數據集預測 `NCAA` 男子籃球錦標賽比賽的獲勝者。



## NCAA March Madness
National Collegiate Athletic Association（NCAA）每年在美國舉辦兩次主要的男子和女子大學籃球比賽。在三月份的 `NCAA` 男子錦標賽中，有 68 支隊伍參加了單淘汰賽，一支隊伍退出了總冠軍。`NCAA` 提供了一個公開數據集，其中包含本賽季和最終比賽的男子和女子籃球比賽和球員的統計數據。比賽數據涵蓋了回溯到 2009 年的逐項比賽得分和藍框得分，以及回溯到 1996 年的最終得分。在某些球隊的情況下，有關勝利與失敗的其他數據可以追溯到 1894-5 賽季。

## Find the NCAA public dataset in BigQuery
移至 `BigQuery` 控制台中，在左側尋找 `Resources` 並點擊 `+ ADD DATA` 按鈕，再選擇 `Explore public datasets`。在搜索框中輸入 `NCAA Basketball`，並確認，此時會彈出一個資源，選擇它點擊 `VIEW DATASET`。

![](https://cdn.qwiklabs.com/%2FgAZEPTmXfmnaIVbkOe83a6vSKExh%2BMXEaF2GXW4Fd8%3D)

這將打開一個新的 `BigQuery` 頁面，並加載數據集。可繼續在該頁面中工作或關閉它並在另一個頁面中刷新 `BigQuery` 控制台以顯示數據集。

點擊 `ncaa_basketball` 數據集旁邊的箭頭以顯示其表

![](https://cdn.qwiklabs.com/8KcqsNVZe7zQShgg9mW3mA24mxG85fIgDQt2B2LBh6Q%3D)

在數據集中應該看到 10 個表。點擊 `mbb_historical_tournament_games`，再單擊`Preview`以查看數據行。然後點擊 `Details` 以獲取有關表的元數據。該頁面應類似於下圖：

![](https://cdn.qwiklabs.com/%2BhvXktqBBwvBhhkWJQedHJ2uCaSAbU67XeXWBR6smSI%3D)

## Write a query to determine available seasons and games
現在，將編寫一個簡單的 `SQL` 查詢，確定在我們的 `mbb_historical_tournament_games` 表中可以探索多個賽季和比賽。

```sql
SELECT
  season,
  COUNT(*) as games_per_tournament
  FROM
 `bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`
 GROUP BY season
 ORDER BY season # default is Ascending (low to high)
```

結果如下

![](https://cdn.qwiklabs.com/ioeUqs6h6bLvrrRPh1eaAe8r9C7xehh30xTFOVOKN4A%3D)

## Understand machine learning features and labels
該實驗室的最終目標是根據以往的歷史比賽知識，預測特定 NCAA 男子籃球比賽的獲勝者。在機器學習中，有助於我們判斷結果(錦標賽比賽的輸或贏)數據中的每一列稱為*特徵*。將嘗試預測的數據列稱為*標籤*。機器學習模型*學習*特徵之間的關聯以預測標籤的結果。

歷史數據集的特徵可能包括
- Season
- Team name
- Opponent team name
- Team seed (ranking)
- Opponent team seed

## Create a labeled machine learning dataset
建立機器學習模型需要大量高品質的訓練數據。幸運的是，我們 NCAA 數據集的量很足夠，因此我們可以依靠它來建立有效的模型。返回 `BigQuery` 控制台。


在左側選單中，透過單擊表名稱打開 `mbb_historical_tournament_games` 表。加載後，單擊 `Preview`，內容類似於下圖

![](https://cdn.qwiklabs.com/DQEXLhYhdou2EGCXI5qaSPhkfKdI0KNUNDT5N5WIBj4%3D)

查看數據集後，會注意到數據集中的一行包含 `win_market` 和 `Los_market` 的列。需要將單個比賽的記錄分成每個團隊的記錄，以便將每一行標記為`winner` 或 `loser`。

```sql
# create a row for the winning team
SELECT
  # features
  season, # ex: 2015 season has March 2016 tournament games
  round, # sweet 16
  days_from_epoch, # how old is the game
  game_date,
  day, # Friday

  'win' AS label, # our label

  win_seed AS seed, # ranking
  win_market AS market,
  win_name AS name,
  win_alias AS alias,
  win_school_ncaa AS school_ncaa,
  # win_pts AS points,

  lose_seed AS opponent_seed, # ranking
  lose_market AS opponent_market,
  lose_name AS opponent_name,
  lose_alias AS opponent_alias,
  lose_school_ncaa AS opponent_school_ncaa
  # lose_pts AS opponent_points

FROM `bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`

UNION ALL

# create a separate row for the losing team
SELECT
# features
  season,
  round,
  days_from_epoch,
  game_date,
  day,

  'loss' AS label, # our label

  lose_seed AS seed, # ranking
  lose_market AS market,
  lose_name AS name,
  lose_alias AS alias,
  lose_school_ncaa AS school_ncaa,
  # lose_pts AS points,

  win_seed AS opponent_seed, # ranking
  win_market AS opponent_market,
  win_name AS opponent_name,
  win_alias AS opponent_alias,
  win_school_ncaa AS opponent_school_ncaa
  # win_pts AS opponent_points

FROM
`bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`
```

結果

![](https://cdn.qwiklabs.com/0WkXm28I7FayRabxTC1Jx5UU8PhQCDXeWCb0dxKFTB8%3D)


## Part 1: Create a machine learning model to predict the winner based on seed and team name
### Choosing a model type
對於此問題，將構建*分類模型*。結果是兩個類別，輸或贏。因此也稱為二進制分類模型，一支球隊可以輸一場比賽或贏一場比賽。如果想在實驗之後，可以使用預測模型預測團隊將獲得的總分數，但這並不是我們要關注的重點。判斷是要進行預測還是分類的一種簡便方法是查看要預測數據的標籤類型：

- If it's a numeric column (like units sold or points in a game), you're doing forecasting
- If it's a string value you're doing classification (this row is either this in class or this other class)
- ... and If you have more than two classes (like win, lose, or tie) you are doing multi-class classification

我們的分類模型將使用稱為 `Logistic Regression` 的統計模型進行機器學習。我們需要一個為每個可能的離散標籤值生成機率的模型，此案例中，該值是`win`或 `loss`。


### Creating a machine learning model with BigQuery ML

要在 `BigQuery` 中創建分類模型，只需要編寫 `SQL` 語句 `CREATE MODEL` 並提供一些選項。但是，在建立模型前，需要在項目中建立一個位置以儲存模型。在 `BigQuery` 控制台左側點選專案，在右側點擊 `CREATE DATASET` 同時跳出一個視窗，並輸入 `bracketology` 至 `Dataset ID` 欄位，其餘預設。

```sql
CREATE OR REPLACE MODEL
  `bracketology.ncaa_model`
OPTIONS
  ( model_type='logistic_reg') AS

# create a row for the winning team
SELECT
  # features
  season,

  'win' AS label, # our label

  win_seed AS seed, # ranking
  win_school_ncaa AS school_ncaa,

  lose_seed AS opponent_seed, # ranking
  lose_school_ncaa AS opponent_school_ncaa

FROM `bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`
WHERE season <= 2017

UNION ALL

# create a separate row for the losing team
SELECT
# features
  season,

  'loss' AS label, # our label
  lose_seed AS seed, # ranking
  lose_school_ncaa AS school_ncaa,

  win_seed AS opponent_seed, # ranking
  win_school_ncaa AS opponent_school_ncaa

FROM
`bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`

# now we split our dataset with a WHERE clause so we can train on a subset of data and then evaluate and test the model's performance against a reserved subset so the model doesn't memorize or overfit to the training data.

# tournament season information from 1985 - 2017
# here we'll train on 1985 - 2017 and predict for 2018
WHERE season <= 2017
```

在代碼中，會注意到創建模型只是幾行 `SQL`。最重要的選項之一是為我們的分類任務選擇模型類型 `logistic_reg`。


### View model training details
現在已經在模型 `Details` 訊息中，向下滾動到`Training options`部分，然後查看模型執行訓練的實際迭代。如果有機器學習的經驗，需注意可以透過在`OPTIONS` 語句中定義它們的值來自定義所有這些超參數，該設定需在模型運行之前設置。可[參考這邊](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create#model_option_list)。

### View model training stats
機器學習模型學習已知特徵和未知標籤之間的關聯。`ranking seed` 或 `school name`之類的某些特徵可能比其他特徵更有助於確定贏還是輸。機器學習模型在沒有這種直覺的情況下開始訓練過程，並且通常會隨機化每個特徵的權重。在訓練過程中，模型將優化一條路徑，以對每個特徵進行最佳加權，每次運行時，它試圖將訓練數據 `Loss` 和評估數據 `Loss` 最小化。如果發現評估的最終損失比訓練要高得多，那麼模型就是*過度擬合*或評估訓練集中函有訓練的數據。

可以點擊 `Model Stats` 或 `Training` 查看模型進行了多少次訓練，如下圖所示

![](https://i.imgur.com/3Q7iSkB.png)

### See what the model learned about our features
訓練後，可以透過檢查權重來查看哪些特徵為模型提供了最大的價值。
```sql
SELECT
  category,
  weight
FROM
  UNNEST((
    SELECT
      category_weights
    FROM
      ML.WEIGHTS(MODEL `bracketology.ncaa_model`)
    WHERE
      processed_input = 'seed')) # try other features like 'school_ncaa'
      ORDER BY weight DESC
```

結果

![](https://i.imgur.com/OjWs9VX.png)

如果團隊的 `seed` 非常低(1、2、3)或非常高(14、15、16)，則模型會在確定勝利失敗的結果時給予很大的權重(最大值為 1)。機器學習的真正魔力在於，我們沒有在 `SQL` 中創建大量的硬編碼 `IF THEN` 語句來告訴模型 `IF` `seed` 是否為 1 `THEN` 從而使團隊獲勝的機會增加了80%。機器學習消除了硬編碼的規則和邏輯，透過自己學習了這些關係。有關[權重資訊](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-weights)資料。


## Evaluate model performance
評估模型的性能，可以對經過訓練的模型運行簡單的 `ML.EVALUATE`。

```sql
SELECT
  *
FROM
  ML.EVALUATE(MODEL   `bracketology.ncaa_model`)
```

結果

![](https://i.imgur.com/d49Rtbc.png)

該值的準確性約為 69%，雖然比硬幣翻轉更好，但仍有改進空間。

## Making predictions
已經訓練了一個模型，該模型使用了直到 `2017` 年之前的歷史數據，現在可對 2018 年季節進行預測了。進行預測就像在經過訓練的模型上調用 `ML.PREDICT` 並傳遞要預測的數據集一樣簡單。

```sql
CREATE OR REPLACE TABLE `bracketology.predictions` AS (

SELECT * FROM ML.PREDICT(MODEL `bracketology.ncaa_model`,

# predicting for 2018 tournament games (2017 season)
(SELECT * FROM `data-to-insights.ncaa.2018_tournament_results`)
)
)
```

最後將看到原始數據集，增加了三個新欄位
- Predicted label
- Predicted label options
- Predicted label probability

## How many did our model get right for the 2018 NCAA tournament?

```sql
SELECT * FROM `bracketology.predictions`
WHERE predicted_label <> label
```

結果

![](https://i.imgur.com/VMmcYvv.png)

## Models can only take you so far...
任何其他三月錦標賽的勝利和驚人的失誤都有許多其他因素和特徵，而模型將很難預測。根據模型，讓我們找到 2017 年錦標賽的最大障礙。我們將看看該模型以 80% 以上的置信度進行預測的地方和錯誤。

```sql
SELECT
  model.label AS predicted_label,
  model.prob AS confidence,

  predictions.label AS correct_label,

  game_date,
  round,

  seed,
  school_ncaa,
  points,

  opponent_seed,
  opponent_school_ncaa,
  opponent_points

FROM `bracketology.predictions` AS predictions,
UNNEST(predicted_label_probs) AS model

WHERE model.prob > .8 AND predicted_label <> predictions.label
```

結果
![](https://i.imgur.com/gxzXc8J.png)
## Recap
- 創建了一個機器學習模型來預測比賽結果
- 使用 `seed` 和 `team name` 作為主要特徵評估了性能並擁有了 69% 的準確性
- 預測了2018年錦標賽的結果
- 分析了結果以獲取見解

## Part 2: Using skillful ML model features
使用新提供的詳細特徵構建第二個 `ML` 模型。也已經熟悉使用 `BigQuery ML` 構建 `ML` 模型，數據科學團隊提供了一個新的詳細數據集，他們在其中創建了新的團隊指標供學習模型。包括以下
- Scoring efficiency over time based on historical play-by-play analysis.
- Possession of the basketball over time.

## Create a new ML dataset with these skillful features
```sql
# create training dataset:
# create a row for the winning team
CREATE OR REPLACE TABLE `bracketology.training_new_features` AS
WITH outcomes AS (
SELECT
  # features
  season, # 1994

  'win' AS label, # our label

  win_seed AS seed, # ranking # this time without seed even
  win_school_ncaa AS school_ncaa,

  lose_seed AS opponent_seed, # ranking
  lose_school_ncaa AS opponent_school_ncaa

FROM `bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games` t
WHERE season >= 2014

UNION ALL

# create a separate row for the losing team
SELECT
# features
  season, # 1994

  'loss' AS label, # our label

  lose_seed AS seed, # ranking
  lose_school_ncaa AS school_ncaa,

  win_seed AS opponent_seed, # ranking
  win_school_ncaa AS opponent_school_ncaa

FROM
`bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games` t
WHERE season >= 2014

UNION ALL

# add in 2018 tournament game results not part of the public dataset:
SELECT
  season,
  label,
  seed,
  school_ncaa,
  opponent_seed,
  opponent_school_ncaa
FROM
  `data-to-insights.ncaa.2018_tournament_results`

)

SELECT
o.season,
label,

# our team
  seed,
  school_ncaa,
  # new pace metrics (basketball possession)
  team.pace_rank,
  team.poss_40min,
  team.pace_rating,
  # new efficiency metrics (scoring over time)
  team.efficiency_rank,
  team.pts_100poss,
  team.efficiency_rating,

# opposing team
  opponent_seed,
  opponent_school_ncaa,
  # new pace metrics (basketball possession)
  opp.pace_rank AS opp_pace_rank,
  opp.poss_40min AS opp_poss_40min,
  opp.pace_rating AS opp_pace_rating,
  # new efficiency metrics (scoring over time)
  opp.efficiency_rank AS opp_efficiency_rank,
  opp.pts_100poss AS opp_pts_100poss,
  opp.efficiency_rating AS opp_efficiency_rating,

# a little feature engineering (take the difference in stats)

  # new pace metrics (basketball possession)
  opp.pace_rank - team.pace_rank AS pace_rank_diff,
  opp.poss_40min - team.poss_40min AS pace_stat_diff,
  opp.pace_rating - team.pace_rating AS pace_rating_diff,
  # new efficiency metrics (scoring over time)
  opp.efficiency_rank - team.efficiency_rank AS eff_rank_diff,
  opp.pts_100poss - team.pts_100poss AS eff_stat_diff,
  opp.efficiency_rating - team.efficiency_rating AS eff_rating_diff

FROM outcomes AS o
LEFT JOIN `data-to-insights.ncaa.feature_engineering` AS team
ON o.school_ncaa = team.team AND o.season = team.season
LEFT JOIN `data-to-insights.ncaa.feature_engineering` AS opp
ON o.opponent_school_ncaa = opp.team AND o.season = opp.season
```


## Preview the new features

點擊 `Go to table` 按鈕，然後點擊 `Preview`，結果如下圖

![](https://i.imgur.com/kugMyz7.png)    

## Interpreting selected metrics
### opp_efficiency_rank
在所有球隊中，隨著時間的推移，我們的對手能有效得分（每 100 顆籃球積分），越低越好。

### opp_pace_rank
在所有球隊中，我們的對手對籃球的持球等級（40 分鐘內的持球次數），越低越好。

此時可訓練第二個模型，為了保護模型免受`memorizing good teams from the past`的影響，下一個模型中排除 `team name` 和 `seed`，並僅關注上述指標。

## Train the new model
```sql
CREATE OR REPLACE MODEL
  `bracketology.ncaa_model_updated`
OPTIONS
  ( model_type='logistic_reg') AS

SELECT
  # this time, dont train the model on school name or seed
  season,
  label,

  # our pace
  poss_40min,
  pace_rank,
  pace_rating,

  # opponent pace
  opp_poss_40min,
  opp_pace_rank,
  opp_pace_rating,

  # difference in pace
  pace_rank_diff,
  pace_stat_diff,
  pace_rating_diff,


  # our efficiency
  pts_100poss,
  efficiency_rank,
  efficiency_rating,

  # opponent efficiency
  opp_pts_100poss,
  opp_efficiency_rank,
  opp_efficiency_rating,

  # difference in efficiency
  eff_rank_diff,
  eff_stat_diff,
  eff_rating_diff

FROM `bracketology.training_new_features`

# here we'll train on 2014 - 2017 and predict on 2018
WHERE season BETWEEN 2014 AND 2017 # between in SQL is inclusive of end points
```

## Evaluate the new model's performance
```sql
SELECT
  *
FROM
  ML.EVALUATE(MODEL     `bracketology.ncaa_model_updated`)
```

結果

![](https://i.imgur.com/5avcL3P.png)

訓練了具有不同特徵的新模型，並將準確性提高到原始模型的 75% 或 5%。這是機器學習中最大的經驗教訓之一，即高品質的特徵數據集可以極大地提高模型的準確性。

## Inspect what the model learned
該模型在贏或輸結果中權重最大的特徵是什麼？
```sql
SELECT
  *
FROM
  ML.WEIGHTS(MODEL     `bracketology.ncaa_model_updated`)
ORDER BY ABS(weight) DESC
```

![](https://cdn.qwiklabs.com/qXAA8wHbsHo7knP7xDLDsUooKa%2Flxr%2Bed96fxseIFQU%3D)

如結果中看到的那樣，前 3 個是 `progress_stat_diff`、`eff_stat_diff` 和 `eff_rating_diff`。


### pace_stat_diff

團隊之間的實際統計數據(持球/40分鐘)有多大差異。根據模型，這是選擇比賽結果的最大推動力。

### eff_stat_diff

團隊之間的實際統計(淨得分(net points )/100擁有)有何不同。

### eff_rating_diff

團隊之間的得分效率歸一化評級有何不同。


該模型在預測中沒有考慮什麼？是季節。在上面的排序權重輸出中，它排在最後，該模型的意思是季節(2013、2014、2015)對預測比賽結果沒有太大幫助。一個有趣的見解是，該模型評估了團隊的步調(他們控制球的能力)與團隊得分效率的關係。


## Prediction time!
```sql
CREATE OR REPLACE TABLE `bracketology.ncaa_2018_predictions` AS

# let is add back our other data columns for context
SELECT
  *
FROM
  ML.PREDICT(MODEL     `bracketology.ncaa_model_updated`, (

SELECT
* # include all columns now (the model has already been trained)
FROM `bracketology.training_new_features`

WHERE season = 2018

))
```

## Prediction analysis:
```sql
SELECT * FROM `bracketology.ncaa_2018_predictions`
WHERE predicted_label <> label
```

![](https://i.imgur.com/Hb4syT3.png)

從查詢返回的記錄中可以看出，該模型在錦標賽對戰總數中有 48 個對戰錯誤(24 場比賽)，2018 年的準確性為 64%。

## Where were the upsets in March 2018?
```sql
SELECT
CONCAT(school_ncaa, " was predicted to ",IF(predicted_label="loss","lose","win")," ",CAST(ROUND(p.prob,2)*100 AS STRING), "% but ", IF(n.label="loss","lost","won")) AS narrative,
predicted_label, # what the model thought
n.label, # what actually happened
ROUND(p.prob,2) AS probability,
season,

# us
seed,
school_ncaa,
pace_rank,
efficiency_rank,

# them
opponent_seed,
opponent_school_ncaa,
opp_pace_rank,
opp_efficiency_rank

FROM `bracketology.ncaa_2018_predictions` AS n,
UNNEST(predicted_label_probs) AS p
WHERE
  predicted_label <> n.label # model got it wrong
  AND p.prob > .75  # by more than 75% confidence
ORDER BY prob DESC
```

![](https://i.imgur.com/zTrILDG.png)

重大變化與我們之前的模型相同：UMBC 與 Virginia。2018 年總體來說是令人沮喪的一年。2019年會一樣嗎？

## Comparing model performance
原始的模型（比較 `seeds`）在哪裡弄錯了，而高級模型在哪裡弄錯了呢？
```sql
SELECT
CONCAT(opponent_school_ncaa, " (", opponent_seed, ") was ",CAST(ROUND(ROUND(p.prob,2)*100,2) AS STRING),"% predicted to upset ", school_ncaa, " (", seed, ") and did!") AS narrative,
predicted_label, # what the model thought
n.label, # what actually happened
ROUND(p.prob,2) AS probability,
season,

# us
seed,
school_ncaa,
pace_rank,
efficiency_rank,

# them
opponent_seed,
opponent_school_ncaa,
opp_pace_rank,
opp_efficiency_rank,

(CAST(opponent_seed AS INT64) - CAST(seed AS INT64)) AS seed_diff

FROM `bracketology.ncaa_2018_predictions` AS n,
UNNEST(predicted_label_probs) AS p
WHERE
  predicted_label = 'loss'
  AND predicted_label = n.label # model got it right
  AND p.prob >= .55  # by 55%+ confidence
  AND (CAST(opponent_seed AS INT64) - CAST(seed AS INT64)) > 2 # seed difference magnitude
ORDER BY (CAST(opponent_seed AS INT64) - CAST(seed AS INT64)) DESC
```
結果

![](https://i.imgur.com/SLijfss.png)


新模型會根據速度和射籃效率等新的熟練特徵正確預測意外。

## Predicting for the 2019 March Madness tournament
現在我們知道了 2019 年 3 月的球隊和種子排名，讓我們預測未來比賽的結果。

### Explore the 2019 data
```sql
SELECT * FROM `data-to-insights.ncaa.2019_tournament_seeds` WHERE seed = 1
```

![](https://cdn.qwiklabs.com/SpQJxX0bF8lBc4JcyPK4%2BCmyBThm1XfTuzefMk%2FnsPc%3D)

## Create a matrix of all possible games
由於我們不知道隨著比賽的進行哪支球隊會互相對戰，因此我們只會讓他們全部對戰對方。

```sql
SELECT
  NULL AS label,
  team.school_ncaa AS team_school_ncaa,
  team.seed AS team_seed,
  opp.school_ncaa AS opp_school_ncaa,
  opp.seed AS opp_seed
FROM `data-to-insights.ncaa.2019_tournament_seeds` AS team
CROSS JOIN `data-to-insights.ncaa.2019_tournament_seeds` AS opp
# teams cannot play against themselves :)
WHERE team.school_ncaa <> opp.school_ncaa
```

## Add in 2018 team stats (pace, efficiency)
```sql
CREATE OR REPLACE TABLE `bracketology.ncaa_2019_tournament` AS

WITH team_seeds_all_possible_games AS (
  SELECT
    NULL AS label,
    team.school_ncaa AS school_ncaa,
    team.seed AS seed,
    opp.school_ncaa AS opponent_school_ncaa,
    opp.seed AS opponent_seed
  FROM `data-to-insights.ncaa.2019_tournament_seeds` AS team
  CROSS JOIN `data-to-insights.ncaa.2019_tournament_seeds` AS opp
  # teams cannot play against themselves :)
  WHERE team.school_ncaa <> opp.school_ncaa
)

, add_in_2018_season_stats AS (
SELECT
  team_seeds_all_possible_games.*,
  # bring in features from the 2018 regular season for each team
  (SELECT AS STRUCT * FROM `data-to-insights.ncaa.feature_engineering` WHERE school_ncaa = team AND season = 2018) AS team,
  (SELECT AS STRUCT * FROM `data-to-insights.ncaa.feature_engineering` WHERE opponent_school_ncaa = team AND season = 2018) AS opp

FROM team_seeds_all_possible_games
)

# Preparing 2019 data for prediction
SELECT

  label,

  2019 AS season, # 2018-2019 tournament season

# our team
  seed,
  school_ncaa,
  # new pace metrics (basketball possession)
  team.pace_rank,
  team.poss_40min,
  team.pace_rating,
  # new efficiency metrics (scoring over time)
  team.efficiency_rank,
  team.pts_100poss,
  team.efficiency_rating,

# opposing team
  opponent_seed,
  opponent_school_ncaa,
  # new pace metrics (basketball possession)
  opp.pace_rank AS opp_pace_rank,
  opp.poss_40min AS opp_poss_40min,
  opp.pace_rating AS opp_pace_rating,
  # new efficiency metrics (scoring over time)
  opp.efficiency_rank AS opp_efficiency_rank,
  opp.pts_100poss AS opp_pts_100poss,
  opp.efficiency_rating AS opp_efficiency_rating,

# a little feature engineering (take the difference in stats)

  # new pace metrics (basketball possession)
  opp.pace_rank - team.pace_rank AS pace_rank_diff,
  opp.poss_40min - team.poss_40min AS pace_stat_diff,
  opp.pace_rating - team.pace_rating AS pace_rating_diff,
  # new efficiency metrics (scoring over time)
  opp.efficiency_rank - team.efficiency_rank AS eff_rank_diff,
  opp.pts_100poss - team.pts_100poss AS eff_stat_diff,
  opp.efficiency_rating - team.efficiency_rating AS eff_rating_diff

FROM add_in_2018_season_stats
```

## Make predictions
```sql
CREATE OR REPLACE TABLE `bracketology.ncaa_2019_tournament_predictions` AS

SELECT
  *
FROM
  # let's predicted using the newer model
  ML.PREDICT(MODEL     `bracketology.ncaa_model_updated`, (

# let's predict on March 2019 tournament games:
SELECT * FROM `bracketology.ncaa_2019_tournament`
))
```


## Get your predictions
```sql
SELECT
  p.label AS prediction,
  ROUND(p.prob,3) AS confidence,
  school_ncaa,
  seed,
  opponent_school_ncaa,
  opponent_seed
FROM `bracketology.ncaa_2019_tournament_predictions`,
UNNEST(predicted_label_probs) AS p
WHERE p.prob >= .5
AND school_ncaa = 'Duke'
ORDER BY seed, opponent_seed
```

結果

![](https://i.imgur.com/imwUEE5.png)


在這裡，我們過濾了模型結果以查看 `Duke` 的所有可能比賽，找到 `Duke vs North Dakota St.` 比賽，Insight: Duke (1) is 88.5% favored to beat North Dakota St. (16) on 3/22/19。通過更改上面的 `school_ncaa` 過濾條件來預測括號中的對戰。