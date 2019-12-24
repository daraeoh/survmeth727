---
title: "Food For Thought"
author: "Rebecca Oh, Alexa Zmich"
date: "December 5, 2019"
output:
  pdf_document: 
    toc: yes
subtitle: SURVMETH 727 Term Project
geometry: margin=0.75in
---

```{r, include = FALSE}
library(RSocrata)
library(ggmap)
library(yelpr)
library(knitr)
library(markdown)
library(tidyverse)
library(httr)
library(RSocrata)
library(tidyverse)
library(stringr)
library(noncensus)
library(readr)
library(dplyr)
library(lubridate)
library(corrplot)
library(magrittr)
library(factoextra)
library(cluster)
library(censusapi)
library(ggplot2)
library(Hmisc)
library(gridExtra)
library(broom)
```

## Introduction

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Rising obesity rates in the United States have become a large concern, as obesity is now the leading preventable cause of death worldwide and is associated with many other causes of death. The dietary choices we make every day affect the potential of developing obesity, including the types of food establishments we choose to eat at. Thus, we are interested in seeing the possible effect these food establishments can have on our health, leading to our research question: “Do the main cuisines of food establishments affect obesity rates within Michigan?” In our study, we additionally seek to delve further into this question by analyzing geographic location (by county), which may also be impacting the obesity rates.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Prior research for obesity is heavily focused on fast food restaurants specifically. Studies have found that the location of fast food reference, in regards to both quantity and closeness to heavily frequented buildings (i.e. schools), can have a significant effect on the obesity rates in the area (Jeffery et al. 2006; Davis and Carpenter 2011). The significance found on location greatly influenced our decision to include a location focus along with our research question, to see if results would be different for non-fast-food restaurants. Currie et al. (2010) separated fast food restaurants from non-fast-food restaurants and found there to be a lack of correlation between non-fast-food restaurants and obesity, however, the study focused primarily on the fast food restaurants and did not further break down the non-fast-food-restaurants due to the initial lack of correlation. Our research question will continue research into a much needed focus on non-fast-food-restaurants, especially as Yelp tends to lack full inclusion of area fast food restaurants.

## Data

### _Cuisines and Counties Data_

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In order to answer our main research question, we first needed to gather data regarding main cuisines in Michigan counties. We chose Yelp as our data gathering source, as it stores key restaurant information beyond the restaurant name, such as cuisine type and location, which will be used to test for significant effects against obesity. After gaining access to Yelp API, we stored the key as an object to be used in the following API queries that ultimately formed our data set.

```{r}
#create a token for use with API request
client_id <- "tuUnFmtQQZAX7YHwwbdz6g"
client_secret <- "PQL5AXms_2Yr6szyGBy2V6yH1sdeFS1g5S3od1WZxFKAD3IK0DEHC0mt1B8atTzO5mspVGcqxvA9gbAb_3u3_u07umTpSr0_YTdcTH9B4sP1UEXrvn713CzqPhzTXXYx"

res <- POST("https://api.yelp.com/oauth2/token",
            body = list(grant_type = "client_credentials",
                        client_id = client_id,
                        client_secret = client_secret))

token <- content(res)$access_token
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We encountered a few unexpected challenges with using the Yelp API. Gathering data by searching a whole county within the US is not supported, insisting that users search for a specific city. Additionally, the API has a limit of 50 queries. Thus, we collected the top 20 most popular food establishments within (or near) the most populated city for each of the 83 counties in the state of Michigan. We searched “food” as the term of interest and the location that we searched was the most populated city for each county. Then, we appended all 83 datasets to have a full dataset listing the most popular food establishments, the cuisine type of each establishment, and the city each establishment is located in.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The data from Yelp provide us information regarding location (city, state, longitude, latitude), but lacks county. Since we were ultimately hoping to analyze the data at the county level, we needed to create a new object providing us with this information, by mapping from the establishment’s corresponding city. We will show an example of this using Wayne County:

```{r}
#search url and collect data per county
yelp <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=detroit+MI" #wayne
url <- modify_url(yelp, path = c("v3", "businesses", "search"))
res <- GET(url, add_headers('Authorization' = paste("bearer", client_secret)))
results <- content(res)

#parse data
yelp_httr_parse <- function(x) {

  parse_list <- list(id = x$id, 
                     name = x$name, 
                     categories = x$categories, 
                     rating = x$rating, 
                     review_count = x$review_count, 
                     latitude = x$coordinates$latitude, 
                     longitude = x$coordinates$longitude, 
                     address1 = x$location$address1, 
                     city = x$location$city, 
                     state = x$location$state, 
                     distance = x$distance)
  
  parse_list <- lapply(parse_list, FUN = function(x) ifelse(is.null(x), "", x))
  
  df <- tibble(id=parse_list$id,
                   name=parse_list$name, 
                   categories = parse_list$categories, 
                   rating = parse_list$rating, 
                   review_count = parse_list$review_count, 
                   latitude=parse_list$latitude, 
                   longitude = parse_list$longitude, 
                   address1 = parse_list$address1, 
                   city = parse_list$city, 
                   state = parse_list$state, 
                   distance= parse_list$distance)
  df
}

results_list <- lapply(results$businesses, FUN = yelp_httr_parse)
food <- do.call("rbind", results_list)
county <- rep("Wayne",length(food$city))
food_data <- cbind(food, county)
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We then repeated this process for the remaining 82 counties in Michigan.

