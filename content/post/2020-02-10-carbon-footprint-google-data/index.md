---
title: "Assessing Your Carbon Footprint with Google Location Data"
author: "Anthony Schmidt"
date: '2020-02-10'
tags:
  - Data Science
  - Climate Change
subtitle: ''
summary: ''
authors: []
lastmod: ""
featured:
image:
  caption: ''
focal_point: ''
preview_only: no
projects: []
---
  
Google collects *a lot* of data on us. If you have Google Maps, chances are your location is being tracked, too. Unless, of course, you have it disabled. But, if you don't, you'd be surprised by the amount of location data (and its accuracy) contained in your Google Timeline. [Many worry about Google's tracking](https://www.nytimes.com/interactive/2019/12/19/opinion/location-tracking-cell-phone.html), but for those of us who don't, there is some potential fun we can have with our own data.

For example, we can **estimate** our carbon emissions. I say **estimate** because it is not exact and it is likely very difficult to account for (or even remember) all of your trips, who was with you, and how to split your emissions. Was your flight full? How "clean" was the train? How much of an Uber trip's emissions count for you vs the driver?

Those are difficult questions, and the real answer is elusive. But, we can come close. Using Google Location data, We can get the date and time of travel, start and end location, the distance traveled, the place, and most importantly, the activity type.

Follow along and I'll show you what I did. It's no doubt imperfect, but for a beginner's effort, I'd say it's not too bad.

# Get Your Data

Navigate to [Google Takeout](https://takeout.google.com/?pli=1), and first click "Deselect all" at the right. Then, scroll down to "Location History" and click on "Multiple Formats". Choose JSON and click "OK". Check the checkbox and scroll down to the bottom. Select "Next step". Here you can export your data once or set a schedule. Click "create export". Dependning on the amount of data, the export can take a few minutes (or "days" as Google claims). I got a year's worth of data in about 3 minutes. Just refresh the page to check on the status. 

When its ready, download your zip file. Extract your zip file somewhere convenient. Drill into the extracted folder until you find "Semantic Location Data". That is what we are after. This will contain a folder for each year and a JSON file for each month in that year.

# Load and Process

You'll need at minimum the following packages

* `jsonlite`
* `tidyverse`
* `lubridate`
* `ggplot2`

## Load Data

You can load your data like so:

```
df <- data.frame(fromJSON("file location/file", flatten=TRUE))
```

JSON comes as a list format. I have no clue how to work with lists. So, I flattened it out to a data frame. I know how to work with those.

I was going to write a for-loop to process the file list and automatically load the data, but since I was only working with 12 files (2019), I just did it like this:

```
jan2019 <- data.frame(fromJSON("2019/2019_JANUARY.json", flatten = TRUE))
feb2019 <- data.frame(fromJSON("2019/2019_FEBRUARY.json", flatten = TRUE))
mar2019 <- data.frame(fromJSON("2019/2019_MARCH.json", flatten = TRUE))
april2019 <- data.frame(fromJSON("2019/2019_APRIL.json", flatten = TRUE))
may2019 <- data.frame(fromJSON("2019/2019_MAY.json", flatten = TRUE))
```
...and so on.

## Process Data

I thought I would simply `rbind` the data and work with a single large data set, but many files had different numbers of columns, making it difficult to use `rbind`. Thankfully, the data sets had the same variable names, so I wrote a function to process the data, selecting the variables I wanted and making a few changes (see [below](#function)).

## Become Familiar with the Variables

Before processing the data, you should become familiar with the column names and what you may want to focus on. There are *a lot* of variables in the JSON file. There are variables for start and end location, path variables, distance, place name, accuracy estimates, etc. Here are the ones I was most interested in:

* timelineObjects.activitySegment.distance
  + distance travelled in feet
* timelineObjects.activitySegment.activityType
  + flying, walking, in a car, on a train, etc.
* timelineObjects.activitySegment.startLocation.latitudeE7 and timelineObjects.activitySegment.startLocation.longitudeE7
  + starting location
* timelineObjects.activitySegment.endLocation.latitudeE7 and timelineObjects.activitySegment.endLocation.longitudeE7
  + ending location
* timelineObjects.placeVisit.location.name
  + the place I went to
* timelineObjects.placeVisit.duration.startTimestampMs
  + the start time

Thanks to [Shirin's playgRound](https://shiring.github.io/maps/2016/12/30/Standortverlauf_post), I had some clues about what to process in R, namely:

* How to convert long/late from IE7 to GPS (divide by `1e7`)
* How to convert time from POSIX milliseconds to human readable time `as.POSIXct(as.numeric(time_variable)/1000, origin = "1970-01-01")`

One problem I noticed was that many rows were offset, meaning data was split between two or three rows. For example, time might have been in row 1 and location, and activity were in the next row. Sometimes, it was three rows. I used `lead()` to grab data from the next row, and `zoo::na.locf()` to grab data from the last non-empty row when `lead()` wouldn't do the trick. This is not a perfect solution and no doubt cause some minor innacurrcies, but for the most part, for such a large data set, it seemed to do the trick.

## Select and Mutate {#function}

The following function selects key variables and then mutates them, creating variables with human-readable names. It also converts the distance to miles and kilometers. You could take it a step further and then select only those variables, dropping the original JSON columns, but I decided to leave them in out of laziness.

The function takes a data frame name as its only input.

```
##process data function
json_process <- function(dfin){
  dfin %>% select(timelineObjects.activitySegment.distance,
                  timelineObjects.activitySegment.activityType,
                  timelineObjects.activitySegment.startLocation.latitudeE7,
                  timelineObjects.activitySegment.startLocation.longitudeE7,
                  timelineObjects.activitySegment.endLocation.latitudeE7,
                  timelineObjects.activitySegment.endLocation.longitudeE7,
                  timelineObjects.placeVisit.location.name,
                  timelineObjects.placeVisit.duration.startTimestampMs) %>%
    mutate(
      start_lat = timelineObjects.activitySegment.startLocation.latitudeE7 / 1e7,
      star_long = timelineObjects.activitySegment.startLocation.longitudeE7 / 1e7,
      end_lat = timelineObjects.activitySegment.endLocation.latitudeE7 / 1e7,
      end_long = timelineObjects.activitySegment.endLocation.longitudeE7 / 1e7,
      time = lead(as.POSIXct(as.numeric(timelineObjects.placeVisit.duration.startTimestampMs)/1000, origin = "1970-01-01"), 3L),
      year = year(time),
      month = ifelse(is.na(time), month(zoo::na.locf(time)), month(time)),
      activity = ifelse(is.na(timelineObjects.activitySegment.activityType), zoo::na.locf(timelineObjects.activitySegment.activityType), timelineObjects.activitySegment.activityType),
      place = lead(timelineObjects.placeVisit.location.name, 3L),
      miles = timelineObjects.activitySegment.distance/1609,
      km = timelineObjects.activitySegment.distance/1000)
  }
```
I called the function like so:

```
jan <- json_process(jan2019)
feb <- json_process(feb2019)
mar <- json_process(mar2019)
```

And then combined the data frames:

```
data <- rbind(jan, feb, mar, april, may, june, july, aug, sep, oct, nov, dec)
```

# Analyze and Visualize

First, you should check what activities are listed in your data:

```
data %>% group_by(activity) %>% count
```
The activities included FLYING, IN_FERRY, IN_BUS, IN_SUBWAY, etc. However, I focused on only two carbon-emitting activities in my data - those that I could easily account carbon emissions for - and created a small data frame to use for filtering:

```
carbon_activities <- data.frame(activity = c("FLYING",
                              "IN_PASSENGER_VEHICLE"))
```

I then created a new data frame that dropped missing values, selected only flying and car activities, and then calculated emissions.

```
prep <- data %>% drop_na(activity) %>%
  filter(activity %in% carbon_activities$activity) %>%
  mutate(
    emissions = ifelse(activity == "FLYING", km*.18, km*.14)
  ) %>% drop_na(emissions)
```

How did I get the figures for emissions? Well, it turns out that calculating carbon emissions from flying is quite difficult. Here are some notes from the [ICAO website](https://www.icao.int/environmental-protection/Carbonoffset/Pages/default.aspx):

* Step 1: Estimation of the aircraft fuel burn
* Step 2: Calculation of the passengers' fuel burn based on a passenger/freight factor which is derived from RTK data
* Step 3: Calculation of seats occupied (assumption: all aircraft are entirely configured with economic seats). Seat occupied = Total seats x Load Factor
* Step 4: CO2 emissions per passenger = (Passengers' fuel burn x 3.16) / Seat occupied
* Note: for flights above 3000 km, CO2 emissions per passenger in premium cabin = 2 x CO2 emissions per passenger in economy

Their [methodology paper](https://www.icao.int/environmental-protection/CarbonOffset/Documents/Methodology%20ICAO%20Carbon%20Calculator_v10-2017.pdf) *probably* has enough of this information, including numerous tables, to make these calculations. While I intend to make a better attempt at this in the future, using some suggestions from [Sheila Saia](https://sheilasaia.rbind.io/post/2019-04-19-carbon-cost-calcs/) I just wanted a rough, liberal estimate of my emissions.

I found [this paper from the Environmental Change Institute](https://www.eci.ox.ac.uk/research/energy/downloads/jardine09-carboninflights.pdf), which explained several different methods of calculation. The simplest is based on the World Resource Institute's liberal estimate of 0.18 kgCO2/km. I chose this because it was simple to calculate.

For driving, I used [this online carbon footprint calculator](https://www.carbonfootprint.com/calculator.aspx). I selected "Car", entered in 1000km and my car's average MPG (38MPG - though I often get 40-45 on longer trips), and the result was "0.14 metric tons:	1000 km in a petrol vehicle doing 38 mpg (US)." That is, 140 kgCO2/1000 km, which is the same as .14 kgCO2/km.

Finally, I can visualize the result:

```
prep %>%
  group_by(month, activity) %>%
  summarize(sum = sum(emissions)) %>%
  ggplot()+
  geom_bar(aes(x=as.factor(month), y=sum, fill=activity), stat="identity")+
  #facet_wrap(~activity, scales="free")+
  scale_y_continuous(labels = function(x) format(x, scientific = FALSE))+
  labs(title = "Personal CO2 Emissions from Transportation",
       caption = "kgC02/km for flights calculated based on liberal WRI estimate: km * .18. \nkgCO2/km for car calculated based on .14kg per 1000km",
       x = "Month",
       y = "kgCO2")
```

![My Personal Emissions - 2019](https://anthonyschmidt.netlify.com/post/2020-02-10-carbon-footprint-google-data/personal-emissions.png)

# Conclusion

This is a just a simple example of what one can do with their location data. I will be using my data in an interactive Tableau dashboard that also includes utility usage and will allow me to drill down by month, day, and hour. It will also allow me to compare my 2020 emissions to 2019. **My goal is to have lower emissions.**

I think there are a lot of possibilities with Google Location data (for tracking carbon emissions, for making cool visualizations), and even with liberal estimates of kgCO2, you still get a sense of your impact and have a visual goal you can compare against for the future.