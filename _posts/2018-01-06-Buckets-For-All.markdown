---
layout: post
title:  "Buckets For All"
date:   2018-01-04 15:00:00 +0000
categories: jekyll update
---

_This brief tutorial shows how you can use the Hoopsville package to create custom video playlists on NBA.com. If you have any questions or feedback, find me on [Twitter](http://www.twitter.com/worville)._

As someone who's trying to learn about basketball, I'm fortunate that there are a lot of #content* out there that I can lean on to get a better understanding about all areas of the sport.

For example, Zach Lowe's weekly [Ten Things](http://www.espn.co.uk/nba/story/_/id/21961715/zach-lowe-10-things-like-including-lebron-james-kevin-love-nba) pieces have taught me a lot, as has his [Lowe Post podcast](http://www.espn.com/espnradio/podcast/archive/_/id/10528553). My favourite place to read/watch/learn about basketball though is [Cleaning The Glass](http://cleaningtheglass.com/). It's a subscription based website, but you get stats, in-depth articles and a discussion forum for a measly $5 a month.

(_*I'm not going to list loads of links in this post, but if you want to learn more about the NBA and find out what to read/watch/listen to and who to follow then you won't find a more in depth list than the r/NBA "PhD Program" [here](https://www.reddit.com/r/nba/comments/76rluz/the_nba_phd_program_year_4/)._)

One of the main ways to learn about a sport is to actually watch the games (duh) but to do this you need video. Footage of full games can be easily found online - if you know where to look - but these are usually a few gigabytes in size and not always what you want when focusing on a specific facet of a given player or team's play.

YouTube [highlight channels](https://www.youtube.com/channel/UCEjOSbbaOfgnfRODEEMYlCw) showcase the made buckets and big moments of every game, but again these are never tailored to a specific player (unless they've had a noteworthy game).

Using my [Hoopsville python package](https://github.com/worville/hoopsville) and [NBA.com](nba.com) I'm going to show how you can create custom playlists. Don't worry if you don't have any Python experience, I'll post the files you need online ([skip to the files here](#player-and-game-data-dumps)).

### Finding the Video

Thanks to [Ben Falk](https://twitter.com/bencfalk) (founder of the aforementioned Cleaning The Glass site) I was pointed in the direction of [this page](http://stats.nba.com/events/?flag=3&CFID=&CFPARAMS=&PlayerID=204001&TeamID=1610612752&GameID=0021700550&ContextMeasure=FGA&Season=2017-18&SeasonType=Regular%20Season&RangeType=0&StartPeriod=1&EndPeriod=10&StartRange=0&EndRange=28800&section=game&sct=plot) on the NBA.com site, after asking Ben how he finds footage efficiently for his articles.

<br>

<p align="center">
<img src="/assets/nba_playlist_1.png" width="600">
</p>

It's part of the site that allows you to query any player/team/game and watch the video of their on ball events (think assists, rebounds, field goal attempts, etc.). Quite a powerful tool, which I'm fortunate to have access to a similar sort of thing for [football in my day job](http://www.optasportspro.com/products/provision/).

To change the playlist, you just need to change out the separate parts of the following URL:

[http://stats.nba.com/events/?flag=3&CFID=&CFPARAMS=&PlayerID=204001&TeamID=1610612752&GameID=0021700550&ContextMeasure=FGA&Season=2017-18&SeasonType=Regular%20Season&RangeType=0&StartPeriod=1&EndPeriod=10&StartRange=0&EndRange=28800&section=game&sct=plot](http://stats.nba.com/events/?flag=3&CFID=&CFPARAMS=&PlayerID=204001&TeamID=1610612752&GameID=0021700550&ContextMeasure=FGA&Season=2017-18&SeasonType=Regular%20Season&RangeType=0&StartPeriod=1&EndPeriod=10&StartRange=0&EndRange=28800&section=game&sct=plot)

The parameters that can be changed in the URL are:

- PlayerID - 204001 here is Kristaps Porzingis
- TeamID - 1610612752 here is the New York Knicks
- GameID - 0021700550 here is the game between the New York Knicks and the San Antonio Spurs
- ContextMeasure - FGA here is Field Goal Attempts (i.e. all 2 or 3 point shots)
- Season - 2017-18 here is the current 17/18 season

There are a couple more that could be tweaked, but these aren't relevant right now.

## Pulling the Data

This link is great if you're really interested in looking at Porzingis' shot attempts, but pointless if you want to look at Chris Paul's season high 14 assists against the Brooklyn Nets back in November.

To get a playlist of Paul's assists I need his PlayerID and GameID. To grab both of these things, I'll use the `Hoopsville` package. First, I'll get Paul's PlayerID.

After installing the [Hoopsville](https://github.com/worville/hoopsville) package, fire up iPython and import the package:

`import hoopsville as hv`

You can then get a pandas DataFrame of all of the players who've played in the league this season by running:

`players = hv.get_players()`

The first few rows and columns of the `players` DataFrame looks like this:

<style>
.tablelines table, .tablelines td, .tablelines th {
        border: 0.5px solid grey;
        }
</style>

`players.iloc[0:4, 0:7].set_index('PERSON_ID')`

| PERSON_ID | DISPLAY_LAST_COMMA_FIRST | DISPLAY_FIRST_LAST | ROSTERSTATUS | FROM_YEAR | TO_YEAR | PLAYERCODE    |
|-----------|--------------------------|--------------------|--------------|-----------|---------|---------------|
| 203518    | Abrines, Alex            | Alex Abrines       | 1            | 2016      | 2017    | alex_abrines  |
| 203112    | Acy, Quincy              | Quincy Acy         | 1            | 2012      | 2017    | quincy_acy    |
| 203500    | Adams, Steven            | Steven Adams       | 1            | 2013      | 2017    | steven_adams  |
| 1628389   | Adebayo, Bam             | Bam Adebayo        | 1            | 2017      | 2017    | bam_adebayo   |
| 201167    | Afflalo, Arron           | Arron Afflalo      | 1            | 2007      | 2017    | arron_afflalo |
{: .tablelines}
<br>
I shortened this down so that it'd fit properly on the page, but this is the only information that is needed to continue.

The `PLAYERCODE` here looks to be the combination of a player's first name, an underscore and their last name. In order to get Chris Paul's data I should only have to filter down the DataFrame just using `chris_paul`.

First I'll store the filtered down DataFrame containing just Paul's `PLAYERCODE`:

`cp3 = players.loc[players.PLAYERCODE == 'chris_paul']`

Next I'll take a look at the table, again just taking the first seven columns:

`cp3.iloc[:, 0:7].set_index('PERSON_ID')`

| PERSON_ID | DISPLAY_LAST_COMMA_FIRST | DISPLAY_FIRST_LAST | ROSTERSTATUS | FROM_YEAR | TO_YEAR | PLAYERCODE |
|-----------|--------------------------|--------------------|--------------|-----------|---------|------------|
| 101108    | Paul, Chris              | Chris Paul         | 1            | 2005      | 2017    | chris_paul |
{: .tablelines}
<br>

Perfect, Paul's `PERSON_ID` is 101108. Now all that's needed is the `Game_ID` for the game against the Nets.

Using the `get_teams()` function in `Hoopsville` I can quickly look up what the Houston Rockets `Team_ID` is:

`hv.get_teams().set_index('TEAM_ID').head(10)`

| TEAM_ID    | ABBREVIATION |
|------------|:------------:|
| 1610612737 |      ATL     |
| 1610612738 |      BOS     |
| 1610612739 |      CLE     |
| 1610612740 |      NOP     |
| 1610612741 |      CHI     |
| 1610612742 |      DAL     |
| 1610612743 |      DEN     |
| 1610612744 |      GSW     |
| 1610612745 |      HOU     |
| 1610612746 |      LAC     |
{: .tablelines}
<br>
The `Team_ID` for Rockets is 1610612745. I can then plug this into the `get_team_games` function to get data on their games this season:

`HOU = hv.get_team_games(tid=1610612745)`

Again, I'll use a shortened version of the DataFrame:

`HOU.iloc[17:18,0:4].set_index('Team_ID')`

| Team_ID    | Game_ID    | GAME_DATE    | MATCHUP     |
|------------|------------|--------------|-------------|
| 1610612745 | 0021700294 | NOV 27, 2017 | HOU vs. BKN |
{: .tablelines}

<br>


## Getting The Video

Now that I have Paul's `Player_ID` and `Game_ID` I can plug them into the URL used previously to get his assists vs the Nets. Note I also changed the `ContextMeasure` to `AST` to just get the footage on his assists:

[http://stats.nba.com/events/?flag=3&CFID=&CFPARAMS=&PlayerID=101108&TeamID=1610612745&GameID=0021700294&ContextMeasure=AST&Season=2017-18&SeasonType=Regular%20Season&RangeType=0&StartPeriod=1&EndPeriod=10&StartRange=0&EndRange=28800&section=game&sct=plot](http://stats.nba.com/events/?flag=3&CFID=&CFPARAMS=&PlayerID=101108&TeamID=1610612745&GameID=0021700294&ContextMeasure=AST&Season=2017-18&SeasonType=Regular%20Season&RangeType=0&StartPeriod=1&EndPeriod=10&StartRange=0&EndRange=28800&section=game&sct=plot
)

<p align="center">
<img src="/assets/nba_playlist_2.png" width="600">
</p>

It works! Now you can repeat the same process for any player, team or game in the past few seasons.

<br>

## Player and Game data dumps

_I'll look to add these to a GitHub gist in the next day or so. Sorry!_
