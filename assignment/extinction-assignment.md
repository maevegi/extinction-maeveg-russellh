### Examining the Evidence for a Sixth Mass Extinction

### Maeve Gilbert, Russell Huang

### Introduction to the Data

In this module, we are attempting to examine and recreate the
conclusions drawn by Ceballos et al. in their 2015 paper which states
that “the average rate of vertebrate species loss over the last century
is up to 100 times higher than the background rate” (Ceballos et
al. 2015). This paper builds off the work of Barnosky et al. (2011),
which examined the fossil record to determine that current extinction
rates are higher than the rate that would be expected by the fossil
record. In order to qualify as a “Mass Extinction”, the rate of
extinction must be higher than that of the background rate and results
in a loss of over 75% of estimated species within a relatively short
geological period. This duration is usually less than 2 million years
(Barnosky et al. 2011).

## Retrieving and Compiling the Data

In order to examine current extinction data, we need to access the IUCN
Red List historical data on species extinctions. This data can be found
in the following file, as our own attempt to call the URL caused the
database to crash. This file allows us to quickly access the data from
an external source (in this case a github page) rather than attempting
to access it from the API every time we ran our code.

    download.file("https://github.com/espm-157/extinction-template/releases/download/data2.0/extinction-data.zip", "extinction-data.zip")
    unzip("extinction-data.zip")

This chunk uses the REST API information from the IUCN website. This
code breaks the URL of the data into pieces that can be read by the
computer. This API call uses a universal base used by all of the
different calls within the API. In this case, we only want to look at
the total number of species in the database, so our endpoint is “species
count”. We are using a public token (query) to access the data.

    base <- "https://apiv3.iucnredlist.org/api/v3"
    endpoint <- "speciescount"
    token <- "9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee"
    url<- (glue("{base}/{endpoint}?token={token}"))

    req <- GET(url)
    x <- content(req)
    x$speciescount

    [1] "150388"

Next, we want to retrieve all pages (0-15) with species data from the
API.

    species_endpoint <- "species"
    page <- paste0("page/", 0:15)
    all_pages<- glue ("{base}/{species_endpoint}/{page}?token={token}")

Because we are retrieving the data for multiple species and pages, we
can apply the map function to perform the function on each page, rather
than doing it individually. This also stops the map function if the
status code returned is above 200, that is to say, it cannot retrieve
the page.

    if(!file.exists("all_species.rds")){
    all_species<-map(all_pages, GET, .progress=TRUE)
    map(all_species, stop_for_status)
    write_rds(all_species, "all_species.rds")
    }

This code will then read out the results from the cached data in the rds
file.

    all_species_rds<-read_rds("all_species.rds")

We then need to extract the response objects from the 16 URLs we
identified earlier.

    all_resp<- map(all_species_rds, content, encoding="UTF-8")

From the response object, we need to extract the relevant information
needed to describe extinct species. This includes the scientific name,
category (ex: threatened, endangered, extinct) and class (ex: mammalia,
amphibia, aves). This is then put into list format. Lastly, we want to
put each of these lists into one table (all\_species\_table).

    sci_name_list<-map(all_resp, \(page) map_chr(page$result, "scientific_name"))|> list_c()

    category<- map(all_resp, \(page) map_chr(page$result, "category"))|> list_c()

    class<-map(all_resp, \(page) map_chr(page$result, "class_name"))|> list_c()

    all_species_table<- tibble(sci_name=sci_name_list, category, class)

Of all of the species we have just compiled into one table, we want to
know which ones have gone extinct.

    extinct_species<- 
      all_species_table|> 
      dplyr::filter(category=="EX")