```{r, include = FALSE}
#search url and collect data per county
yelp2 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=troy+MI" #oakland
yelp3 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=warren+MI" #macomb
yelp4 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=grand+rapids+MI" #kent
yelp5 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=flint+MI" #genesee
yelp6 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ann+arbor+MI" #washtenaw
yelp7 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=lansing+MI" #ingham
yelp8 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=allendale+MI" #ottawa
yelp9 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=kalamazoo+MI" #kalamazoo
yelp10 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=howell+MI" #livingston
yelp11 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=saginaw+MI" #saginaw
yelp12 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=muskegon+MI" #muskegon
yelp13 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=port+huron+MI" #st. clair
yelp14 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=jackson+MI" #jackson
yelp15 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=niles+MI" #berrien
yelp16 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=monroe+MI" #monroe
yelp17 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=battle+creek+MI" #calhoun
yelp18 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=holland+MI" #allegan
yelp19 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=charlotte+MI" #eaton
yelp20 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=bay+city+MI" #bay
yelp21 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=adrian+MI" #lenawee
yelp22 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=traverse+city+MI" #grand traverse
yelp23 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=lapeer+MI" #lapeer
yelp24 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=midland+MI" #midland
yelp25 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=saint+johns+MI" #clinton
yelp26 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=south+haven+MI" #van buren
yelp27 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=mount+pleasant+MI" #isabella
yelp28 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=owosso+MI" #shiawassee
yelp29 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=marquette+MI" #marquette
yelp30 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ionia+MI" #ionia
yelp31 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=greenville+MI" #montcalm
yelp32 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=hastings+MI" #barry
yelp33 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=sturgis+MI" #st. joseph
yelp34 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=caro+MI" #tuscola
yelp35 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=dowagiac+MI" #cass
yelp36 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=fremont+MI" #newaygo
yelp37 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=hillsdale+MI" #hillsdale
yelp38 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=coldwater+MI" #branch
yelp39 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=big+rapids+MI" #mecosta
yelp40 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=sandusky+MI" #sanilac
yelp41 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=alma+MI" #gratiot
yelp42 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=sault+ste+marie+MI" #chippewa
yelp43 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=hancock+MI" #houghton
yelp44 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=escanaba+MI" #delta
yelp45 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=cadillac+MI" #wexford
yelp46 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=petosky+MI" #emmet
yelp47 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=bad+axe+MI" #huron
yelp48 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=clare+MI" #clare
yelp49 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ludington+MI" #mason
yelp50 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=alpena+MI" #alpena
yelp51 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=hart+MI" #oceana
yelp52 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=boyne+city+MI" #charlevoix
yelp53 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=cheboygan+MI" #cheboygan
yelp54 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=iron+mountain+MI" #dickinson
yelp55 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=gladwin+MI" #gladwin
yelp56 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=east+tawas+MI" #iosco
yelp57 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=gaylord+MI" #otsego
yelp58 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=manistee+MI" #manistee
yelp59 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=roscommon+MI" #roscommon
yelp60 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=elk+rapids+MI" #antrim
yelp61 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=reed+city+MI" #osceola
yelp62 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=menominee+MI" #menominee
yelp63 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=greilickville+MI" #leelanau
yelp64 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=west+branch+MI" #ogemaw
yelp65 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=kalkaska+MI" #kalkaska
yelp66 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=frankfort+MI" #benzie
yelp67 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=lake+city+MI" #missaukee
yelp68 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ironwood+MI" #gogebic
yelp69 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=standish+MI" #arenac
yelp70 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=grayling+MI" #crawford
yelp71 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=rogers+city+MI" #presque isle
yelp72 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=baldwin+MI" #lake
yelp73 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=iron+river+MI" #iron
yelp74 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=st+ignace+MI" #mackinac
yelp75 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=harrisville+MI" #alcona
yelp76 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=hillman+MI" #montmorency
yelp77 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=munising+MI" #alger
yelp78 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=baraga+MI" #baraga
yelp79 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=big+creek+township+MI" #oscoda
yelp80 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=manistique+MI" #schoolcraft
yelp81 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=newberry+MI" #luce
yelp82 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ontonagon+MI" #ontonagon
yelp83 <- "https://api.yelp.com/v3/businesses/search/?limit=20&offset=0&sort=0&term=food&location=ahmeek+MI" #keweenaw

url2 <- modify_url(yelp2, path = c("v3", "businesses", "search"))
url3 <- modify_url(yelp3, path = c("v3", "businesses", "search"))
url4 <- modify_url(yelp4, path = c("v3", "businesses", "search"))
url5 <- modify_url(yelp5, path = c("v3", "businesses", "search"))
url6 <- modify_url(yelp6, path = c("v3", "businesses", "search"))
url7 <- modify_url(yelp7, path = c("v3", "businesses", "search"))
url8 <- modify_url(yelp8, path = c("v3", "businesses", "search"))
url9 <- modify_url(yelp9, path = c("v3", "businesses", "search"))
url10 <- modify_url(yelp10, path = c("v3", "businesses", "search"))
url11 <- modify_url(yelp11, path = c("v3", "businesses", "search"))
url12 <- modify_url(yelp12, path = c("v3", "businesses", "search"))
url13 <- modify_url(yelp13, path = c("v3", "businesses", "search"))
url14 <- modify_url(yelp14, path = c("v3", "businesses", "search"))
url15 <- modify_url(yelp15, path = c("v3", "businesses", "search"))
url16 <- modify_url(yelp16, path = c("v3", "businesses", "search"))
url17 <- modify_url(yelp17, path = c("v3", "businesses", "search"))
url18 <- modify_url(yelp18, path = c("v3", "businesses", "search"))
url19 <- modify_url(yelp19, path = c("v3", "businesses", "search"))
url20 <- modify_url(yelp20, path = c("v3", "businesses", "search"))
url21 <- modify_url(yelp21, path = c("v3", "businesses", "search"))
url22 <- modify_url(yelp22, path = c("v3", "businesses", "search"))
url23 <- modify_url(yelp23, path = c("v3", "businesses", "search"))
url24 <- modify_url(yelp24, path = c("v3", "businesses", "search"))
url25 <- modify_url(yelp25, path = c("v3", "businesses", "search"))
url26 <- modify_url(yelp26, path = c("v3", "businesses", "search"))
url27 <- modify_url(yelp27, path = c("v3", "businesses", "search"))
url28 <- modify_url(yelp28, path = c("v3", "businesses", "search"))
url29 <- modify_url(yelp29, path = c("v3", "businesses", "search"))
url30 <- modify_url(yelp30, path = c("v3", "businesses", "search"))
url31 <- modify_url(yelp31, path = c("v3", "businesses", "search"))
url32 <- modify_url(yelp32, path = c("v3", "businesses", "search"))
url33 <- modify_url(yelp33, path = c("v3", "businesses", "search"))
url34 <- modify_url(yelp34, path = c("v3", "businesses", "search"))
url35 <- modify_url(yelp35, path = c("v3", "businesses", "search"))
url36 <- modify_url(yelp36, path = c("v3", "businesses", "search"))
url37 <- modify_url(yelp37, path = c("v3", "businesses", "search"))
url38 <- modify_url(yelp38, path = c("v3", "businesses", "search"))
url39 <- modify_url(yelp39, path = c("v3", "businesses", "search"))
url40 <- modify_url(yelp40, path = c("v3", "businesses", "search"))
url41 <- modify_url(yelp41, path = c("v3", "businesses", "search"))
url42 <- modify_url(yelp42, path = c("v3", "businesses", "search"))
url43 <- modify_url(yelp43, path = c("v3", "businesses", "search"))
url44 <- modify_url(yelp44, path = c("v3", "businesses", "search"))
url45 <- modify_url(yelp45, path = c("v3", "businesses", "search"))
url46 <- modify_url(yelp46, path = c("v3", "businesses", "search"))
url47 <- modify_url(yelp47, path = c("v3", "businesses", "search"))
url48 <- modify_url(yelp48, path = c("v3", "businesses", "search"))
url49 <- modify_url(yelp49, path = c("v3", "businesses", "search"))
url50 <- modify_url(yelp50, path = c("v3", "businesses", "search"))
url51 <- modify_url(yelp51, path = c("v3", "businesses", "search"))
url52 <- modify_url(yelp52, path = c("v3", "businesses", "search"))
url53 <- modify_url(yelp53, path = c("v3", "businesses", "search"))
url54 <- modify_url(yelp54, path = c("v3", "businesses", "search"))
url55 <- modify_url(yelp55, path = c("v3", "businesses", "search"))
url56 <- modify_url(yelp56, path = c("v3", "businesses", "search"))
url57 <- modify_url(yelp57, path = c("v3", "businesses", "search"))
url58 <- modify_url(yelp58, path = c("v3", "businesses", "search"))
url59 <- modify_url(yelp59, path = c("v3", "businesses", "search"))
url60 <- modify_url(yelp60, path = c("v3", "businesses", "search"))
url61 <- modify_url(yelp61, path = c("v3", "businesses", "search"))
url62 <- modify_url(yelp62, path = c("v3", "businesses", "search"))
url63 <- modify_url(yelp63, path = c("v3", "businesses", "search"))
url64 <- modify_url(yelp64, path = c("v3", "businesses", "search"))
url65 <- modify_url(yelp65, path = c("v3", "businesses", "search"))
url66 <- modify_url(yelp66, path = c("v3", "businesses", "search"))
url67 <- modify_url(yelp67, path = c("v3", "businesses", "search"))
url68 <- modify_url(yelp68, path = c("v3", "businesses", "search"))
url69 <- modify_url(yelp69, path = c("v3", "businesses", "search"))
url70 <- modify_url(yelp70, path = c("v3", "businesses", "search"))
url71 <- modify_url(yelp71, path = c("v3", "businesses", "search"))
url72 <- modify_url(yelp72, path = c("v3", "businesses", "search"))
url73 <- modify_url(yelp73, path = c("v3", "businesses", "search"))
url74 <- modify_url(yelp74, path = c("v3", "businesses", "search"))
url75 <- modify_url(yelp75, path = c("v3", "businesses", "search"))
url76 <- modify_url(yelp76, path = c("v3", "businesses", "search"))
url77 <- modify_url(yelp77, path = c("v3", "businesses", "search"))
url78 <- modify_url(yelp78, path = c("v3", "businesses", "search"))
url79 <- modify_url(yelp79, path = c("v3", "businesses", "search"))
url80 <- modify_url(yelp80, path = c("v3", "businesses", "search"))
url81 <- modify_url(yelp81, path = c("v3", "businesses", "search"))
url82 <- modify_url(yelp82, path = c("v3", "businesses", "search"))
url83 <- modify_url(yelp83, path = c("v3", "businesses", "search"))

res2 <- GET(url2, add_headers('Authorization' = paste("bearer", client_secret)))
res3 <- GET(url3, add_headers('Authorization' = paste("bearer", client_secret)))
res4 <- GET(url4, add_headers('Authorization' = paste("bearer", client_secret)))
res5 <- GET(url5, add_headers('Authorization' = paste("bearer", client_secret)))
res6 <- GET(url6, add_headers('Authorization' = paste("bearer", client_secret)))
res7 <- GET(url7, add_headers('Authorization' = paste("bearer", client_secret)))
res8 <- GET(url8, add_headers('Authorization' = paste("bearer", client_secret)))
res9 <- GET(url9, add_headers('Authorization' = paste("bearer", client_secret)))
res10 <- GET(url10, add_headers('Authorization' = paste("bearer", client_secret)))
res11 <- GET(url11, add_headers('Authorization' = paste("bearer", client_secret)))
res12 <- GET(url12, add_headers('Authorization' = paste("bearer", client_secret)))
res13 <- GET(url13, add_headers('Authorization' = paste("bearer", client_secret)))
res14 <- GET(url14, add_headers('Authorization' = paste("bearer", client_secret)))
res15 <- GET(url15, add_headers('Authorization' = paste("bearer", client_secret)))
res16 <- GET(url16, add_headers('Authorization' = paste("bearer", client_secret)))
res17 <- GET(url17, add_headers('Authorization' = paste("bearer", client_secret)))
res18 <- GET(url18, add_headers('Authorization' = paste("bearer", client_secret)))
res19 <- GET(url19, add_headers('Authorization' = paste("bearer", client_secret)))
res20 <- GET(url20, add_headers('Authorization' = paste("bearer", client_secret)))
res21 <- GET(url21, add_headers('Authorization' = paste("bearer", client_secret)))
res22 <- GET(url22, add_headers('Authorization' = paste("bearer", client_secret)))
res23 <- GET(url23, add_headers('Authorization' = paste("bearer", client_secret)))
res24 <- GET(url24, add_headers('Authorization' = paste("bearer", client_secret)))
res25 <- GET(url25, add_headers('Authorization' = paste("bearer", client_secret)))
res26 <- GET(url26, add_headers('Authorization' = paste("bearer", client_secret)))
res27 <- GET(url27, add_headers('Authorization' = paste("bearer", client_secret)))
res28 <- GET(url28, add_headers('Authorization' = paste("bearer", client_secret)))
res29 <- GET(url29, add_headers('Authorization' = paste("bearer", client_secret)))
res30 <- GET(url30, add_headers('Authorization' = paste("bearer", client_secret)))
res31 <- GET(url31, add_headers('Authorization' = paste("bearer", client_secret)))
res32 <- GET(url32, add_headers('Authorization' = paste("bearer", client_secret)))
res33 <- GET(url33, add_headers('Authorization' = paste("bearer", client_secret)))
res34 <- GET(url34, add_headers('Authorization' = paste("bearer", client_secret)))
res35 <- GET(url35, add_headers('Authorization' = paste("bearer", client_secret)))
res36 <- GET(url36, add_headers('Authorization' = paste("bearer", client_secret)))
res37 <- GET(url37, add_headers('Authorization' = paste("bearer", client_secret)))
res38 <- GET(url38, add_headers('Authorization' = paste("bearer", client_secret)))
res39 <- GET(url39, add_headers('Authorization' = paste("bearer", client_secret)))
res40 <- GET(url40, add_headers('Authorization' = paste("bearer", client_secret)))
res41 <- GET(url41, add_headers('Authorization' = paste("bearer", client_secret)))
res42 <- GET(url42, add_headers('Authorization' = paste("bearer", client_secret)))
res43 <- GET(url43, add_headers('Authorization' = paste("bearer", client_secret)))
res44 <- GET(url44, add_headers('Authorization' = paste("bearer", client_secret)))
res45 <- GET(url45, add_headers('Authorization' = paste("bearer", client_secret)))
res46 <- GET(url46, add_headers('Authorization' = paste("bearer", client_secret)))
res47 <- GET(url47, add_headers('Authorization' = paste("bearer", client_secret)))
res48 <- GET(url48, add_headers('Authorization' = paste("bearer", client_secret)))
res49 <- GET(url49, add_headers('Authorization' = paste("bearer", client_secret)))
res50 <- GET(url50, add_headers('Authorization' = paste("bearer", client_secret)))
res51 <- GET(url51, add_headers('Authorization' = paste("bearer", client_secret)))
res52 <- GET(url52, add_headers('Authorization' = paste("bearer", client_secret)))
res53 <- GET(url53, add_headers('Authorization' = paste("bearer", client_secret)))
res54 <- GET(url54, add_headers('Authorization' = paste("bearer", client_secret)))
res55 <- GET(url55, add_headers('Authorization' = paste("bearer", client_secret)))
res56 <- GET(url56, add_headers('Authorization' = paste("bearer", client_secret)))
res57 <- GET(url57, add_headers('Authorization' = paste("bearer", client_secret)))
res58 <- GET(url58, add_headers('Authorization' = paste("bearer", client_secret)))
res59 <- GET(url59, add_headers('Authorization' = paste("bearer", client_secret)))
res60 <- GET(url60, add_headers('Authorization' = paste("bearer", client_secret)))
res61 <- GET(url61, add_headers('Authorization' = paste("bearer", client_secret)))
res62 <- GET(url62, add_headers('Authorization' = paste("bearer", client_secret)))
res63 <- GET(url63, add_headers('Authorization' = paste("bearer", client_secret)))
res64 <- GET(url64, add_headers('Authorization' = paste("bearer", client_secret)))
res65 <- GET(url65, add_headers('Authorization' = paste("bearer", client_secret)))
res66 <- GET(url66, add_headers('Authorization' = paste("bearer", client_secret)))
res67 <- GET(url67, add_headers('Authorization' = paste("bearer", client_secret)))
res68 <- GET(url68, add_headers('Authorization' = paste("bearer", client_secret)))
res69 <- GET(url69, add_headers('Authorization' = paste("bearer", client_secret)))
res70 <- GET(url70, add_headers('Authorization' = paste("bearer", client_secret)))
res71 <- GET(url71, add_headers('Authorization' = paste("bearer", client_secret)))
res72 <- GET(url72, add_headers('Authorization' = paste("bearer", client_secret)))
res73 <- GET(url73, add_headers('Authorization' = paste("bearer", client_secret)))
res74 <- GET(url74, add_headers('Authorization' = paste("bearer", client_secret)))
res75 <- GET(url75, add_headers('Authorization' = paste("bearer", client_secret)))
res76 <- GET(url76, add_headers('Authorization' = paste("bearer", client_secret)))
res77 <- GET(url77, add_headers('Authorization' = paste("bearer", client_secret)))
res78 <- GET(url78, add_headers('Authorization' = paste("bearer", client_secret)))
res79 <- GET(url79, add_headers('Authorization' = paste("bearer", client_secret)))
res80 <- GET(url80, add_headers('Authorization' = paste("bearer", client_secret)))
res81 <- GET(url81, add_headers('Authorization' = paste("bearer", client_secret)))
res82 <- GET(url82, add_headers('Authorization' = paste("bearer", client_secret)))
res83 <- GET(url83, add_headers('Authorization' = paste("bearer", client_secret)))

results2 <- content(res2)
results3 <- content(res3)
results4 <- content(res4)
results5 <- content(res5)
results6 <- content(res6)
results7 <- content(res7)
results8 <- content(res8)
results9 <- content(res9)
results10 <- content(res10)
results11 <- content(res11)
results12 <- content(res12)
results13 <- content(res13)
results14 <- content(res14)
results15 <- content(res15)
results16 <- content(res16)
results17 <- content(res17)
results18 <- content(res18)
results19 <- content(res19)
results20 <- content(res20)
results21 <- content(res21)
results22 <- content(res22)
results23 <- content(res23)
results24 <- content(res24)
results25 <- content(res25)
results26 <- content(res26)
results27 <- content(res27)
results28 <- content(res28)
results29 <- content(res29)
results30 <- content(res30)
results31 <- content(res31)
results32 <- content(res32)
results33 <- content(res33)
results34 <- content(res34)
results35 <- content(res35)
results36 <- content(res36)
results37 <- content(res37)
results38 <- content(res38)
results39 <- content(res39)
results40 <- content(res40)
results41 <- content(res41)
results42 <- content(res42)
results43 <- content(res43)
results44 <- content(res44)
results45 <- content(res45)
results46 <- content(res46)
results47 <- content(res47)
results48 <- content(res48)
results49 <- content(res49)
results50 <- content(res50)
results51 <- content(res51)
results52 <- content(res52)
results53 <- content(res53)
results54 <- content(res54)
results55 <- content(res55)
results56 <- content(res56)
results57 <- content(res57)
results58 <- content(res58)
results59 <- content(res59)
results60 <- content(res60)
results61 <- content(res61)
results62 <- content(res62)
results63 <- content(res63)
results64 <- content(res64)
results65 <- content(res65)
results66 <- content(res66)
results67 <- content(res67)
results68 <- content(res68)
results69 <- content(res69)
results70 <- content(res70)
results71 <- content(res71)
results72 <- content(res72)
results73 <- content(res73)
results74 <- content(res74)
results75 <- content(res75)
results76 <- content(res76)
results77 <- content(res77)
results78 <- content(res78)
results79 <- content(res79)
results80 <- content(res80)
results81 <- content(res81)
results82 <- content(res82)
results83 <- content(res83)

#parse data
yelp_httr_parse <- function(x) {

  parse_list <- list(id = x$id, 
                     name = x$name, 
                     categories = x$categories, 
                     rating = x$rating, 
                     review_count = x$review_count, 
                     latitude = x$coordinates$latitude, 
                     longitude = x$coordinates$longitude, 
                     address1 = x$location$address1, 
                     city = x$location$city, 
                     state = x$location$state, 
                     distance = x$distance)
  
  parse_list <- lapply(parse_list, FUN = function(x) ifelse(is.null(x), "", x))
  
  df <- tibble(id=parse_list$id,
                   name=parse_list$name, 
                   categories = parse_list$categories, 
                   rating = parse_list$rating, 
                   review_count = parse_list$review_count, 
                   latitude=parse_list$latitude, 
                   longitude = parse_list$longitude, 
                   address1 = parse_list$address1, 
                   city = parse_list$city, 
                   state = parse_list$state, 
                   distance= parse_list$distance)
  df
}

results_list2 <- lapply(results2$businesses, FUN = yelp_httr_parse)
results_list3 <- lapply(results3$businesses, FUN = yelp_httr_parse)
results_list4 <- lapply(results4$businesses, FUN = yelp_httr_parse)
results_list5 <- lapply(results5$businesses, FUN = yelp_httr_parse)
results_list6 <- lapply(results6$businesses, FUN = yelp_httr_parse)
results_list7 <- lapply(results7$businesses, FUN = yelp_httr_parse)
results_list8 <- lapply(results8$businesses, FUN = yelp_httr_parse)
results_list9 <- lapply(results9$businesses, FUN = yelp_httr_parse)
results_list10 <- lapply(results10$businesses, FUN = yelp_httr_parse)
results_list11 <- lapply(results11$businesses, FUN = yelp_httr_parse)
results_list12 <- lapply(results12$businesses, FUN = yelp_httr_parse)
results_list13 <- lapply(results13$businesses, FUN = yelp_httr_parse)
results_list14 <- lapply(results14$businesses, FUN = yelp_httr_parse)
results_list15 <- lapply(results15$businesses, FUN = yelp_httr_parse)
results_list16 <- lapply(results16$businesses, FUN = yelp_httr_parse)
results_list17 <- lapply(results17$businesses, FUN = yelp_httr_parse)
results_list18 <- lapply(results18$businesses, FUN = yelp_httr_parse)
results_list19 <- lapply(results19$businesses, FUN = yelp_httr_parse)
results_list20 <- lapply(results20$businesses, FUN = yelp_httr_parse)
results_list21 <- lapply(results21$businesses, FUN = yelp_httr_parse)
results_list22 <- lapply(results22$businesses, FUN = yelp_httr_parse)
results_list23 <- lapply(results23$businesses, FUN = yelp_httr_parse)
results_list24 <- lapply(results24$businesses, FUN = yelp_httr_parse)
results_list25 <- lapply(results25$businesses, FUN = yelp_httr_parse)
results_list26 <- lapply(results26$businesses, FUN = yelp_httr_parse)
results_list27 <- lapply(results27$businesses, FUN = yelp_httr_parse)
results_list28 <- lapply(results28$businesses, FUN = yelp_httr_parse)
results_list29 <- lapply(results29$businesses, FUN = yelp_httr_parse)
results_list30 <- lapply(results30$businesses, FUN = yelp_httr_parse)
results_list31 <- lapply(results31$businesses, FUN = yelp_httr_parse)
results_list32 <- lapply(results32$businesses, FUN = yelp_httr_parse)
results_list33 <- lapply(results33$businesses, FUN = yelp_httr_parse)
results_list34 <- lapply(results34$businesses, FUN = yelp_httr_parse)
results_list35 <- lapply(results35$businesses, FUN = yelp_httr_parse)
results_list36 <- lapply(results36$businesses, FUN = yelp_httr_parse)
results_list37 <- lapply(results37$businesses, FUN = yelp_httr_parse)
results_list38 <- lapply(results38$businesses, FUN = yelp_httr_parse)
results_list39 <- lapply(results39$businesses, FUN = yelp_httr_parse)
results_list40 <- lapply(results40$businesses, FUN = yelp_httr_parse)
results_list41 <- lapply(results41$businesses, FUN = yelp_httr_parse)
results_list42 <- lapply(results42$businesses, FUN = yelp_httr_parse)
results_list43 <- lapply(results43$businesses, FUN = yelp_httr_parse)
results_list44 <- lapply(results44$businesses, FUN = yelp_httr_parse)
results_list45 <- lapply(results45$businesses, FUN = yelp_httr_parse)
results_list46 <- lapply(results46$businesses, FUN = yelp_httr_parse)
results_list47 <- lapply(results47$businesses, FUN = yelp_httr_parse)
results_list48 <- lapply(results48$businesses, FUN = yelp_httr_parse)
results_list49 <- lapply(results49$businesses, FUN = yelp_httr_parse)
results_list50 <- lapply(results50$businesses, FUN = yelp_httr_parse)
results_list51 <- lapply(results51$businesses, FUN = yelp_httr_parse)
results_list52 <- lapply(results52$businesses, FUN = yelp_httr_parse)
results_list53 <- lapply(results53$businesses, FUN = yelp_httr_parse)
results_list54 <- lapply(results54$businesses, FUN = yelp_httr_parse)
results_list55 <- lapply(results55$businesses, FUN = yelp_httr_parse)
results_list56 <- lapply(results56$businesses, FUN = yelp_httr_parse)
results_list57 <- lapply(results57$businesses, FUN = yelp_httr_parse)
results_list58 <- lapply(results58$businesses, FUN = yelp_httr_parse)
results_list59 <- lapply(results59$businesses, FUN = yelp_httr_parse)
results_list60 <- lapply(results60$businesses, FUN = yelp_httr_parse)
results_list61 <- lapply(results61$businesses, FUN = yelp_httr_parse)
results_list62 <- lapply(results62$businesses, FUN = yelp_httr_parse)
results_list63 <- lapply(results63$businesses, FUN = yelp_httr_parse)
results_list64 <- lapply(results64$businesses, FUN = yelp_httr_parse)
results_list65 <- lapply(results65$businesses, FUN = yelp_httr_parse)
results_list66 <- lapply(results66$businesses, FUN = yelp_httr_parse)
results_list67 <- lapply(results67$businesses, FUN = yelp_httr_parse)
results_list68 <- lapply(results68$businesses, FUN = yelp_httr_parse)
results_list69 <- lapply(results69$businesses, FUN = yelp_httr_parse)
results_list70 <- lapply(results70$businesses, FUN = yelp_httr_parse)
results_list71 <- lapply(results71$businesses, FUN = yelp_httr_parse)
results_list72 <- lapply(results72$businesses, FUN = yelp_httr_parse)
results_list73 <- lapply(results73$businesses, FUN = yelp_httr_parse)
results_list74 <- lapply(results74$businesses, FUN = yelp_httr_parse)
results_list75 <- lapply(results75$businesses, FUN = yelp_httr_parse)
results_list76 <- lapply(results76$businesses, FUN = yelp_httr_parse)
results_list77 <- lapply(results77$businesses, FUN = yelp_httr_parse)
results_list78 <- lapply(results78$businesses, FUN = yelp_httr_parse)
results_list79 <- lapply(results79$businesses, FUN = yelp_httr_parse)
results_list80 <- lapply(results80$businesses, FUN = yelp_httr_parse)
results_list81 <- lapply(results81$businesses, FUN = yelp_httr_parse)
results_list82 <- lapply(results82$businesses, FUN = yelp_httr_parse)
results_list83 <- lapply(results83$businesses, FUN = yelp_httr_parse)

food2 <- do.call("rbind", results_list2)
county <- rep("Oakland", length(food2$city))
food_data2 <- cbind(food2, county)

food3 <- do.call("rbind", results_list3)
county <- rep("Macomb", length(food3$city))
food_data3 <- cbind(food3, county)

food4 <- do.call("rbind", results_list4)
county <- rep("Kent", length(food4$city))
food_data4 <- cbind(food4, county)

food5 <- do.call("rbind", results_list5)
county <- rep("Genesee", length(food5$city))
food_data5 <- cbind(food5, county)

food6 <- do.call("rbind", results_list6)
county <- rep("Washtenaw", length(food6$city))
food_data6 <- cbind(food6, county)

food7 <- do.call("rbind", results_list7)
county <- rep("Ingham", length(food7$city))
food_data7 <- cbind(food7, county)

food8 <- do.call("rbind", results_list8)
county <- rep("Ottawa", length(food8$city))
food_data8 <- cbind(food8, county)

food9 <- do.call("rbind", results_list9)
county <- rep("Kalamazoo", length(food9$city))
food_data9 <- cbind(food9, county)

food10 <- do.call("rbind", results_list10)
county <- rep("Livingston", length(food10$city))
food_data10 <- cbind(food10, county)

food11 <- do.call("rbind", results_list11)
county <- rep("Saginaw", length(food11$city))
food_data11 <- cbind(food11, county)

food12 <- do.call("rbind", results_list12)
county <- rep("Muskegon", length(food12$city))
food_data12 <- cbind(food12, county)

food13 <- do.call("rbind", results_list13)
county <- rep("St. Clair", length(food13$city))
food_data13 <- cbind(food13, county)

food14 <- do.call("rbind", results_list14)
county <- rep("Jackson", length(food14$city))
food_data14 <- cbind(food14, county)

food15 <- do.call("rbind", results_list15)
county <- rep("Berrien", length(food15$city))
food_data15 <- cbind(food15, county)

food16 <- do.call("rbind", results_list16)
county <- rep("Monroe", length(food16$city))
food_data16 <- cbind(food16, county)

food17 <- do.call("rbind", results_list17)
county <- rep("Calhoun", length(food17$city))
food_data17 <- cbind(food17, county)

food18 <- do.call("rbind", results_list18)
county <- rep("Allegan", length(food18$city))
food_data18 <- cbind(food18, county)

food19 <- do.call("rbind", results_list19)
county <- rep("Eaton", length(food19$city))
food_data19 <- cbind(food19, county)

food20 <- do.call("rbind", results_list20)
county <- rep("Bay", length(food20$city))
food_data20 <- cbind(food20, county)

food21 <- do.call("rbind", results_list21)
county <- rep("Lenawee", length(food21$city))
food_data21 <- cbind(food21, county)

food22 <- do.call("rbind", results_list22)
county <- rep("Grand Traverse", length(food22$city))
food_data22 <- cbind(food22, county)

food23 <- do.call("rbind", results_list23)
county <- rep("Lapeer", length(food23$city))
food_data23 <- cbind(food23, county)

food24 <- do.call("rbind", results_list24)
county <- rep("Midland", length(food24$city))
food_data24 <- cbind(food24, county)

food25 <- do.call("rbind", results_list25)
county <- rep("Clinton", length(food25$city))
food_data25 <- cbind(food25, county)

food26 <- do.call("rbind", results_list26)
county <- rep("Van Buren", length(food26$city))
food_data26 <- cbind(food26, county)

food27 <- do.call("rbind", results_list27)
county <- rep("Isabella", length(food27$city))
food_data27 <- cbind(food27, county)

food28 <- do.call("rbind", results_list28)
county <- rep("Shiawassee", length(food28$city))
food_data28 <- cbind(food28, county)

food29 <- do.call("rbind", results_list29)
county <- rep("Marquette", length(food29$city))
food_data29 <- cbind(food29, county)

food30 <- do.call("rbind", results_list30)
county <- rep("Ionia", length(food30$city))
food_data30 <- cbind(food30, county)

food31 <- do.call("rbind", results_list31)
county <- rep("Montcalm", length(food31$city))
food_data31 <- cbind(food31, county)

food32 <- do.call("rbind", results_list32)
county <- rep("Barry", length(food32$city))
food_data32 <- cbind(food32, county)

food33 <- do.call("rbind", results_list33)
county <- rep("St. Joseph", length(food33$city))
food_data33 <- cbind(food33, county)

food34 <- do.call("rbind", results_list34)
county <- rep("Tuscola", length(food34$city))
food_data34 <- cbind(food34, county)

food35 <- do.call("rbind", results_list35)
county <- rep("Cass", length(food35$city))
food_data35 <- cbind(food35, county)

food36 <- do.call("rbind", results_list36)
county <- rep("Newaygo", length(food36$city))
food_data36 <- cbind(food36, county)

food37 <- do.call("rbind", results_list37)
county <- rep("Hillsdale", length(food37$city))
food_data37 <- cbind(food37, county)

food38 <- do.call("rbind", results_list38)
county <- rep("Branch", length(food38$city))
food_data38 <- cbind(food38, county)

food39 <- do.call("rbind", results_list39)
county <- rep("Mecosta", length(food39$city))
food_data39 <- cbind(food39, county)

food40 <- do.call("rbind", results_list40)
county <- rep("Sanilac", length(food40$city))
food_data40 <- cbind(food40, county)

food41 <- do.call("rbind", results_list41)
county <- rep("Gratiot", length(food41$city))
food_data41 <- cbind(food41, county)

food42 <- do.call("rbind", results_list42)
county <- rep("Chippewa", length(food42$city))
food_data42 <- cbind(food42, county)

food43 <- do.call("rbind", results_list43)
county <- rep("Houghton", length(food43$city))
food_data43 <- cbind(food43, county)

food44 <- do.call("rbind", results_list44)
county <- rep("Delta", length(food44$city))
food_data44 <- cbind(food44, county)

food45 <- do.call("rbind", results_list45)
county <- rep("Wexford", length(food45$city))
food_data45 <- cbind(food45, county)

food46 <- do.call("rbind", results_list46)
county <- rep("Emmet", length(food46$city))
food_data46 <- cbind(food46, county)

food47 <- do.call("rbind", results_list47)
county <- rep("Huron", length(food47$city))
food_data47 <- cbind(food47, county)

food48 <- do.call("rbind", results_list48)
county <- rep("Clare", length(food48$city))
food_data48 <- cbind(food48, county)

food49 <- do.call("rbind", results_list49)
county <- rep("Mason", length(food49$city))
food_data49 <- cbind(food49, county)

food50 <- do.call("rbind", results_list50)
county <- rep("Alpena", length(food50$city))
food_data50 <- cbind(food50, county)

food51 <- do.call("rbind", results_list51)
county <- rep("Oceana", length(food51$city))
food_data51 <- cbind(food51, county)

food52 <- do.call("rbind", results_list52)
county <- rep("Charlevoix", length(food52$city))
food_data52 <- cbind(food52, county)

food53 <- do.call("rbind", results_list53)
county <- rep("Cheboygan", length(food53$city))
food_data53 <- cbind(food53, county)

food54 <- do.call("rbind", results_list54)
county <- rep("Dickinson", length(food54$city))
food_data54 <- cbind(food54, county)

food55 <- do.call("rbind", results_list55)
county <- rep("Gladwin", length(food55$city))
food_data55 <- cbind(food55, county)

food56 <- do.call("rbind", results_list56)
county <- rep("Iosco", length(food56$city))
food_data56 <- cbind(food56, county)

food57 <- do.call("rbind", results_list57)
county <- rep("Otsego", length(food57$city))
food_data57 <- cbind(food57, county)

food58 <- do.call("rbind", results_list58)
county <- rep("Manistee", length(food58$city))
food_data58 <- cbind(food58, county)

food59 <- do.call("rbind", results_list59)
county <- rep("Roscommon", length(food59$city))
food_data59 <- cbind(food59, county)

food60 <- do.call("rbind", results_list60)
county <- rep("Antrim", length(food60$city))
food_data60 <- cbind(food60, county)

food61 <- do.call("rbind", results_list61)
county <- rep("Osceola", length(food61$city))
food_data61 <- cbind(food61, county)

food62 <- do.call("rbind", results_list62)
county <- rep("Menominee", length(food62$city))
food_data62 <- cbind(food62, county)

food63 <- do.call("rbind", results_list63)
county <- rep("Leelanau", length(food63$city))
food_data63 <- cbind(food63, county)

food64 <- do.call("rbind", results_list64)
county <- rep("Ogemaw", length(food64$city))
food_data64 <- cbind(food64, county)

food65 <- do.call("rbind", results_list65)
county <- rep("Kalkaska", length(food65$city))
food_data65 <- cbind(food65, county)

food66 <- do.call("rbind", results_list66)
county <- rep("Benzie", length(food66$city))
food_data66 <- cbind(food66, county)

food67 <- do.call("rbind", results_list67)
county <- rep("Missaukee", length(food67$city))
food_data67 <- cbind(food67, county)

food68 <- do.call("rbind", results_list68)
county <- rep("Gogebic", length(food68$city))
food_data68 <- cbind(food68, county)

food69 <- do.call("rbind", results_list69)
county <- rep("Arenac", length(food69$city))
food_data69 <- cbind(food69, county)

food70 <- do.call("rbind", results_list70)
county <- rep("Crawford", length(food70$city))
food_data70 <- cbind(food70, county)

food71 <- do.call("rbind", results_list71)
county <- rep("Presque Isle", length(food71$city))
food_data71 <- cbind(food71, county)

food72 <- do.call("rbind", results_list72)
county <- rep("Lake", length(food72$city))
food_data72 <- cbind(food72, county)

food73 <- do.call("rbind", results_list73)
county <- rep("Iron", length(food73$city))
food_data73 <- cbind(food73, county)

food74 <- do.call("rbind", results_list74)
county <- rep("Mackinac", length(food74$city))
food_data74 <- cbind(food74, county)

food75 <- do.call("rbind", results_list75)
county <- rep("Alcona", length(food75$city))
food_data75 <- cbind(food75, county)

food76 <- do.call("rbind", results_list76)
county <- rep("Montmorency", length(food76$city))
food_data76 <- cbind(food76, county)

food77 <- do.call("rbind", results_list77)
county <- rep("Alger", length(food77$city))
food_data77 <- cbind(food77, county)

food78 <- do.call("rbind", results_list78)
county <- rep("Baraga", length(food78$city))
food_data78 <- cbind(food78, county)

food79 <- do.call("rbind", results_list79)
county <- rep("Oscoda", length(food79$city))
food_data79 <- cbind(food79, county)

food80 <- do.call("rbind", results_list80)
county <- rep("Schoolcraft", length(food80$city))
food_data80 <- cbind(food80, county)

food81 <- do.call("rbind", results_list81)
county <- rep("Luce", length(food81$city))
food_data81 <- cbind(food81, county)

food82 <- do.call("rbind", results_list82)
county <- rep("Ontonagon", length(food82$city))
food_data82 <- cbind(food82, county)

food83 <- do.call("rbind", results_list83)
county <- rep("Keweenaw", length(food83$city))
food_data83 <- cbind(food83, county)
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;To have a complete dataset, we appended as follows, and called our final merged dataset for cuisines and counties: “all _establishments.”

```{r}
#append 83 datasets
all_establishments <- rbind(
  food_data, food_data2, food_data3, food_data4, food_data5, food_data6,
  food_data7, food_data8, food_data9, food_data10, food_data11, food_data12,
  food_data13, food_data14, food_data15, food_data16, food_data17, food_data18,
  food_data19, food_data20, food_data21, food_data22, food_data23, food_data24,
  food_data25, food_data26, food_data27, food_data28, food_data29, food_data30,
  food_data31, food_data32, food_data33, food_data34, food_data35, food_data36,
  food_data37, food_data38, food_data39, food_data40, food_data41, food_data42,
  food_data43, food_data44, food_data45, food_data46, food_data47, food_data48,
  food_data49, food_data50, food_data51, food_data52, food_data53, food_data54,
  food_data55, food_data56, food_data57, food_data58, food_data59, food_data60,
  food_data61, food_data62, food_data63, food_data64, food_data65, food_data66,
  food_data67, food_data68, food_data69, food_data70, food_data71, food_data72,
  food_data73, food_data74, food_data75, food_data76, food_data77, food_data78,
  food_data79, food_data80, food_data81, food_data82, food_data83)
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;One of the columns of interest in “all_establishments,” ‘categories,’ requires some formatting clean-up. The ‘categories’ used by Yelp were very specific, whereas we were hoping to use the more general cuisine types for our analyses. Additionally, we needed to place the categories into new, appropriate buckets to make the analyses relevant. Thus, we reformatted that column in order to make it more understandable and clear. We show a few examples of classifying ‘categories’ in the following chunk:

