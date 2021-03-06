Count and size vs PISCO
================

``` r
# Load some packages
library(ggplot2)
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

Size and count data is seperated by season so I want to do the same thing to PISCO data, putting a season for each row with the function `dataToNum` defined before.

``` r
# first add a column to PISCO data
allLocationNDateS <- mutate(allLocationNDate, season = c(0), year = c(0))
for (i in 1:nrow(allLocationNDateS)) {
  curDate <- (dateToNum(allLocationNDateS$end[i]) + dateToNum(allLocationNDateS$begin[i])) / 2
  curSeason <- curDate %% 1
  curYear <- curDate - curSeason
  # curDate <- dateToNum('0000-09-06')
  rt <- 0
  if (curSeason >= dateToNum('0000-12-01') & curSeason <= dateToNum('0000-02-29')) {
    rt <- 4 # Winter
  } else if (curSeason >= dateToNum('0000-03-01') & curSeason <= dateToNum('0000-05-31')) {
    rt <- 1 # Spring
  } else if (curSeason >= dateToNum('0000-06-01') & curSeason <= dateToNum('0000-08-31')) {
    rt <- 2 # Summer
  } else {
    rt <- 3 # Fall
  }
  allLocationNDateS$season[i] <- rt
  allLocationNDateS$year[i] <- curYear
}
```

``` r
write.csv(allLocationNDateS, '../data/PISCOwSeason.csv')
```

``` r
seastarkat <- read.csv("../data/seastarkat_size_count_totals_download.csv", stringsAsFactors = FALSE)
```

I first examine the species data inside California and later to all the data.

``` r
seastarkat_ca <- seastarkat[seastarkat$state_province == 'California', ]
seastarkat_ca_g <- seastarkat_ca %>% 
  group_by(season_sequence, marine_common_year, target_assemblage, latitude, longitude) %>%
  summarise(total_sum = sum(total))
```
  
For each size and count data entry, I find a nearest PISCO location(latitude and longitude), using the Euclidean distance, in the same season and year. If the minimum Euclidean distance is bigger than 1 or the seasons and the years do not match, I assume this size and count data entry doesn't have a corresponding PISCO dataset.

``` r
nearPisWSeason <- function(i, sea_star_dt, pisco_dt) {
  ss <- sea_star_dt[i, ]
  ss_loc <- c(ss$longitude, ss$latitude)
  ss_year <- ss$marine_common_year
  ss_season <- ss$season_sequence
  
  min_dis <- 100000
  which_pis <- -1
  for (j in 1:nrow(pisco_dt)) {
    cur_pisco <- pisco_dt[j, ]
    cur_pisco_year <- cur_pisco$year
    cur_pisco_season <- cur_pisco$season
    if (ss_year != cur_pisco_year | ss_season != cur_pisco_season) {
      next
    }
    
    cur_pisco_loc <- c(cur_pisco$longitude, cur_pisco$latitude)
    dis <- (ss_loc[1] - cur_pisco_loc[1])**2 + (ss_loc[2] - cur_pisco_loc[2])**2
    if (dis < min_dis) {
      min_dis <- dis
      which_pis <- j
    }
  }
  if (min_dis > 1) {
    return(-1)
  }
  
  return(which_pis)
}
```

Call the function defined above on size and count data to obtain the indicies of all corresponding PISCO datasets.

``` r
ca_which_pis <- c()
for (m in 1:nrow(seastarkat_ca_g)) {
  ca_which_pis <- c(ca_which_pis, nearPisWSeason(m, seastarkat_ca_g, allLocationNDateS))
}
```

Add a column of these indicies to size and count data frame and write the new data frame into a csv file.

``` r
seastarkat_ca_g$pis_ind <- ca_which_pis
write.csv(seastarkat_ca_g, '../data/scca_seastarka.csv')
```

We can see that more than half of size and count rows are discarded because of wrong season and year or wrong location and this can be shown with two plots below.

``` r
nrow(filter(seastarkat_ca_g, pis_ind == -1))
```

``` r
#, season_sequence, marine_common_year
ggplot() +
  geom_point(data = filter(seastarkat_ca_g, pis_ind == -1), aes(x = latitude, y = longitude, color = "Discarded size and count")) +
  geom_point(data = allLocationNDateS, aes(x = latitude, y = longitude, color = "PISCO")) + 
  theme_minimal() +
  labs(title = "Locations of discarded size and count data entries", color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("Discarded size and count", "PISCO"),
                      values = c("light green", "light blue")) +
  ylim(-125, -115)
