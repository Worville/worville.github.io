---
layout: post
title:  "Cleaning the MLS Salary Dump"
date:   2018-05-13 22:00:00 +0000
categories: jekyll update
---
Last week the [MLS Players Union](https://twitter.com/MLSPlayersUnion) released salary data for the 2018 MLS season. The data's great and does exactly what it says on the tin - it tells you who's getting paid what - but the ability to use this data for any sort of analysis is miniscule in its raw form.

The salary information itself is interesting, but when combined with roster information (e.g. Designated Player information, Senior/Reserve/Supplemental roster slot designations) there's a lot more you can do.

Below is the code that I used to:
- Scrape the [MLS Roster pages](https://www.mlssoccer.com/rosters/2018)
- Clean the roster data
- Merge the roster data with the salary information (albeit through some pretty hacky methods...)

Feel free to re-run the data yourself (you'll need to ensure that you have the CSV of [raw salary data](https://gist.github.com/Worville/1835de67da09a1c17f186c3086511e09) saved in your current working directory). I've included a few comments, but I'd recommend running the code line by line.

This script could definitely be improved with functions and I didn't really go back to re-write any of the earlier parts of this script - but hopefully it contains a few useful snippets you can take away and use elsewhere!

If you don't want to run through the code, and just want to get the data to play with you can find it [here](https://t.co/7py34jC5uL). Click `Download Zip` and unzip that file to get the CSV.

Any questions/thoughts, find me on twitter [@Worville](http://www.twitter.com/Worville).

```
# MLS roster scraping and salary merge

# Load in the salary data
library(readr)
library(dplyr)

salaries <- read_csv("mls_salaries_05-18.csv") %>%
            filter(Club != "Major League Soccer L.L.C")

# Create some basic ID columns and add them on to the salaries
salary_team_ids <- salaries %>%
                  select(Club) %>%
                  unique() %>%
                  arrange(Club) %>%
                  mutate(team_id = seq(1, n()))

# Merge the team_ids back into the data
salaries <- merge(salaries, salary_team_ids, by = "Club")

# rename the name fields
salaries <- salaries %>% rename(first_name = `First Name`,
                                last_name = `Last Name`,
                                base_salary = `Base Salary`,
                                total_compensation = `Total Compensation`)

# Trying to download MLS Roster Data first...

library(tidyverse)
library(rvest)
library(stringr)

# Need to get a vector of teams of which to loop through and get
# roster data on

other_url <- "https://www.mlssoccer.com/rosters/2018"

team_roster_urls <- read_html(other_url) %>%
  # filter down to just the HTML tags we care about
  html_nodes('.field-items, .field-item even,
             .u24container section group a,
             .col span_1_of_3 u24box a,
             .u24overlay a') %>%
  # get the href attribute, i.e. the web address
  html_attr("href") %>%
  as.character() %>%
  # drop the NAs
  na.omit() %>%
  # keep just the 24 teams' addresses
  .[0:23]

# store the MLS URL
mls_url <- "https://www.mlssoccer.com"

rosters_df <- data.frame()
# for the team in the vector of team roster urls...
for(team in team_roster_urls){
  print(team)
  Sys.sleep(sample(seq(0.5, 1.5, by=0.001), 1)) # To prevent denial of service
  # ...create the url to use
  team_roster_url <- paste(mls_url, team, sep="")

  print(team_roster_url)
  # ...store the team name
  name <- strsplit(team, "/")[[1]][4] %>%
    str_replace(., "-", " ") %>%
    str_replace(., "-", " ") %>%
    str_to_title()

  # ...grab the roster data
  team_data <- read_html(team_roster_url) %>%
    html_node('.activethirty') %>%
    html_table(fill = TRUE) %>%
    data.frame(.) %>% head(30)

  team_data <- team_data %>% mutate(team = name)

  # ...bind the data together
  rosters_df <- rbind(rosters_df, team_data)
}

# rename the rosters_df columns

full_rosters <- rosters_df[c(1:6, 11)] %>% setNames(c("player", "shirt_num",
                                      "pos", "roster status", "category", "notes",
                                      "team")) %>% filter(shirt_num >= 0)

# split out the player column to first name and surname

full_rosters <- full_rosters %>%
  mutate(first_name = sapply(strsplit(full_rosters$player, ", "), function(x) x[2]),
         last_name = sapply(strsplit(full_rosters$player, ", "), function(x) x[1]))

# Create some basic ID columns and add them to the teams
full_rosters_team_ids <- full_rosters %>%
                          select(team) %>%
                          unique() %>%
                          arrange(team) %>%
                          mutate(team_id = seq(1, n()))

# merge the team_ids back into the full_rosters data
full_rosters <- merge(full_rosters, full_rosters_team_ids, by="team")

# Initial join, solves ~95% of the names...
full_rosters_w_salary <- merge(full_rosters, salaries,
                               by.x = c("first_name", "last_name", "team_id"),
                               by.y = c("first_name", "last_name", "team_id"),
                               all.x = TRUE)

# Just those players where the join wasn't successful with the last names being wrong
left_over <- merge(full_rosters, salaries,
                            by.x = c("first_name", "last_name", "team_id"),
                            by.y = c("first_name", "last_name", "team_id"),
                            all.x = TRUE) %>% filter(is.na(base_salary) == TRUE)

left_over_fill <- merge(left_over[0:10], salaries %>% select(-last_name),
                   by.x = c("first_name", "team_id"),
                   by.y = c("first_name", "team_id"))

full_rosters_w_salary <- rbind(full_rosters_w_salary, left_over_fill) %>%
                          filter(is.na(base_salary) == FALSE)

# Now get the left_over players again who still remain
merged_players <- left_over_fill %>% select(player) %>% pull()

still_left_over <- left_over %>% filter(!player %in% merged_players)

# remove Oscar Garcia from the data as he's still being a pain
still_left_over <- still_left_over %>% filter(!first_name == "Boniek" |
                                                is.na(first_name) == TRUE)

wrong_named_players <- merge(still_left_over[0:10],
                             salaries %>% select(-first_name),
                             by.x = c("last_name", "team_id"),
                             by.y = c("last_name", "team_id"))

# merge in the guys with wrong names to our overall data
full_rosters_w_salary <- rbind(full_rosters_w_salary,
                               wrong_named_players) %>%
                          filter(is.na(base_salary) == FALSE)

# get a list of names of guys just added
wrong_names_players <- wrong_named_players %>% select(last_name, team_id)

# get a final list of the players who haven't been merged
final_left_over <- still_left_over %>%
  filter(!last_name %in% wrong_names_players$last_name)

# manually recode the names in the final list, so that the salary data
# can easily be merged onto it.
final_left_over <- final_left_over %>%
  mutate(first_name = ifelse(first_name == "Alejandro",
                             "Alejandro 'Kaku'",first_name),
         first_name = ifelse(first_name == "Bismark", "Nana", first_name),
         last_name = ifelse(last_name == "Ndam","Fouapon", last_name),
         first_name = ifelse(first_name == "Mohamed", "Mohammed", first_name),
         last_name = ifelse(last_name == "Ilsinho", "Dias", last_name),
         first_name = ifelse(last_name == "Dias", "Ilson", first_name),
         last_name = ifelse(last_name == "Ibson", "da Silva", last_name),
         first_name = ifelse(last_name == "da Silva", "Ibson", first_name),
         last_name = ifelse(last_name == "Fabinho", "Alves Macedo", last_name),
         first_name = ifelse(last_name == "Alves Macedo", "Fabinho", first_name),
         last_name = ifelse(last_name == "PC", "Giro", last_name),
         first_name = ifelse(last_name == "Giro", "Victor", first_name),
         last_name = ifelse(last_name == "Vako", "Qazaishvili", last_name),
         first_name = ifelse(last_name == "Qazaishvili","Valeri 'Vako'", first_name),
         first_name = ifelse(last_name == "Artur", "Artur", first_name),
         last_name = ifelse(last_name == "Artur", "de Lima", last_name),
         first_name = ifelse(last_name == "Leonardo", "Leonardo", first_name),
         last_name = ifelse(last_name == "Leonardo", "Da Silva", last_name),
         first_name = ifelse(last_name == "Maximiano", "Luiz", first_name),
         last_name = ifelse(first_name == "Luiz", "Fernando", last_name))

# get the salary data for the players above
final_players <- rbind(merge(final_left_over[0:10], salaries %>% select(-last_name),
                       by.x = c("first_name", "team_id"),
                       by.y = c("first_name", "team_id")),
                       merge(final_left_over[0:10], salaries %>% select(-first_name),
                             by = c("last_name", "team_id"))) %>% unique()

# merge them into the overall dataset
full_rosters_w_salary <- rbind(full_rosters_w_salary, final_players) %>%
  filter(is.na(base_salary) == FALSE)

# isolate boniek garcia and add him back in. Reason he was messing things up was
# due to there being two Garcia's on Houston
boniek <- left_over %>%
  filter(!player %in% merged_players) %>%
  filter(first_name == "Boniek") %>% mutate(first_name = "Oscar Boniek")

boniek_salary <- merge(boniek[0:10], salaries %>% select(-last_name),
                       by = c("first_name", "team_id"))

# FINAL DATAFRAME!
# Missing two players as their salaries aren't included in the release
full_rosters_w_salary <- rbind(full_rosters_w_salary, boniek_salary)

# store the roster status as a boolean

reserve_cols = c("ReserveRES",
                 "ReserveRES, Disabled ListDL",
                 "ReserveRES, On loanOL",
                 "ReserveRES, Season-Ending InjurySEI")

senior_cols = c("SeniorSR",
                "SeniorSR, Disabled ListDL",
                "SeniorSR, On loanOL",
                "SeniorSR, Season-Ending InjurySEI")

supplemental_cols = c("SupplementalSUP",
                      "SupplementalSUP, Disabled ListDL",
                      "SupplementalSUP, On loanOL")

disabled_cols = c("ReserveRES, Disabled ListDL",
                  "SeniorSR, Disabled ListDL",
                  "SupplementalSUP, Disabled ListDL")

on_loan_cols = c("ReserveRES, On loanOL",
                 "SeniorSR, On loanOL",
                 "SupplementalSUP, On loanOL")

season_end_cols = c("SeniorSR, Season-Ending InjurySEI")

full_rosters_w_salary <- full_rosters_w_salary %>%
  mutate(reserve = ifelse(`roster status` %in% reserve_cols, 1, 0),
         senior = ifelse(`roster status` %in% senior_cols, 1, 0),
         supplemental = ifelse(`roster status` %in% supplemental_cols, 1, 0),
         disabled_list = ifelse(`roster status` %in% disabled_cols, 1, 0),
         on_loan = ifelse(`roster status` %in% on_loan_cols, 1, 0),
         season_end_injury = ifelse(`roster status` %in% season_end_cols, 1, 0))

# store the player category as a boolean

dp_cols = c("DP", "DP, INTL")
intl_cols = c("DP, INTL", "INTL", "INTL, GA",
              "INTL, HG", "Young DPYDP, INTL")
ga_cols = c("GA", "GA-C", "INTL, GA")
hg_cols = c("HG", "INTL, HG")
ydp = c("Young DPYDP, INTL")

full_rosters_w_salary <- full_rosters_w_salary %>%
  mutate(DP = ifelse(category %in% dp_cols, 1, 0),
         INTL = ifelse(category %in% intl_cols, 1, 0),
         GA = ifelse(category %in% ga_cols, 1, 0),
         HG = ifelse(category %in% hg_cols, 1, 0),
         YDP = ifelse(category %in% ydp, 1, 0))

# change the salary columns to numeric
final_roster_info <- full_rosters_w_salary %>%
  select(-category, -notes, -`roster status`, -Position) %>%
  mutate(base_salary = as.numeric(gsub("[\\$,]", "", base_salary)),
         total_compensation = as.numeric(gsub("[\\$,]", "", total_compensation))) %>%
  rename(club = Club)

# #SaveTheData
write.csv(final_roster_info, "mls-full-salary-info-05-18.csv", row.names=FALSE)
```
