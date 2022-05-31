---
title: "Case Study - Cyclistic"
author: "Alex Hollomby"
date: 2022-05-31T17:58:02+01:00
draft: true
---

### Notes on the project

[Here](https://github.com/alexdata1/Case-Study-Cyclistic) is a link to my GitHub repository for this project, which includes this markdown document along with the code used in the project.

The [Tableau Public page](https://public.tableau.com/app/profile/alex.hollomby/viz/Cyclistic_16534254417140/CyclisticInsights) for this project contains all visualisations I created as part of this project (these are also shown in Step 6 of the markdown).


### Introduction

As the culmination of my study of Google's Data Analytics Professional Certificate, I carried out a case study as a capstone project to showcase the technical abilities that I learned on the course, as well as my understanding of the data analysis process.

For my case study, I chose the scenario for Cyclistic, a fictitious bike share company located in Chicago.

This scenario had me assume the role of a junior data analyst in a marketing team at Cyclistic, tasked with creating a data analysis project to provide insights for a new marketing campaign.

The main question behind this project - "How do annual members and casual riders use Cyclistic bikes differently?"

### Phase 1 - Ask

The scenario's first step was understanding the questions asked by the project's stakeholders, and the problems that needed to be solved. I identified the business task and set out the aims of the project as follows:

*The goal of this project is to discover the main differences in usage of Cyclistic bikes between annual members and casual riders (i.e. customers who have purchased a single-ride or day pass). This is the first part of a larger objective to develop marketing strategies aimed at converting casual riders into annual members.*

*This project will identify differences in trip distance, duration, and frequency by using the past 12 months of historical data. After identifying these differences, this project aims to provide recommendations based on the analysis on how casual riders can be converted to annual members.*

### Phase 2 - Prepare

#### Choosing the tools for the project 

With a plan of action in place, I began preparing the data for analysis.

The data was made available [here](https://divvy-tripdata.s3.amazonaws.com/index.html) from Motivate International Inc. under this [license](https://ride.divvybikes.com/data-license-agreement). 

As mentioned previously, i decided that using the last twelve months' data would be best. This would ensure the data was recent, and provide a year-long scope for the project so that trends across months and seasons could be analysed.

Having downloaded the .zip files and unpacked the .csv files from each of them, I realised that I was working with a large amount of data (almost 1 gigabyte in total across just 12 .csv files!). As a result, this project could not easily be done via Excel or Google Sheets, so I decided that using the R programming language would be best. Along with the benefit of R being able to work with large data sets, I could find packages to work with the data the way I wanted, and write up the steps taken in markdown as I went along.

I installed VStudio code on the recommendation of a friend, and began setting things up to start my analysis.

#### Setting up my environment

To use R, I started by installing and loading all the packages I knew would be necessary for this project.

```{r loading packages, eval=FALSE}

install.packages("here")
install.packages("tidyverse")

library(tidyverse)
library(dplyr)
library(lubridate)
library(here)
```

The 'Tidyverse' package contains several libraries I used in this project:

-   *readr* - for importing the data
-   *lubridate* - for working with date and time
-   *dplyr* - for manipulating the data


#### Importing the raw data

Then came importing the raw data, which consists of twelve .csv files containing bike trip data from May 2021 to April 2022. The files were imported to the project folder and named using the *here* and *readr* packages.

```{r importing data, eval=FALSE}
data_2021_05 <- read_csv(here("raw_data/202105-divvy-tripdata.csv"))
...
data_2022_04 <- read_csv(here("raw_data/202204-divvy-tripdata.csv"))
```

#### Merging the data

As the raw data I needed was separated into twelve datasets, it was necessary to merge them all into a single data frame before any meaningful analysis could be performed. This meant the next step was to make sure that each the twelve datasets would match before merging them into one data frame. Performing the colnames() and str() functions gave me a quick look at each dataset.

```{r checking data, eval=FALSE}
str(data_2021_05)
colnames(data_2021_05)
...
str(data_2022_04)
colnames(data_2022_04)
```

Each was shown to have 13 columns that were identically named and in the same order. The format of the data within them also seemed to match. Satisfied with this, I proceeded to merge them into a new data frame.

```{r merging data, eval=FALSE}
cyclistic_rides <- rbind(data_2021_05, ... , data_2022_04)
```

Taking a look (again via the str() function )at the shiny new data frame revealed that the process was complete - thirteen columns with the same formatting as before, now with all 5,757,551 rows. I saved this as a new .csv file just in case.

```{r saving data frame, eval=FALSE}
write.csv(cyclistic_rides, "cyclistic_rides.csv", row.names = FALSE)
```

### Phase 3 - Process 

With the data now prepared for analysis, it was time to start transforming it into something I could work with. 

The first and most obvious step was to add some new columns that made sense of some of the existing ones. By taking the time difference between the started_at and ended_at columns, I could work out the duration of each trip. I thought it would be useful to have the day and month of each trip as new columns. The mutate() function from *dplyr* was used here to add the new columns, and *lubridate* was used to convert the existing columns from character format to time format.

```{r adding new columns for time duration, eval=FALSE}
cyclistic_rides <- cyclistic_rides %>% mutate(across(c(started_at, ended_at), ymd_hms), ride_duration_mins = round(difftime(ended_at, started_at, units="mins"))) %>% 
 mutate(hour_of_ride = hour(started_at)) %>% 
 mutate(time_of_day = case_when(am(started_at) ~ "a.m.", pm(started_at) ~ "p.m.", TRUE ~ "something's wrong..")) %>% 
 mutate(day_of_ride = wday(started_at, label=TRUE, abbr = FALSE)) %>% 
 mutate(month_of_ride = month(started_at, label=TRUE, abbr = FALSE))
```

Similarly, the distance of each trip could be worked out using the longitude/latitude columns. I knew that the *lubridate* package could assist with the former task, however for the latter I was unsure. Some searching online showed me the *Geodist* package could help me out. Having installed and loaded the package, I used it to add a column that would show the distance of each trip in km.

```{r adding new columns for distance, eval=FALSE}
install.packages("geodist")
library(geodist)
cyclistic_rides <- cyclistic_rides %>% mutate(ride_distance_km = round(
    geodist::geodist_vec(x1 = start_lng, y1 = start_lat, x2 = end_lng, y2 = end_lat, paired = TRUE, measure = "haversine")
    /1000, digits = 2))
```

#### Cleaning the data

Next it was time to clean the data by removing any rows containing incorrect or unnecessary values from the data frame, along with any obvious outliers. 

To that end, I decided that the rows should be removed as follows:
-   Any rows where start_lat, start_lng, end_lat or end_lng were missing values;
-   Any rows where started_at or ended_at were missing values;
-   Any rows where ride_distance_km was either less than 0.1 km or greater than 10 km;
-   Any rows where ride_duration_mins was either less than 2 minutes or greater than 240 minutes (i.e. 4 hours); and
-   Any rows where member_casual was neither 'casual' nor 'member'.

Although useful to filter out any aborted trips, I realised that this approach would also remove any circular trips (i.e. trips that started and ended at the same location). However I thought that this would still be the best approach due to the significant number of aborted trips, which could skew the findings of the data. This realisation showed another potential problem with the analysis - ride_distance_km only gives the distance between the start and end points of the trip as the crow flies - not the actual distance travelled. 

I also considered removing any rows missing a start_station_name or end_station_name value, but given the large amount of trips missing these, I thought it best to include them so as not to dilute the available data too much.

*dplyr* was used to filter the data frame. I saved a new copy afterwards.

```{r cleaning the data, eval=FALSE}
cyclistic_rides_cleaned <- cyclistic_rides %>% 
 filter(across(c(start_lat, start_lng, end_lat, end_lng, started_at, ended_at), ~ !is.na(.))) %>%
 filter(ride_duration_mins >= 2 & ride_duration_mins <= 240 & ride_distance_km <= 10 & ride_distance_km >= 0.1) %>%
 filter(member_casual %in% c("member", "casual"))

write.csv(cyclistic_rides_cleaned, "cyclistic_rides_cleaned.csv", row.names = FALSE)
```

### Phase 4 - Analyse & Share

With the data now cleaned, I took the next step - creating visualisations to share the findings of the data.

I opted for Tableau Public to visualise the project's data. Its ability to work with large datasets and easily access and share visualisations made it a natural choice, compared with R libraries that were less accessible and less comprehensive. 

This data story comprises my exploration of the data. Click through each header to follow along on my [Tableau Public page](https://public.tableau.com/app/profile/alex.hollomby/viz/Cyclistic_16534254417140/CyclisticInsights). Casual riders are colour-coded blue and annual members are colour-coded red for the sake of consistency and visibility throughout.

#### Insights from the data

* The first slide of the data story is a simple pie chart that shows the split between casual riders (42.71%) and annual members (57.29%) among the total number of trips taken. From this we can glean that members make significantly more trips than casuals.

* The second slide shows the differences in trip duration across each day of the week for casuals and members. It is apparent that casuals take much longer rides on average, with a notable increase in average trip duration on weekends across all riders. 

* Slide three shows the number of trips taken throughout the day. Overall, riders take more than twice as many trips after midday than before. Members seem to take much more trips than casuals between 5 a.m. and 10 a.m. Both types of rider take the most amount of trips between 3 p.m. and 7 p.m. This could indicate a surge for trips taken to get to and from work, with casuals more inclined to take trips home from work in the evening than trips to get there in the morning. 

* Slide four shows the number of trips per day of the week. Interestingly, member rides are highest mid-week, peaking on Wednesday, with similarly high amount of trips on Tuesday and Thursday. Conversely, casual riders take the most trips on the weekend, with weekday trips noticably dwindling in the midweek.

* Slide five shows the number of trips across each month of the year. This is largely consistent across casuals and members, with rides peaking in the summer months and fewer rides taken in winter. The peak for members is steadier across June to October, however, with a slightly sharper peak for casuals across June to September.

* The sixth and final slide shows the distance riders travelled across each day of the week. Once again, this is fairly consistent for all riders, with longer trips being taken on weekends compared to weekdays.

### Phase 5 - Act

My conclusions based on my analysis of the data are as follows:

* Annual members take more trips, take shorter trips, and are more consistent with their trips than casual riders.

* Annual members are much more interested in weekday trips than casual riders, and casual riders are much more interested in weekend trips than annual members.

* Summer months are the peak months for all riders, and winter months are noticably less popular.

My recommendations based on these findings are as follows:

1 - A strong marketing campaign could advertise something like 'weekend adventures' (possibly with family, friends, partners, etc.) as part of the benefits of an annual membership. This could be especially effective leading up to summer months when rides are at their highest.

2 - Focusing advertising on evening trips, especially the ability to get home from work quickly and easily, could bring in more annual members.

3 - Marketing could also centre on casual members' inclination to take shorter trips, advertising an annual membership as a means to have a quick means of transport on-demand at any time of day.
