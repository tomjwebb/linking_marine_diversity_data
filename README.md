
<!-- README.md is generated from README.Rmd. Please edit that file -->

## Linking dimensions of data on global marine animal diversity

This is the code for reproducing the figures and analysis reported in
Webb & Vanhoorne *Linking dimensions of data on global marine animal
diversity*, Phil Trans R Soc B, https://doi.org/10.1098/rstb.2019.0445

First, load required packages (see bottom of this script for installed
versions used):

``` r
library(tidyverse) # For reading, manipulating and plotting data
library(pscl) # For fitting hurdle models
library(broom) # For tidying model outputs
library(patchwork) # For multi-panel figure layouts
library(ggmosaic) # For mosaic plots
library(ggbeeswarm) # For beeswarm-style plots
```

The dataset used in this analysis is available under a Creative Commons
Attribution 4.0 International License in the Marine Data Archive (MDA).
The preferred route to access the data is via IMIS (Integrated Marine
Inforamtion System), which provides full meta-data. The IMIS record for
our dataset is here: <https://doi.org/10.14284/417> The code below uses
the direct link to MDA to download and unzip the two component csv files
into the data folder of this project directory: the full dataset and a
second csv which provides a full description of each variable.

``` r
# download zipped data from MDA
curl::curl_download(url = "https://mda.vliz.be/download.php?file=VLIZ_00000106_5f1ff6c0cf0a1110164358",
                    destfile = "data/worms_animals_links.zip")
# unzip
unzip("data/worms_animals_links.zip", exdir = "data")
```

The main dataset can now be read in:

``` r
worms_valid <- readr::read_csv("data/worms_animals_links.csv")
```

This contains 206849 observations of 28 variables. Each observation is a
valid marine animal species. Variables are described fully in the meta
data here:

``` r
worms_valid_meta <- readr::read_csv("data/meta.csv")
```

For each variable, this gives its class (`class`), the number of non-NA
observations (`n_obs`), the number of missing (NA) observations
(`n_miss`), the percent missing observations (`pct_miss`), a description
of the variables (`Description`), and relevant notes (`Notes`). For
info: the missing value scores were generated using
`naniar::miss_var_summary`.

## Visualising data availability across phyla and functional groups

Next, we create a phylum-level summary dataset, which summarises the
number of species per phylum, together with the number of species with
records in OBIS, GenBank, neither, or both. It also gives phylum-level
median number of non-zero OBIS and GenBank records:

``` r
phylum_obis_gen_summary <- worms_valid %>% group_by(phylum) %>% 
  summarise(
    n_sp = n(),
    n_obis = sum(!is.na(OBIS) & is.na(GenBank_nuc)),
    n_obis_gen = sum(!is.na(OBIS) & !is.na(GenBank_nuc)),
    n_gen = sum(is.na(OBIS) & !is.na(GenBank_nuc)),
    n_neither = sum(is.na(OBIS) & is.na(GenBank_nuc)),
    med_obis = median(OBIS, na.rm = TRUE),
    med_gen = median(GenBank_nuc, na.rm = TRUE)
  ) %>% 
  rownames_to_column() %>% 
  rename(phylum_id = rowname) %>% 
  mutate(phylum_id = as.numeric(phylum_id))
```

We then create a long version of this summary data frame ready for
plotting, adding a scaling factor so that the number of species per
phylum is rescaled onto 0.25-1, for nice plotting of bar widths:

``` r
phylum_obis_gen_long <- phylum_obis_gen_summary %>%
  pivot_longer(
    cols = n_obis:n_neither,
    names_to = "data_source",
    values_to = "n_species"
  ) %>% 
  mutate(data_source = ordered(data_source, c("n_neither", "n_gen", "n_obis_gen", "n_obis")),
         p_species = n_species / n_sp,
         scaled_sp = ((n_sp / 76448) + 0.25))
```

Create a colour palette for plotting this chart of proportion of species
per phylum with data in each
database:

``` r
data_source_pal <- tibble(data_source = c("n_neither", "n_gen", "n_obis_gen", "n_obis"),
                          ds_colours = c("#FCFAF1", "#DB565D", "#633231", "#FACCAD"),
                          ds_labels = c("no data", "GenBank only", "OBIS and GenBank", "OBIS only")
                          )
```

