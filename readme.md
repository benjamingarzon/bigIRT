bigIRT
================
Charles Driver

<!-- badges: start -->
[![R-CMD-check](https://github.com/benjamingarzon/bigIRT/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/cdriveraus/bigIRT/actions/workflows/R-CMD-check.yaml)
<!-- badges: end -->

## Install

If building on windows, first ensure you have Rtools installed.
<https://cran.r-project.org/bin/windows/Rtools/>

Run the following code to install the package:

``` r
remotes::install_github('cdriveraus/bigIRT', INSTALL_opts = "--no-multiarch", dependencies = c("Depends", "Imports"))
```

## Example

This example generates data from a 3 parameter logistic model, fits the
model using mirt and bigIRT, and compares the parameter estimates.

``` r
library(bigIRT)
library(data.table)
library(mirt)
library(ggplot2)

#generate data
dat <- simIRT(Nsubs = 400,Nitems = 50,Nscales = 1,
  logitCMean = -2,logitCSD = 1,AMean = 1,ASD = .3,
  BMean=0,BSD = .5,
  AbilityMean = 0,AbilitySD = 1)

#normalise true parameters and store
trueitempars <- normaliseIRT(B = dat$B, A = dat$A, Ability = dat$Ability)

#fit using bigIRT
bigirtfit <- fitIRT(dat$dat, pl=3, cores=2, dropPerfectScores = FALSE, stochastic=TRUE,
  score = 'score', id = 'id', scale = 'Scale', item = 'Item')

#view item and person parameter estimates
#print(bigirtfit$itemPars)
#print(bigirtfit$personPars)

#get normalized parameter estimates (normalised on ability)
bigirtpars <- normaliseIRT(B = bigirtfit$itemPars$B, A = bigirtfit$itemPars$A, Ability = bigirtfit$pars$Ability)

#convert to wide data for mirt
wdat <- data.frame(dcast(data.table(dat$dat),formula = 'id ~ Item',value.var='score')[,-1])

#fit using mirt
mirtfit <- mirt(wdat,model=1, itemtype = '3PL',method = "EM")

#get normalized parameter estimates
mirtItempars <- coef(mirtfit)
mirtItempars$GroupPars <- NULL
mirtItempars <- rbindlist(lapply(mirtItempars, function(x) as.data.frame(x)))
mirtPersonpars <- fscores(mirtfit)[,1]
mirtpars<- normaliseIRT(B = -mirtItempars$d / mirtItempars$a1, A = mirtItempars$a1, Ability = mirtPersonpars)


#combine parameter outputs
itempars <- rbind(
  data.table(Estimator = 'mirt', Parameter = 'A', Estimate=c(mirtpars$A), TrueValue=c(trueitempars$A)),
  data.table(Estimator = 'mirt', Parameter = 'B', Estimate=c(mirtpars$B), TrueValue=c(trueitempars$B)),
  data.table(Estimator = 'mirt', Parameter = 'C', Estimate=c(mirtItempars$g), TrueValue=c(dat$C)),
  data.table(Estimator = 'bigIRT', Parameter = 'A', Estimate=c(bigirtpars$A), TrueValue=c(trueitempars$A)),
  data.table(Estimator = 'bigIRT', Parameter = 'B', Estimate=c(bigirtpars$B), TrueValue=c(trueitempars$B)),
  data.table(Estimator = 'bigIRT', Parameter = 'C', Estimate=c(bigirtfit$itemPars$C), TrueValue=c(dat$C))
)

personpars <- rbind(
  data.table(Estimator = 'mirt', Parameter = 'Ability', Estimate=c(mirtpars$Ability), TrueValue=c(dat$Ability)),
  data.table(Estimator = 'bigIRT', Parameter = 'Ability', Estimate=c(bigirtpars$Ability), TrueValue=c(dat$Ability))
)

pars <- rbind(itempars,personpars)

#plot parameter estimates
ggplot(pars,aes(x=TrueValue,y=Estimate-TrueValue,color=Estimator)) + 
  geom_point() + 
  facet_wrap(~Parameter,scales='free')+
  theme_bw()+
  geom_hline(yintercept=0,linetype='dashed')
```

![](readme_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

``` r
#plot average absolute error
ggplot(pars,aes(x=TrueValue,y=abs(Estimate-TrueValue),colour=Estimator, fill=Estimator))+
  geom_smooth()+
  facet_wrap(~Parameter,scales='free')+
  theme_bw()+
  coord_cartesian(ylim=c(0,NA))+
  geom_hline(yintercept=0,linetype='dashed')
```

![](readme_files/figure-gfm/unnamed-chunk-1-2.png)<!-- -->

``` r
#plot average error
ggplot(pars,aes(x=TrueValue,y=Estimate-TrueValue,colour=Estimator, fill=Estimator))+
  geom_smooth()+
  facet_wrap(~Parameter,scales='free')+
  theme_bw()+geom_hline(yintercept=0,linetype='dashed')
```

![](readme_files/figure-gfm/unnamed-chunk-1-3.png)<!-- -->
