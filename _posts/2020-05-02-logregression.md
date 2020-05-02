---
title: "Logistic Regression Part 1: Scraping the data"
date: 2020-05-02
---
The ongoing COVID-19 pandemic has, at the very least, suspended the 2020 sports season worldwide. On the bright side, that gives us time to make predictions of just how any season might turn out.

In this project, using Python, I first create a web scraper to gather MLB data from previous seasons, then use logistic regression to calculate each team's probability of making the 2020 postseason.

Keep in mind that I am still somewhat of an amateur at writing code, especially in Python so this all a learning process for me, that includes the analysis.

## Data Description
The data will be scraped from [Baseball Reference]("https://www.baseball-reference.com"). It contains data from as far back as the 1870s, however, because the structure of the playoffs, and baseball as a whole, has changed often and drastically since, the data I use goes back only to 2012, the first year that the current playoff structure was implemented. The changes deal primarily with the amount of teams allowed to make it to the playoffs, and in 2012 the MLB expanded it to 10 teams.    

Therefore, we will be gathering batting, and pitching data from the 2012 - 2019 seasons.

## Building the web scraper
The URL that contains the data we need:

<img src="{{ site.url }}{{ site.baseurl }}/images/bsbrefurl.png" alt="URL being scraped">

Fortunately, all the data we need are on one page, so the logic for sending requests to the url is straight forward, given its format. It just requires replacing 2019 with the year we need data for.

All the libraries I used:
```python
from bs4 import BeautifulSoup,Comment
import pandas as pd
import time
import requests
```

I used BeautifulSoup to parse the HTML.
```python
   url = "https://www.baseball-reference.com/leagues/MLB/{}.shtml".format(year)
   headers = {'user-agent': "Mozilla/5.0 (X11; U; Linux i686) Gecko/20071127 Firefox/2.0.0.11"}
   page = requests.get(url, headers=headers)
   soup = BeautifulSoup(page.text, 'lxml')
```
The HTML code for the batting data table:

<img src="{{ site.url }}{{ site.baseurl }}/images/battingtable.png" alt="HTML">
```python
   batting_table = soup.find("div", attrs={"id": "div_teams_standard_batting"})
```
It is essentially the same for the pitching data table, however there is an issue where the code for it is commented in for some reason, so I had to add a few lines of code to get what I needed out of it (Thanks to StackOverflow for helping me figure this out).

```python
    comments = soup.find_all(text=lambda text: isinstance(text, Comment))
    pitching_html = comments[19]
    pitching_table = BeautifulSoup(pitching_html, 'lxml')
```
## Code
The rest of the code consists of extracting the stats I needed, creating a dataframe, and then exporting it to a csv file. Here it is in its entirety:

```python
from bs4 import BeautifulSoup,Comment
import pandas as pd
import time
import requests


def batting_stats(bstat):
    tables = batting_table.find_all("td", attrs={"data-stat": bstat})

    b_stats = []
    for table in tables:
        b_stat = table.text
        b_stat = float(b_stat)
        b_stats.append(b_stat)

    b_stats = b_stats[:-2] #exclude total and average

    return b_stats

def pitching_stats(pstat):
    tables = pitching_table.find_all("td", attrs={"data-stat": pstat})

    p_stats = []
    for table in tables:
        p_stat = table.text
        if p_stat == '':
           p_stat = "empty"
        else:
            p_stat = float(p_stat)

        p_stats.append(p_stat)

    p_stats = p_stats[:-2]

    return p_stats


def postseason(team_names, yr): # 1 if a team made the playoff that year 0 if not
    playoff_teams = []

    go = True
    while go:
        team = input("Enter playoff teams for {}. Enter any number when all are in \n".format(yr))
        playoff_teams.append(team)

        if team.isdigit():
            go = False
            print("{} Postseason teams: ".format(yr), playoff_teams[:-1])
        else:
            print(playoff_teams)

    playoff = []
    for a in range(0, len(team_names)):
        if team_names[a] in playoff_teams:
            playoff.append(1)
        else:
            playoff.append(0)

    return playoff


all_years = []

for year in range(2012, 2020):

    url = "https://www.baseball-reference.com/leagues/MLB/{}.shtml".format(year)
    headers = {'user-agent': "Mozilla/5.0 (X11; U; Linux i686) Gecko/20071127 Firefox/2.0.0.11"}
    page = requests.get(url, headers=headers)
    soup = BeautifulSoup(page.text, 'lxml')

    batting_table = soup.find("div", attrs={"id": "div_teams_standard_batting"})

    comments = soup.find_all(text=lambda text: isinstance(text, Comment))
    pitching_html = comments[19]
    pitching_table = BeautifulSoup(pitching_html, 'lxml')

    name_table = batting_table.find_all("th", attrs={"data-stat": "team_ID"})

    names = []
    for name in name_table: #Get team names
        n = name.text
        names.append(n)
    names = names[:-3]
    names = names[1:]

    years = []
    for i in range(0, len(names)): #Get the year of the season
        years.append(year)

    playoffs = postseason(names, year)

    b_dict = {'Tm': names, 'Yr': years, 'Playoff': playoffs}

    b_stat_names = ['batters_used','age_bat','runs_per_game','G','PA','AB','R','H','2B','3B','HR','RBI',
                    'SB','CS','BB','SO','batting_avg','onbase_perc','slugging_perc','onbase_plus_slugging','TB'
                  ,'GIDP','HBP','SH','SF','IBB','LOB']

    p_stat_names = ['pitchers_used', 'age_pitch', 'runs_allowed_per_game', 'W', 'L', 'win_loss_perc', 'earned_run_avg',
                    'G', 'GS', 'GF', 'CG', 'SHO_team', 'SHO_cg', 'SV', 'IP', 'H', 'R', 'ER', 'HR', 'BB', 'IBB', 'SO',
                    'HBP', 'BK','WP', 'batters_faced', 'fip', 'whip', 'hits_per_nine', 'home_runs_per_nine',
                    'bases_on_balls_per_nine','strikeouts_per_nine', 'strikeouts_per_base_on_balls', 'LOB']

    p_dict = {'Tm': names, 'Yr': years,'Playoff': playoffs}
    for x in p_stat_names:
        p_dict[x] = pitching_stats(x)

    df_pitching = pd.DataFrame.from_dict(p_dict)

    for i in b_stat_names:
        b_dict[i] = batting_stats(i)

    df_batting = pd.DataFrame.from_dict(b_dict)

    df_batting_pitching = df_pitching.merge(df_batting,on=['Yr','Tm','Playoff'],how='left',suffixes=('_pitching','_batting'))
    all_years.append(df_batting_pitching)

    time.sleep(6)

df_all = pd.concat(all_years)
print(df_all)

df_all.to_csv('mlb_since_2012.csv')
```

We now have all the data we need for this project. I decided to also use each team's payroll as an additional predictor. I simply entered each datapoint in manually.

In my next post I will get into the analysis, and build the logistic regression model.
