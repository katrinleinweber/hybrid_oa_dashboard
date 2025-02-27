

# Abbildungen Präsentation Bibliothekartag

## Crossref Lizenzabdeckung


```r
library(tidyverse)
library(ggalt)
library(scales)
library(jsonlite)
hybrid_cr <- readr::read_csv("../../data/hybrid_publications.csv") %>%
  mutate(year = factor(issued, levels = c(
    "2013", "2014", "2015", "2016", "2017", "2018"
  ))) %>%
  arrange(desc(yearly_publisher_volume))
jn_facets <-
  jsonlite::stream_in(file("../../data/jn_facets_df.json"), verbose = FALSE)
# number of articles

n_journals_df <- jn_facets %>%
  distinct(journal_title, publisher) %>%
  group_by(publisher) %>%
  summarise(n_journals = n_distinct(journal_title))
#' all journals from open apc dataset for which we retrieved facet counts
#' AND from licensing info from crossref
n_hoa_df <- hybrid_cr %>%
  distinct(journal_title, publisher) %>%
  group_by(publisher) %>%
  summarise(n_hoa_journals = n_distinct(journal_title))
#' merge them into one dataframe
cvr_df <- left_join(n_journals_df, n_hoa_df, by = "publisher") %>%
  #' and prepare analysis of top 10 publishers
  tidyr::replace_na(list(n_hoa_journals = 0)) %>%
  arrange(desc(n_journals)) %>%
  mutate(publisher = forcats::as_factor(publisher)) %>%
  mutate(publisher = forcats::fct_other(publisher, drop = publisher[11:length(publisher)])) %>%
  ungroup() %>%
  group_by(publisher) %>%
  summarise(n_journals = sum(n_journals),
            n_hoa_journals = sum(n_hoa_journals))
#
cvr_df_2 <- tidyr::gather(cvr_df, group, value, -publisher)

#' plot
gg <- ggplot(cvr_df, aes(y = publisher)) +
  geom_point(data = cvr_df_2, aes(x = value, color = group), size = 3.5) +
  ggalt::geom_dumbbell(
    aes(x = n_journals, xend = n_hoa_journals),
    colour = "#30638E",
    colour_xend = "#EDAE49",
    colour_x = "#30638E",
    size_x = 3.5,
    alpha = 0.9,
    size_xend = 3.5
  ) +
  scale_y_discrete(limits = rev(levels(cvr_df$publisher))) +
  scale_x_continuous(breaks = seq(0, 1500, by = 250)) +
  labs(x = "Anzahl hybride OA-Journale je Verlag",
       y = NULL,
       title = "Hybrid Open Access Journals:\nSind Open-Content-Lizenzen über Crossref verfügbar?") +
  scale_color_manual(
    name = "",
    values = c("#EDAE49", "#30638E"),
    labels = c("Mit Open-Content-Lizenz", "Open APC")
  ) +
  theme_minimal(base_family = "Roboto", base_size = 12) +
  theme(plot.margin = margin(30, 30, 30, 30)) +
  theme(panel.grid.minor = element_blank()) +
  theme(axis.ticks = element_blank()) +
  theme(panel.grid.major.y = element_blank()) +
  theme(panel.border = element_blank()) +
  theme(legend.position = "top")
gg
```

![plot of chunk unnamed-chunk-2](figure/unnamed-chunk-2-1.png)

```r
ggsave(
  file = "publisher_coverage_d.png",
  width = 9,
  height = 5,
  dpi = 300
)
```

## Springer Nature Abdeckung im Vergleich


```r
library(tidyverse)
hybrid_cr <- readr::read_csv("../../data/hybrid_publications.csv") %>%
  mutate(year = factor(issued, levels = c(
    "2013", "2014", "2015", "2016", "2017", "2018"
  ))) %>%
  arrange(desc(yearly_publisher_volume))
o_apc_df <- readr::read_csv("../../data/oapc_hybrid.csv") %>%
  mutate(year = factor(period, levels = c(
    "2013", "2014", "2015", "2016", "2017", "2018"
  )))
hybrid_springer <- hybrid_cr %>%
  filter(publisher == "Springer Nature")

hybrid_sub <- hybrid_springer %>%
  group_by(year) %>%
  summarize(n = n_distinct(doi_oa)) %>%
  mutate(source = "Crossref")

o_apc_sub <- o_apc_df %>%
  filter(
    journal_full_title %in% hybrid_springer$journal_title,
    period %in% hybrid_springer$year
  ) %>%
  group_by(year) %>%
  summarize(n = n()) %>%
  mutate(source = "Open APC")

my_df <- bind_rows(o_apc_sub, hybrid_sub)
ggplot(my_df, aes(year, n, fill = source)) +
  geom_bar(stat = "identity", position = "dodge") +
  # coord_flip() +
  scale_fill_manual("", values = c("#EDAE49", "#30638E")) +
  scale_y_continuous(
    labels = function(x)
      format(x, big.mark = " ", scientific = FALSE)
  ) +
  theme_minimal("Roboto", base_size = 16) +
  theme(plot.margin = margin(30, 30, 30, 30)) +
  theme(panel.grid.minor = element_blank()) +
  theme(panel.border = element_blank()) +
  theme(legend.position = "top") +
  labs(x = NULL,
       y = "Artikel",
       title = "Springer Nature Abdeckung OA-Artikel im hybriden Modell")
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png)

```r
ggsave(
  file = "springer_coverage.png",
  width = 9,
  height = 5,
  dpi = 300
)
```


### Lizenzinfos


```r
hybrid_cr %>%
  mutate(license = gsub("http://creativecommons.org/licenses/", "cc-", license)) %>%
  mutate(license = gsub("/3.0*", "", license)) %>%
  mutate(license = gsub("/4.0", "", license)) %>%
  mutate(license = gsub("/2.0*", "", license)) %>%
  mutate(license = gsub("/uk/legalcode", "", license)) %>%
  mutate(license = gsub("/igo", "", license)) %>%
  mutate(license = gsub("/legalcode", "", license)) %>%
  mutate(
    license = gsub(
      "http://pubs.acs.org/page/policy/authorchoice_termsofuse.html",
      "ACS Author Choice",
      license
    )
  ) %>%
  mutate(license = toupper(license)) %>%
  mutate(license = forcats::fct_lump(license, n = 5)) -> tt

tt %>%
  count(license) %>%
  mutate(prop = n / sum(n)) %>%
  
  ggplot(aes(reorder(license, prop), prop)) +
  geom_bar(stat = "identity", fill = c(rep("grey60", 5), "#56B4E9")) +
  # scale_fill_manual(values = c("#56B4E9", rep("grey50", 5))) +
  coord_flip() +
  theme_minimal("Arial Narrow", base_size = 16) +
  theme(plot.margin = margin(30, 30, 30, 30)) +
  theme(panel.grid.minor = element_blank()) +
  theme(panel.border = element_blank()) +
  theme(legend.position = "top") +
  labs(x = NULL,
       y = NULL,
       title = "Anteil Open-Content-Lizenzen für hybride OA Artikel")
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)

```r
ggsave(
  file = "license_prop.png",
  width = 9,
  height = 5,
  dpi = 300
)
```
