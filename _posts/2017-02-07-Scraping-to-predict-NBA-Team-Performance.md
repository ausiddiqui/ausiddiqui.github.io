---
layout: post
title: "Predicting what makes the ultimate NBA Team?"
comments: false
description: "Quick exploratory models in sports performance using BeautifulSoup, pandas and Random Forest"
keywords: "random forest, linear regression, nba, scraping, pandas"
---
![MJ Dunks](/assets/images/jordan-dunk.jpg)

### **Sum of Parts**
{: .text-justify}
Team sports like basketball require a balance of skills and technical acumen from different specialists in order to succeed at the highest level. I know this is especially true in the case of soccer which I follow religiously, but given the smaller squad size and the dynamic and often erratic nature of basketball games and scoring I always wondered if all it takes is a superstar. Growing up like many kids in the 90s the Michael Jordan era was an unbelievable time to follow the NBA and support the Bulls - and in subsequent eras we've seen teams continue to be built around superstar players and dominate the game - Kobe/Shaq's Lakers, Lebron/Wade in Miami, etc. So is a team's performance greater than the sum of parts or are one or two superstars all you need?  

In order to test this out, I look at some stats around players from [Basketball-Reference.com](http://www.basketball-reference.com/), a site dedicated to a plethora of, as you guessed it stats! The idea is that there might be a model that combines the player stats for each of the positions on a roster to predict how the team will perform overall.

The data I needed were the following:  

+ Franchise Teams and their range of activity (_used all 30 active franchises in 2016/17_)  
+ Team rosters for each year and the season results
+ Player stats for each year  

The idea would be to _**look at this year's roster**_, get the player stats for the **players most likely to play** the most number of minutes for each of the starting positions and then **_predict the Win-Loss Percentage for this season_** based on that.

---

### **The Rabbit Hole of Scraping**  
Scraping can be a pretty fun activity - (a) you are kind of getting something (_i.e. the data_) out of nothing (b) given how tedious it can be - there's so much joy when you catch those ```<div>```s and ```<td>```s.

In order to get the data from Basketball-Reference, I used the **BeautifulSoup**, **requests** and **fake_useragent** python libraries. Getting the franchises table was pretty easy. It was however much harder to get through the tables that had seasonal stats per year for each team for all the players.

These tables looked like:

![Roster Table](/assets/images/roster_table.png)
![Advanced Table](/assets/images/advanced_table.png)
![Advanced Table HTML](/assets/images/advanced_html.png)
>>  These tables are from the [2016-17 Golden State Warriors](http://www.basketball-reference.com/teams/GSW/2017.html) page. The last image is the html code for the Advanced Table.

In scraping this data, there were two major hurdles:  

+ Table data appeared under the comments tag ```<!-- -->```
+ Unicode content from BeautifulSoup had to be handled for Python 2.7

To overcome the first I used the following regular expression code snippet to substitute these comment tags out with an empty string:

```python
import re
comm = re.compile("<!--|-->")
clean_soup = BeautifulSoup(re.sub("<!--|-->","",response.text),'lxml')
```

For the next challenge it was a number of transformations:

```python
table_t = unicode.join(u'\n',map(unicode, clean_soup.findAll("table", {"id": "roster"})[0].findAll('tr'))).encode('ascii','ignore')
```

What this does is map all the extractions to ```unicode``` at every stage until the very end, where it is re-encoded as ```ascii``` ignoring any errors for characters that are not found.

In total the following ranges of years for these 30 franchises was scraped, totaling 1,453 pages for each individual season. Out of each page the 'Rosters', 'Per Game', 'Totals' and 'Advanced' tables were collected for stats on individual players in that team in that season.  

Franchise | From | To | Years
------------ | ------------ | ------------ | ------------
Atlanta Hawks | 1950 | 2017 | 68
Boston Celtics | 1947 | 2017 | 71
Brooklyn Nets | 1968 | 2017 | 50
Charlotte Hornets | 1989 | 2017 | 27
Chicago Bulls | 1967 | 2017 | 51
Cleveland Cavaliers | 1971 | 2017 | 47
Dallas Mavericks | 1981 | 2017 | 37
Denver Nuggets | 1968 | 2017 | 50
Detroit Pistons | 1949 | 2017 | 69
Golden State Warriors | 1947 | 2017 | 71
Houston Rockets | 1968 | 2017 | 50
Indiana Pacers | 1968 | 2017 | 50
Los Angeles Clippers | 1971 | 2017 | 47
Los Angeles Lakers | 1949 | 2017 | 69
Memphis Grizzlies | 1996 | 2017 | 22
Miami Heat | 1989 | 2017 | 29
Milwaukee Bucks | 1969 | 2017 | 49
Minnesota Timberwolves | 1990 | 2017 | 28
New Orleans Pelicans | 2003 | 2017 | 15
New York Knicks | 1947 | 2017 | 71
Oklahoma City Thunder | 1968 | 2017 | 50
Orlando Magic | 1990 | 2017 | 28
Philadelphia 76ers | 1950 | 2017 | 68
Phoenix Suns | 1969 | 2017 | 49
Portland Trail Blazers | 1971 | 2017 | 47
Sacramento Kings | 1949 | 2017 | 69
San Antonio Spurs | 1968 | 2017 | 50
Toronto Raptors | 1996 | 2017 | 22
Utah Jazz | 1975 | 2017 | 43
Washington Wizards | 1962 | 2017 | 56

---

### **Transformations**  

With all this scraping out of the way, the data looked like this as a ```pandas``` table:  

![Players Data Frame](/assets/images/players_df1.png)
![Players Data Frame Continued](/assets/images/players_df2.png)
>> The images above show the first four players in the rosters list from the Atlanta Hawks 2017 season along with their biographic and season stats information.  

At this stage the dataframe contained all the players in all the current 30 active franchises from start to finish with 22,919 rows and 68 numerical stats that could be considered in the feature set. There were certain assumptions made to clean and simply the dataset for building out a model:  

+ Every player's stat is shifted by 1 year, i.e. consider being at the beginning of the season and only looking at the past year of stats for the players - rookies would thus be discarded.
+ If a player moved teams, mid-season only consider where he played the most minutes. For e.g. Vince Carter moved from the Toronto Raptors to the New Jersey Nets in 2005 and then again from Orlando to Phoenix mid-season in 2011.
+ All positions were simplified into either Guard, Forward or Center. Players with positions 'F-C' or 'F-G' were considered as Forwards, 'G-F' as Guards and 'C-F' as Centers, respectively.
+ The players with the top two total minutes played for the position of Guard or Forward from last season (irrespective of what team they played for) were picked to be in the model. Similarly, the Center with the top total minutes played from last season was picked for that position.
+ Missing data for the 'Player Efficiency Rating (PER)', 'Win-Share (WS)' and 'Value Over Replacement Player (VORP)' were filled in with the sample mean from across all player/years.
+ For all other stats missing values were treated as zero.

---

### **Feature Pruning**
![A useless pairplot](/assets/images/sns_plot.png)
>> Plotting all the pairwise distributions of features.  

Before setting up the data for each of the 5 most influential players per team per season by position, some of these 68 features needed to be reduced.

The following 14 were considered to be the most relevant:  

+ Games Played (g)
+ Games Started (gs)
+ Minutes Played per Game (mp_per_g)
+ Field Goals per Game (fg_per_g)
+ 3-Pt Field Goals per Game (fg3_per_g)
+ Rebounds per Game (trb_per_g)
+ Assists per Game (ast_per_g)
+ Blocks per Game (blk_per_g)
+ Total Minutes Played (mp)
+ Total Points Scored (pts)
+ Player Efficiency Rating (per)
+ True Shooting Percentage (ts_pct)
+ Win-Share (ws)
+ Value-Over-Replacement-Player (vorp)

I included a mix of features that were at the per game level, total for the season level and advanced statistics that compare against other players in the team or the league (e.g. vorp, ws).

Once these features were reduced the dataframe was pivoted based on the Team, Season and Player Position and I ended up with 1,393 rows. Each row of this dataframe is a Team in a particular season, and the columns are these 14 features x 5 positions (2 Guards, 2 Forwards and 1 Center) who played the most in their squad last year.

![Final Dataframe](/assets/images/pivot_df.png)

### **Finally the models come out**  

The dataset is finally ready to be run through a few models. The predictors are these 14 x 5 - 70 features which represent the player's in the roster this year and their performance last year and the target is the Win-Loss Percentage of the team at the end of the current year.

![Games Started vs Season Win-Loss Pct](/assets/images/gs_scatter.png)
![Total Points vs Season Win-Loss Pct](/assets/images/pts_scatter.png)
>> The top figure is the distribution of the number of Games Started by position versus the Season Win-Loss Percentage. The bottom figure shows the Total Points versus the target.  

A number of features ended up being quite uniformly distributed, such as the Games Started when plotted against the target. In other instances, some of the anomalies and outliers in the dataset show up. For example, the Total Points should be a good indicator for a team's performance - i.e. teams with players who score a lot, should also perform better - however it isn't obvious how much the Center would influence this. But looking at the scatter it turns out Wilt Chamberlain who was a Center, had record breaking seasons while playing for the Philadelphia Warriors, in 1960-61 he scored 3,033 points, the next season he scored 4,029 followed by 3,586 the year after.

The models fit eventually were using standard Linear Regression with and without Lasso regularization. Additionally, tree based regressors were run using Random Forest and Gradient Boosting.

The dataset was split using cross validation into a 70/30 train/test split with no separate hold-out other than the test set.

The best fit was found using GridSearchCV using Lasso - with a parameter of ```alpha = 1.0e-05```.

Model | Parameter | R²
------------ | ------------ | ------------
Linear Regression w/Lasso | alpha=0.00001; all 70 features | 0.5204
Linear Regression | all 70 features | 0.5101
Gradient Boosting Regression | estimators=2,000; learning rate=0.005; max depth=6; subsample=0.8; min leaf=3; all 70 features | 0.4557
Random Forest Regression | estimators=1,000, max features=8, min sample leafs=4; all 70 features | 0.4247

A number of models were run using a smaller subset of features based on the regularization, low or zero coefficient features were dropped, but it did not end up significantly improving the models.

The most important features for the Linear Regression models were:
+ True Shooting Pct:
  + Guard 1 (17.8)
  + Guard 2 (-7.0)
  + Forward 1 (-26.5)
  + Forward 2 (-13.0)
+ Field Goals per Game: Guard 1 (1.6)
+ 3 Pt Field Goals per Game: Center (1.3)

For the Tree Based Models the Win-Share was 3 times more important than any other features.

The ```ts_pct``` already embeds much of the data for Field Goals and 3-Pt Field Goals, so it is likely one of the reasons why this dominates. The negative coefficients for the non-Guard 1 players seems to suggest that the model is using the regularization to compensate for over-estimating the importance of Guard 1's True Shooting Percentage.

---

### **Some Predictions**  

![Best Model Fit](/assets/images/pred_scatter.png)

Based on the best model, I predicted the current season's outcomes for total wins given the players on the roster out of 82 games in the 2016-17 regular season.

So far out of 35 games played, the model predicts the top 3 but is closest in predictions to the Warriors who lead the pack. A number of other combinations of features and feature engineering might end up improving this and building a more robust prediction engine for team performance.

Team | Regular Season | Predicted Pct | Current Pct (Actual) | Current Rank (by Win Pct)
------------ | ------------ | ------------ | ------------ | ------------
Golden State Warriors | 70 | 85.4% | 84.3% | 1
San Antonio Spurs | 57 | 69.5% | 76.5% | 2
Cleveland Cavaliers | 54 | 65.9% | 70.0% | 3
Toronto Raptors | 50 | 61.0% | 60.4% | 7
Los Angeles Clippers | 48 | 58.5% | 59.6% | 8

---

#### **Sources**
[1] Data Source: [Basketball-Reference.com](http://www.basketball-reference.com)  
[2] Image Source: [Science ABC](http://www.scienceabc.com)