```{r, warning = FALSE}
#clean up 'categories'
all_establishments$categories <- all_establishments$categories %>%
  str_replace_all(., "list\\(alias = ", "") %>%
  str_replace_all(., "\\)", "") %>%
  str_replace_all(., ", title = ", "") %>%
  str_replace_all(., "\"", "") %>%
  str_replace_all(., "burgersBurgers", "American") %>%
  str_replace_all(., "newamericanAmerican \\(New", "American") %>%
  str_replace_all(., "beergardensBeer Gardens", "Bars/Breweries")
```


```{r, include = FALSE}
#clean up the remainder of 'categories'
all_establishments$categories <- all_establishments$categories %>%
  str_replace_all(., "barsBars", "Bars/Breweries") %>%
  str_replace_all(., "bakeriesBakeries", "Cafes/Dessert") %>%
  str_replace_all(., "african", "") %>%
  str_replace_all(., "bubbleteaBubble Tea", "Cafes/Dessert") %>%
  str_replace_all(., "coffeeCoffee & Tea", "Cafes/Dessert") %>%
  str_replace_all(., "tradamericanAmerican \\(Traditional", "American") %>%
  str_replace_all(., "cafesCafes", "Cafes/Dessert") %>%
  str_replace_all(., "breweriesBreweries", "Bars/Breweries") %>%
  str_replace_all(., "hotdogHot Dogs", "American") %>%
  str_replace_all(., "icecreamIce Cream & Frozen Yogurt", "Cafes/Dessert") %>%
  str_replace_all(., "teaTea Rooms", "Cafes/Dessert") %>%
  str_replace_all(., "sportsbarsSports Bars", "Bars/Breweries") %>%
  str_replace_all(., "brewpubsBrewpubs", "Bars/Breweries") %>%
  str_replace_all(., "hotdogsFast Food", "American") %>%
  str_replace_all(., "chocolateChocolatiers & Shops", "Cafes/Dessert") %>%
  str_replace_all(., "candyCandy Stores", "Cafes/Dessert") %>%
  str_replace_all(., "donutsDonuts", "Cafes/Dessert") %>%
  str_replace_all(., "juicebarsJuice Bars & Smoothies", "Cafes/Dessert") %>%
  str_replace_all(., "cocktailbarsCocktail Bars", "Bars/Breweries") %>%
  str_replace_all(., "coffeeroasteriesCoffee Roasteries", "Cafes/Dessert") %>%
  str_replace_all(., "beer_and_wineBeer, Wine & Spirits", "Bars/Breweries") %>%
  str_replace_all(., "bbqBarbeque", "American") %>%
  str_replace_all(., "italian", "") %>%
  str_replace_all(., "wineriesWineries", "Bars/Breweries") %>%
  str_replace_all(., "vietnameseVietnamese", "Southeast Asian") %>%
  str_replace_all(., "burmeseBurmese", "Southeast Asian") %>%
  str_replace_all(., "thaiThai", "Southeast Asian") %>%
  str_replace_all(., "teaTea Rooms", "Cafes/Dessert") %>%
  str_replace_all(., "sushiSushi Bars", "Japanese/Hawaiian") %>%
  str_replace_all(., "pokePoke", "Japanese/Hawaiian") %>%
  str_replace_all(., "ramenRamen", "Japanese/Hawaiian") %>%
  str_replace_all(., "japaneseJapanese", "Japanese/Hawaiian") %>%
  str_replace_all(., "grocery", "") %>%
  str_replace_all(., "greekGreek", "Mediterranean/Middle Eastern") %>%
  str_replace_all(., "fishnchipsFish & Chips", "Misc. European") %>%
  str_replace_all(., "frenchFrench", "Misc. European") %>%
  str_replace_all(., "germanGerman", "Misc. European") %>%
  str_replace_all(., "distilleriesDistilleries", "Bars/Breweries") %>%
  str_replace_all(., "divebarsDive Bars", "Bars/Breweries") %>%
  str_replace_all(., "dessertsDesserts", "Cafes/Dessert") %>%
  str_replace_all(., "delisDelis", "Delis/Sandwiches") %>%
  str_replace_all(., "cupcakesCupcakes", "Cafes/Dessert") %>%
  str_replace_all(., "creperiesCreperies", "Cafes/Dessert") %>%
  str_replace_all(., "mexican", "") %>%
  str_replace_all(., "britishBritish", "Misc. European") %>%
  str_replace_all(., "breakfast_brunch", "") %>%
  str_replace_all(., "chinese", "") %>%
  str_replace_all(., "koreanKorean", "Misc. Asian") %>%
  str_replace_all(., "asianfusionAsian Fusion", "Misc. Asian") %>%
  str_replace_all(., "tex-mexTex-Mex", "Mexican") %>%
  str_replace_all(., "tacosTacos", "Mexican") %>%
  str_replace_all(., "southernSouthern", "American") %>%
  str_replace_all(., "pizzaPizza", "Pizza") %>%
  str_replace_all(., "noodlesNoodles", "Japanese/Hawaiian") %>%
  str_replace_all(., "panasianPan Asian", "Misc. Asian") %>%
  str_replace_all(., "modern_europeanModern European", "Misc. European") %>%
  str_replace_all(., "mideasternMiddle Eastern", "Mediterranean/Middle Eastern") %>%
  str_replace_all(., "sandwichesSandwiches", "Delis/Sandwiches") %>%
  str_replace_all(., "pubsPubs", "Bars/Breweries") %>%
  str_replace_all(., "polishPolish", "Misc. European") %>%
  str_replace_all(., "mediterraneanMediterranean", "Mediterranean/Middle Eastern") %>%
  str_replace_all(., "lebaneseLebanese", "Mediterranean/Middle Eastern") %>%
  str_replace_all(., "steakSteakhouses", "Steakhouses") %>%
  str_replace_all(., "smokehouseSmokehouse", "Steakhouses") %>%
  str_replace_all(., "streetvendorsStreet Vendors", "Misc. Asian") %>%
  str_replace_all(., "saladSalad", "Salad") %>%
  str_replace_all(., "scandinavianScandinavian", "Misc. European") %>%
  str_replace_all(., "seafoodSeafood", "Seafood") %>%
  str_replace_all(., "seafoodmarketsSeafood Markets", "Seafood") %>%
  str_replace_all(., "wrapsWraps", "Delis/Sandwiches") %>%
  str_replace_all(., "venezuelanVenezuelan", "Latin/Spanish") %>%
  str_replace_all(., "veganVegan", "Vegan/Vegetarian") %>%
  str_replace_all(., "vegetarianVegetarian", "Vegan/Vegetarian") %>%
  str_replace_all(., "tapasTapas Bars", "Latin/Spanish") %>%
  str_replace_all(., "tapasmallplatesTapas/Small Plates", "Latin/Spanish") %>%
  str_replace_all(., "popuprestaurantsPop-Up Restaurants", "American") %>%
  str_replace_all(., "latinLatin American", "Latin/Spanish") %>%
  str_replace_all(., "wine_barsWine Bars", "Bars/Breweries") %>%
  str_replace_all(., "bagelsBagels", "Delis/Sandwiches") %>%
  str_replace_all(., "bedbreakfastBed & Breakfast", "Breakfast & Brunch") %>%
  str_replace_all(., "belgianBelgian", "Misc. European") %>%
  str_replace_all(., "beerbarBeer Bar", "Bars/Breweries") %>%
  str_replace_all(., "butcherButcher", "Meat") %>%
  str_replace_all(., "meatsMeat Shops", "Meat") %>%
  str_replace_all(., "buffetsBuffets", "Buffets") %>%
  str_replace_all(., "cajunCajun/Creole", "Cajun/Caribbean") %>%
  str_replace_all(., "caribbeanCaribbean", "Cajun/Caribbean") %>%
  str_replace_all(., "cafeteriaCafeteria", "American") %>%
  str_replace_all(., "chicken_wingsChicken Wings", "Chickens/Wings") %>%
  str_replace_all(., "chickenshopChicken Shop", "Chickens/Wings") %>%
  str_replace_all(., "comfortfoodComfort Food", "American") %>%
  str_replace_all(., "customcakesCustom Cakes", "Cafes/Dessert") %>%
  str_replace_all(., "cubanCuban", "Latin/Spanish") %>%
  str_replace_all(., "dinersDiners", "Diners") %>%
  str_replace_all(., "gluten_freeGluten-Free", "Vegan/Vegetarian") %>%
  str_replace_all(., "vacation_rentalsVacation Rentals", "Non-restaurant") %>%
  str_replace_all(., "venuesVenues & Event Spaces", "Non-restaurant") %>%
  str_replace_all(., "theaterPerforming Arts", "Non-restaurant") %>%
  str_replace_all(., "souvenirsSouvenir Shops", "Non-restaurant") %>%
  str_replace_all(., "servicestationsGas Stations", "Non-restaurant") %>%
  str_replace_all(., "partysuppliesParty Supplies", "Non-restaurant") %>%
  str_replace_all(., "musicvideoMusic & DVDs", "Non-restaurant") %>%
  str_replace_all(., "newcanadianCanadian \\(New", "Bars/Breweries") %>%
  str_replace_all(., "marketsFruits & Veggies", "Fruits & Veggies") %>%
  str_replace_all(., "importedfoodImported Food", "Misc. European") %>%
  str_replace_all(., "hotelsHotels", "Non-restaurant") %>%
  str_replace_all(., "halalHalal", "Mediterranean/Middle Eastern") %>%
  str_replace_all(., "hairHair Salons", "Non-restaurant") %>%
  str_replace_all(., "healthmarketsHealth Markets", "Grocery") %>%
  str_replace_all(., "indonesianIndonesian", "Southeast Asian") %>%
  str_replace_all(., "indpakIndian", "Indian") %>%
  str_replace_all(., "himalayanHimalayan/Nepalese", "Misc. Asian") %>%
  str_replace_all(., "loungesLounges", "Non-restaurant") %>%
  str_replace_all(., "soulfoodSoul Food", "American") %>%
  str_replace_all(., "soupSoup", "American") %>%
  str_replace_all(., "golfGolf", "Non-restaurant") %>%
  str_replace_all(., "giftshopsGift Shops", "Non-restaurant") %>%
  str_replace_all(., "gastropubsGastropubs", "Bars/Breweries") %>%
  str_replace_all(., "foodtrucksFood Trucks", "Mexican") %>%
  str_replace_all(., "foodstandsFood Stands", "American") %>%
  str_replace_all(., "flowersFlowers & Gifts", "Non-restaurant") %>%
  str_replace_all(., "farmersmarketFarmers Market", "Grocery") %>%
  str_replace_all(., "drugstoresDrugstores", "Grocery") %>%
  str_replace_all(., "deptstoresDepartment Stores", "Grocery") %>%
  str_replace_all(., "convenienceConvenience Stores", "Grocery") %>%
  str_replace_all(., "cateringCaterers", "American") %>%
  str_replace_all(., "brazilianBrazilian", "Latin/Spanish") %>%
  str_replace_all(., "bowlingBowling", "Non-restaurant") %>%
  str_replace_all(., "bookstoresBookstores", "Non-restaurant") %>%
  str_replace_all(., "gourmetSpecialty Food", "Other") %>%
  str_replace_all(., "restaurantsRestaurants", "Other") %>%
  str_replace_all(., "arcadesArcades", "Other") %>%
  str_replace_all(., "winetastingroomWine Tasting Room", "Bars/Breweries") %>%
  str_replace_all(., "cheeseCheese Shops", "Other")
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Our initially cleaned data set of cuisines and counties (all_establisments) had 1568 observations, since the most populated areas of certain counties had less than 20 food establishment options on Yelp. However, when looking through this dataset, we noticed more areas of data-cleaning that needed to be addressed. For instance, Yelp has captured establishments that don’t align with our restaurant-specific analyses (i.e. bowling). We grouped these establishments into a category called “Non-restaurant.” We pre-determined to not include grocery stores (“Grocery” in our dataset) in our analyses as they are a fairly different food establishment type than a restaurant and it can be more difficult to determine the choices a consumer made or the impact those choices will have on their obesity. Lastly, we had a small group of establishments that did not fit into any categories, so we placed them in a category we called “Other.” As these three categories may not be very useful in our analyses, we removed them from our dataset, shown below:

```{r}
all_establishments <- all_establishments %>% 
  filter(!categories == "Non-restaurant") %>%
  filter(!categories == "Grocery") %>%
  filter(!categories == "Other")
