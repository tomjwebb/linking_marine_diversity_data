
<!-- README.md is generated from README.Rmd. Please edit that file -->

## Linking dimensions of data on global marine animal diversity

This is the code for reproducing the figures and analysis reported in
Webb & Vanhoorne *Linking dimensions of data on global marine animal
diversity*, Phil Trans R Soc B, doi: 10.1098/rstb.2019.0445

First, load required packages (see bottom of this script for installed
versions used):

``` r
library(tidyverse) # For data manipulation and plotting
library(patchwork) # For multi-panel figure layouts
library(broom) # For tidying model outputs
library(ggmosaic) # For mosaic plots
library(ggbeeswarm) # For beeswarm-style plots
```

## Reproducibility

<details>

<summary>Reproducibility receipt</summary>

``` r
## datetime
Sys.time()
```

    ## [1] "2020-07-28 15:13:54 BST"

``` r
## repository
git2r::repository()
```

    ## Local:    master /Users/tom/Google Drive/Linking and Enriching Marine Data/linking_marine_diversity_data
    ## Remote:   master @ origin (https://github.com/tomjwebb/linking_marine_diversity_data)
    ## Head:     [d5faf3d] 2020-07-28: Initial commit to set up

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
    ##  [1] ggbeeswarm_0.6.0 ggmosaic_0.2.0   broom_0.5.3      patchwork_1.0.0 
    ##  [5] forcats_0.4.0    stringr_1.4.0    dplyr_1.0.0      purrr_0.3.3     
    ##  [9] readr_1.3.1      tidyr_1.0.0      tibble_2.1.3     ggplot2_3.2.1   
    ## [13] tidyverse_1.3.0 
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_1.0.3         lubridate_1.7.4    lattice_0.20-38    assertthat_0.2.1  
    ##  [5] digest_0.6.24      R6_2.4.1           cellranger_1.1.0   plyr_1.8.5        
    ##  [9] backports_1.1.5    reprex_0.3.0       evaluate_0.14      httr_1.4.1        
    ## [13] pillar_1.4.3       rlang_0.4.6        lazyeval_0.2.2     readxl_1.3.1      
    ## [17] rstudioapi_0.10    data.table_1.12.8  rmarkdown_2.0      htmlwidgets_1.5.1 
    ## [21] munsell_0.5.0      compiler_3.6.2     vipor_0.4.5        modelr_0.1.5      
    ## [25] xfun_0.12          pkgconfig_2.0.3    htmltools_0.4.0    tidyselect_1.1.0  
    ## [29] fansi_0.4.1        viridisLite_0.3.0  crayon_1.3.4       dbplyr_1.4.2      
    ## [33] withr_2.1.2        grid_3.6.2         nlme_3.1-142       jsonlite_1.6.1    
    ## [37] gtable_0.3.0       lifecycle_0.2.0    DBI_1.1.0          git2r_0.26.1      
    ## [41] magrittr_1.5       scales_1.1.0       cli_2.0.1          stringi_1.4.6     
    ## [45] fs_1.3.1           xml2_1.2.2         generics_0.0.2     vctrs_0.3.1       
    ## [49] tools_3.6.2        glue_1.4.1         beeswarm_0.2.3     productplots_0.1.1
    ## [53] hms_0.5.3          yaml_2.2.1         colorspace_1.4-1   rvest_0.3.5       
    ## [57] plotly_4.9.2.1     knitr_1.26         haven_2.2.0
