---
title: "Salinity Beeswarm Plots"
author: "Kim Cressman"
date: "2018-05-24"
output: 
    html_document:
        keep_md: true
        toc: no
---




It's been a while since we've posted a Plot of the Month here! You all know how things can get. I still like playing with data though, and these plots were inspired by #TidyTuesday on Twitter. (Those of you looking for more frequent "assignments" to practice your R skills should check it out.)  


So what I'm doing here is making a _beeswarm plot_. It's really cool - like a histogram, but way more fun, because you get interesting shapes. There's a point on the graph for every measurement. Where the blob is wider, there are more readings of that particular value. Thomas Mock made one for this week's #TidyTuesday, and it seemed like a perfect way to visualize salinity data. So here we go....


Open up some libraries:


```r
library(SWMPr)  # for easy import and QC of SWMP data
library(tidyverse)  # for lots of data manipulation functions
library(ggbeeswarm)  # for cool plotting
library(knitr)  # for pretty tables
```


# Bring in the data  

Every so often, I do a zip download of all Grand Bay's data from the CDMO's Advanced Query System: http://cdmo.baruch.sc.edu/aqs/ . I keep the up-to-date yearly files in a folder called "Data-latest" so I can just pull them from anywhere else that I'm working in R.  

1.  I'm using `SWMPr::import_local` to pull in and qaqc my data files.  
    +  Remember that you can import all files from a site by only specifying the site name, without a year. I only care about 2017 here, but if I wanted all the data, I'd just say `import_local(datapath, "gndblwq")`.  
    +  Also remember that the default behavior for the `qaqc()` function is to only keep data that's flagged 0. I like to keep suspect (1) and corrected (5) data too, so I've had to specify all three flags using `qaqc_keep()`. 
2.  The `select` function from `dplyr` lets me pick and choose by column name. I'm using a "-" in front of column names that represent data we don't have: select everything MINUS these columns. If I only wanted to keep a column or two, it would read `select(datetimestamp, sal)` and be a smaller data frame.  
3.  The pipe function, `%>%`, lets me do multiple things at once. I think of it like _"and then"_ - so these lines of code say, "import this data file, _and then_ run the qaqc step on it, _and then_ remove the columns that I don't want." 


```r
# look for data here
datapath <- "C:/Users/kimberly.cressman/Desktop/Main Docs/Data-latest"

# import data
dat_bl <- import_local(datapath, "gndblwq2017") %>%
    qaqc(qaqc_keep = c(0, 1, 5)) %>%
    select(-level, -clevel, -chlfluor)

dat_bh <- import_local(datapath, "gndbhwq2017") %>%
    qaqc(qaqc_keep = c(0, 1, 5)) %>%
    select(-level, -clevel, -chlfluor)

dat_bc <- import_local(datapath, "gndbcwq2017") %>%
    qaqc(qaqc_keep = c(0, 1, 5)) %>%
    select(-level, -clevel, -chlfluor)

dat_pc <- import_local(datapath, "gndpcwq2017") %>%
    qaqc(qaqc_keep = c(0, 1, 5)) %>%
    select(-level, -clevel, -chlfluor)
```

<br>

Next, I want to get all of the salinities into a single data frame, with the site name as the column name, so I can easily put them all on a plot and color by site. Surely there's a better way, but here's what I've gotten to work.

I'm reading the code in my head like this: "bl_sal will be the end result. Take dat_bl, _and then:_ select the datetimestamp and sal columns, _and then:_ rename the sal column to be BL." See what I mean about the pipe letting you string a bunch of commands together easily?

And I'm just doing that for each site.


```r
bl_sal <- dat_bl %>%
    select(datetimestamp, sal) %>%
    rename(BL = sal)

bh_sal <- dat_bh %>%
    select(datetimestamp, sal) %>%
    rename(BH = sal)

bc_sal <- dat_bc %>%
    select(datetimestamp, sal) %>%
    rename(BC = sal)

pc_sal <- dat_pc %>%
    select(datetimestamp, sal) %>%
    rename(PC = sal)
```

<br>

Now I'm going to glue them all together, adding one at a time through dplyr's left_join function. Read this as: "all_sal will be the output. Take bl_sal, _and then_ left_join bh_sal to it; _and then_ left_join bc_sal to that; _and then_ left_join pc_sal."


```r
all_sal <- bl_sal %>%
    left_join(bh_sal) %>%
    left_join(bc_sal) %>%
    left_join(pc_sal)
```

<br>

We'll look at the head of the data frame to make sure it worked. I got a little worried when I saw all NAs at the top of BL's list, but then I remembered the sonde's batteries died during its deployment, which had started in December 2016. So this isn't actually a problem with anything I've done in R.


```r
kable(head(all_sal, 10), align = "c", caption = "first 10 salinity readings of 2017 for each site")
```



Table: first 10 salinity readings of 2017 for each site

    datetimestamp       BL     BH      BC      PC  