```

    ## Warning: Removed 7 rows containing missing values (geom_point).

![](knb-sckat_files/figure-markdown_github/unnamed-chunk-11-1.png)

``` r
ggplot() +
  geom_point(data = filter(seastarkat_ca_g, pis_ind == -1), aes(x = season_sequence, y = marine_common_year, color = "Discarded size and count"), alpha = 0.1) +
  geom_point(data = allLocationNDateS, aes(x = season, y = year, color = "PISCO"), alpha = 0.1) + 
  theme_minimal() +
  labs(title = "Season and year of discarded size and count data entries", color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("Discarded size and count", "PISCO"),
                      values = c("light green", "light blue")) +
  ylim(1990, 2020)
```

    ## Warning: Removed 12 rows containing missing values (geom_point).

![](knb-sckat_files/figure-markdown_github/unnamed-chunk-12-1.png)

Adding PISCO datasets with the same locations
---------------------------------------------

When I find a nearest PISCO location, I only considered one dataset with the closest location(latitude and longitude), restrained by the season and year. However, for each season, year and location, there should be multiple datasets. Thus I should modify the function of finding the nearest PISCO datasets and use matrix as my data structure.
Also, since I've found that some data entries don't have a corresponding PISCO dataset, I can discard them. So only 421 rows of size and count data are useful for now.

``` r
sk_ca_filtered <- filter(seastarkat_ca_g, pis_ind != -1)
nrow(sk_ca_filtered)
```

    ## [1] 421

Define two matricies and put all related PISCO datasets for each row into the matricies, one for IDs, the other for the row indicies.

``` r
needed_pisID <- matrix(list(), nrow=421, ncol=1) # each row for each size and count data row
needed_pisIND <- matrix(list(), nrow=421, ncol=1)
for (i in 1:nrow(sk_ca_filtered)) {
  cur_pis <- sk_ca_filtered$pis_ind[i] # index of the corresponding pisco dataset of the ith species data entry
  cur_pis_dt <- allLocationNDateS[cur_pis, ]
  these_pis <- filter(allLocationNDateS, 
                      latitude == cur_pis_dt$latitude, 
                      longitude == cur_pis_dt$longitude, 
                      season == cur_pis_dt$season, 
                      year == cur_pis_dt$year)
  needed_pisID[[i, 1]] <- these_pis$ID
  needed_pisIND[[i, 1]] <- these_pis$X
}
```

What is the maximum number of corresponding PISCO datasets for each row in `needed_pis`?

``` r
max(sapply(needed_pisID, function(row) {length(row)}))
```

    ## [1] 18

Download PISCO datasets
-----------------------

``` r
library(dataone)
library(XML)
cn <- CNode("PROD")

downl_pis <- function(id) {
  #id <- dataset_w_pop[i, 3]

  # download the metadata file to find the data table
  metadata <- rawToChar(getObject(mn, id))
  doc = xmlRoot(xmlTreeParse(metadata, asText=TRUE, trim = TRUE, ignoreBlanks = TRUE))

  # now extract the node that has the data table's information
  node <- getNodeSet(doc, "//entityName")
  table_id <- xmlValue(node[[1]])
  
  if (grepl("\\.(TXT|txt)", table_id)) {
    table_id <- gsub("\\.(TXT|txt)", "", table_id)
  }
  
  # we can see that the ids have the pattern
  dataRaw <- getObject(mn, paste0("doi:10.6085/AA/", table_id))
  dataChar <- rawToChar(dataRaw)
  theData <- textConnection(dataChar)
  df <- read.csv(theData, stringsAsFactors=FALSE, header = TRUE, sep = " ", row.names=NULL)
  return(df)
}
```

Because downloading PISCO datasets is pretty slow, I want to reduce unneccessary downloading as much as possible.
Ealier I have downloaded plenty of PISCO datasets stored in a hard drive, from the 1st to the 1260th except for some of them, so the indicies of already dowloaded PISCO datasets are:

``` r
dlded_pis <- (1:1260)[!(1:1260) %in% c(7, 8, 9, 15, 23, 28, 64, 77, 93, 105, 106, 110, 113, 121, 132, 133, 135, 186, 201, 223, 257, 301, 340, 345, 359, 360, 414, 418, 430, 503, 519, 537, 552, 562, 573, 580, 617, 653, 655, 673, 676, 688, 692, 694, 697, 701, 710, 727, 778, 786, 798, 814, 817, 836, 851, 863, 870, 878, 879, 885, 900, 912, 915, 927, 953, 991, 1003, 1015, 1082)]
```

However the downloading process took to much time that I couldn't finish it for just one time. So checking if there exists a local file is better.
Here I define a function to obtain temperature information of one PISCO dataset such that I only download one PISCO dataset once by checking if a dataset is already downloaded.

``` r
# returns temperature info of cur_pis
down_pis_temp <- function(cur_pis) {
  ### if you've downloaded this dataset
  if (file.exists(paste0("../data/downloaded/d", cur_pis, ".csv"))) { 
    pisco_df <- read.csv(file = paste0("../data/downloaded/d", cur_pis, ".csv"), stringsAsFactors = FALSE)
  } else {
    cur_pisID <- needed_pisID[[i, 1]][j]
    pisco_df <- downl_pis(cur_pisID)
    write.csv(pisco_df, file = paste0("../data/downloaded/d", cur_pis, ".csv"))
  }
  # discard if temp == 9999.00, which is equivalent to NA for PISCO datasets
  return(filter(pisco_df, temp_c != 9999.00)$temp_c)
}
```
  
Initialize the mean\_temp column with a huge unrealistic value so that it works as `NA`.
For each row of size and count data frame, use `tryCatch` to obtain the temperature information from the related PISCO datasets and print error information.

``` r
#sk_ca_filtered$mean_temp <- c(9999.0)
sk_ca_filtered <- read.csv("../data/generated/seastarkat_ca_temp1.csv", stringsAsFactors = FALSE)
for (i in 280:nrow(needed_pisID)) {
  all_temp <- c()
  # indicies of needed pisco datasets in allLocationNDateS
  these_pis <- needed_pisIND[[i, 1]]
  for (j in 1:length(these_pis)) {
    tryCatch(all_temp <- c(all_temp, down_pis_temp(these_pis[j])), 
             error = function(e) {print(paste("row", i, "PISCO", j, "invalid"));
                                  NaN})
}
    
  sk_ca_filtered$mean_temp[i] <- mean(all_temp)
}
write.csv(sk_ca_filtered, file = paste0("../data/seastarkat_ca_temp1.csv"))
```
  
I want to make a plot of water temperature for each location in time order, so I first extract the unique locations and store them in a data frame. We can tell there are 27 unique locations, hence I will make 27 plots.

``` r
sk_ca_filtered <- read.csv(file = "../data/seastarkat_ca_temp1.csv", stringsAsFactors = FALSE)
uni_loc_sk <- sk_ca_filtered %>% 
  group_by(longitude, latitude) %>%
  count()
