Twitter handles: US Lawmakers
-----------------------------

*Tweets of Congress* (TOC) handles – Available @
<a href="https://github.com/alexlitel/congresstweets" class="uri">https://github.com/alexlitel/congresstweets</a>
–

-   We do not use any of the tweet sets TOC make available – which
    starts at June 2017, ie, half-way through the first session of
    congress 115 –

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

-   We use the GWU *tweet-set* for congress 115 and for congress 116 up
    until 5/2020, where GWU stops –

-   So, we update our tweet-set via `rtweet` using the same
    twitter-handles that seed GWU tweet-set –

-   GWU accounts are made available by congress and chamber, so we have
    access to this data, which TOC lacks –

-   Final twitter-handle data set, then:: GWU handles, with a TOC join -
    which gets us to bioguide\_id’s

*ADD"* TOC meta/bioguide\_id to GWU twitter-handle table –

``` r
full <- gwu_accounts %>% left_join(tweets_of_congress) %>%
  group_by(screen_name) %>%
  mutate(n = n()) %>%
  ungroup() %>%
  filter(!(n == 2 & handle_type == 'prev_names')) %>%
  select(-n) %>%
  na.omit()
```

*Lastly: re adding information from the VoteView* –

``` r
# Package is designed with two predominant use-cases in mind --
#   (1) Returns & ideology, &
#   (2) Returns & Twitter --
# in other words, align election returns, tweets, and VoteView --
```

Simple Vote-View build –

``` r
vv_meta <- lapply(c('115', '116'), function(x) {
  Rvoteview::download_metadata(congress = x, 
                               type = 'members', 
                               chamber = 'both')} )  %>%
  data.table::rbindlist() %>%
  distinct() %>%
  filter(chamber != 'President') %>%
  mutate(party_name = case_when(party_code == '100' ~ 'Deomcratic Party',
                                party_code == '200' ~ 'Republican Party',
                                party_code == '328' ~ 'Independent')) %>%
  select(bioguide_id, congress:chamber, 
         state_abbrev, district_code, bioname, 
         party_name, born, nominate_dim1)
```

    ## [1] "/tmp/RtmpYnx1Ey/HS115_members.csv"
    ## [1] "/tmp/RtmpYnx1Ey/HS116_members.csv"
