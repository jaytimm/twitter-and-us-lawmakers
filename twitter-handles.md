Twitter handles for US lawmakers
================================

2020-09-22

[Twitter handle data
set](https://github.com/jaytimm/twitter-and-us-lawmakers/blob/master/data/lawmaker-twitter-handles.csv)

Tweets of Congress
------------------

The [Tweets of Congress
(TOC)](https://github.com/alexlitel/congresstweets) project makes
available a fantastic list of Twitter handles for US lawmakers. Some
notes on its composition/utility:

-   Lawmakers from both chambers & both congresses (115 + 116) are
    included; these distinctions, however, are not made in the data set.

-   While not utilized here, the TOC handle list includes caucus-,
    committee- & party-related handles.

-   The list includes handles for both campaign & office accounts (the
    latter, importantly, funded by taxpayers). This distinction is
    generally not addressed from a methods perspective and, worse, the
    two are often confounded.

-   The data set identifies each lawmaker by their Bioguide ID – a
    fairly common legislator identifier – which allows us to cross
    Twitter details to other data sets without having to make manual
    edits.

-   The list also accounts for screen name changes.

------------------------------------------------------------------------

The json file is available on Git Hub
[here](https://github.com/alexlitel/congresstweets-automator/blob/master/data/historical-users-filtered.json).
The code below details the json extraction process.

``` r
setwd(ldir)
toc_accounts <- 
  jsonlite::fromJSON('historical-users-filtered.json') %>% 
  filter(type == 'member')

ids <- data.frame(member = toc_accounts$name,
                  bioguide_id = toc_accounts$id$bioguide)

names(toc_accounts$accounts) <- toc_accounts$name 

tweets_of_congress <- toc_accounts$accounts %>% 
  bind_rows(.id = 'member') %>%
  
  select(member, account_type, screen_name, prev_names) %>% 
  mutate(screen_name = toupper(screen_name),
         prev_names = unlist(toupper(prev_names))) %>%
  
  pivot_longer(screen_name:prev_names,
               names_to = "handle_type", 
               values_to = "screen_name") %>%
  filter(screen_name != 'NULL')  %>%
  left_join(ids) %>%
  select(bioguide_id, member, account_type, handle_type, screen_name) 
```

GWU Twitter handles
-------------------

The GWU Twitter handle list is the source of handles for our Twitter
corpus. It also breaks down lawmaker handles by chamber & congress (by
virtue of having separate meta data for each), which the TOC does not.
So, we extract these details from GWU, and combine the two data sets
(via Twitter handle).

``` r
setwd(ldir)
gfiles <- list.files(path = ldir, 
                     pattern = "csv", 
                     recursive = TRUE) 

cs <- gsub('(^.*congress)(...)(.*$)', '\\2', gfiles)
ss <- stringr::str_to_title(
  gsub('(^.*[0-9]-)(.*)(-accounts.*$)', '\\2', gfiles)
  )

gwu_accounts <- lapply(1:length(gfiles), function(x) {
  read.csv(gfiles[x]) %>% mutate(congress = cs[x],
                                 chamber = as.character(ss[x])) 
  } ) %>% 
  data.table::rbindlist() %>%
  mutate(screen_name = toupper(Token)) %>%
  select(congress, chamber, screen_name)
```

Full Twitter list
-----------------

A small sample of the new Twitter handle data set is presented below.

``` r
full <- gwu_accounts %>% 
  left_join(tweets_of_congress) %>%
  group_by(screen_name) %>%
  mutate(n = n()) %>%
  ungroup() %>%
  filter(!(n == 2 & handle_type == 'prev_names')) %>%
  select(-n) %>%
  na.omit()
```

|  congress| chamber | screen\_name     | bioguide\_id | member             | account\_type | handle\_type |
|---------:|:--------|:-----------------|:-------------|:-------------------|:--------------|:-------------|
|       115| House   | KYCOMER          | C001108      | James Comer        | campaign      | prev\_names  |
|       115| House   | REPJACKYROSEN    | R000608      | Jacky Rosen        | office        | prev\_names  |
|       115| House   | REPESPAILLAT     | E000297      | Adriano Espaillat  | office        | screen\_name |
|       115| House   | REPTREY          | H001074      | Trey Hollingsworth | office        | screen\_name |
|       115| House   | REPDWIGHTEVANS   | E000296      | Dwight Evans       | office        | screen\_name |
|       115| House   | ROGERMARSHALLMD  | M001198      | Roger Marshall     | campaign      | screen\_name |
|       115| House   | USREPLONG        | L000576      | Billy Long         | office        | screen\_name |
|       115| House   | REPBYRNE         | B001289      | Bradley Byrne      | office        | screen\_name |
|       115| House   | ROBERT\_ADERHOLT | A000055      | Robert Aderholt    | office        | screen\_name |
|       115| House   | USREPGARYPALMER  | P000609      | Gary Palmer        | office        | screen\_name |