```

```{r, include = FALSE}
all_establishments %>% mutate_if(is.factor, as.character) -> all_establishments
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;After removing the "Non-restaurant", "Grocery", and "Other" categories from all_establishments, our final dataset of cuisines and counties includes 1479 unique observations.

### _Obesity Data_

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Obesity data was collected from the Institute for Health Metrics and Evaluation website, in the form of an Excel file (saved as a CSV) listing counties with both a female and male obesity rate for each county. The file was then uploaded to our Github repository and we read in the raw url file.

```{r, message = FALSE}
obesity <- read_csv("https://raw.githubusercontent.com/daraeoh/survmeth727/master/obesity.csv")
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We took the average of the male and female obesity rates and created a new column called “obesity_avg” to use as our depended variable of interest, as shown below. We will refer to this variable as “aggregate rate of obesity.”

```{r}
obesity_avg <- 
  (obesity$female_obesity_prevalence_percent + obesity$male_obesity_prevalence_percent) / 2
avg_obesity <- cbind(obesity, obesity_avg)
```

```{r, include = FALSE}
avg_obesity %>% mutate_if(is.factor, as.character) -> avg_obesity
```

### _Master Data_

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Now that we have gathered both the “cuisine and counties” data set and the “obesity” data set, we merged them together by “county” to get our master dataset (entitled “master”) for analysis.

```{r, warning = FALSE}
master <-inner_join(all_establishments, avg_obesity, by = "county")
```

## Results

### _Data Exploration_

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In our exploratory analyses, we started by creating a basic barplot for “categories” to get an idea of the prevalence of different types of the most popular cuisines on Yelp for the state of Michigan.

```{r}
#basic barplot
ggplot(master) + 
  geom_bar(aes(x = categories)) + 
  labs(
    x = "Cuisine Category", 
    y = "Frequency", 
    title = "Frequency of Cuisine Categories on Yelp"
    ) + 
  coord_flip()
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;According to our barplot; American, Bars/Breweries, Cafes/Dessert, and Pizza are the most popular types of cuisine on Yelp for Michigan.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Since the obesity data we collected outlined the obesity rates for both males and females, we were also interested in seeing how the separate sexes obesity rates differed by county. We created bar plots outlining the sex obesity percent rate by county. The bar graphs listed below show that each sex had a fairly different line-up for obesity percent by county.

