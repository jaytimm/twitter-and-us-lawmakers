``` r
library(tidyverse)
setwd('/home/jtimm/jt_work/GitHub/twitter-and-us-lawmakers/data')
dta <- read.csv('lawmaker-twitter-handles.csv')
```

To VoteView –
-------------

Simple Vote-View build –

``` r
vv_meta <- lapply(c('115', '116'), function(x) {
  Rvoteview::download_metadata(congress = x, 
                               type = 'members', 
                               chamber = 'both')} )  %>%
  data.table::rbindlist() %>%
  distinct() %>%
  filter(chamber != 'President') %>%
  mutate(
    party_name = case_when(party_code == '100' ~ 'Deomcratic Party',
                           party_code == '200' ~ 'Republican Party',
                           party_code == '328' ~ 'Independent')) %>%
  select(bioguide_id, congress:chamber, 
         state_abbrev, district_code, bioname, 
         party_name, born, nominate_dim1)
```

    ## [1] "/tmp/RtmpuNb1S9/HS115_members.csv"
    ## [1] "/tmp/RtmpuNb1S9/HS116_members.csv"