And now create the plot (Figure 1A):

``` r
p_species_data_source <- ggplot(phylum_obis_gen_long,
                                aes(x = phylum, y = p_species,
                                    fill = data_source, width = scaled_sp)) +
  geom_bar(stat = "identity", position = "stack", alpha = 0.8, colour = "black", size = 0.2) +
  scale_fill_manual(name = "P(species with data)",
                    values = pull(data_source_pal, ds_colours),
                    labels = pull(data_source_pal, ds_labels)) +
  labs(x = "", y = "", tag = "A") +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, vjust = 1)) +
  scale_y_continuous(breaks=c(0, 0.5, 1)) +
  geom_hline(yintercept = c(0, 0.5, 1), size = 0.25)
```

To create Figure 1 B and C, we use this palette for functional groups:

``` r
functional_group_palette <- tibble(
  functional_group = c("benthos", "zooplankton", "nekton",
                       "fish", "mammals", "birds", "reptiles",
                       "other/unknown"),
  fg_pal = c("#077893", "#FF9465", "#00AE91",
             "#45BEC6", "#9F4440", "#FFD37B", "#EE9BE4",
             "#E3DECA")
)
```

Then generate the figures, plotting the number of OBIS records and then
GenBank nucleotides for each species (excluding 0 records) in each
phylum:

``` r
obis_by_phylum <- ggplot(worms_valid, aes(x = phylum, y = log10(OBIS))) +
  geom_jitter(aes(colour = functional_group), size = 1/3, alpha = 3/5, na.rm = TRUE) +
  scale_colour_manual(name = "functional group",
                      values = pull(functional_group_palette, fg_pal),
                      labels = pull(functional_group_palette, functional_group)) +
  labs(x = "", y = expression(log[10]*"(OBIS records)"), tag = "B") +
  theme_minimal() +
  theme(axis.title.x = element_blank(), axis.text.x = element_blank(), legend.position = "none") +
  geom_boxplot(na.rm = TRUE, fill = NA, size = 0.25, outlier.shape = NA,
               fatten = 0, varwidth = TRUE) +
  geom_point(data = phylum_obis_gen_summary, aes(x = phylum, y = log10(med_obis)),
             shape = 4, size = 1.5, stroke = 1.1)

genbank_by_phylum <- ggplot(worms_valid, aes(x = phylum, y = log10(GenBank_nuc))) +
    geom_jitter(aes(colour = functional_group), size = 1/3, alpha = 3/5, na.rm = TRUE) +
    scale_colour_manual(name = "functional group",
                        values = pull(functional_group_palette, fg_pal),
                        labels = pull(functional_group_palette, functional_group)) +
    labs(x = "", y = expression(log[10]*"(GenBank nucleotides)"), tag = "C") +
    theme_minimal() +
    theme(axis.title.x = element_blank(), axis.text.x = element_blank()) +
    guides(colour = guide_legend(override.aes = list(size = 5))) +
    geom_boxplot(na.rm = TRUE, fill = NA, size = 0.25, outlier.shape = NA,
                 fatten = 0, varwidth = TRUE) +
    geom_point(data = phylum_obis_gen_summary, aes(x = phylum, y = log10(med_gen)),
               shape = 4, size = 1.5, stroke = 1.1)
```

We use the `patchwork` package to assemble these three
panels:

``` r
(linking_fig1 <- genbank_by_phylum / obis_by_phylum / p_species_data_source +
  plot_layout(heights = c(2, 2, 1))
)
```