```{r fig.height = 10, fig.width = 10}
#male obesity barplot by county
male <- aggregate(
  male_obesity_prevalence_percent ~ county, master, mean) %>%
  arrange(desc(male_obesity_prevalence_percent))
male_plot <- ggplot(male, 
                    aes(x = reorder(county, male_obesity_prevalence_percent), 
                        y = male_obesity_prevalence_percent)) + 
  geom_bar(
    stat = "identity", 
    color = 'skyblue', 
    fill = 'steelblue') + 
  labs(
    x = "County", 
    y = "Male Obesity Percent", 
    title = "Male Obesity Percent by County"
    ) +
  coord_flip()

#female obesity barplot by county
female <- aggregate(
  female_obesity_prevalence_percent ~ county, master, mean) %>%
  arrange(desc(female_obesity_prevalence_percent))
female_plot <- ggplot(female, 
                    aes(x = reorder(county, female_obesity_prevalence_percent), 
                        y = female_obesity_prevalence_percent)) + 
  geom_bar(
    stat = "identity", 
    color = 'lightpink', 
    fill = 'pink') + 
  labs(
    x = "", 
    y = "Female Obesity Percent", 
    title = "Female Obesity Percent by County"
    ) +
  coord_flip()

#print male and female obesity barplots
grid.arrange(male_plot, female_plot, ncol = 2)
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Next, we were interested in observing the top cuisine categories with the highest values of average aggregate rate of obesity. In order to obtain these statistics, we first found the mean percentage of aggregate rate of obesity for each cuisine category. Then we listed the top ten cuisine categories with the largest percentage of aggregate rate of obesity for each cuisine category. When we listed the top ten cuisine categories with the largest percentage of aggregate rate of obesity, we found that “African,” “Fruits and & Veggies,” “Diners,” “Meat,” and “Indian” cuisines were among the cuisine categories that yielded the highest obesity rates, as shown in the table below. It is worth noting that “African” and “Fruits & Veggies” were both cuisine types with a small sample size, which could be affecting the received results.

```{r, results = "hide"}
#top 10 cuisine categories arranged by highest average aggregate rate of obesity
top_categories <- aggregate(
  obesity_avg ~ categories, master, mean) %>%
  arrange(desc(obesity_avg))
