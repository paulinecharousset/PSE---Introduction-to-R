---
title: "Class 1: Introduction to data wrangling with the tidyverse"
output: 
 html_document:
    number_sections: true
    theme: cosmo
    code_folding: hide
---

<style>
  body {background-color:lavender}
</style>

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
require("tidyverse")
require("sf")
require("DT")
iris <- readRDS("Output/iris.rds")
arrd <- readRDS("Output/arrd.rds") %>% select(Arrondissement=COM, P13_POP0014_pc:P13_POP75P_pc) %>% 
  mutate(Arrondissement = substr(Arrondissement, 4, 5)) %>%
  mutate(Arrondissement = paste0("Paris ", Arrondissement))
```

# Today's exercise

We will use the tidyverse to extract and transform sociodemographic data of local areas in Paris. At the end of the class, we will produce maps of the 1000 IRISes in Paris, coloured by sociodemographic variables. For example, we will able to see which areas of Paris have the highest number of qualified professionals, the highest number of immigrants, or the highest number of young people. 

We aim to create two data frames, one by IRIS and the other by arrondissement, that look something like this: 

```{r arrdtab, echo = FALSE, warning = FALSE, message = FALSE, error = FALSE}
datatable(arrd)
```

We then aim to plot this data on maps of Paris. 

# Set-up

There are many different libraries in the R universe for a wide variety of tasks. In this first class, we will be covering methods to import and transform data in R using the `tidyverse`.

![Libraries in the tidyverse](Data/images/tidyverse.png)

## R projects and structuring your code

R projects are good for managing your data and scripts in a particular folder on your computer. Using R-Studio, click File -> New project to create a new R project in a new or existing folder. A good name for a new folder is something like "Class1", which you can save somewhere logical on your computer, such as in a folder called "Introdution_to_R".

Within the folder "Class1", create 3 subfolders, "Data", "Scripts" and "Output". We will save all R code in the folder "Scripts". A key advantage of using R projects is that all paths leading to our input data and output files will be relative to the location of the R project (the folder "Class1").

Create a new R script by clicking File -> New file -> R Script. This should be saved in the folder "Scripts". You can call this script something like "cleaning_paris_data".

## Commenting code

It is always a good idea to comment lines of code. Use `#` at the start of a line in order place a comment or in order to disactivate the line so that it does not run. 

## Installing packages 

Installing packages in R is done with the command `install.packages("package_name")`. To install the tidyverse package, we can thus type `install.packages("tidyverse")`. Once the package has been installed, it needs to be loaded every session using the code `library("package_name")`. Thus, the first lines of our script will be:

```{r installtidyverse, warning=FALSE, message=FALSE, error=FALSE}
### Installing and loading packages
# install.packages("tidyverse")
library("tidyverse")
```

A useful piece of code to install packages only if they are not already installed, then load them, is:

```{r installifnotinstalled, warning=FALSE,message=FALSE,error=FALSE}
### installs if necessary and loads tidyverse and sf, another package which we will be using today
list.of.packages <- c("tidyverse", "sf")
new.packages <- list.of.packages[!(list.of.packages %in% installed.packages()[,"Package"])]
if(length(new.packages)) install.packages(new.packages, repos = "http://cran.us.r-project.org")

invisible(lapply(list.of.packages, library, character.only = TRUE))
```

# Reading data 

All the data for today's exercise can be downloaded from [here](https://drive.google.com/open?id=115sQqs8LVqQmsKYRORTIAu3KZvUHczh0). Although I provide sources, you do not need to download the data from a given source. 

