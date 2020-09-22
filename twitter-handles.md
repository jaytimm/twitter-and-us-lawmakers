Twitter handles for US lawmakers
--------------------------------

[Twitter
handles](https://github.com/jaytimm/twitter-and-us-lawmakers/blob/master/data/lawmaker-twitter-handles.csv):

*Tweets of Congress* (TOC) handles – Available @
<a href="https://github.com/alexlitel/congresstweets" class="uri">https://github.com/alexlitel/congresstweets</a>
–

-   TOC contains lawmakers from both chambers for both 115/116
    congresses; BUT these distinctions are not made in the data set.
    Neither chamber nor congress – so, we have to do this manually –

-   TOC also contains caucus-, committee- & party-related handles (in
    TYPE column), which is cool, but which we do not need/use here.

-   A distinction is also made between account types: campaign & office
    – the latter, importantly, funded by taxpayers – which I think is
    important & methodologically useful – a distinction folks do not
    seem to acknowledge or address –

-   TOS contains `bioguid_id` column – which is super-valuable, as it
    gets us back to all other meta/data sets, and ultimately the reason
    why we incorporate this resource at all at this point – !!

-   TOC contains “previous” twitter-handles as well – I do not know
    exactly what this means, but we include such handles, and preserve
    the distinction –

``` r
setwd(ldir)
toc_accounts <- 
  jsonlite::fromJSON('historical-users-filtered.json') %>% 
  filter(type == 'member')


###
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

*GWU* handles – which are included as part of the *Harvard Dataverse* –

-   Per Harvard stamp, this is the most defensible collection of
    twitter-handles.

-   GWU accounts are made available by congress and chamber, so we have
    access to this data, which TOC lacks

-   Final twitter-handle data set, then:: GWU handles, with a TOC join -
    which gets us to bioguide\_id’s

``` r
setwd(ldir)
gfiles <- list.files(path = ldir, 
                     pattern = "csv", 
                     recursive = TRUE) 
cs <- gsub('(^.*congress)(...)(.*$)', '\\2', gfiles)
ss <- stringr::str_to_title(gsub('(^.*[0-9]-)(.*)(-accounts.*$)', '\\2', gfiles))

gwu_accounts <- lapply(1:length(gfiles), function(x) {
  read.csv(gfiles[x]) %>% mutate(congress = cs[x],
                                 chamber = as.character(ss[x])) 
  } ) %>% 
  data.table::rbindlist() %>%
  mutate(screen_name = toupper(Token)) %>%
  select(congress, chamber, screen_name)
```

*ADD"* TOC meta/bioguide\_id to GWU twitter-handle table –

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

``` r
setwd('/home/jtimm/jt_work/GitHub/twitter-and-us-lawmakers/data')
dta <- read.csv('lawmaker-twitter-handles.csv')
```

``` r
dta %>% slice(1:10) %>% knitr::kable()
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