head(top_categories, 10)
```

<!-- Table of top 10 cuisine categories arranged by highest average aggregate rate of obesity -->
Cuisine Categories | Average Aggregate Rate of Obesity (%)
------------------ | -------------------------------------
African	| 40.87500		
Fruits & Veggies	| 39.97500		
Diners	| 39.41268		
Meat	| 39.00550		
Indian	| 38.99000		
Buffets	| 38.98750		
Chinese	| 38.93091		
Pizza	| 38.71260		
Delis/Sandwiches	| 38.64617		
Bars/Breweries	| 38.47065	

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Similarly, we were interested in observing the top counties with the highest values of average aggregate rate of obesity, so we first found the mean percentage of aggregate rate of obesity for each county and then listed the top ten counties with the largest percentage of aggregate rate of obesity. These counties are listed in the table below:

```{r, results = "hide"}
#top 10 counties arranged by highest average aggregate rate of obesity
top_counties <- aggregate(
  obesity_avg ~ county, master, mean) %>%
  arrange(desc(obesity_avg))
head(top_counties, 10)
```

<!-- Table of top 10 counties arranged by highest average aggregate rate of obesity -->
County | Average Aggregate Rate of Obesity (%)
------ | -------------------------------------
Saginaw	| 44.85		
Genesee	| 43.65		
Lake	| 43.25		
Gratiot	| 43.20		
Ionia	| 42.45		
Calhoun	| 42.20		
Arenac	| 42.10		
Monroe	| 41.60		
Oceana	| 41.35		
Wayne	| 41.30

### _Analysis_

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Based on our data exploration and our research question, we decided to fit linear models to our data to analyze the relationship of the variables we collected on the obesity rates. In order to find the model that best fit our data, we began with a simple approach, modeling the obesity average as our dependent variable against the single “categories” variable.

```{r, results = "hide"}
#model with 'cateogories' as the only predictor
master %>%
  lm(obesity_avg ~ factor(categories), 
     data = .) %>%
  tidy()