---------------------  ----  ------  ------  ------
 2017-01-01 00:00:00    NA    21.8    26.1    27.3 
 2017-01-01 00:15:00    NA    21.2    26.9    27.3 
 2017-01-01 00:30:00    NA    20.1    26.4    27.4 
 2017-01-01 00:45:00    NA    20.6    26.4    27.5 
 2017-01-01 01:00:00    NA    20.4    26.7    27.4 
 2017-01-01 01:15:00    NA    20.5    23.4    27.3 
 2017-01-01 01:30:00    NA    20.6    22.9    27.3 
 2017-01-01 01:45:00    NA    20.0    23.7    27.2 
 2017-01-01 02:00:00    NA    19.7    21.4    27.1 
 2017-01-01 02:15:00    NA    14.9    21.4    27.0 

<br> 

Now for a little more reshaping; I forgot that for easy grouping in ggplot, we need data in a long format. This is pretty easy using the `dplyr` package.


```r
all_sal_long <- all_sal %>%
    gather(key = "site", value = "sal", -datetimestamp)

kable(head(all_sal_long), align = "c")
```

    datetimestamp       site    sal 
---------------------  ------  -----
 2017-01-01 00:00:00     BL     NA  
 2017-01-01 00:15:00     BL     NA  
 2017-01-01 00:30:00     BL     NA  
 2017-01-01 00:45:00     BL     NA  
 2017-01-01 01:00:00     BL     NA  
 2017-01-01 01:15:00     BL     NA  

```r
kable(tail(all_sal_long), align = "c")
```

             datetimestamp       site    sal  
-------  ---------------------  ------  ------
140155    2017-12-31 22:30:00     PC     25.4 
140156    2017-12-31 22:45:00     PC     25.4 
140157    2017-12-31 23:00:00     PC     25.4 
140158    2017-12-31 23:15:00     PC     25.3 
140159    2017-12-31 23:30:00     PC     25.2 
140160    2017-12-31 23:45:00     PC     25.2 

<br>

# Make the plot  

Finally! 

I've made this plot using the `geom_quasirandom` function from the `ggbeeswarm` package. This package plays nicely with `ggplot2`, which is my favorite package for making complicated graphs.  

You might notice I didn't specifically load `ggplot2` at the beginning of this; rather I used `library(tidyverse)` - this is a group of packages, including `ggplot2` and `dplyr`, that have useful functions for working with data. I've grown to like loading them all rather than figuring out specifically which ones I'll be using in any given script.  


__ANYWAY. Here's the plot!__



```r
ggplot(all_sal_long) +
    geom_quasirandom(aes(x = site, y = sal, color = site), 
                     na.rm = TRUE, # ignore missing data
                     show.legend = FALSE) +  # don't need a legend for colors by site
    labs(title = "2017 salinity distribution by site", 
         x = "site", 
         y = "salinity (psu)") +
    theme_bw()
```

![](salinity_beeswarm_files/figure-html/unnamed-chunk-7-1.png)<!-- -->



We've got 35,000+ measurements going into each of these, so the graph looks pretty solid. If you had a smaller dataset, you could play around with the size and shape of the points to get a different look.  

<br>

For an example, I'll pull out data from April 2017. The `filter` function comes from the `dplyr` package, and lets you specify conditions for rows you want to keep. Remember, if you need more detail on a function, you can type `?filter` (or whatever the function is) into the console, and it will pull up the help file.


```r
april_sal <- all_sal_long %>%
    filter(datetimestamp >= "2017-04-01 0:00",
           datetimestamp <= "2017-04-30 23:45")
```

<br>


Here's the plot with default settings. It's still a lot of data.


```r
ggplot(april_sal) +
    geom_quasirandom(aes(x = site, y = sal, color = site), 
                     na.rm = TRUE, # ignore missing data
                     show.legend = FALSE) +  # don't need a legend for colors by site
    labs(title = "April 2017 salinity distribution by site", 
         x = "site", 
         y = "salinity (psu)") +
    theme_bw()
```

![](salinity_beeswarm_files/figure-html/unnamed-chunk-9-1.png)<!-- -->


<br>

Here it is again, specifying `size = 0.5`. This produces smaller points, and a bit of whitespace:


```r
ggplot(april_sal) +
    geom_quasirandom(aes(x = site, y = sal, color = site), 
                     size = 0.5,
                     na.rm = TRUE, # ignore missing data
                     show.legend = FALSE) +  # don't need a legend for colors by site
    labs(title = "April 2017 salinity distribution by site", 
         x = "site", 
         y = "salinity (psu)") +
    theme_bw()
```

![](salinity_beeswarm_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

<br>

Here, I'll go up to a bigger size, and specify a different shape (see  http://www.sthda.com/english/wiki/ggplot2-point-shapes for shapes to choose from):


```r
ggplot(april_sal) +
    geom_quasirandom(aes(x = site, y = sal, color = site), 
                     size = 2,
                     shape = 0,
                     na.rm = TRUE, # ignore missing data
                     show.legend = FALSE) +  # don't need a legend for colors by site
    labs(title = "April 2017 salinity distribution by site", 
         x = "site", 
         y = "salinity (psu)") +
    theme_bw()
```

![](salinity_beeswarm_files/figure-html/unnamed-chunk-11-1.png)<!-- -->


I kind of want to make one of these for every year now. And then loop through them in an animation. There are ways. But that's best left for some time in the future when I have a bit more time to play.   

I hope you've found this interesting and helpful. If you recreate it with data from your Reserve, please post the results here!