The data we configured has thus far given us the species scientific
name, category, and class, in this case extinct. However, we want to get
more specific information about these species regarding when they went
extinct. We need to construct another REST API to obtain this data.

    sci_name<- extinct_species$sci_name
    ex_urls<- glue("{base}/species/narrative/{sci_name}?token={token}")|>
      URLencode()
    ex_urls[[1]]

    [1] "https://apiv3.iucnredlist.org/api/v3/species/narrative/Mirogrex%20hulensis?token=9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee"

    GET(ex_urls[[1]])

    Response [https://apiv3.iucnredlist.org/api/v3/species/narrative/Mirogrex%20hulensis?token=9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee]
      Date: 2023-11-30 03:01
      Status: 200
      Content-Type: application/json; charset=utf-8
      Size: 1.65 kB

As before, we can make the retrieval of this data go faster by using the
map function and caching the data so that the function does not need to
run every time.

    if(!file.exists("ex_narrative.rds")){
      ex_narrative<- map(ex_urls, GET)
      map(ex_narrative, stop_for_status, .progress=TRUE)
      write_rds(ex_narrative, "ex_narrative.rds")
    }

    ex_narrative<-read_rds("ex_narrative.rds")

We then need to extract the contents of this .rds file containing
extinction dates. Within the description of each species, the extinction
date can be found either in the population description or the rationale.
The two examples below show how the information appears in text form
containing the extinction date.

    narrative_contents<- map(ex_narrative, content)

    narrative_population<-map(narrative_contents, \(content)     map_chr(content$result[1], "population", .default=""))


    narrative_rationale<-map(narrative_contents, \(content) content$result[[1]]$rationale)

    narrative_population[[1]]

    [1] "This species is now extinct; it was last recorded in 1975."

    narrative_rationale[[1]]

    [1] "The Hula lake and adjacent marshes were drained in the 1950s. Drainage of the lake resulted in the species being restricted to marsh and pond areas in Hula nature reserve. The reserve management switched from using spring water to fishpond water in the species' habitat, resulting in decline and the extinction of the fish within six years of doing so. The species was last recorded in 1975."

However, the dates listed in sections of text can’t be read into a
table. In order to make a table of the data, we need to extract only the
date that the species was last recorded. Once again, we can apply the
map function to extract all four digit integers from each of the
narrative population data and list them.

    last_seen<-narrative_population |> 
      map_chr(str_extract, "\\d{4}")|> 
      as.integer()

Once we have the list of last seen dates, they can then be put into a
tibble with the scientific name for each species.

    gone<-tibble(sci_name, last_seen)|>
      distinct()

Finally, we can join the table “gone”, containing species and extinction
dates, with the earlier “all\_species\_table”, which has species names,
category, and class. From this large table, we can determine the number
of species in each class we are going to use for our final figure. In
this case, we are focusing on mammals, birds, amphibians, reptiles, and
bony fish.

    combined<-dplyr::left_join(all_species_table, gone)

    Joining with `by = join_by(sci_name)`

    total_sp<-combined|>
      filter(class%in% c("MAMMALIA", "AVES", "AMPHIBIA", "REPTILIA", "ACTINOPTERYGII"))|>
      count(class, name="total")

## Examining Extinction Rates by Class

For our final table, we need to perform a number of functions in order
to filter by the correct categories and determine extinction rates. From
our combined table, we can filter by only species that are extinct. We
can then filter by the relevant classes of species. However, there are
many NA values in the data. These are species that have gone extinct but
whose precise extinction date is unknown. We are therefore assigning the
year 2023 to all NA values. The extinctions with known dates are
assigned a century of extinction by filtering by the first two digits of
the year. We then need to join this table with the total\_sp table,
which gives a total count of each species in each class. By grouping the
classes together, we can calculate the percentage of each class that
went extinct in each year by dividing the number of extinctions (n) by
the total number of species in the class (total). However, because we
are basing this off of extinction rates of million species years, we
would need to multiply by one million and then divide by one hundred. We
then need to divide by one hundred again in order to get a percentage of
the total species evaluated by the IUCN. This means that the ultimate
calculation will only be multiplying by 100. From there, we calculated
the cumulative sum of extinctions by class for each century. Using these
numbers, we could calculate the cumulative percentages by dividing the
cumulative number of extinct species in each class by the total number
of species in each class and multiplying by 100.

    cumulative<-combined|>
      filter(category=="EX")|>
      filter(class%in% c("MAMMALIA", "AVES", "AMPHIBIA", "REPTILIA", "ACTINOPTERYGII"))|>
      mutate(last_seen= replace_na(last_seen, 2023), century= str_extract(last_seen, "\\d{2}"))|>
      count(century, class)|>
      left_join(total_sp)|>
      group_by(class)|>
      mutate(percent_extinct=((n/total)*100))|>
      mutate(cumulative_extinct=(cumsum(n))) |>
      mutate(percent_cumulative_extinct=((cumulative_extinct/total)*100)) 

    Joining with `by = join_by(class)`

## Graphing the Results

Once we have the cumulative percentages for each class, we can graph
these results. The x-axis units are centuries and the y axis is the
cumulative percentage of extinct species. The classes are differentiated
by color.

    extinctions_plot<- cumulative|>
      ggplot()+
      geom_line(aes(x=century, y=percent_cumulative_extinct, color=class, group=class))+
      ggtitle("Cumulative Species Extinctions of IUCN Evaluated Species")+
      xlab("Century")+
      ylab("Cumulative Extinctions (%)")
    extinctions_plot

![](extinction-assignment_files/figure-markdown_strict/unnamed-chunk-19-1.png)
Note that due to the methods used to separate extinctions by year,
century refers to the first two numbers of the year. (ex: 14th century
actually refers to the years with 14 as the first two digits, rather
than the 1300s).

## Discussion

It is important to note that in this data set, not all of the years are
included in the total count. The method of filtering dates kept only
those with four digits, which excluded extinction dates that may have
been characterized by a century, for example, “This species went extinct
in the 18th century”. In these cases, the data would not have been
retrieved because it was not a four-digit integer. Additionally, the NA
dates that were replaced with 2023 may inaccurately represent the data
of extinction and weigh the data more heavily towards the present.
Another limitation of the data is that the species assessed by the IUCN
are incomplete and do not reflect all species on Earth, especially those
that have yet to be described by science. There are also instances where
extinction dates may be inaccurate as the last sighting of a species
does not necessarily reflect their absence on Earth.

Due to sampling biases and lack of data, there are major differences in
the amount of data for mammals and birds compared to other classes.
These are usually the two best-described classes due to their visibility
and preference by humans. There is therefore a longer record of the
species, while the data for amphibians, reptiles, and bony fish only
dates back to the 1800s. These classes may also be subject to some of
the other issues with the data as described above.

With these limitations in mind, we can then compare the results from our
table to those generated by Ceballos et al. (2015), shown below. There
are some noticeable differences between the two, mainly that our graph
is divided by class, while Ceballos et al. compared mammal and bird
extinctions on their own compared to that of fish and amphibians (other
vertebrates), compared to the total cumulative extinction rates of all
categories combined (vertebrates). Ceballos et al. used a background
rate of extinction (dashed black line) of 2 species extinctions per
million years, while our graph used a value of one extinction per
million species years. However, both graphs represent increasing numbers
of cumulative extinctions across all classes that exceed the background
rate. It is especially high in mammals and birds.

![](https://espm-157.carlboettiger.info/img/extinctions.jpg)

## References:

-   Ceballos et al. (2015): <http://doi.org/10.1126/sciadv.1400253>
-   Barnosky et al. (2011):(<http://doi.org/10.1038/nature09678>)
