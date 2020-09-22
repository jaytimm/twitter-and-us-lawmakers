Aligning data sets
==================

2020-09-22

``` r
library(tidyverse)
setwd(ldir)
handles <- read.csv('lawmaker-twitter-handles.csv')
```

VoteView – DW-NOMINATE
----------------------

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
         party_name, born, nominate_dim1)
```

    ## [1] "/tmp/RtmpAIeIVe/HS115_members.csv"
    ## [1] "/tmp/RtmpAIeIVe/HS116_members.csv"

Plus tweets
-----------

``` r
full <- handles %>%
  left_join(vv_meta)

set.seed(99)
full %>%
  filter(chamber == 'House' & account_type == 'office' &
           handle_type == 'screen_name' &
           congress == '116') %>%
  select(screen_name, bioguide_id, bioname, 
         state_abbrev, 
         district_code, party_name) %>%
  sample_n(7) %>%
  knitr::kable()
```

| screen\_name   | bioguide\_id | bioname                | state\_abbrev |  district\_code| party\_name      |
|:---------------|:-------------|:-----------------------|:--------------|---------------:|:-----------------|
| REPROSSSPANO   | S001210      | SPANO, Ross            | FL            |              15| Republican Party |
| REPANNAESHOO   | E000215      | ESHOO, Anna Georges    | CA            |              18| Democratic Party |
| REPHORSFORD    | H001066      | HORSFORD, Steven       | NV            |               4| Democratic Party |
| REPJOHNJOYCE   | J000302      | JOYCE, John            | PA            |              13| Republican Party |
| SPEAKERPELOSI  | P000197      | PELOSI, Nancy          | CA            |              12| Democratic Party |
| REPTIMBURCHETT | B001309      | BURCHETT, Timothy      | TN            |               2| Republican Party |
| REPMCEACHIN    | M001200      | MCEACHIN, Aston Donald | VA            |               4| Democratic Party |

Election returns & census data
------------------------------

Via state abbreviation and district number.