```

```{r, results = "hide"}
#code including R-squared value
model1 <- lm(obesity_avg ~ factor(categories), 
             data = master)
summary(model1)
```

<!-- Table replicating model summary output, showing just the estimate and p-values -->
term | estimate | p.value
---- | -------- | -------
(Intercept)	| 40.875000	| 1.654985e-74
factor(categories)American	| -2.599681	|	2.193781e-01
factor(categories)Bars/Breweries	| -2.404350	|	2.574884e-01
factor(categories)Breakfast & Brunch	| -3.141583	|	1.433369e-01
factor(categories)Buffets	| -1.887500	|	4.653939e-01
factor(categories)Cafes/Dessert	| -2.473144	|	2.442854e-01
factor(categories)Cajun/Caribbean	| -2.819444	|	2.271220e-01
factor(categories)Chickens/Wings	| -3.175000	|	1.698905e-01
factor(categories)Chinese	| -1.944091	|	3.657209e-01
factor(categories)Delis/Sandwiches	| -2.228828	|	2.985644e-01
... with 17 more rows | |

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The R-squared value of our preliminary model was quite low (R-squared = 0.048), indicating poor model fit. To improve upon the model, we added county to the same model as above.

```{r, results = "hide"}
#model with 'categories' and 'county' as the predictors
master %>%
  lm(obesity_avg ~ factor(categories) + factor(county), data = .) %>%
  tidy()