```
  
For each location, I extract its temperature data from the original data frame. Take the first location for example:

``` r
cur_uni <- uni_loc_sk[1, ]
cur_df <- sk_ca_filtered %>%
  filter(latitude == cur_uni$latitude, longitude == cur_uni$longitude) %>%
  arrange(marine_common_year) %>%
  select(4:ncol(sk_ca_filtered)) %>%
  mutate(ind = row_number())

ggplot() +
geom_line(data = cur_df, aes(x = ind, y = mean_temp, color = "mean temperature")) +
geom_line(data = cur_df, aes(x = ind, y = total_sum, color = "sum")) + 
theme_minimal() +
labs(title = paste("species sum and mean temperature of", cur_df$latitude[1], cur_df$longitude[1]), color = "Datasets\n") +
scale_colour_manual("", 
                    breaks = c("mean temperature", "sum"),
                    values = c("light green", "light blue")) 
```

![](knb-sckat_files/figure-markdown_githubunnamed-chunk-21-1.png) 

Notice that the sum is a lot larger than temperature so I divide temperature by 20 in order for a clearer trend.

``` r
ggplot() +
  geom_line(data = cur_df, aes(x = ind, y = mean_temp, color = "mean temperature")) +
  geom_line(data = cur_df, aes(x = ind, y = total_sum/20, color = "sum")) + 
  theme_minimal() +
  labs(title = paste("species sum and mean temperature of", cur_df$latitude[1], cur_df$longitude[1]), color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("mean temperature", "sum"),
                      values = c("light green", "light blue")) +
  annotate("text", x = cur_df$ind, y = rep(1:2, len = length(cur_df$ind)), label = cur_df$marine_common_year)
```

![](knb-sckat_files/figure-markdown_githubunnamed-chunk-22-1.png) 

For a specific season in one year, I plot a map with species count information, using `ggmap`.

Download `ggmap`:

``` r
if(!requireNamespace("devtools")) install.packages("devtools")
devtools::install_github("dkahle/ggmap", ref = "tidyup", force=TRUE)
```

Setup for `ggmap`:

``` r
# Load ggmap
library("ggmap")
```

    ## Google's Terms of Service: https://cloud.google.com/maps-platform/terms/.

    ## Please cite ggmap if you use it! See citation("ggmap") for details.

``` r
# Set your API Key
ggmap::register_google(key = "AIzaSyAL0x-tMwrTQVrQZVhfbfwOFS9-ajoyLnA")
```

``` r
myLocation <- c(lon = cur_df$longitude[1], lat = cur_df$latitude[1])
myMap <- get_map(location = myLocation, source="stamen", maptype="watercolor", crop=FALSE)
```

    ## Source : https://maps.googleapis.com/maps/api/staticmap?center=37.04425,-122.23493&zoom=10&size=640x640&scale=2&maptype=terrain&key=xxx-tMwrTQVrQZVhfbfwOFS9-ajoyLnA

    ## Source : http://tile.stamen.com/watercolor/10/163/397.jpg

    ## Source : http://tile.stamen.com/watercolor/10/164/397.jpg

    ## Source : http://tile.stamen.com/watercolor/10/165/397.jpg

    ## Source : http://tile.stamen.com/watercolor/10/163/398.jpg

    ## Source : http://tile.stamen.com/watercolor/10/164/398.jpg

    ## Source : http://tile.stamen.com/watercolor/10/165/398.jpg

    ## Source : http://tile.stamen.com/watercolor/10/163/399.jpg

    ## Source : http://tile.stamen.com/watercolor/10/164/399.jpg

    ## Source : http://tile.stamen.com/watercolor/10/165/399.jpg

``` r
ggmap(myMap) +
 geom_point(aes(x = latitude, y = longitude, size = total_sum),
 data = cur_df, alpha = 1, color="darkred")
