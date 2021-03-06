Lesson 09: Geospatial Visualization
================

In ggplot2 you can draw maps using polygons to indicate geographical
areas. To draw a polygon, use `geom_polygon` and a dataset of x-y
coordinates:

``` r
ggplot(data = tibble(x = c(1, 2, 3), y = c(1, 3, 1))) + 
  geom_polygon(aes(x = x, y = y)) #geom polygon connects by default the start and end points
```

![](README_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

``` r
ggplot(data = tibble(x = c(1, 2, 3, 1), y = c(1, 3, 1, 1))) + 
  geom_path(aes(x = x, y = y))
```

![](README_files/figure-gfm/unnamed-chunk-1-2.png)<!-- -->

You can think about each geographical entities (e.g., state, country) as
polygons defined by longitudinal/latitudinal coordinates. Thus, to plot
a map, you need a dataset with the lat/long information of the edges of
the geographical areas. The package `maps` (included in tidyverse)
contains some datasets of the most used geographical areas, and you can
use the function `map_data` to retrieve dataset of long-lat coordinates
for a country or regions.

\#\#Choropleth mapping

Maps can be very useful to show the geographical variance of a variable
at a glance. For instance, say we want to show on a map the rates of
`fivethirtyeight::insurance_premiums`. First, to plot a choropleth map
in ggplot, we need to define each region as a polygon. For instance for
a map of the US by state:

``` r
states <- map_data("state")
ggplot(data = states) + 
  geom_polygon(aes(x = long, y = lat, group = group), color = "white") + 
  coord_fixed(1.3)
```

![](README_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

Now we can map an aesthetics in `geom_polygon` to `insurance_premiums`.
For a choropleth map, we want `fill`. In our dataset, each insurance
premiums refers to a specific state, that we will merge with the dataset
of the states coordinates. For simplicity we can assign `state` and
`insurance_premium` to a new object `premiums`:

``` r
premiums <- select(bad_drivers, state, insurance_premiums)
top_n(premiums, 5)
```

    ## # A tibble: 5 x 2
    ##   state                insurance_premiums
    ##   <chr>                             <dbl>
    ## 1 District of Columbia              1274.
    ## 2 Florida                           1160.
    ## 3 Louisiana                         1282.
    ## 4 New Jersey                        1302.
    ## 5 New York                          1234.

The next step is to join the dataset with the polygons shapefiles
`states` with `premiums`. That way, we can map `insurance_premiums` to
the fill of each
state:

``` r
mutate(premiums, state = tolower(state)) %>% #reducing to lower-case to make sure that stateanames match
  right_join(states, by = c('state' = 'region')) %>% 
                      ggplot() +
                      geom_polygon(aes(x = long, y = lat, group = group, fill = insurance_premiums), color = "white") + 
                      #note how we need to group by state!
                      coord_fixed(1.3) +
                      labs(title = 'Insurance premiums in the US') -> mapInsurance 
mapInsurance 
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

Not bad, let’s to some final twicks. For instance, it is unclear from
the legend which type of variable we are plotting. Since premiums are in
dollars, we can add a dollar sign. Also, we override the default labels
to include the highest and lowest value on the color-gradient legend:

``` r
if (!require('scales')) {
  install.packages('scales')
  library('scales')}
#create a vector of 4 "pretty breaks" based on min/max values
bks <- cbreaks(c(min(bad_drivers$insurance_premiums), max(bad_drivers$insurance_premiums)), pretty_breaks(4))
```

``` r
mapInsurance +
 scale_fill_gradient(limits = c(min(bks$breaks), max(bks$breaks)), 
                     breaks = bks$breaks, labels = dollar_format(), 
                     low = 'white', high = '#a50e39', name = 'Premiums') +
  #axes names and anchros are meaningless: we can hide them
  theme(panel.background = element_blank(), 
        axis.title = element_blank(), 
        axis.text = element_blank(), 
        axis.ticks = element_blank(),
        legend.title = element_text(face = 'bold'))
```

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

And of course, we can use facetting too. For instance for plotting both
`perc_speeding` and `perc_speeding`

``` r
select(bad_drivers, state, perc_speeding, perc_alcohol) %>% 
  mutate(state = tolower(state)) %>% 
  right_join(states, by = c('state'='region')) %>% 
  gather(hazardType, perc, perc_speeding, perc_alcohol) %>% 
  mutate(hazardType = case_when(hazardType == 'perc_speeding' ~ "% speeding",
                                T ~ "% alcohol"),
         perc = perc/100) -> dt 
  ggplot(dt) +
  geom_polygon(aes(x = long, y = lat, group = group, fill = perc), color = "white") +
  coord_fixed(1.3) +
  scale_fill_gradient(labels = percent_format(), low = 'white', high = '#a50e39', name = 'Percent') +
  facet_wrap(~ hazardType, ncol = 1,  strip.position = 'left') +
  theme(strip.text.y = element_text(angle=90),         #you can save theme to an object to create a "style"
        panel.background = element_blank(), 
        axis.title = element_blank(), 
        axis.text = element_blank(), 
        axis.ticks = element_blank(),
        legend.title = element_text(face = 'bold'))
```

![](README_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

Similarly to any other geoms, you can map more variables to the other
aes available form `geom_polygon`. For instance, you can map a variable
to the transparency using `alpha`

``` r
#data source: https://uselectionatlas.org/
dtEl <- readxl::read_xlsx(here::here('data/US_presidentialElection.xlsx'), sheet = 2, skip=1) %>% slice(-1)
dtEl <- dtEl %>% select(State, `Clinton, Hillary - Democratic`, `Trump, Donald - Republican`, PVI) %>% 
  gather('candidate', 'share', -State, -PVI)

dtEl <- mutate(dtEl, State = tolower(State))
dtEl <- dtEl %>% 
          mutate(State = case_when(State == 'washington dc' ~ 'district of columbia',
                          T ~ State)) %>% 
          filter(!is.na(State))
map_data('state') %>% anti_join(dtEl, by = c('region' = 'State'))  #double check whether any states has missing value in dtEl
```

    ## [1] long      lat       group     order     region    subregion
    ## <0 rows> (or 0-length row.names)

``` r
map_data('state') %>% left_join(dtEl, by = c('region' = 'State')) %>% 
ggplot() +
  geom_polygon(aes( long, lat, group = group, fill = PVI, alpha = share)) +
  scale_fill_manual(values = c('#0057e7', '#d62d20')) +
  coord_fixed(1.3) +
   theme(panel.background = element_blank(), 
        axis.title = element_blank(), 
        axis.text = element_blank(), 
        axis.ticks = element_blank(),
        legend.title = element_text(face = 'bold')) +
  ggtitle('Map of the US elections')
```

![](README_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

However, the issue with the previous map is that bigger shape somewhat
magnetize the user’s attention. Another way to plot the geographical
locations while making the `premiums` more readable, is to standardize
the states shapes as in the following:

``` r
if (!require('geofacet')) install.packages('geofacet')


bad_drivers %>% left_join(tibble(abb = state.abb,state = state.name)) %>% 
    mutate(abb = case_when(state=='District of Columbia' ~ 'DC', T ~ abb)) %>% 
    mutate(x = 1) %>% # size of the bar being plotted. All bars should be same size to make perfect squares
    mutate(label_y = .5) %>%  # this location of state labels
    mutate(label_x = 1) %>% 
    ggplot() +
    geom_col(mapping=aes(x=x, y = x, fill=insurance_premiums))  +
    facet_geo(~ state, grid = "us_state_grid1") +
    geom_text(aes(x=label_x, y=label_y, label=abb), color='#ffffff', size=8) +
    scale_fill_continuous(low = '#ccdbe5', high= "#114365", guide = guide_colorbar(title = 'Premiums'), labels=dollar_format()) + 
    ggtitle('The US elections') +
    theme_classic() + # theme classic removes many ggplot defaults (grey background, etc)
    theme(#plot.title = element_text(size = 28), # format the plot
          plot.margin = unit(c(1,1,1,1), "cm"),
          legend.text=element_text(size=16),
          legend.title = element_text(size=16),
          axis.title=element_blank(),
          axis.text=element_blank(),
          axis.ticks = element_blank(),
          axis.line = element_blank(),
          strip.text.x = element_blank())
```

![](README_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

Another way of standardizing sizes is to map variables to points, for
instance to their size. Say we had a dataset of premiums by city,
instead of state:

``` r
citypremium <- tibble(city = c('Dallas', 'Baton Rouge', 'San Diego'),
                      state = c('TX', 'LA', 'CA'),
                      premium = c(850, 1100, 780))
citypremium
```

    ## # A tibble: 3 x 3
    ##   city        state premium
    ##   <chr>       <chr>   <dbl>
    ## 1 Dallas      TX        850
    ## 2 Baton Rouge LA       1100
    ## 3 San Diego   CA        780

First, we need the coordinates for city names to draw a point on the map
for each city. The dataset zipcode form the package `zipcode` containts
all US cities:

``` r
library(zipcode)
data("zipcode") #load the dataset
```

We can merge `citypremium` with `zipcode`, and extract any long/lat
coordinates for each city. Note that the join will give multiple
matches, but for this specific example matching a specific zipcode is
useless, and we can just keep a random one using
`distinct`:

``` r
citypremium <- citypremium %>% left_join(zipcode, by = c('city', 'state')) %>% distinct(city, .keep_all = T) 
citypremium
```

    ## # A tibble: 3 x 6
    ##   city        state premium zip   latitude longitude
    ##   <chr>       <chr>   <dbl> <chr>    <dbl>     <dbl>
    ## 1 Dallas      TX        850 75201     32.8     -96.8
    ## 2 Baton Rouge LA       1100 70801     30.4     -91.2
    ## 3 San Diego   CA        780 92101     32.7    -117.

``` r
 ggplot(states) +
  geom_polygon(aes(x = long, y = lat, group = group), color = "white", alpha = .2) + 
  geom_point(data = citypremium, aes(x = longitude, y = latitude, size = premium), color = 'orange', alpha = .8) +
  geom_text(data = citypremium, aes(x = longitude, y = latitude, label = scales::dollar(premium)), color = 'darkblue', show.legend = F) +
  coord_fixed(1.3) +
  labs(title = 'Insurance premiums in the US')  +
    theme(strip.text.y = element_text(angle=90),        
          panel.background = element_blank(), 
          axis.title = element_blank(), 
          axis.text = element_blank(), 
          axis.ticks = element_blank(),
          legend.title = element_text(face = 'bold')) +
  ggtitle('Car insurance premiums')
```

![](README_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

\#\#Exercise

Use the `map_data("world")` and the dataset on [alcohol consumption in
the
world](https://github.com/fivethirtyeight/data/blob/master/alcohol-consumption)
facetting by beverage type: `beer_servings`, `spirit_servings`,
`wine_servings` and `total_litres_of_pure_alcohol`. Some countries in
`map_data('world')` may not have a match in the dataset of alcohol
consumption: feel free to ignore them and leave them as
`NA`.

``` r
dt <- read_csv('https://raw.githubusercontent.com/fivethirtyeight/data/master/alcohol-consumption/drinks.csv')
```
