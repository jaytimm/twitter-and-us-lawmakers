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

-   The list also accounts for any changes in Twitter handles.

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

GWU breaks down lawmaker handles by chamber & congress, which the TOC
does not. So, we extract these details from GWU, and combine the two
data sets (via Twitter handle).

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

TOC + GWU Twitter list
----------------------

A small sample of the new Twitter handle data set is presented below.

``` r
handles <- gwu_accounts %>% 
  left_join(tweets_of_congress) %>%
  group_by(screen_name) %>%
  mutate(n = n()) %>%
  ungroup() %>%
  filter(!(n == 2 & handle_type == 'prev_names')) %>%
  select(-n) %>%
  na.omit()
```

|  congress| chamber | screen\_name   | bioguide\_id | member             | account\_type | handle\_type |
|---------:|:--------|:---------------|:-------------|:-------------------|:--------------|:-------------|
|       115| House   | KYCOMER        | C001108      | James Comer        | campaign      | prev\_names  |
|       115| House   | REPJACKYROSEN  | R000608      | Jacky Rosen        | office        | prev\_names  |
|       115| House   | REPESPAILLAT   | E000297      | Adriano Espaillat  | office        | screen\_name |
|       115| House   | REPTREY        | H001074      | Trey Hollingsworth | office        | screen\_name |
|       115| House   | REPDWIGHTEVANS | E000296      | Dwight Evans       | office        | screen\_name |

VoteView & lawmaker information
-------------------------------

The `Rvoteview` package makes accessible a host of historical
legislative data, including over 200 years of roll call data and
DW-NOMINATE ideal point estimates. The package additionally makes
available a uniform set of details about each legislator to hold office
in the history of the US congress, including party affiliation, DOB,
ideology scores, and state/district information.

``` r
vv_meta <- lapply(c('115', '116'), function(x) {
  Rvoteview::download_metadata(congress = x, 
                               type = 'members', 
                               chamber = 'both')} )  %>%
  data.table::rbindlist() %>%
  distinct() %>%
  filter(chamber != 'President') %>%
  mutate(
    party_name = case_when(party_code == '100' ~ 'Democratic Party',
                           party_code == '200' ~ 'Republican Party',
                           party_code == '328' ~ 'Independent')) %>%
  select(bioguide_id, congress:chamber, 
         state_abbrev, district_code, bioname, 
         party_name, born) # nominate_dim1
```

    ## [1] "/tmp/RtmpeCJwAq/HS115_members.csv"
    ## [1] "/tmp/RtmpeCJwAq/HS116_members.csv"

**Via the Biodguide identifier**, we can easily add these details to our
GWU/TOC Twitter list.

``` r
full <- handles %>%
  left_join(vv_meta)
```

    ## Joining, by = c("congress", "chamber", "bioguide_id")

``` r
full %>% head()
```

    ##   congress chamber     screen_name bioguide_id             member account_type
    ## 1      115   House         KYCOMER     C001108        James Comer     campaign
    ## 2      115   House   REPJACKYROSEN     R000608        Jacky Rosen       office
    ## 3      115   House    REPESPAILLAT     E000297  Adriano Espaillat       office
    ## 4      115   House         REPTREY     H001074 Trey Hollingsworth       office
    ## 5      115   House  REPDWIGHTEVANS     E000296       Dwight Evans       office
    ## 6      115   House ROGERMARSHALLMD     M001198     Roger Marshall     campaign
    ##   handle_type state_abbrev district_code                          bioname
    ## 1  prev_names           KY             1                     COMER, James
    ## 2  prev_names           NV             3            ROSEN, Jacklyn Sheryl
    ## 3 screen_name           NY            13            ESPAILLAT, Adriano J.
    ## 4 screen_name           IN             9 HOLLINGSWORTH, Joseph Albert III
    ## 5 screen_name           PA             2                    EVANS, Dwight
    ## 6 screen_name           KS             1            MARSHALL, Roger Wayne
    ##         party_name born
    ## 1 Republican Party 1972
    ## 2 Democratic Party 1957
    ## 3 Democratic Party 1954
    ## 4 Republican Party 1983
    ## 5 Democratic Party 1954
    ## 6 Republican Party 1960