```

```{r, results = "hide"}
#code including R-squared value
model2 <- lm(obesity_avg ~ factor(categories) + factor(county), data = master)
summary(model2)
```

<!-- Table replicating model summary output, showing just the estimate and p-values -->
term | estimate | p.value
---- | -------- | -------
(Intercept)	| 3.865000e+01	|	0.000000000
factor(categories)American	| -1.073765e-13	|	0.021576634
factor(categories)Bars/Breweries	| -1.340980e-13	|	0.004236776
factor(categories)Breakfast & Brunch	| -1.164266e-13	|	0.014129448
factor(categories)Buffets	| -4.075038e-14	|	0.475383464
factor(categories)Cafes/Dessert	| -1.227356e-13	|	0.009033133
factor(categories)Cajun/Caribbean	| -8.744369e-14	|	0.090267795
factor(categories)Chickens/Wings	| -1.355913e-13	|	0.008058154
factor(categories)Chinese	| -9.941365e-14	|	0.036456553
factor(categories)Delis/Sandwiches |	-1.062960e-13	|	0.024849817
... with 17 more rows of cuisine categories | |
factor(county)Alger	| -5.500000e-01	|	0.000000000
factor(county)Allegan	| -9.500000e-01	|	0.000000000
factor(county)Alpena	| -1.700000e+00	|	0.000000000
factor(county)Antrim	| -3.150000e+00	|	0.000000000
factor(county)Arenac	| 3.450000e+00	|	0.000000000
factor(county)Baraga	| 5.500000e-01	|	0.000000000
factor(county)Barry	| 1.950000e+00	|	0.000000000
factor(county)Bay	| 2.600000e+00	|	0.000000000
factor(county)Benzie	| -4.350000e+00	|	0.000000000
factor(county)Berrien	| 1.100000e+00	|	0.000000000
... with 73 more rows of counties | |

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The R-squared value of the second model was much higher than the first (R-squared = 1), indicating a better model fit. Due to this, we decided to make two additional bar plots outlining the aggregate obesity rate by county and cuisine, respectively. We found the bar plots to be an interesting comparison to the results featured in our models.

```{r fig.height = 10, fig.width = 13}
#aggregate rate of obesity plot by county
countyobes <- aggregate(
  obesity_avg ~ county, master, mean) %>%
  arrange(desc(obesity_avg))
county_plot <- ggplot(countyobes, 
                    aes(x = reorder(county, obesity_avg), 
                        y = obesity_avg)) + 
  geom_bar(
    stat = "identity", 
    color = 'violet', 
    fill = 'purple') + 
  labs(
    x = "County", 
    y = "Aggregate Rate of Obesity (%)", 
    title = "Aggregate Rate of Obesity by County"
    ) +
  coord_flip()

#aggregate rate of obesity plot by cuisine category
catobes <- aggregate(
  obesity_avg ~ categories, master, mean) %>%
  arrange(desc(obesity_avg))
categories_plot <- ggplot(catobes, 
                    aes(x = reorder(categories, obesity_avg), 
                        y = obesity_avg)) + 
  geom_bar(
    stat = "identity", 
    color = 'green', 
    fill = 'darkgreen') + 
  labs(
    x = "Cuisine Category", 
    y = "Aggregate Rate of Obesity (%)", 
    title = "Aggregate Rate of Obesity by Cuisine Category"
    ) +
  coord_flip()

#print both plots
grid.arrange(county_plot, categories_plot, ncol = 2)
```

## Discussion

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Our first model explored the effect of cuisine on obesity, without including the county variable. This model had poor fit and the variable outputs implied a lack of significance for the effect of cuisine on obesity, with the exception of three cuisine types: “Misc. Asian,” “Misc. European,” and “Vegan/Vegetarian.” Adding county data into the model helped to explain the data in a better manner, as we expected. At a 0.05 threshold for significance, the majority of the cuisine types had a significant effect on the model, with the exception of the following cuisine types: "Buffets," "Cajun/Caribbean," "Mediterranean/Middle Eastern," "Misc. Asian," "Steakhouses," and "Vegan/Vegetarian." All the counties in Michigan, except for Luce county, had a significant effect on obesity. Our results indicate evidence to support the main focus of our research question exploring the effect of cuisine types on obesity rates. Additionally, the strong significance of the county data on the obesity rates reveals potential focus for future data.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;A few limitations are throughout our research. Working with the Yelp API proved to be difficult for our dataset in a few different ways. As Yelp is constantly changing due to user generated input, our results would slightly change as well whenever we would re-run a model. Additionally, the cuisine data was not as concise as we had originally expected, which did lead to small sample sizes for each cuisine type. Though our cuisine types were not significant, future research with much larger sample sizes would still benefit from testing cuisine types to see if different results are received with more available data. Lastly, our model does not include weights, which does not account for the vastly different population sizes of some of the counties and the lack of available restaurants in the smaller counties.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In addition to the recommendations listed above, future research could benefit from exploring the effect of sex on obesity. Our data exploration found women to generally have higher obesity rates than men. When comparing the “Obesity Percent by County” bar plots for each sex to the “Aggregate Rate of Obesity by County,” the “Aggregate Rate of Obesity by County” bar plot is closer aligned with the female plot than the male plot.  Our research question specifically focused on restaurant and location level data, leading to our decision to not include health and socio-demographic data, as that would alter the focus of our research.

## Link to GitHub Repository

https://github.com/daraeoh/survmeth727

## References

Currie, Janet, Stefano DellaVigna, Enrico Moretti, and Vikram Pathania. 2010. "Effect of Fast Food Restaurants on Obesity and Weight Gain."

Davis, Brennan, and Christopher Carpenter. 2011. "Proximity of Fast-Food Restaurants to Schools and Adolescent Obesity."

Institute for Health Metrics and Evaluations. (n.d.). US County Profiles. Retrieved from http://www.healthdata.org/us-county-profiles.

Jeffery, Robert W., Judy Baxter, Maureen McGuire, and Jennifer Linde. 2006. "Are Fast Food Restaurants an Environmental Risk Factor for Obesity?"
