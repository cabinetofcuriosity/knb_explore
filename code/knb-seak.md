Location
================

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
  ca_which_pis <- c(ca_which_pis, nearPisWSeason(m, seastarkat_ca_g, allLocationNDate))
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
  geom_point(data = allLocationNDate, aes(x = latitude, y = longitude, color = "PISCO")) + 
  theme_minimal() +
  labs(title = "Locations of discarded size and count data entries", color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("Discarded size and count", "PISCO"),
                      values = c("light green", "light blue")) +
  ylim(-125, -115)
```

``` r
ggplot() +
  geom_point(data = filter(seastarkat_ca_g, pis_ind == -1), aes(x = season_sequence, y = marine_common_year, color = "Discarded size and count"), alpha = 0.1) +
  geom_point(data = allLocationNDate, aes(x = season, y = year, color = "PISCO"), alpha = 0.1) + 
  theme_minimal() +
  labs(title = "Season and year of discarded size and count data entries", color = "Datasets\n") +
  scale_colour_manual("", 
                      breaks = c("Discarded size and count", "PISCO"),
                      values = c("light green", "light blue")) +
  ylim(1990, 2020)
```

Adding PISCO datasets with the same locations
---------------------------------------------

When I find a nearest PISCO location, I only considered one dataset with the closest location(latitude and longitude), restrained by the season and year. However, for each season, year and location, there should be multiple datasets. Thus I should modify the function of finding the nearest PISCO datasets and use matrix as my data structure.
Also, since I've found that some data entries don't have a corresponding PISCO dataset, I can discard them. So only 421 rows of size and count data are useful for now.

``` r
sk_ca_filtered <- filter(seastarkat_ca_g, pis_ind != -1)
nrow(sk_ca_filtered)
```

``` r
needed_pis <- matrix(list(), nrow=421, ncol=1) # each row for each size and count data row
  
for (i in 1:nrow(sk_ca_filtered)) {
  cur_pis <- sk_ca_filtered$pis_ind[i] # index of pisco datasets in allLocationNDateS
  if (cur_pis == -1) {
    spe_dt$mean_temp[i] <- 999999.0
    next
  }
    
  cur_pis_dt <- allLocationNDateS[cur_pis, ]
  these_pis <- filter(allLocationNDateS, 
                      latitude == cur_pis_dt$latitude, 
                      longitude == cur_pis_dt$longitude, 
                      season == cur_pis_dt$season, 
                      year == cur_pis_dt$year)
  needed_pis[[i, 1]] <- these_pis$ID
}
```

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

``` r
get_temp <- function(spe_dt) {
  pis_ind <- spe_dt$pis_ind
  spe_dt$mean_temp <- c()
  needed_pis <- c()
  
  for (i in 1:nrow(spe_dt)) {
    cur_pis <- pis_ind[i] # index of pisco datasets in allLocationNDateS
    if (cur_pis == -1) {
      spe_dt$mean_temp[i] <- 999999.0
      next
    }
    
    #if (cur_pis >= 926) {
    #  cur_pis <- cur_pis + 1
    #}
    
    cur_pis_dt <- allLocationNDateS[cur_pis, ]
    these_pis <- filter(allLocationNDateS, 
                        latitude == cur_pis_dt$latitude, 
                        longitude == cur_pis_dt$longitude, 
                        season == cur_pis_dt$season, 
                        year == cur_pis_dt$year)
    all_temp <- c()
    for (j in 1:nrow(these_pis)) {
      if (these_pis[j, ]$ID %in% stored) {
        
      } else
      pisco_df <- downl_pis(these_pis[j, ]$ID)
      
      ### if you've downloaded all the datasets:
      ### pisco_df <- read.csv(file = paste0("../data/d", cur_pis, ".csv"), stringsAsFactors = FALSE)
    
      # discard if temp == 9999.00, which is equivalent to NA for PISCO datasets
      all_temp <- c(all_temp, filter(pisco_df, temp_c != 9999.00)$temp_c)
    }
    
    spe_dt$mean_temp[i] <- mean(all_temp)
  }
  return(spe_dt)
}
```

``` r
get_temp(seastarkat_ca_g)
```

``` r
dt1_tem <- filter(dt1, date == '2002-11-03', temp_c != 9999.00)

ggplot(data = dt1_tem, aes(x=X, y=temp_c)) +
  geom_bar(stat="identity", fill = "lightblue") +
  theme_minimal()
```