```

    ## Warning: Removed 21 rows containing missing values (geom_point).


![](knb-sckat_files/figure-markdown_githubunnamed-chunk-25-1.png) 

The locations are out of map range so I should enlarge the map range.


``` r
myMap <- get_map(location = myLocation, source = "stamen", maptype = "watercolor", zoom = 6)
```

    ## Source : https://maps.googleapis.com/maps/api/staticmap?center=37.04425,-122.23493&zoom=6&size=640x640&scale=2&maptype=terrain&key=xxx-tMwrTQVrQZVhfbfwOFS9-ajoyLnA

    ## Source : http://tile.stamen.com/watercolor/6/9/23.jpg

    ## Source : http://tile.stamen.com/watercolor/6/10/23.jpg

    ## Source : http://tile.stamen.com/watercolor/6/11/23.jpg

    ## Source : http://tile.stamen.com/watercolor/6/9/24.jpg

    ## Source : http://tile.stamen.com/watercolor/6/10/24.jpg

    ## Source : http://tile.stamen.com/watercolor/6/11/24.jpg

    ## Source : http://tile.stamen.com/watercolor/6/9/25.jpg

    ## Source : http://tile.stamen.com/watercolor/6/10/25.jpg

    ## Source : http://tile.stamen.com/watercolor/6/11/25.jpg

    ## Source : http://tile.stamen.com/watercolor/6/9/26.jpg

    ## Source : http://tile.stamen.com/watercolor/6/10/26.jpg

    ## Source : http://tile.stamen.com/watercolor/6/11/26.jpg

``` r
ggmap(myMap) +
  scale_x_continuous(limits = c(min(uni_loc_sk$longitude), max(uni_loc_sk$longitude)), expand = c(0, 0)) +
  scale_y_continuous(limits = c(min(uni_loc_sk$latitude), max(uni_loc_sk$latitude)), expand = c(0, 0)) +
  geom_point(aes(x = longitude, y = latitude, size = total_sum), data = filter(sk_ca_filtered, marine_common_year == 2001, season_sequence == 1), color="darkred")
```

    ## Scale for 'x' is already present. Adding another scale for 'x', which
    ## will replace the existing scale.

    ## Scale for 'y' is already present. Adding another scale for 'y', which
    ## will replace the existing scale.

    ## Warning: Removed 1 rows containing missing values (geom_rect).

![](knb-sckat_files/figure-markdown_github/unnamed-chunk-26-1.png)

``` r
par(mfrow=c(27,1)) 
for (i in 1:nrow(uni_loc_sk)) {
#for (i in 1:2) {
  cur_uni <- uni_loc_sk[i, ]
  cur_df <- filter(sk_ca_filtered, latitude == cur_uni$latitude, longitude == cur_uni$longitude) %>%
    arrange(marine_common_year) %>%
    select(4:ncol(sk_ca_filtered)) %>%
    mutate(ind = row_number())
  cur_plot <- ggplot() +
  geom_line(data = cur_df, aes(x = ind, y = mean_temp, color = "mean temperature")) +
  geom_line(data = cur_df, aes(x = ind, y = total_sum/20, color = "sum")) + 
  theme_minimal() +
  labs(title = paste("species sum and mean temperature of", cur_df$latitude[1], cur_df$longitude[1]), color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("mean temperature", "sum"),
                      values = c("light green", "light blue")) 
  print(cur_plot)
  
  #ylim(-125, -115)
}
```

![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-1.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-2.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-3.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-4.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-5.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-6.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-7.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-8.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-9.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-10.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-11.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-12.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-13.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-14.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-15.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-16.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-17.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-18.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-19.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-20.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-21.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-22.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-23.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-24.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-25.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-26.png)![](knb-sckat_files/figure-markdown_github/unnamed-chunk-27-27.png)
