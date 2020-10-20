Twitter handles for US lawmakers
================================

2020-10-19

[Twitter handle data
set](https://github.com/jaytimm/twitter-and-us-lawmakers/blob/master/data/lawmaker-twitter-handles-voteview.csv)

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

handles <- toc_accounts$accounts %>% 
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

``` r
handles <- gwu_accounts %>% 
  left_join(handles) %>%
  group_by(screen_name) %>%
  mutate(n = n()) %>%
  ungroup() %>%
  filter(!(n == 2 & handle_type == 'prev_names')) %>%
  select(-n) %>%
  mutate(congress = as.integer(congress),
         chamber = ifelse(chamber == 'Senators', 'Senate', chamber)) %>%
  na.omit()
```

``` r
handles %>%
  head() %>%
  knitr::kable()
```

|  congress| chamber | screen\_name    | bioguide\_id | member             | account\_type | handle\_type |
|---------:|:--------|:----------------|:-------------|:-------------------|:--------------|:-------------|
|       115| House   | KYCOMER         | C001108      | James Comer        | campaign      | prev\_names  |
|       115| House   | REPJACKYROSEN   | R000608      | Jacky Rosen        | office        | prev\_names  |
|       115| House   | REPESPAILLAT    | E000297      | Adriano Espaillat  | office        | screen\_name |
|       115| House   | REPTREY         | H001074      | Trey Hollingsworth | office        | screen\_name |
|       115| House   | REPDWIGHTEVANS  | E000296      | Dwight Evans       | office        | screen\_name |
|       115| House   | ROGERMARSHALLMD | M001198      | Roger Marshall     | campaign      | screen\_name |

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
  
  group_by(congress, chamber, state_abbrev) %>%
  mutate(x = length(unique(district_code))) %>%
  ungroup() %>%
  mutate(district_code = ifelse(x==1, 0, district_code))
```

    ## [1] "/tmp/RtmpIpnR0F/HS115_members.csv"
    ## [1] "/tmp/RtmpIpnR0F/HS116_members.csv"

``` r
## at-large here ???
vv_meta1 <- vv_meta %>%
  mutate(district_code = stringr::str_pad (district_code, 2, pad = 0),
         district_code = ifelse(chamber == 'Senate', 
                                'statewide', 
                                district_code),
         party_name = case_when(party_code == '100' ~ 'democratic',
                                party_code == '200' ~ 'republican',
                                party_code == '328' ~ 'independent')) %>%
  select(bioguide_id, congress:icpsr, 
         state_abbrev, district_code, party_name) # nominate_dim1
```

Full Twitter list
-----------------

**Via the Biodguide identifier**, we can easily add these details to our
TOC/GWU Twitter list. Data are available as a
[csv](https://github.com/jaytimm/twitter-and-us-lawmakers/blob/master/data/lawmaker-twitter-handles-voteview.csv),
and also included as a table in the [`uspols` R
package](https://github.com/jaytimm/uspols).

``` r
handles %>%
  left_join(vv_meta1, 
            by = c("congress", "chamber", "bioguide_id")) %>%
  head() %>%
  knitr::kable()
```

<table>
<colgroup>
<col style="width: 6%" />
<col style="width: 6%" />
<col style="width: 12%" />
<col style="width: 9%" />
<col style="width: 14%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 4%" />
<col style="width: 9%" />
<col style="width: 10%" />
<col style="width: 8%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: right;">congress</th>
<th style="text-align: left;">chamber</th>
<th style="text-align: left;">screen_name</th>
<th style="text-align: left;">bioguide_id</th>
<th style="text-align: left;">member</th>
<th style="text-align: left;">account_type</th>
<th style="text-align: left;">handle_type</th>
<th style="text-align: right;">icpsr</th>
<th style="text-align: left;">state_abbrev</th>
<th style="text-align: left;">district_code</th>
<th style="text-align: left;">party_name</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">KYCOMER</td>
<td style="text-align: left;">C001108</td>
<td style="text-align: left;">James Comer</td>
<td style="text-align: left;">campaign</td>
<td style="text-align: left;">prev_names</td>
<td style="text-align: right;">21565</td>
<td style="text-align: left;">KY</td>
<td style="text-align: left;">01</td>
<td style="text-align: left;">republican</td>
</tr>
<tr class="even">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">REPJACKYROSEN</td>
<td style="text-align: left;">R000608</td>
<td style="text-align: left;">Jacky Rosen</td>
<td style="text-align: left;">office</td>
<td style="text-align: left;">prev_names</td>
<td style="text-align: right;">21743</td>
<td style="text-align: left;">NV</td>
<td style="text-align: left;">03</td>
<td style="text-align: left;">democratic</td>
</tr>
<tr class="odd">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">REPESPAILLAT</td>
<td style="text-align: left;">E000297</td>
<td style="text-align: left;">Adriano Espaillat</td>
<td style="text-align: left;">office</td>
<td style="text-align: left;">screen_name</td>
<td style="text-align: right;">21715</td>
<td style="text-align: left;">NY</td>
<td style="text-align: left;">13</td>
<td style="text-align: left;">democratic</td>
</tr>
<tr class="even">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">REPTREY</td>
<td style="text-align: left;">H001074</td>
<td style="text-align: left;">Trey Hollingsworth</td>
<td style="text-align: left;">office</td>
<td style="text-align: left;">screen_name</td>
<td style="text-align: right;">21725</td>
<td style="text-align: left;">IN</td>
<td style="text-align: left;">09</td>
<td style="text-align: left;">republican</td>
</tr>
<tr class="odd">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">REPDWIGHTEVANS</td>
<td style="text-align: left;">E000296</td>
<td style="text-align: left;">Dwight Evans</td>
<td style="text-align: left;">office</td>
<td style="text-align: left;">screen_name</td>
<td style="text-align: right;">21566</td>
<td style="text-align: left;">PA</td>
<td style="text-align: left;">02</td>
<td style="text-align: left;">democratic</td>
</tr>
<tr class="even">
<td style="text-align: right;">115</td>
<td style="text-align: left;">House</td>
<td style="text-align: left;">ROGERMARSHALLMD</td>
<td style="text-align: left;">M001198</td>
<td style="text-align: left;">Roger Marshall</td>
<td style="text-align: left;">campaign</td>
<td style="text-align: left;">screen_name</td>
<td style="text-align: right;">21734</td>
<td style="text-align: left;">KS</td>
<td style="text-align: left;">01</td>
<td style="text-align: left;">republican</td>
</tr>
</tbody>
</table>
