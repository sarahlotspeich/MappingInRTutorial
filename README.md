# Introduction to geocoding, calculating distances, and map-making in `R`
Files and datasets for R Ladies tutorial - 12 Sept 2017
Sarah Lotspeich

## A few packages for spatial analysis and visualization in R

  1.  [`ggmap`](https://cran.r-project.org/web/packages/ggmap/ggmap.pdf): A collection of functions to visualize spatial data and models
on top of static maps from various online sources (e.g Google Maps and Stamen Maps). It includes tools common to those tasks, including functions for geolocation and routing.
  2.  [`rgdal`](https://cran.r-project.org/web/packages/rgdal/rgdal.pdf): The Geospatial Data Abstraction Library, access to projection/ transformation operations.
  3.  [`ggplot2`](https://cran.r-project.org/web/packages/ggplot2/ggplot2.pdf): A system for 'declaratively' creating graphics, based on "The Grammar of Graphics". You provide the data, tell 'ggplot2' how to map variables to aesthetics, what graphical primitives to use, and it takes care of the details.
  4.  [`ggsn`](https://cran.r-project.org/web/packages/ggsn/ggsn.pdf): Adds north symbols (18 options) and scale bars in kilometers to maps in geographic or metric coordinates created with 'ggplot2' or 'ggmap'.
  5.  [`broom`](https://cran.r-project.org/web/packages/broom/vignettes/broom.html): The broom package takes the messy output of built-in functions in R, such as lm, nls, or t.test, and turns them into tidy data frames. We will use it here to turn imported shapefiles into data frames.
  6.  [`geosphere`](https://cran.r-project.org/web/packages/geosphere/geosphere.pdf): Spherical trigonometry for geographic applications. That is, compute distances and re- lated measures for angular (longitude/latitude) locations.
  7.  [`gmapsdistance`](https://cran.r-project.org/web/packages/gmapsdistance/gmapsdistance.pdf): Get distance and travel time between two points from Google Maps. Four possible modes of transportation (bicycling, walking, driving and public transportation).

```{r, message=FALSE, warning=FALSE}
library(ggmap)
library(rgdal)
library(ggplot2)
library(ggsn)
library(broom)
library(geosphere)
library(gmapsdistance)
library(wesanderson)
```

# Introduction to geocoding

In geospatial analyses, string addresses hold very little meaning just as in our everyday lives the precise latitude and longitude coordinates of our homes are not all that useful. That's where geocoding comes in. The basis of geocoding is the following (respectfully copy-pasted from Google about their [Google Maps Geocoding API](https://developers.google.com/maps/documentation/geocoding/start)): 

"Geocoding is the process of converting addresses (like a street address) into geographic coordinates (like latitude and longitude), which you can use to place markers on a map, or position the map."

Before we begin, here are a few "best practices" when preparing addresses to be geocoded (taken from the [Harvard University Center for Geographic Analysis](http://gis.harvard.edu/services/blog/geocoding-best-practices)): 

  1.  Remove suite/ apartment numbers as they will simply create "ties". 
  2.  Be mindful of special characters like @, #, ?, ;, -
  3.  Buckle down and be prepared to spend a good long while "cleaning" addresses. I spent two days on my pharmacy addresses. I would highly recommend doing it with a quality latte (or perhaps something stronger) in hand.

## Example 1: [City of Austin Public Art Collection](https://data.austintexas.gov/Fun/City-of-Austin-Public-Art-Collection/yqxj-7evp)

*Background:* This file contains the names and address of artworks commissioned through the Austin Art in Public Places Program. Established by the City in 1985, the Art in Public Places (AIPP) program collaborates with local & nationally-known artists to include the history and values of our community into cultural landmarks that have become cornerstones of Austin's identity. The City of Austin was the first municipality in Texas to make a commitment to include works of art in construction projects (taken from [here](http://www.publicartarchive.org/austinaipp)). 

```{r, warning=FALSE, tidy=TRUE}
AustinPublicArt <- read.csv("https://query.data.world/s/93ujuqzxbfjkwnda3y2p8mgv9", header=TRUE, stringsAsFactors=FALSE)
colnames(AustinPublicArt) 
AustinPublicArt$art_full_address <- paste(AustinPublicArt$art_location_street_address,AustinPublicArt$art_location_city,AustinPublicArt$art_location_state,AustinPublicArt$art_location_zip, sep = " ")
unique(AustinPublicArt$art_full_address) #let's see what we're working with here
```

##Keys for cleaning addresses: all hail [`sub()` and `gsub()`](http://rfunction.com/archives/2354)

If you just want to get rid of the first occurrence of something, use: 

  -   Use `sub("What you don't want", "What you do want", "Where you want it")`
  
If you want to get rid of every occurrence of something, use: 

  -   Use `gsub("What you don't want", "What you do want", "Where you want it")`
  
```{r, tidy=TRUE}
AustinPublicArt$art_full_address <- gsub(";", "", AustinPublicArt$art_full_address) #get rid of special character ";" at the end of each address
AustinPublicArt$art_full_address <- sub(".* - ", "", AustinPublicArt$art_full_address) #get rid of "Possum Point - " at the beginning
AustinPublicArt$art_full_address <- gsub("NA", "", AustinPublicArt$art_full_address) #remove NA zipcodes
```

Many geocoders exist today (both open-source and proprietary), but the basis is the same. The algorithm begins by taking a complete street address and breaking it down into its component parts. For example, if we wanted to geocode the Vanderbilt Biostatistics Department we would begin with 

2525 West End Avenue, Nashville, TN 37203 $\rightarrow$ 2525 West End Avenue | Nashville | TN | 37203

The geocoder then works its way down the geographic hierarchy that you specify (depending on the scope of your data) to find increasingly more precise coordinates based on matching the address's: 

  1.  State
  2.  City
  3.  Zip code
  4.  Street address

Fortunately, the R community has blessed us with yet another package: `ggmap`! [`ggmap`](https://cran.r-project.org/web/packages/ggmap/ggmap.pdf) contains a quick function called `geocode()` -- tricky to remember, I know -- that will take in a string address and output the latitude and longitude coordinates utilizing the Google Maps API (`source = "google"`) or the Data Science Toolkit online (`source = "dsk"`). Let's try this function out on the second artwork in the data set (the first cannot be geocoded).

```{r, message=TRUE, tidy=TRUE}
#view second full address
AustinPublicArt$art_full_address[2]
#geocode second full address using Data Science Toolkit
geocode(AustinPublicArt$art_full_address[2], source="dsk") 
#geocode second full address using Google Maps
geocode(AustinPublicArt$art_full_address[2], source="google") 
```

We can see that the geocoded addresses from each source are quite close. I have not (yet) found much information differentiating between the two, so for now I chose to proceed with Google Maps. It took around 3 minutes for the `r nrow(AustinPublicArt)` addresses to geocode, so to save time an updated csv with the latitude and longitude can be found [here](https://raw.githubusercontent.com/sarahlotspeich/MappingInRTutorial/master/AustinPublicArtWithLatLong.csv). 

```{r, tidy=TRUE}
#import completely geocoded AustinPublicArt dataset from the link above
AustinPublicArt.gc <- read.csv("https://raw.githubusercontent.com/sarahlotspeich/MappingInRTutorial/master/AustinPublicArtWithLatLong.csv", header=TRUE, stringsAsFactors=FALSE)
```

This is what is referred to as \underline{batch geocoding}. Now, the unfortunate truth is that we will rarely be able to successfully geocode an entire set of addresses due to either unforgiveable type-os or missing data. In our case, we were unable to precisely pinpoint the location of `r round((nrow(AustinPublicArt) - nrow(AustinPublicArt.gc))/nrow(AustinPublicArt),2)*100`$\%$ of the works of art. Excluding those that were unable to be geocoded leaves us with a subset of `r nrow(AustinPublicArt.gc)` works of art, and now that we have geocoded our addresses we are ready to build our first map! 

#It all starts with a shapefile

Now that we have data that we're interested in visualizing on a map, we need a blank canvas. One option for this blank canvas is called a ["shapefile."](https://en.wikipedia.org/wiki/Shapefile) This is the same file type used for Geographic Information Systems (GIS) software, so I prefer it because there is an abundance of data available in this format. The first place I look for maps of the United States is the [U.S. Census Bureau.](https://www.census.gov/geo/maps-data/data/tiger-line.html)

Using the [`readOGR()`](https://www.rdocumentation.org/packages/rgdal/versions/1.2-8/topics/readOGR) function in the `rgdal` package, we can read in the shapefile we've downloaded as a usable spatial layer. 

```{r, warning=FALSE, message=FALSE, tidy=TRUE}
UrbanAreasUS.shp <- readOGR(dsn = path.expand("tl_2016_us_uac10/tl_2016_us_uac10.shp"), layer="tl_2016_us_uac10")
class(UrbanAreasUS.shp)
```

What the heck is a `spatial polygons data frame`? 

## Classes of spatial data

  1.  `SpatialPoints`: simply describe locations (with no attributes)
  2.  `SpatialPointsDataFrame`: describes locations + attributes for them
  2.  `SpatialPolygonsDataFrame`: describes locations + shapes + attributes for them
  
We can access the familiar data frame component of the `UrbanAreas_US` spatial polygons data frame as follows: 

```{r}
#view the head of the UrbanAreas_US data frame
head(UrbanAreasUS.shp@data) 
head(data.frame(UrbanAreasUS.shp))
```

By taking advantage of this data frame, we can focus our map on the Austin, TX urban area using our usual data subsetting techniques. Be careful not to select a subset of only the data frame, because if you lose the spatial polygon part R will not plot your map properly (i.e. as a map). 

```{r, tidy=TRUE}
UrbanAreasUS.shp@data[which(UrbanAreasUS.shp@data$NAME10 == "Austin, TX"),]
AustinTX.shp <- subset(UrbanAreasUS.shp, UrbanAreasUS.shp@data$NAME10 == "Austin, TX")
```

### Spatial data in `ggplot2`

To map Austin in `ggplot2` we need to use the `tidy()` function in the `broom` package to convert the spatial polygon data frame to a traditional data frame. Previously, the `fortify()` function could be used for this, but this function is likely to become deprecated. 

```{r, tidy=TRUE}
AustinTX.df <- tidy(AustinTX.shp, region = "NAME10") 
#or AustinTX.df <- fortify(AustinTX.shp, region="NAME10")
```

### Creating Maps with `ggplot2`

Once the shapefile has been formatted appropriately, we can utilize our knowledge of the "grammar of graphics" to map the urban area for Austin, TX. Begin with the `geom_polygon()` function to plot the latitude and longitude coordinates for the outline of the Austin, TX urban area (`aes(x = long, y = lat)`) and use `group = group` in your aesthetic to tell `ggplot2` to treat this area as one polygon. Feel free to select your own `fill` color, and outline color, `col`. 

```{r, tidy=TRUE, fig.align="center"}
(AustinTX.map <- ggplot() + geom_polygon(data = AustinTX.df, aes(x = long, y = lat, group=group), fill = "white", col="black"))
```

By adding `+ coord_equal(ratio = 1)` to the end, we can fix the skew of the plot. 

```{r, tidy=TRUE, fig.align="center"}
(AustinTX.map <- AustinTX.map + coord_equal(ratio=1))
```

Use `ggtitle()` to add a main title describing your map. 

```{r, tidy=TRUE, fig.align="center"}
(AustinTX.map <- AustinTX.map + ggtitle("Ausin Public Art Collection"))
```

We will need the additional `ggsn` package to add north arrows and map scales for orientation. The `north()` function takes a `location` parameter for where to place it in the map. The `scalebar()` function also takes a `location` parameter, in addition to a `dist` input (distance in km per segment of the scale bar), `dd2km` (set to TRUE for a map in latitude and longitude, to FALSE for maps in meters), `model` (describing the projection used for the map, most commonly `= 'WGS84'`), and `st.size` which sets the text size for the map scale. 

```{r, tidy=TRUE, fig.align="center"}
(AustinTX.map <- AustinTX.map + north(AustinTX.df, location = "topleft")) 
(AustinTX.map <- AustinTX.map + scalebar(AustinTX.df, location = "bottomright", dist = 10, dd2km = TRUE, model = 'WGS84', st.size=2.5))
```

To make this look less like a plot and more like a traditional map, we can add the following chunk of to the end of our plot: 

`+ theme(axis.text.x=element_blank(), axis.text.y=element_blank(), axis.ticks=element_blank(),axis.title.x=element_blank(),axis.title.y=element_blank())`

to remove the x and y axis titles, tick lines, and values. 

```{r, tidy=TRUE, fig.align="center"}
(AustinTX.map <- AustinTX.map + theme(axis.text.x=element_blank(), axis.text.y=element_blank(), axis.ticks=element_blank(),axis.title.x=element_blank(),axis.title.y=element_blank()))
```

## Map of the City of Austin's Public Art Collection 

Now we can begin with our base map from above, and overlay it with the locations of works of street art! We can use the `geom_point()` function with the latitude and longitude coordinates and the original `AustinPublicArt` data frame. 

```{r, fig.align="center", tidy=TRUE}
(AustinArtCollection.map <- AustinTX.map + geom_point(data = AustinPublicArt.gc,aes(x = art_location_long, y = art_location_lat, color = "orange")) + theme(legend.position="none"))
```

From the map above, we should be able to make some preliminary observations about the spatial distribution of public artwork across the urban area for the city of Austin, Texas. Perhaps policy makers requested this analysis so that they could assess how the City of Austin Public Arts Collection is bringing enrichment to the people of Austin and if the collection is accessible to everyone. How would we answer their question? 

How about estimating the average distance between works of art as a gauge of availability/ accessibility? 

# Calculating distances between places on a map

When we think about distances traveled, how do we measure them? 

  1.  Distance (shortest route possible between two points)
    a.  [Haversine Formula:](http://www.movable-type.co.uk/scripts/latlong.html) calculates the "great-circle" distance between two points (i.e. "as the crow flies"). Performs well even at short distances. 
$$d(i,j) = 2r\text{arcsin}(\text{sin}^2(\frac{\text{lat}_j-\text{lat}_i}{2}) + \text{cos}(\text{lat}_i)\text{cos}(\text{lat}_j)\text{sin}^2(\frac{\text{long}_j - \text{long}_i}{2}))$$
    b.  [Spherical Law of Cosines:](http://www.movable-type.co.uk/scripts/latlong.html) gives reliable estimates down to a few meters. Formula is simpler than Haversine Formula. 
$$d(i,j) = r \cdot \text{arccos}(\text{sin}(\text{lat}_i)\text{sin}(\text{lat}_j) + \text{cos}(\text{lat}_i)\text{cos}(\text{lat}_j)\text{cos}(\text{long}_j-\text{long}_j))$$
    c.  Where $r = 3959$, the radius of the Earth in miles. If we were interested in calculating distances in kilometers or some other unit, we would replace $r$ with the radius of the Earth in the desired output units.   
  2.  Distance (incorporating transportation networks to measure true route between two points)
  3.  Travel time (incorporating transportation networks to measure true route between two points)
  
Which of these measures you choose might depend on the nature or scope of your project, but since our data set is reasonably manageable I thought we could experiment with all three. 

##Using the `geosphere` package for Haversine Formula and Spherical Law of Cosines

The `geosphere` package contains functions `disthaversine()` and `distcosine()` to calculate the distance between either two vectors of latitude/ longitude coordinates or matrices with two columns for latitude/ longitude, using the Haversine Formula and Spherical Law of Cosines, respectively. 

```{r, tidy=TRUE}
#Recall how to access coordinates from a spatialpointsdataframe
head(AustinPublicArt.gc[,9:10])

#Input: lat/long coordinates of the 1st and 2nd works of art
#Output: distance between 1st and 2nd works of art (in miles)
distHaversine(AustinPublicArt.gc[1,9:10],  AustinPublicArt.gc[2,9:10], r = 3959)

#Input: lat/long coordinates of the 1st vs. all other works of art
#Output: distances between the 1st and all other works of art (in miles)
distHaversine(AustinPublicArt.gc[1,9:10], AustinPublicArt.gc[-1,9:10], r = 3959)
```

We can take the mean of this last command's output to calculate the average distance to other works of art for the first piece in the dataset (our proposed gauge of accessability). To find the average distance from each of the `r nrow(AustinPublicArt)` works of art, we can construct a distance matrix using the `distm()` function in `geosphere()` and then take the `colmeans()` of that matrix.

```{r, tidy=TRUE}
DistanceMatrix.Haversine <- distm(AustinPublicArt.gc[,9:10], AustinPublicArt.gc[,9:10], distHaversine)*0.000621371 #convert km --> mi by multipying * 0.000621371
DistanceMatrix.Haversine[1,] #Look at the first row of distance matrix output
diag(DistanceMatrix.Haversine) <- NA #remove 0 values for distance b/t a work of art and itself
AustinPublicArt.gc$avg_distance_haversine <- colMeans(DistanceMatrix.Haversine, na.rm=TRUE)
```

Just for fun, let's look at the histogram of average distance from each piece of art to the rest of the collection. 

```{r, echo=FALSE, fig.align="center", message=FALSE, tidy=TRUE}
qplot(AustinPublicArt.gc$avg_distance_haversine, xlab="miles", ylab="pieces of art", main="Average Distance between Artwork and the Remainder of Collection", geom="histogram", fill=I("orange"))
```

The inputs for the `distCosine()` function are the same, and the calculations will be similar for most reasonable distances. We could repeat all of the previous steps with the Spherical Law of Cosines simply by substituting `distCosine()` in for `distHaversine()`. 

```{r, tidy=TRUE}
#Input: lat/long coordinates of the 1st and 2nd works of art
#Output: distance between 1st and 2nd works of art (in miles)
distCosine(AustinPublicArt.gc[1,9:10],  AustinPublicArt.gc[2,9:10], r = 3959)
```

##Using the `gmapsdistance` package for travel times and distances based on road networks

Depending on the focus of the spatial analysis, it may be more prudent to incorporate networks of roads and sidewalks to calculate distances or travel times directly rather than "as the crow flies" (the shortest possible distance between two points). With the gmapsdistance package this can be accomplished using the Google Maps API directly in R to utilize Google Maps data to calculate travel distances (output in meters) and times (output in seconds) by a variety of modes (walking, biking, driving, or taking public transportation), at different times of day, and under different traffic conditions ("pessimistic", "optimistic", "best guess"). 

The inputs for the `gmapsdistance()` function are a little different. With this command, the origin and destination can either be named places or pairs of latitude and longitude coordinates of the forms `CITY+STATE` or `LATITUDE+LONGITUDE`, respectively. To simplify the code, I elected to use the `paste()` function to input coordinates in this format without altering the original spatial polygons data frame. 

```{r, tidy=TRUE}
AustinPublicArt.gc[1,9:10] #access one pair of lat/long coordinates from the spatial polygons data frame
paste(AustinPublicArt.gc[1,10],AustinPublicArt.gc[1,9], sep="+") #reformat this pair to be of the form "lat+long"
```

Given what we've observed from the preliminary map of the dispersal of public art in Austin, it could be insightful to incorporate sidewalk and road networks to calculate distances for pedestrians rather than cars (as street art could be more easily enjoyed on foot). Let's begin by calculating the walking distance between the first and second works of art in the dataset. 

```{r google maps distance, tidy=TRUE}
gmapsdistance(origin = paste(AustinPublicArt.gc[1,10],AustinPublicArt.gc[1,9], sep="+"), destination = paste(AustinPublicArt.gc[2,10],AustinPublicArt.gc[2,9], sep="+"), mode = "walking")
```

From this output, we see that there are two accesible objects: the `Time` and `Distance` between the `origin` and `destination`. Recall that `Time` is output in seconds and `Distance` in meters, so you will likely need to convert these. The following are additional function inputs that could be used to get more precise estimates of these: 
  - `avoid`: when the `mode = "driving"`, we can specify routes avoiding `"tolls"`, `"highways"`, `"ferries"`, and `"indoor"` segments. 
  - `traffic model`: when the `mode = "driving"`, we can specify the anticipated level of traffic using `"optimistic"`, `"pessimistic"`, or `"best_guess"`. 

## Example 2: [Places of Worship](http://opendata.dc.gov/datasets/b134de8f8eaa49499715a38ba97673c8_5)

*Background:* The dataset contains locations and attributes of Places of Worship, created as part of the DC Geographic Information System (DC GIS) for the D.C. Office of the Chief Technology Officer (OCTO) and participating D.C. government agencies. Information provided by various sources identified Places of Worship such as churches and faith based organizations. DC GIS staff geo-processed this data from a variety of sources.

```{r, tidy=TRUE}
PlacesOfWorship.df <- read.csv("https://query.data.world/s/enqqu329pdhaq78usawjway3a", header=TRUE, stringsAsFactors=FALSE)
head(PlacesOfWorship.df)
```

We begin by repeating the subset process to get our base layer shape for the state of Washington, D.C.. We can find this outline by downloading the [States (and equivalent) shapefile](https://www.census.gov/cgi-bin/geo/shapefiles/index.php) from the United States Census Bureau. Helpful tidbit: the [FIPS code](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standard_state_code) for the District of Columbia is 11. By using the FIPS code instead of trying to use a string "District of Columbia" we can be careful about capitalizations or abbreviations. 

```{r, tidy=TRUE}
USStates.shp <- readOGR(path.expand("tl_2016_us_state/tl_2016_us_state.shp"), "tl_2016_us_state")
WashingtonDC.shp <- subset(USStates.shp, USStates.shp@data$STATEFP == 11)
```

In `ggplot2`, we can plot it using either of the following commands. The first repeats the steps from the Austin, Texas example to `fortify()` the shapefile and then plot the latitude and longitude coordinates as a polygon using `geom_polygon()`. The second is new! Within `ggplot2`, there is a command called `map_data()` that allows you to directly read in common map shapes. We couldn't use this before because we were focusing on a small area, but now would be a fun time to explore it to plot Washington D.C.! 

```{r, fig.align = "center", message=FALSE, warning=FALSE, tidy=TRUE}
WashingtonDC.df <- tidy(WashingtonDC.shp, region = "NAME")
WashingtonDC.gg <- subset(map_data("state"), map_data("state")$region == "district of columbia")
DCShapefile <- ggplot() + geom_polygon(data = WashingtonDC.df, aes(x = long, y = lat, group = group), fill = "white", col="black") + coord_equal(ratio = 1) + theme(axis.text.x=element_blank(), axis.text.y=element_blank(), axis.ticks=element_blank(),axis.title.x=element_blank(),axis.title.y=element_blank()) + north(WashingtonDC.df, location = "topleft") + scalebar(WashingtonDC.df, location = "bottomright", dist = 3, dd2km = TRUE, model = 'WGS84', st.size=2.5)

DCMapData <- ggplot() + geom_polygon(data = WashingtonDC.gg, aes(x = long, y = lat, group = region), fill="white", col="black") + coord_equal(ratio = 1) + theme(axis.text.x=element_blank(), axis.text.y=element_blank(), axis.ticks=element_blank(),axis.title.x=element_blank(),axis.title.y=element_blank())+ north(WashingtonDC.gg, location = "topleft") + scalebar(WashingtonDC.gg, location = "bottomright", dist = 3, dd2km = TRUE, model = 'WGS84', st.size=2.5)
```
```{r, echo=FALSE, fig.align="center"}
library(gridExtra)
grid.arrange(DCShapefile + ggtitle("From US Census"), DCMapData + ggtitle("From ggplot"), ncol=2)
```

Personally, I prefer the precision of the shapefile to that of the `map_data()` plot. But for maps of larger areas (i.e. all of the states) this is definitely a great option!

## Mapping location and attributes together

To illustrate incorporating attributes with spatial data, we will be creating a map of the places of worship in Washington, D.C. where the religion of each location can be identified from the unique color. To do so, instead of setting the `col` for the `geom_point()` statement we set `col = religion`. I was not a huge fan of the automatic color scheme, so I used `scale_color_manual()` with `values` from the `wesanderson` package color scheme `"Fantastic Fox"`. 

```{r, tidy=TRUE, fig.align="center"}
(PlacesOfWorship.map <- DCShapefile + geom_point(data = PlacesOfWorship.df, aes(x = longitude, y = latitude, col=religion)) + scale_color_manual(values=wes_palette(n=5, name="FantasticFox")))
```

With a majority Christian locations, I elected to create a map with only non-Christian locations to provide more insight. 

```{r, tidy=TRUE, fig.align="center"}
(PlacesOfWorship.map <- DCShapefile + geom_point(data = PlacesOfWorship.df[which(PlacesOfWorship.df$religion != "CHRISTIAN"),], aes(x = longitude, y = latitude, col=religion)) + scale_color_manual(values=wes_palette(n=5, name="FantasticFox")))
```

# Example 3: [Claim-Based Medicare Spending by State](https://data.world/dartmouthatlas/claims-based-medicare-spending-by-state-level)

Claims-based Medicare spending: Price, age, sex and race-adjusted (20% sample for 2003-09: 100% sample for 2010-13). Dartmouth Atlas data have been used for many years to compare utilization and expenditures across hospital referral regions (HRRs). 

```{r, tidy=TRUE}
Medicare <- read.csv("https://query.data.world/s/l6qwxpDqQw5RX8Urdo8G_hKq6P6i5a", header=TRUE, stringsAsFactors = FALSE)
head(Medicare)
```

##Getting colorful with choropleth maps 

A choropleth is a map that shades geographic areas (counties, census blocks, urban areas, etc.) by their relative levels of a given attribute. Across the United States, we can use a choropleth to examine the relative per capita Medicare reimbursements by state. To do so, we will be go back to our US States shapefile and use the `tidy()` function to transform it to a `ggplot`-friendly data frame. 

```{r, tidy=TRUE}
USStates.df <- tidy(USStates.shp, region = "NAME")
ContUSStates.df <- USStates.df[which(USStates.df$id!="Alaska"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="Hawaii"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="United States Virgin Islands"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="Puerto Rico"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="Guam"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="Commonwealth of the Northern Mariana Islands"),]
ContUSStates.df <- ContUSStates.df[which(ContUSStates.df$id!="American Samoa"),]
ContUSStates.df <- ContUSStates.df[is.na(ContUSStates.df$long)==FALSE,]
```

Now that we have state-level attributes (number of Medicare enrollees and total Medicare reimbursements per enrollee) and state-level polygons we need to `merge()` the two data frames. 

```{r, tidy=TRUE}
MedicareStates <- merge(ContUSStates.df, Medicare, by.x = "id", by.y = "StateName", all.x = TRUE)
```

Now we can create a choropleth of the merged data frame `MedicareStates`. 

```{r, tidy=TRUE, fig.align="center"}
(MedicareEnrollees.map <- ggplot() + geom_polygon(data = MedicareStates, aes(x = long, y = lat, group = group, fill=MedicareEnrollees), color="black") + ggtitle("Number of Medicare Enrollees by U.S. State") + coord_equal(ratio = 1) + theme(axis.text.x=element_blank(), axis.text.y=element_blank(), axis.ticks=element_blank(),axis.title.x=element_blank(),axis.title.y=element_blank())+ north(MedicareStates, location = "bottomleft") + scalebar(MedicareStates, location = "bottomright", dist = 500, dd2km = TRUE, model = 'WGS84', st.size=2.5))
```

Now go forth and map! 