![](README_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

This can then be saved in your required
format:

``` r
ggsave("figures/webb_philtrans_fig1.pdf", linking_fig1, width = 10, height = 8, units = "in")  
```

## Data availablity across fish habitat groups

For fish we also explored data availability across habitat groups
derived from FishBase, using the `rfishbase` package to add habitat
affinities to our dataset (stored in `functional_group2`). Filter the
dataset to fish only, and order the categories:

``` r
fish <- worms_valid %>% filter(functional_group == "fish") %>%
  mutate(functional_group2 = ifelse(
    functional_group2 == "fish_unspecified_habitat", "habitat unspecified", functional_group2)) %>% 
  mutate(functional_group2 = ordered(functional_group2,
                                            c("bathydemersal", "bathypelagic",
                                            "benthopelagic", "demersal",
                                            "pelagic-oceanic", "pelagic-neritic",
                                            "reef-associated", "habitat unspecified")))
```

Then create a summary of data availablity across these groups:

``` r
fish_obis_gen_summary <- fish %>% group_by(functional_group2) %>% 
  summarise(
    n_sp = n(),
    n_obis = sum(!is.na(OBIS) & is.na(GenBank_nuc)),
    n_obis_gen = sum(!is.na(OBIS) & !is.na(GenBank_nuc)),
    n_gen = sum(is.na(OBIS) & !is.na(GenBank_nuc)),
    n_neither = sum(is.na(OBIS) & is.na(GenBank_nuc)),
    med_obis = median(OBIS, na.rm = TRUE),
    med_gen = median(GenBank_nuc, na.rm = TRUE)
  ) %>% 
  rownames_to_column() %>% 
  rename(habitat_id = rowname)
```

Make this long for plotting:

``` r
fish_obis_gen_long <- fish_obis_gen_summary %>%
  pivot_longer(
    cols = n_obis:n_neither,
    names_to = "data_source",
    values_to = "n_species"
  ) %>% 
  mutate(data_source = ordered(data_source, c("n_neither", "n_gen", "n_obis_gen", "n_obis")),
         p_species = n_species / n_sp,
         scaled_sp = (n_sp / (6248/0.75)) + 0.25)
```

Create the first panel of the figure - proportion of species in each
habiat class with data in each database:

``` r
p_species_data_source_fish <- ggplot(fish_obis_gen_long,
                                      aes(x = functional_group2, y = p_species,
                                          fill = data_source, width = scaled_sp)) +
    geom_bar(stat = "identity", position = "stack", alpha = 0.8, colour = "black", size = 0.2) +
    scale_fill_manual(name = "P(species with data)",
                      values = pull(data_source_pal, ds_colours),
                      labels = pull(data_source_pal, ds_labels)) +
    labs(x = "", y = "", tag = "A") +
    theme_minimal() +
    theme(axis.text.x = element_text(angle = 60, hjust = 1, vjust = 1)) +
    scale_y_continuous(breaks=c(0, 0.5, 1)) +
    geom_hline(yintercept = c(0, 0.5, 1), size = 0.25)
```

For the remaining panels, use this palette for fish habitats:

``` r
fish_habitat_palette <- tibble(
  habitat = c("bathydemersal", "bathypelagic",
                        "benthopelagic", "demersal",
                        "pelagic-oceanic", "pelagic-neritic",
                        "reef-associated", "habitat unspecified"),
  fg_pal = c("#003E51", "#2B4D50", "#3C3768", "#486D87",
             "#007DBA", "#2E94AC", "#9DB6B5", "#C5B9AC")
)
```

Plot number of OBIS records per species by habitat class:

``` r
obis_by_fish_fg <- 
  ggplot(fish, aes(x = functional_group2, y = log10(OBIS))) +
  geom_quasirandom(aes(colour = functional_group2), size = 1/3, alpha = 3/5, na.rm = TRUE) +
  scale_colour_manual(name = "habitat",
                      values = pull(fish_habitat_palette, fg_pal),
                      labels = pull(fish_habitat_palette, habitat)) +
  labs(x = "", y = expression(log[10]*"(OBIS records)"), tag = "B") +
  theme_minimal() +
  theme(axis.title.x = element_blank(), axis.text.x = element_blank(), legend.position = "none") +
  geom_boxplot(na.rm = TRUE, fill = NA, size = 0.25, outlier.shape = NA,
               fatten = 0, varwidth = TRUE) +
  geom_point(data = fish_obis_gen_summary, aes(x = functional_group2, y = log10(med_obis)),
             shape = 4, size = 1.5, stroke = 1.1)
```

And the same for GenBank nucleotides:

``` r
genbank_by_fish_fg <- 
  ggplot(fish, aes(x = functional_group2, y = log10(GenBank_nuc))) +
  geom_quasirandom(aes(colour = functional_group2), size = 1/3, alpha = 3/5, na.rm = TRUE) +
  scale_colour_manual(name = "habitat",
                      values = pull(fish_habitat_palette, fg_pal),
                      labels = pull(fish_habitat_palette, habitat)) +
  labs(x = "", y = expression(log[10]*"(Genbank nucleotides)"), tag = "C") +
  theme_minimal() +
  theme(axis.title.x = element_blank(), axis.text.x = element_blank()) +
  guides(colour = guide_legend(override.aes = list(size = 5))) +
  geom_boxplot(na.rm = TRUE, fill = NA, size = 0.25, outlier.shape = NA,
               fatten = 0, varwidth = TRUE) +
  geom_point(data = fish_obis_gen_summary, aes(x = functional_group2, y = log10(med_gen)),
             shape = 4, size = 1.5, stroke = 1.1)
```

Assemble Figure
S1:

``` r
(linking_fig1_fish_habs <- genbank_by_fish_fg / obis_by_fish_fg / p_species_data_source_fish +
    plot_layout(heights = c(2, 2, 1))
)
```

![](README_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

And save to file:

``` r
ggsave("figures/webb_philtrans_figS1.pdf", linking_fig1_fish_habs,
       width = 10, height = 8, units = "in")
```

## Modelling data availability across functional groups

Because of the large number of zeros in our data, we use a hurdle
modelling approach to model data availability across functional groups.
First, we create a couple of convenience variables to aid modelling:

``` r
worms_valid <- worms_valid %>% 
  mutate(n_obis = ifelse(is.na(OBIS), 0, OBIS),
         n_gb = ifelse(is.na(GenBank_nuc), 0, GenBank_nuc),
         fg = factor(functional_group))
```

We then run a single hurdle model, modelling zeros as binomial and
non-zero counts as negative binomial, using `pscl::hurdle`:

``` r
fm_obis_hurdle <- hurdle(n_obis ~ fg - 1, data = worms_valid,
                         dist = "negbin", zero.dist = "binomial")
```

To derive estimated non-zero counts for each functional group, we first
get P(non-zero count) for each functional
group:

``` r
zero_coefs <- coef(fm_obis_hurdle)[which(str_detect(names(coef(fm_obis_hurdle)), "zero"))]
p_nonzero <- exp(zero_coefs) / (exp(zero_coefs) + 1)
```

Mean count (including zeros) is just the response prediction:

``` r
mean_count <- predict(fm_obis_hurdle, type = "response",
                      newdata = data.frame(fg = levels(worms_valid$fg)))
```

So predicted counts for non-zero species are:

``` r
count_given_present <- mean_count / p_nonzero
```

Create a simple summary table for functional groups:

``` r
fg_summary <- worms_valid %>% 
  group_by(functional_group) %>% 
  summarise(n_sp = n(), n_obis = sum(in_obis), n_genb = sum(in_genb)) %>% 
  mutate(functional_group = ordered(functional_group,
                                    c("benthos", "zooplankton", "nekton", "fish",
                                      "mammals", "birds", "reptiles", "other/unknown")))
```

For convenience we run the zero model as a separate binomial GLM to
facilitate calculation of
CIs:

``` r
fm_in_obis <- glm(in_obis ~ functional_group - 1, family = binomial, data = worms_valid)
```

Now create a tidy version of the
coefficients:

``` r
fm_in_obis_tidy <- tidy(fm_in_obis, conf.int = TRUE, exponentiate = TRUE) %>% 
  bind_cols(fg_summary)
```

Get empirical means and bootstrap CIs for the non-zero counts. This
function will do this based on a sample of 90 (given 96 species in the
smallest groups):

``` r
get_fg_means <- function(dat, i = 90){
  dat %>% 
    group_by(functional_group) %>% 
    sample_n(i, replace = T) %>% 
    summarise(mean_obis = mean(OBIS, na.rm = TRUE), .groups = "keep")
}
```

Run this 10,000
times:

``` r
bootstrap_fg_means <- purrr::map_dfr(1:10000, function(x) get_fg_means(worms_valid))
```

Get number of species in OBIS for each functional
group:

``` r
n_in_obis <- worms_valid %>% filter(!is.na(OBIS)) %>% count(functional_group)
```

Get dataframe of mean from data and from model:

``` r
mean_counts <- worms_valid %>%
  filter(in_obis == 1) %>%
  group_by(functional_group) %>%
  summarise(mean_from_data = mean(OBIS)) %>% 
  mutate(est_from_model = count_given_present)
```

Create a summary data frame:

``` r
bootstrap_obis_summary <- bootstrap_fg_means %>%
  filter(!is.na(mean_obis)) %>% 
  summarise(obis_non_zero = mean(mean_obis),
            low_ci = quantile(mean_obis, 0.025),
            high_ci = quantile(mean_obis, 0.975)) %>% 
  left_join(mean_counts, by = "functional_group") %>% 
  left_join(n_in_obis, by = "functional_group") %>% 
  rename(n_obis_sp = n) %>% 
  mutate(functional_group = ordered(functional_group,
                                    c("benthos", "zooplankton", "nekton", "fish",
                                      "mammals", "birds", "reptiles", "other/unknown")))
```

Now create the figure. First, zero
coefficients:

``` r
in_obis_binomial <- ggplot(fm_in_obis_tidy, aes(x = functional_group, y = estimate,
                                                colour = functional_group)) +
    geom_point(aes(size = log10(n_sp))) +
    geom_segment(aes(x = functional_group, xend = functional_group,
                     y = conf.low, yend = conf.high)) +
    scale_colour_manual(values = pull(functional_group_palette, fg_pal)) +
    labs(x = "", y = "Binomial coefficient (OBIS)", tag = "A") +
    theme_minimal() +
    theme(legend.position = "none") +
    coord_flip()
```

Now for the zero-truncated means:

``` r
obis_n_empirical <- ggplot(bootstrap_obis_summary,
                            aes(x = functional_group, y = log10(mean_from_data),
                                colour = functional_group)) +
    geom_point(aes(size = log10(n_obis_sp))) +
    geom_segment(aes(x = functional_group, xend = functional_group,
                     y = log10(low_ci), yend = log10(high_ci))) +
    geom_point(aes(y = log10(est_from_model)), shape = 4, colour = "black") +
    scale_colour_manual(values = pull(functional_group_palette, fg_pal)) +
    labs(x = "", y = expression(log[10]*"(zero-truncated mean OBIS records)"), tag = "B") +
    theme_minimal() +
    theme(legend.position = "none") +
    coord_flip()
```

Now run the same process for GenBank:

``` r
fm_gb_hurdle <- hurdle(n_gb ~ fg - 1, data = worms_valid,
                         dist = "negbin", zero.dist = "binomial")
# work out estimated non-zero counts, first get P(non-zero count) for each functional group
zero_coefs_gb <- coef(fm_gb_hurdle)[which(str_detect(names(coef(fm_gb_hurdle)), "zero"))]
p_nonzero_gb <- exp(zero_coefs_gb) / (exp(zero_coefs_gb) + 1)
# mean count is just the response predictiosn
mean_count_gb <- predict(fm_gb_hurdle, type = "response", newdata = data.frame(fg = levels(worms_valid$fg)))
# so predicted counts for non-zero species are:
count_given_present_gb <- mean_count_gb / p_nonzero_gb

# run the zero model as a separate glm to get CIs:
fm_in_gb <- glm(in_genb ~ functional_group - 1, family = binomial, data = worms_valid)
# now create a tidy version of the coefficients:
fm_in_gb_tidy <- tidy(fm_in_gb, conf.int = TRUE, exponentiate = TRUE) %>% 
  bind_cols(fg_summary)

# get empirical means and bootstrap CIs for the non-zero counts
get_fg_means <- function(dat, i = 75){
  dat %>% 
    group_by(functional_group) %>% 
    sample_n(i, replace = T) %>% 
    summarise(mean_gb = mean(GenBank_nuc, na.rm = TRUE), .groups = "keep")
}

# run this 10,000 times
bootstrap_fg_means_gb <- purrr::map_dfr(1:10000, function(x) get_fg_means(worms_valid))

# get number of species in GenBank for each functional group:
n_in_gb <- worms_valid %>% filter(in_genb == 1) %>% count(functional_group)

# get dataframe of mean from data and from model
mean_counts_gb <- worms_valid %>%
  filter(in_genb == 1) %>%
  group_by(functional_group) %>%
  summarise(mean_from_data = mean(GenBank_nuc)) %>% 
  mutate(est_from_model = count_given_present_gb)

# create a summary data frame
bootstrap_gb_summary <- bootstrap_fg_means_gb %>%
  filter(!is.na(mean_gb)) %>% 
  summarise(gb_non_zero = mean(mean_gb),
            low_ci = quantile(mean_gb, 0.025),
            high_ci = quantile(mean_gb, 0.975)) %>% 
  left_join(mean_counts_gb, by = "functional_group") %>% 
  left_join(n_in_gb, by = "functional_group") %>% 
  rename(n_gb_sp = n) %>% 
  mutate(functional_group = ordered(functional_group,
                                    c("benthos", "zooplankton", "nekton", "fish",
                                      "mammals", "birds", "reptiles", "other/unknown")))
```

Now create the figure. First for zero
coefficients:

``` r
in_gb_binomial <- ggplot(fm_in_gb_tidy, aes(x = functional_group, y = estimate,
                                            colour = functional_group)) +
    geom_point(aes(size = log10(n_sp))) +
    geom_segment(aes(x = functional_group, xend = functional_group,
                     y = conf.low, yend = conf.high)) +
    scale_colour_manual(values = pull(functional_group_palette, fg_pal)) +
    labs(x = "", y = "Binomial coefficient (GenBank)", tag = "C") +
    theme_minimal() +
    theme(legend.position = "none") +
    coord_flip()
```

Then for zero-truncated means:

``` r
gb_n_empirical <- ggplot(bootstrap_gb_summary,
                            aes(x = functional_group, y = log10(mean_from_data),
                                colour = functional_group)) +
    geom_point(aes(size = log10(n_gb_sp))) +
    geom_segment(aes(x = functional_group, xend = functional_group,
                     y = log10(low_ci), yend = log10(high_ci))) +
    geom_point(aes(y = log10(est_from_model)), shape = 4, colour = "black") +
    scale_colour_manual(values = pull(functional_group_palette, fg_pal)) +
    labs(x = "", y = expression(log[10]*"(zero-truncated mean GenBank nucleotides)"), tag = "D") +
    theme_minimal() +
    theme(legend.position = "none") +
    coord_flip()
```

Use `patchwork` to assemble Figure
2:

``` r
(coef_figs <- (in_obis_binomial | obis_n_empirical) / (in_gb_binomial | gb_n_empirical))
```

![](README_files/figure-gfm/unnamed-chunk-40-1.png)<!-- -->

And save to
file:

``` r
ggsave("figures/webb_philtrans_fig2.pdf", coef_figs, width = 10, height = 8, units = "in")
```

## Relationship between OBIS and GenBank data availability

We use mosaic plots to visualise the contingency between OBIS records
and GenBank nucleotides. First, we create log-10 categories for both
OBIS and GenBank:

``` r
worms_valid <- worms_valid %>% mutate(
  obis_cat = as.numeric(cut(log10(OBIS),
                              breaks = c(-1:5, 7),
                              labels = 10^(0:6), include.lowest = TRUE, right = TRUE)),
  gen_cat = as.numeric(cut(log10(GenBank_nuc),
                            breaks = c(-1:5, 7),
                            labels = 10^(0:6), include.lowest = TRUE, right = TRUE))
  ) %>% 
  mutate(
    obis_cat = case_when(
      is.na(obis_cat) ~ 0,
      TRUE ~ 10^(obis_cat - 1)),
    gen_cat = case_when(
      is.na(gen_cat) ~ 0,
      TRUE ~ 10^(gen_cat - 1))
    ) %>% 
  mutate(obis_cat = as.factor(obis_cat),
         gen_cat = as.factor(gen_cat))
```

Create a convenient vector of scale labels:

``` r
p10_scale <- c(0, 1, 10, expression(10^2), expression(10^3),
               expression(10^4), expression(10^5), expression(10^7)
)
```

Mosaic plot of all species:

``` r
(obis_gen_all <- ggplot(data = worms_valid) +
  geom_mosaic(aes(x = product(obis_cat), fill = gen_cat), offset = 0.001) +
  scale_fill_viridis_d(name = "GenBank\nnucleotides",
                       labels = p10_scale) +
  scale_x_productlist(labels = c(p10_scale[1:4], "", expression(10^3*"-"*10^7), "", "")) +
  scale_y_productlist(labels = c(p10_scale[1:3], "", expression(10^2*"-"*10^7), "", "", "")) +
  labs(x = "OBIS records", y = element_blank(), tag = "A") +
  theme_minimal() +
  theme(legend.position = c(0, 0.5),
        legend.justification = "right") +
  coord_fixed()
)
```

![](README_files/figure-gfm/unnamed-chunk-44-1.png)<!-- -->

Create a plot for the subset of species with a large number (\>100) of
OBIS records:

``` r
(obis_gen_hi_obis <- ggplot(data = filter(worms_valid,
                                          obis_cat %in% c("1000", "10000", "1e+05", "1e+06"))) +
  geom_mosaic(aes(x = product(obis_cat), fill = gen_cat), offset = 0.001) +
  scale_fill_viridis_d(name = "GenBank\nnucleotides",
                       labels = p10_scale) +
  scale_x_productlist(labels = c("", "", "", "", p10_scale[5:8])) +
  labs(x = "OBIS records", y = element_blank(), tag = "B") +
  theme_minimal() +
  theme(legend.position = c(0, 0.5), legend.justification = "right",
        axis.text.y = element_blank()) +
  coord_fixed()
  )
```

![](README_files/figure-gfm/unnamed-chunk-45-1.png)<!-- -->

And the same for species with a large number of GenBank nucleotides:

``` r
(obis_gen_hi_gen <- ggplot(data = filter(worms_valid,
                                         gen_cat %in% c("1000", "10000", "1e+05", "1e+06"))) +
  geom_mosaic(aes(x = product(gen_cat), fill = obis_cat), offset = 0.001) +
  scale_fill_viridis_d(name = "OBIS\nrecords",
                       labels = p10_scale) +
  scale_x_productlist(labels = c("", "", "", "", p10_scale[5:8])) +
  labs(x = "GenBank nucleotides", y = element_blank(), tag = "C") +
  theme_minimal() +
    theme(legend.position = c(0, 0.5), legend.justification = "right",
          axis.text.y = element_blank()) +
  coord_fixed()
)
```

![](README_files/figure-gfm/unnamed-chunk-46-1.png)<!-- -->

The final version of figure 3 was assembled and annotated offline from
these three constituent panels, saved here:

``` r
ggsave("figures/webb_philtrans_fig3a.pdf", obis_gen_all,
       width = 7, height = 6, units = "in")
ggsave("figures/webb_philtrans_fig3b.pdf", obis_gen_hi_obis,
       width = 5.5, height = 4.7, units = "in")
ggsave("figures/webb_philtrans_fig3c.pdf", obis_gen_hi_gen,
       width = 5.5, height = 4.7, units = "in")
```

## IUCN and BOLD data availability

Finally consider IUCN assessments and Barcode of Life data availability.
Create a palette for IUCN categories:

``` r
iucn_palette <- tibble(
  iucn_cat = c("Not Assessed", "Data Deficient", "Threatened", "Not Threatened"),
  iucn_pal = c("#E3DECA", "#077893", "#71AB7F", "#E34F33")
)
```

And plot OBIS records against IUCN
categories:

``` r
iucn_n_obis <- ggplot(filter(worms_valid, in_obis == 1), aes(x = IUCN_cat, y = log10(OBIS),
                                                             colour = IUCN_cat)) +
  geom_quasirandom(size = 0.5, alpha = 3/5) +
  scale_colour_manual(name = "IUCN category",
                    values = pull(iucn_palette, iucn_pal),
                    labels = pull(iucn_palette, iucn_cat),
                    drop = FALSE) +
  labs(x = "IUCN Assessment Status", y = expression(log[10]*"(OBIS records)"), tag = "A") +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, vjust = 1),
        legend.position = "none") +
  facet_wrap(~functional_group, ncol = 4, nrow = 2)
```

Do the same for
BOLD:

``` r
bold_n_obis <- ggplot(filter(worms_valid, in_obis == 1), aes(x = in_bold, y = log10(OBIS),
                                                              colour = in_bold)) +
    geom_quasirandom(size = 0.5, alpha = 3/5) +
    scale_colour_manual(name = "In BOLD",
                        values = c("#077893", "#FF9465")) +
    labs(x = "In Barcode of Life Database", y = expression(log[10]*"(OBIS records)"), tag = "B") +
    theme_minimal() +
    theme(legend.position = "none") +
    facet_wrap(~functional_group, ncol = 4, nrow = 2)
```

Assmeble the figure and save to
file:

``` r
(bold_iucn_obis <- iucn_n_obis / bold_n_obis)
```

![](README_files/figure-gfm/unnamed-chunk-51-1.png)<!-- -->

``` r
ggsave("figures/webb_phitrans_fig4.pdf", bold_iucn_obis, width = 10, height = 10)
```

## Reproducibility

<details>

<summary>Reproducibility receipt</summary>

``` r
## datetime
Sys.time()
```

    ## [1] "2020-08-19 16:51:30 BST"

``` r
## repository
git2r::repository()
```

    ## Local:    master /Users/tom/Google Drive/Linking and Enriching Marine Data/linking_marine_diversity_data
    ## Remote:   master @ origin (https://github.com/tomjwebb/linking_marine_diversity_data)
    ## Head:     [24e3410] 2020-07-28: Adding all figure and analysis code to readme

``` r
## session info
sessionInfo()
```

    ## R version 3.6.2 (2019-12-12)
    ## Platform: x86_64-apple-darwin15.6.0 (64-bit)
    ## Running under: macOS Catalina 10.15.5
    ## 
    ## Matrix products: default
    ## BLAS:   /Library/Frameworks/R.framework/Versions/3.6/Resources/lib/libRblas.0.dylib
    ## LAPACK: /Library/Frameworks/R.framework/Versions/3.6/Resources/lib/libRlapack.dylib
    ## 
    ## locale:
    ## [1] en_GB.UTF-8/en_GB.UTF-8/en_GB.UTF-8/C/en_GB.UTF-8/en_GB.UTF-8
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] ggbeeswarm_0.6.0 ggmosaic_0.2.0   patchwork_1.0.0  broom_0.5.3     
    ##  [5] pscl_1.5.5       forcats_0.4.0    stringr_1.4.0    dplyr_1.0.0     
    ##  [9] purrr_0.3.3      readr_1.3.1      tidyr_1.0.0      tibble_2.1.3    
    ## [13] ggplot2_3.2.1    tidyverse_1.3.0 
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_1.0.3         lubridate_1.7.4    lattice_0.20-38    assertthat_0.2.1  
    ##  [5] digest_0.6.24      R6_2.4.1           cellranger_1.1.0   plyr_1.8.5        
    ##  [9] backports_1.1.5    reprex_0.3.0       evaluate_0.14      httr_1.4.1        
    ## [13] pillar_1.4.3       rlang_0.4.6        curl_4.3           lazyeval_0.2.2    
    ## [17] readxl_1.3.1       rstudioapi_0.10    data.table_1.12.8  rmarkdown_2.0     
    ## [21] labeling_0.3       htmlwidgets_1.5.1  munsell_0.5.0      vipor_0.4.5       
    ## [25] compiler_3.6.2     modelr_0.1.5       xfun_0.12          pkgconfig_2.0.3   
    ## [29] htmltools_0.4.0    tidyselect_1.1.0   fansi_0.4.1        viridisLite_0.3.0 
    ## [33] crayon_1.3.4       dbplyr_1.4.2       withr_2.1.2        MASS_7.3-51.4     
    ## [37] grid_3.6.2         nlme_3.1-142       jsonlite_1.6.1     gtable_0.3.0      
    ## [41] lifecycle_0.2.0    DBI_1.1.0          git2r_0.26.1       magrittr_1.5      
    ## [45] scales_1.1.0       cli_2.0.1          stringi_1.4.6      farver_2.0.3      
    ## [49] fs_1.3.1           xml2_1.2.2         ellipsis_0.3.0     generics_0.0.2    
    ## [53] vctrs_0.3.1        tools_3.6.2        glue_1.4.1         beeswarm_0.2.3    
    ## [57] productplots_0.1.1 hms_0.5.3          yaml_2.2.1         colorspace_1.4-1  
    ## [61] rvest_0.3.5        plotly_4.9.2.1     knitr_1.26         haven_2.2.0