For this exercise, we use the French population data at the IRIS level (50k units in metropolitan France) that can be downloaded from the [Insee website](https://www.insee.fr/fr/statistiques/2386737). The shapefiles for the IRIS can be downloaded from [here](http://professionnels.ign.fr/contoursiris). The shapefiles for the arrondissements in Paris can be downloaded from [here](https://opendata.paris.fr/explore/dataset/arrondissements/).

## Reading delimited data

This data is in the form of an .csv (comma-separated values). This can be read using the function `read_delim` the tidyverse package `readr`. To the left of the `<-` sign is the new R object we wish to define, to the right is how we wish to define it. 

```{r readingxl, warning=FALSE,message=FALSE,error=FALSE}
df <- read_delim(file = "Data/base-ic-evol-struct-pop-2013.csv", delim = ",", col_names = TRUE, skip = 5, locale = locale(encoding = "UTF-8"))
```

The options for the function `read_delim` can be found by typing `?? read_delim` in the console. Here, we just present a few frequently used options. 

Argument      | Description
------------- | -------------
file (required)             | path to file (relative to R project)
delim (required)          | delimiter
col_names (TRUE by default) | TRUE if first line is column names, else FALSE or a vector of column names
skip (0 by default)        | the number of lines to skip at the start
locale | control the regional options, importantly the encoding

We can check that our dataframe `df` is how we want it to be by typing `View(df)` in the console, or by clicking on the data frame in the "Environment" panel. 

![Encoding matters](Data/images/geek_martine-ecrit-en-utf-8.jpg)

## Other types of data

There are other packages inside the tidyverse that can be used to read most other classic types of data, for example: `read_csv`, `read_xls`, `read_dta`, `read_sas`, `read_sav`. These functions work similarly. 

## The data frame in R 

A data frame in R is a standard R object used to store databases. A data frame consists of rows and columns, where the columns contain one of five basic classes of data. 

1. logical (e.g., TRUE, FALSE)
2. integer (e.g., 213, as.integer(3))
3. numeric (real or decimal) (e.g, 2, 2.0, pi)
4. complex (e.g, 1 + 0i, 1 + 4i)
5. character (e.g, "hello", "AA231")

When our data set was imported from .csv, R recognised character and numeric columns. We will later learn how to change column types. 

Each of the columns may be accessed by their name, e.g. `df$IRIS`, or by their number , e.g. `df[,2]`.

# The dplyr pipe function

The pipe function is part of the package `dplyr` in the tidyverse, and is used to simply transform a data frame. A cheat sheet for the `dplyr` package can by found by clicking Help -> Cheatsheets -> Data Transformation with dplyr, or [at this link](https://www.rstudio.org/links/data_transformation_cheat_sheet).

![The pipe function](Data/images/pipe.png)


## Selecting rows and columns: select and subset

We want to select only the columns `IRIS`, `COM`, `TYP_IRIS`, `P13_POP` and the age variables `P13_POP0014` through to `P13_POP75P`. We want to select the rows that denote data from Paris only. To select columns, we use the function `select`. To select rows, we use the function `subset`. 


```{r subsetselect, warning=FALSE,message=FALSE,error=FALSE}
iris <- df %>% 
  subset(DEP=="75") %>%
  select(IRIS, COM, TYP_IRIS, P13_POP, P13_POP0014:P13_POP75P) 
```

The order of these two lines matters, if we select the columns first, then we cannot use the variable `DEP` to subset the variables. It is also possible to deselect variables by putting a minus sign before the variable, e.g. `select(-COM)`. 

### Renaming columns

Columns can be renamed by using `rename(new_name=old_name)`, or by integrating the new names into the select function, e.g. `select(new_name_1=old_name_1, new_name_2=old_name_2)`.

### Note on logical statements in R (and most other languages)

In logical statements, e.g. "if A is equal to B, then apply function Y", we use the following notation. Specifically, we use a double equals sign for 'equal to'. 

Statement      | Meaning
------------- | -------------
`==`           | equal to
`>=`, `<=`         | greater than or equal to, less than or equal to
`>`, `<` | greater than, less than 
`!=`      | not equal to
`&` | and
`|` | or

## Mutating variables

We now wish to convert all the population variables to percentages, and the `TYP_IRIS` variable to a factor. To modify one, many or all columns, we use the functions `mutate`, `mutate_at` or `mutate_all`. 

```{r mutate, warning=FALSE,message=FALSE,error=FALSE}
iris <- df %>% 
  subset(DEP=="75") %>%
  select(IRIS, COM, TYP_IRIS, P13_POP, P13_POP0014:P13_POP75P) %>%
  mutate(TYP_IRIS = as.factor(TYP_IRIS)) %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP))
```

### Conditional mutations

We notice that there are some IRIS for which the population is 0. In these cases, when we divide by 0, we obtain the result `NaN` (not a number). We wish to convert these values to 0. We can use `mutate_if` to only mutate columns satifying a particular condition, and we can use the `ifelse` function to replace `NaN` by `0`. The three arguments of the `ifelse` function are: 

1. Logical statement 
2. Action to take if logical statement is true
3. Action to take if logical statement is false

```{r mutateif, warning=FALSE,message=FALSE,error=FALSE}
iris <- df %>% 
  subset(DEP=="75") %>%
  select(IRIS, COM, TYP_IRIS, P13_POP, P13_POP0014:P13_POP75P) %>%
  mutate(TYP_IRIS = as.factor(TYP_IRIS)) %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP)) %>%
  mutate_if(is.numeric, funs(ifelse(is.nan(.), 0, .)))
```

### Some basic string operations

Here we learn two simple functions for string variables, `substr` and `paste0`. 

Say we wish to convert the column `COM` into a more readable string, e.g. instead of "75114", we wish to write "Paris 14". We use the function `substr` to extract from the 4th to the 5th position of the string, and `paste0` to concatenate strings. 

```{r strings, warning=FALSE,message=FALSE,error=FALSE}
iris <- df %>% 
  subset(DEP=="75") %>%
  select(IRIS, COM, TYP_IRIS, P13_POP, P13_POP0014:P13_POP75P) %>%
  mutate(TYP_IRIS = as.factor(TYP_IRIS)) %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP)) %>%
  mutate_if(is.numeric, funs(ifelse(is.nan(.), 0, .))) %>%
  mutate(COM = substr(COM, 4, 5)) %>%
  mutate(COM = paste0("Paris ", COM))
```

## Grouping and aggregating variables

We now wish to group the IRISes by arrondissement, in order to obtain aggregated statistics of the population by arrondissement. Using the function `group_by`, we can group the variables by `COM`, which indicates the arrondissement. We can use the function `summarise_all`, which works in the same way as `mutate_all`, to aggregate our data by group. After this aggregation, we need to `ungroup` our data frame. 

```{r groups, warning=FALSE,message=FALSE,error=FALSE}
arrd <- iris %>% 
  select(COM, P13_POP, P13_POP0014:P13_POP75P) %>%
  group_by(COM) %>%
  summarise_all(funs(sum(.))) %>%
  ungroup %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP)) %>%
  mutate_if(is.numeric, funs(ifelse(is.nan(.), 0, .)))
```

The final two lines are the same as before. 

## Changing from wide to long, and long to wide

Our data is currently in wide format. To change it from wide to long format, we use the function `gather`, and to change it from long to wide format, we use the function `spread`. 

```{r longwide, warning=FALSE,message=FALSE,error=FALSE}
long <- arrd %>%
  gather(key = population_variable, value = value, -COM)

wide <- long %>%
  spread(key = population_variable, value = value)
```

## Writing data 

We can write data in .csv format using `write_csv`. We can also use .rds format (r dataset) in order to preserve the data frame attributes, such as which variables are factor variables. 

```{r savingdata, warning=FALSE,message=FALSE,error=FALSE}
iris <- df %>% 
  subset(DEP=="75") %>%
  select(IRIS, COM, TYP_IRIS, P13_POP, P13_POP0014:P13_POP75P) %>%
  mutate(TYP_IRIS = as.factor(TYP_IRIS)) %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP)) %>%
  mutate_if(is.numeric, funs(ifelse(is.nan(.), 0, .))) %>%
  mutate(name_arrd = substr(COM, 4, 5)) %>%
  mutate(name_arrd = paste0("Paris ", name_arrd)) %>%
  write_csv("Output/iris.csv") %>%
  write_rds("Output/iris.rds") 

arrd <- iris %>% 
  select(COM, P13_POP, P13_POP0014:P13_POP75P) %>%
  group_by(COM) %>%
  summarise_all(funs(sum(.))) %>%
  ungroup %>%
  mutate_at(vars(P13_POP0014:P13_POP75P), funs(pc=./P13_POP)) %>%
  mutate_if(is.numeric, funs(ifelse(is.nan(.), 0, .))) %>%
  write_csv("Output/arrd.csv") %>%
  write_rds("Output/arrd.rds") 

```

## Joins

There are four key types of joins.

Function      | Meaning
------------- | -------------
`left_join(a, b, by="x")`           | Join matching rows from b to a
`right_join(a, b, by="x")`        | Join matching rows from a to b
`inner_join(a, b, by="x")`   | Join data retaining rows in both sets
`full_join(a, b, by="x")`       | Join data retaining all rows

We will apply a join with geographical data, in order to display our variables on a map. 

### Import geographical data

Shapefiles are a common format of geographical data. We can import them using the package `sf`, which is not part of the tidyverse, but follows the same syntax. We select only the variable corresponding to the IRIS code, and call this `IRIS` to match our other data set. We then apply a `right_join` to join our data to the geographical data to the iris data frame that we have created. 

```{r irisshp, warning=FALSE,message=FALSE,error=FALSE}
irisshp <- read_sf(dsn = "Data/iris", layer = "CONTOURS-IRIS") %>%
  select(IRIS=CODE_IRIS) %>%
  right_join(iris, by="IRIS")
```

### Plot data

In the next class, we will plot data in a much nicer way using `ggplot2`. However, for now, we will simply use the `plot` function. 

We wish to plot a demography variable, such as the percentage of people over 75 years old, on a map of Paris. We select only the variable of interest then use the function `plot`.

```{r join, warning=FALSE,message=FALSE,error=FALSE}
iristoplot <- irisshp %>%
  # mutate(P13_POP75P_pc=ifelse(TYP_IRIS=="H", P13_POP75P_pc, NA)) %>%  ### optional line to exclude IRISes with no or few inhabitants
  select(P13_POP75P_pc) 

plot(iristoplot)
```

In order to save the plots, use the following code. 

```{r plotssave, warning=FALSE,message=FALSE,error=FALSE}
### to save plot use these two lines
# dev.copy(pdf, 'Output/age.pdf')
# dev.off()
```

The same plot by arrondissement is given by the following code.

```{r arrd, warning=FALSE,message=FALSE,error=FALSE}
arrdshp <- read_sf(dsn = "Data/arrondissements", layer = "arrondissements") %>%
  select(COM=c_arinsee) %>%
  mutate(COM=as.character(COM)) %>%
  left_join(arrd, by="COM") %>%
  select(P13_POP75P_pc)

plot(arrdshp)
```

# Exercises

1. Create a map of Paris by arrondissement with the percentage of immigrants 
2. Create a map of the percentage of qualified professionals (cadre) in the French city of your choice (C13_POP15P_CS3)

Upload your script [here](https://script.google.com/macros/s/AKfycbzsOfnH_T3lSWFzmsp8VUO0oCa7DLdhiSxB8oWi8zYQpmMl0YBn/exec)
