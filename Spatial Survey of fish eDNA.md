---
title: "Script for 'Optimal sample type and number vary in small shallow lakes when targeting non-native fish environmental DNA'"
author: "Maïlys Picard"
date: "16/03/2023"
output: html_document
---

## Introduction


Goals:
 + Compare detection between surface sediment and surface water samples
 + Compare detection between near-shore and mid-lake sites
 + Will detection change depending on the fish species (rudd = benthic vs perch = pelagic)?
 + How many sites/replicates needed per lake to reliably detect fish eDNA?


Hypotheses:
 + eDNA will be homogeneously distributed across lakes for both species but
 + due to their life history (benthic rudd versus pelagic perch)
    + rudd eDNA will be better detected in sediment samples, while 
    + perch eDNA will be better detected by water samples.

## Methods

+ Three lakes in the North Island of New Zealand (Oceania) – Lakes Pounui, Tomarata, Waitawa
+ 14 sites per lake
    + 2 types of samples per site
    + Surface water: 2 x 1 L – 500 mL analysed
    + Surface sediment: 2 x ~ 3g
+ 2 types of location within a lake
    + Near-shore
    + Mid-lake
+ 2 targets (non-native fish)
    + Rudd (benthic)
    + European Perch (pelagic)
+ eDNA samples analysed by
    + ddPCR (purely quantitative) – species-specific primers

+ Data analysis
    + General graphs (boxplots, heatmap)
    + Occupancy modelling
    + Simulations to calculate how many sites/replicates needed if occupancy = 1


```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)

```


```{r Loading packages, include=FALSE}
require(tidyverse)
require(ggvenn)
require(scales)
require(magrittr)
require(viridis)
require(ggpubr)
require(ggstar)
require(lawstat)

```

## LoQ calculations

```{r LoQ determination}

loq<-read.csv("Picard_LoQ_calculation_data.csv ") #see paper supplementary info
glimpse(loq)
levels(as.factor(loq$Calc.Conc))

loq  %<>% filter(Calc.Conc > 0.000001)
levels(as.factor(loq$Calc.Conc))


SS1 <- loq %>% filter(TargetType == "Perch") %>%
  filter(Calc.Conc > 0.9)

mean(SS1$Calc.Conc)

mean(SS1$Measured.Conc)

SS2 <- loq %>% filter(TargetType == "Rudd") %>%
  filter(Calc.Conc > 0.9)

mean(SS2$Calc.Conc)

mean(SS2$Measured.Conc)

SS <-  data.frame(
  label = c(rep("LoQ", 2)),
  x = c(0.15, 0.009),
  y   = c(20, 12),  
  x1 = c(0.00009, 0.00008),
  y1   = c(0.001, 0.01),
  y2   = c(1.5, 2),
  TargetType = c("Perch", "Rudd")
)

ggplot(data = loq) +
  geom_smooth(aes(x = Calc.Conc, y = Measured.Conc + 1), method = "loess") +
  geom_line(aes(Calc.Conc, Theor.Measured.Conc), alpha = 0.1) +
  geom_point(aes(Calc.Conc, Theor.Measured.Conc), shape = 4, color = "dark grey", size = 3) +
  geom_point(aes(x = Calc.Conc, y = Measured.Conc + 1), size = 2) +
  geom_vline(data=filter(loq, TargetType=="Perch"), aes(xintercept=0.5), colour="#F38D35", linetype = 5, linewidth = 1) + 
  geom_vline(data=filter(loq, TargetType=="Rudd"), aes(xintercept=0.03), colour="#F38D35", linetype = 5, linewidth = 1) + 
  
  geom_text(data = SS,aes(x = x, y = y, label = label), colour="#F38D35", size = 4, hjust   = 0, vjust   = 0) +
  geom_text(data = SS,aes(x = x1, y = y1, label = "Expected trend"), colour="dark grey", size = 4, hjust   = 0, vjust   = 0) +
  geom_text(data = SS,aes(x = x1, y = y2, label = "Measured trend"), colour="dark blue", size = 4, hjust   = 0, vjust   = 0) +
  facet_wrap( .~ TargetType, scales = "free") +
  scale_x_log10() +
  scale_y_log10() +
  labs(x = "Calculated fish DNA concentration (ng/uL)", y = "Measured fish DNA concentration (copies/uL)") +
  theme_bw() +
  theme(#axis.text.y = element_blank(),
    #axis.ticks.y = element_blank(),
    axis.text = element_text(size = 12),
    axis.title = element_text(size = 15),
    strip.text = element_text(size = 15),
    strip.background = element_blank()
  )


ggsave("Supplementary Figure 2 - LoQs.png", height = 6, width = 10)

```


## Data input

```{r Data input}
data <- read.csv("Picard_Spatial_Survey_data.csv") #see paper supplementary info

data %<>%
  mutate(Norm_Perch = case_when(
          SampleType == "Sediment" ~ (Conc.uL_Perch *(22.45/6)*100)/(Sample.Amount.g.mL*Sediment.Content),
          SampleType == "Water" ~ (Conc.uL_Perch *(22.45/6)*100)/Sample.Amount.g.mL),
         Norm_Rudd = case_when(
          SampleType == "Sediment" ~ (Conc.uL_Rudd *(22.45/6)*100)/(Sample.Amount.g.mL*Sediment.Content),
          SampleType == "Water" ~ (Conc.uL_Rudd *(22.45/6)*100)/Sample.Amount.g.mL))

data %<>%
  group_by(Lake, Site, SampleType) %>%
  mutate(Sum_R = sum(Binary_Rudd_ddPCR),
         Sum_P = sum(Binary_Perch_ddPCR)) %>%
  ungroup()

levels(as.factor(data$Lake))
#"Pounui"     "Tomarata"   "Waitawa" 
levels(as.factor(data$SampleType))
#"Sediment"  "Water"

levels(as.factor(data$Location))
#"Littoral" "Pelagic" -> littoral = near shore and pelagic = mid-lake

levels(as.factor(data$Site))
#1 to 14

levels(as.factor(data$Replicate))
#"A" "B"

data_long <- data %>%
  relocate(Location, .after = SampleType) %>%
  select(!c(Binary_Perch_ddPCR:Counts_Rudd_metaB, Sample.ID:Sediment.Content)) %>%
  pivot_longer(
    cols = c(Conc.uL_Perch:Norm_Rudd),
    names_to = c(".value", "Target"),
    names_pattern = "(.+)_(.+)") %>%
  rename(Raw_Conc = Conc.uL,
         Norm_Conc = Norm,
         Raw_PoissonConfMin = PoissonConfMin,
         Raw_PoissonConfMax = PoissonConfMax) %>%
  mutate(Norm_PoissonConfMin = Raw_PoissonConfMin*Norm_Conc/Raw_Conc,
         Norm_PoissonConfMax = Raw_PoissonConfMax*Norm_Conc/Raw_Conc)

```

## Results

### Overall detections

```{r Overall detections}
colnames(data)
glimpse(data)
colnames(data_long)
glimpse(data_long)

#Perch detected in which lake(s)
SS <- data_long%>%filter(Target == "Perch")%>% filter(Raw_Conc >0)
levels(as.factor(SS$Lake))

#Rudd detected in which lake(s)
SS <- data_long%>%filter(Target == "Rudd")%>% filter(Raw_Conc >0)
levels(as.factor(SS$Lake))


#Fish eDNA levels depending on sample type
SS <- data_long%>%filter(SampleType=="Sediment")%>%filter(Raw_Conc>0)%>%select(Raw_Conc, Target)
max(SS$Raw_Conc)
min(SS$Raw_Conc)

SS <- data_long%>%filter(SampleType=="Water")%>%filter(Raw_Conc>0)%>%select(Raw_Conc, Target)
max(SS$Raw_Conc)
min(SS$Raw_Conc)



```


### Detection comparison for Perch and Rudd

```{r Detection comparison for Perch and Rudd}
#For perch
cat("Perch eDNA was detected in ",nrow(data%>%filter(Binary_Perch_ddPCR>0)) / nrow(data) * 100,"% of all biological replicates (", nrow(data%>%filter(Binary_Perch_ddPCR>0))," out of ",nrow(data)," samples) across all lakes and at ",nrow(data%>%filter(Binary_Perch_ddPCR>0) %>% group_by(Lake, Site) %>% summarise())," out of 42 sites (69 %). This included all sites in Lakes Pounui and Waitawa, and one site in Lake Tomarata despite perch not been known to occur in this lake.")

#Testing
#Pounui
kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(Location), data = subset(data, Lake %in% "Pounui") %>%
  drop_na(Conc.uL_Perch)) #chi-squared = 15.296, df = 1, p-value = 9.19e-05

kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(SampleType), data = subset(data, Lake %in% "Pounui") %>%
  drop_na(Conc.uL_Perch)) #chi-squared = 3.4017, df = 1, p-value = 0.06513

SS <- data_long %>% filter(Raw_Conc >0, Lake == "Pounui") %>% select(Raw_Conc, Target, SampleType, Location)

#So differences concentration due to location (Higher in littoral sites) but not sample type


#Waitawa
kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(Location), data = subset(data, Lake %in% "Waitawa") %>%
  drop_na(Conc.uL_Perch)) #chi-squared = 6.4486, df = 1, p-value = 0.166
kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(SampleType), data = subset(data, Lake %in% "Waitawa") %>%
  drop_na(Conc.uL_Perch)) #6.4486, df = 1, p-value = 0.0111

SS <- data_long %>% filter(Raw_Conc >0, Lake == "Waitawa", Target == "Perch") %>% select(Raw_Conc, Target, SampleType, Location, Site)
#the difference in detection is skewed by really high levels in sediment samples of sites 9 and 12. If we remove these:

SS <- data_long %>% filter(Raw_Conc < 5, Lake == "Waitawa", Target == "Perch") %>% select(Raw_Conc, Target, SampleType, Location, Site)
kruskal.test(log10(Raw_Conc+1) ~ as.factor(SampleType), data = SS)
#water samples become significantly higher - chi-squared = 10.067, df = 1, p-value = 0.00151






#For rudd

cat("Rudd eDNA was detected in ",nrow(data%>%filter(Binary_Rudd_ddPCR>0)) / nrow(data) * 100,"% of all biological replicates (", nrow(data%>%filter(Binary_Rudd_ddPCR>0))," out of ",nrow(data)," samples) across all lakes and at ",nrow(data%>%filter(Binary_Rudd_ddPCR>0) %>% group_by(Lake, Site) %>% summarise())," of 42 sites (50%).")


#Tomarata
kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(Location), data = subset(data, Lake %in% "Tomarata") %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 0.86305, df = 1, p-value = 0.3529
kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(SampleType), data = subset(data, Lake %in% "Tomarata") %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 2.3558, df = 1, p-value = 0.1248

SS <- data_long %>% filter(Raw_Conc >0, Lake == "Tomarata") %>% select(Raw_Conc, Target, SampleType, Location)

#So no differences in eDNA concentrations anywhere





#Waitawa
kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(Location), data = subset(data, Lake %in% "Waitawa") %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 1.8447, df = 1, p-value = 0.1744

kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(SampleType), data = subset(data, Lake %in% "Waitawa") %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 20.699, df = 1, p-value = 5.375e-06

SS <- data_long %>% filter(Lake == "Waitawa", Target == "Rudd") %>% select(Raw_Conc, Target, SampleType, Location, Site)
#Only significant difference is in rudd eDNA raw concentration (per sample) depending on the sample type





### Detections per location zone ###
SS <- data %>%
  select(!ID.Counts) %>%
  group_by(Lake, Location, SampleType) %>%
  summarise(Perch = sum(Binary_Perch_ddPCR)) %>% 
  pivot_wider(names_from = Location, values_from = Perch)

SS <- data %>%
  select(!ID.Counts) %>%
  group_by(Lake, Location, SampleType) %>%
  summarise(Rudd = sum(Binary_Rudd_ddPCR)) %>% 
  pivot_wider(names_from = Location, values_from = Rudd)

#Raw eDNA concentrations
#Total min and max
min(data%>%filter(Conc.uL_Perch>0)%>%select(Conc.uL_Perch))
max(data%>%filter(Conc.uL_Perch>0)%>%select(Conc.uL_Perch))
min(data%>%filter(Conc.uL_Rudd>0)%>%select(Conc.uL_Rudd))
max(data%>%filter(Conc.uL_Perch>0)%>%select(Conc.uL_Rudd))

#Min and max per sample type
min(data_long%>%filter(SampleType=="Sediment")%>%filter(Raw_Conc>0)%>%select(Raw_Conc))
min(data_long%>%filter(SampleType=="Water")%>%filter(Raw_Conc>0)%>%select(Raw_Conc))
max(data_long%>%filter(SampleType=="Sediment")%>%filter(Raw_Conc>0)%>%select(Raw_Conc))
max(data_long%>%filter(SampleType=="Water")%>%filter(Raw_Conc>0)%>%select(Raw_Conc))



#Differences in detection rates depending on sample type for perch

cat("In Lake Pounui, sediment samples yielded more detections than water (",
    nrow(data %>% filter(Lake=="Pounui", SampleType=="Sediment", Binary_Perch_ddPCR>0))/28*100
,"% detection in sediment vs. ",
    nrow(data %>% filter(Lake=="Pounui", SampleType=="Water", Binary_Perch_ddPCR>0))/28*100,
"% detection in water), and it was the opposite in Lake Waitawa (",
  nrow(data %>% filter(Lake=="Waitawa", SampleType=="Sediment", Binary_Perch_ddPCR>0))/28*100,"% sediment, "
  ,nrow(data %>% filter(Lake=="Waitawa", SampleType=="Water", Binary_Perch_ddPCR>0))/28*100,"% water). Spatial patchiness depended on the lake: eDNA was patchy in water for Lake Pounui and found at all sites by sediment, while it was the opposite for Lake Waitawa. ")






#Differences in concentrations depending on sample type for perch
cat("Perch eDNA was"
,min(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  max(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0)%>%select(Raw_Conc)),
"to"
,max(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  min(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)),
"times more concentrated in sediment compared to water samples (Figure 2).")


#Differences in concentrations for rudd
cat("Rudd eDNA was"
,min(data_long%>%filter(SampleType=="Sediment", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  max(data_long%>%filter(SampleType=="Water", Target =="Rudd")%>%filter(Raw_Conc>0)%>%select(Raw_Conc)),
"to"
,max(data_long%>%filter(SampleType=="Sediment", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  min(data_long%>%filter(SampleType=="Water", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)),
"times more concentrated in sediment compared to water samples (Figure 2).")


#Rudd raw conc comparison sediment water
cat("Rudd eDNA was"
,min(data_long%>%filter(SampleType=="Sediment", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / max(data_long%>%filter(SampleType=="Water", Target =="Rudd")%>%filter(Raw_Conc>0)%>%select(Raw_Conc)),
"to"
,max(data_long%>%filter(SampleType=="Sediment", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  min(data_long%>%filter(SampleType=="Water", Target =="Rudd")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)),
"times more concentrated in sediment compared to water samples (Figure 2).")


#Perch raw conc comparison sediment water
cat("Perch eDNA was"
,min(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / max(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0)%>%select(Raw_Conc)),
"to"
,max(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  min(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)),
"times more concentrated in sediment compared to water samples (Figure 2).")

cat("Perch eDNA was"
,min(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  max(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0)%>%select(Raw_Conc)),
"to"
,max(data_long%>%filter(SampleType=="Water", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)) / 
  min(data_long%>%filter(SampleType=="Sediment", Target =="Perch")%>%filter(Raw_Conc>0) %>%select(Raw_Conc)),
"times more concentrated in water compared to sediment samples (Figure 2).")



```





```{r Statistical differences raw values (published manuscript)}

#All lakes
kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(Location), data = data %>%
  drop_na(Conc.uL_Perch)) #chi-squared = 6.4135, df = 1, p-value = 0.01133

kruskal.test(log10(Conc.uL_Perch+1) ~ as.factor(SampleType), data = data %>%
  drop_na(Conc.uL_Perch)) #chi-squared = 0.21174, df = 1, p-value = 0.6454

kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(Location), data = data %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 0.22325, df = 1, p-value = 0.6366

kruskal.test(log10(Conc.uL_Rudd+1) ~ as.factor(SampleType), data = data %>%
  drop_na(Conc.uL_Rudd)) #chi-squared = 5.0961, df = 1, p-value = 0.02398

kruskal.test(log10(Raw_Conc+1) ~ as.factor(SampleType), data = data_long %>%
  drop_na(Raw_Conc))
```









```{r Counts boxplots - Raw concentrations}

ggplot(data_long, aes(x = SampleType, y = Raw_Conc)) +
  geom_boxplot(data = data_long, mapping = aes(x = SampleType, y = Raw_Conc, fill = Location)) +
  scale_fill_manual(labels = c("Near-shore", "Mid-lake"), values = c("#BCE123", "#23B1E1")) +
  #scale_y_continuous(trans=scales::pseudo_log_trans(base = 10), 
   #                  breaks = c(0, 10, 100, 1000, 3000)) +
   scale_y_continuous(trans="sqrt") +
  #scale_y_log10() +#expand = expansion(mult = c(0, 0.1))) +
  facet_grid(Lake ~ Target, drop = TRUE) + #, scales = "free") + 
  labs(x = NULL, y = "Target gene levels (gene copy numbers per gram)") +
  theme_bw() +
  #stat_compare_means(method = "anova", label.y = 1)+      # Add global p-value
  #stat_compare_means(label = "p.signif", method = "t.test", hide.ns = TRUE)  +
  theme(panel.grid.minor = element_blank(),
        axis.text = element_text(size = 12),
        axis.title = element_text(size = 15),
        legend.title = element_text(size = 15),
        legend.text = element_text(size = 12),
        legend.position = "bottom",
        strip.text = element_text(size = 15),
        strip.placement = "outside",
        strip.background = element_blank())


ggsave("Figure 2 final.png", width = 10, height = 11)
```



```{r Figure S3 - Counts boxplot normalised concentration}

#And the significant differences across locations are...:


SS <- data_long %>% filter(Lake == "Pounui") %>%
  filter(SampleType == "Sediment", Target == "Perch") %>%
  unite(col = "IDss", c(Site, Replicate), sep = "_", remove = FALSE)
wilcox.test(Raw_Conc~ Location, data = SS, paired = TRUE)
#Pounui Water Litto vs Pel Perch: V = 87, p-value = 0.004166
#Pounui Sediment Lito vs Pel Perch: V = 96, p-value = 0.004028

tidy(SS)

#Waitawa - Sediment = 
#Pounui Sed Lito vs Pel Perch: V = 217, p-value = 0.0001049 -- significant
#Pounui Water Lito vs Pel Perch: V = 84, p-value = 0.007915 -- significant
#Tomarata Sed Lito vs Pel Rudd: V = 5, p-value = 0.08006
#Pounui Water Lito vs Pel Rudd: V = 1, p-value = 1
#Waitawa Sed Lito vs Pel Perch: V = 143, p-value = 0.01305 -- significant
#Waitawa Water Lito vs Pel Perch: V = 65, p-value = 0.4631
#Waitawa Sed Lito vs Pel Rudd: V = 94, p-value = 0.05708
#Waitawa Water Lito vs Pel Rudd: V = 68, p-value = 0.3575

wilcox_test <- tibble(Lake = c("Pounui", "Pounui"),
                           Target = c("Perch", "Perch"),
                           SampleType = c("Sediment", "Water"),
                           .y. = rep("Norm_Conc", 2),
                           group1 = rep("Littoral", 2),
                           group2 = rep("Pelagic", 2),
                           Location = rep("c(Littoral, Pelagic)", 2),
                           label = c("< 0.01", "0.01")) %>% group_by(SampleType, Location)



ggplot(data_long, aes(x = SampleType, y = Raw_Conc, fill = Location)) +
  geom_boxplot() +
  scale_fill_manual(labels = c("Near-shore", "Mid-lake"), values = c("#BCE123", "#23B1E1")) +
  #scale_y_continuous(trans=scales::pseudo_log_trans(base = 10)) + #, 
                     #breaks = c(0, 10, 100, 1000, 3000)) +
  scale_y_continuous(trans="sqrt", 
                     breaks = c(0, 1, 2, 3, 4, 5, 6)) + #,
  facet_grid(Lake ~ Target, drop = TRUE) + #, scales = "free") + 
  labs(x = NULL, y = "Target gene levels (gene copy numbers per microliter)") +
  theme_bw() +
  #stat_compare_means(method = "anova", label.y = 1)+      # Add global p-value
  #stat_compare_means(label = "p.signif", method = "t.test", hide.ns = TRUE)  +
  theme(panel.grid.minor = element_blank(),
        axis.text = element_text(size = 12),
        axis.title = element_text(size = 15),
        legend.title = element_text(size = 15),
        legend.text = element_text(size = 12),
        legend.position = "bottom",
        strip.text = element_text(size = 15),
        strip.placement = "outside",
        strip.background = element_blank())




ggsave("Figure S1.png", width = 10, height = 11)



```


```{r Binary detection}
#Checking binary detections

SS <- data %>%
  filter(Binary_Perch_ddPCR >0) %>%
  distinct(Sample.ID, Site, Lake)

dim(SS) #88 samples with Perch eDNA

SS <- data %>%
  filter(Binary_Perch_ddPCR >0) %>%
  distinct(Site, Location, Lake)

dim(SS) #29 sites with Perch eDNA

SS <- data %>%
  filter(Binary_Rudd_ddPCR >0) %>%
  distinct(Sample.ID,Site, Lake)

dim(SS) #46 samples with Rudd eDNA


SS <- data %>%
  filter(Binary_Rudd_ddPCR >0) %>%
  distinct(Site, Lake)

dim(SS) #21 sites with Rudd eDNA

#Sites with detections per fish species
SS <- data %>%
  select(!ID.Counts) %>%
  group_by(Lake, Site, SampleType) %>%
  summarise(Perch = sum(Binary_Perch_ddPCR)) %>%
  pivot_wider(names_from = SampleType, values_from = Perch) %>%
  filter(Sediment >0 & Water > 0)

SS <- data %>%
  select(!ID.Counts) %>%
  group_by(Lake, Site, SampleType) %>%
  summarise(Rudd = sum(Binary_Rudd_ddPCR)) %>%
  pivot_wider(names_from = SampleType, values_from = Rudd) %>%
  filter(Sediment >0 & Water > 0)
```


```{r Heatmap}
require(hrbrthemes)
require(RColorBrewer)

perch <- c("#F6E0DA", "#EF8F76", "#CB5334")
rudd <- brewer.pal(4, "YlGnBu")


#data$Sum_P <- factor(data$Sum_P, levels(c(3,2,1,0)))
P2 <- data %>%
  mutate(Location = case_when(
    Location == "Littoral" ~ "Near-shore",
    Location == "Pelagic" ~ "Mid-lake")) %>%
ggplot(., aes(SampleType, as.factor(Site), fill= as.factor(Sum_P))) + 
  geom_tile(colour="white", size=0.25) +
  facet_grid(Location ~ Lake, scales = "free_y") +
  scale_fill_manual(name = "Positive detections", values=perch)+
  scale_y_discrete(limits = rev)+
  guides(fill = guide_legend(label.vjust = 0.5)) +
  labs(x=NULL, y = "Site") +
  theme_bw() +
  theme(panel.grid.major = element_blank(),
        panel.grid.minor = element_blank(),
        axis.text = element_text(size = 14),
        axis.title = element_text(size = 16),
        legend.title = element_text(size = 16),
        legend.text = element_text(size = 14),
        strip.text = element_text(size = 17),
        strip.placement = "outside",
        strip.background = element_blank())
        
#ggsave('Figure 3 Heatmap perch.png', device = 'png', width = 12, height = 7)




R2 <- data %>%
  mutate(Location = case_when(
    Location == "Littoral" ~ "Near-shore",
    Location == "Pelagic" ~ "Mid-lake")) %>%
ggplot(., aes(SampleType, as.factor(Site), fill= as.factor(Sum_R))) + 
  geom_tile(colour="white", size=0.25) +
  facet_grid(Location ~ Lake, scales = "free_y") +
  scale_fill_manual(name = "Positive detections", values=rudd)+
  scale_y_discrete(limits = rev)+
  scale_x_discrete(position = "top") +
  guides(fill = guide_legend(label.vjust = 0.5)) +
  labs(x=NULL, y = "Site") +
  theme_bw() +
  theme(panel.grid.major = element_blank(),
        panel.grid.minor = element_blank(),
        axis.text = element_text(size = 14),
        axis.title = element_text(size = 16),
        legend.title = element_text(size = 16),
        legend.text = element_text(size = 14),
        strip.text = element_text(size = 17),
        strip.placement = "outside",
        strip.background = element_blank(),
        strip.text.x = element_blank(),
        axis.text.x = element_blank())
        
#ggsave('Figure 3 Heatmap rudd.png', device = 'png', width = 12, height = 7)


ggarrange(P2, R2, ncol = 1, labels = c("A.", "B."))
ggsave('Figure 3 Heatmap.png', device = 'png', width = 12, height = 10)
```


### Occupancy modelling

#### With the Presence software

Information summarised in Table 3 (see paper) was generated using the Presence software - https://www.mbr-pwrc.usgs.gov/software/presence.html (no script)
Contact me if you'd like to know more


#### Simulations



```{r Custom simulation formula, include=FALSE}

#run the function below

evaldesign <- function(psi,p,s,k,nits=10000,doprint=TRUE,doplot=TRUE) {
  
  t1=proc.time()
  
  mydist<-matrix(NA,0,5)
  jj<-1
  
  for (ii in 1:nits)   
  {
    # generate a history
    Sp<-rbinom(1,s,psi)
    hist<-rbind(matrix(rbinom(Sp*k,1,p),Sp,k),matrix(0,s-Sp,k))
    
    # summarize history (sufficient statistics)
    SD<-sum(rowSums(hist)>0)
    d<-sum(hist)
    
    # count history
    if (jj==1)
    {
      mydist<-rbind(mydist,c(SD,d,1,NA,NA))
    } else 
    {
      temp<-which((mydist[1:nrow(mydist),1]==SD))
      temp2<-which((mydist[temp,2]==d))	
      if (length(temp2)==0)
      {
        mydist<-rbind(mydist,c(SD,d,1,NA,NA))
      } else 
      { 
        #add one count to this {SD,d} history 
        mydist[temp[temp2],3]<-mydist[temp[temp2],3]+1
      }
    }	
    jj<-jj+1
  }		#end for nits
  
  
  # analyze each history type obtained in simulation
  for (ii in 1:nrow(mydist))
  {  
    SD<-mydist[ii,1]
    d<-mydist[ii,2]
    
    if (SD==0){			  #empty histories
      psihat<-NA
      phat<-NA
    } else 
    {	
      if (((s-SD)/s)<((1-d/(s*k))^k)) {	#boundary estimates
        psihat <- 1
        phat <- d/(s*k)
      } else
      {
        params = c(psi,p)   #init vals for optim
        fitted1 = optim(params,loglikf,SD=SD,d=d,s=s,k=k,lin=0)
        psihat <- 1/(1+exp(-fitted1$par[1]))      
        phat <- 1/(1+exp(-fitted1$par[2])) 
      }
    }
    
    # store estimates
    mydist[ii,4]<-psihat
    mydist[ii,5]<-phat   
    mydist[ii,3]<-mydist[ii,3]/nits
  }
  
  # MLE properties: distribution removing empty histories
  mydist2<-mydist[mydist[,1]!=0,]	
  mydist2[,3]<-mydist2[,3]/sum(mydist2[,3])
  mymeanpsi<-sum(mydist2[,3]*mydist2[,4],na.rm='true')
  mybiaspsi<-mymeanpsi-psi
  myvarpsi<-sum((mydist2[,4]-mymeanpsi)^2*mydist2[,3])
  myMSEpsi<-myvarpsi+mybiaspsi^2
  mymeanp<-sum(mydist2[,3]*mydist2[,5],na.rm='true')
  mybiasp<-mymeanp-p
  myvarp<-sum((mydist2[,5]-mymeanp)^2*mydist2[,3])
  myMSEp<-myvarp+mybiasp^2
  mycovar<-sum((mydist2[,5]-mymeanp)*(mydist2[,4]-mymeanpsi)*mydist2[,3])
  mycritA<-myMSEpsi+myMSEp
  mycritD<-myMSEpsi*myMSEp-mycovar^2
  
  # MLE properties: distribution removing also boundary estimates
  mydist3<-mydist2[mydist2[,4]!=1,]	
  mydist3[,3]<-mydist3[,3]/sum(mydist3[,3])
  mymeanpsi_B<-sum(mydist3[,3]*mydist3[,4],na.rm='true')
  mybiaspsi_B<-mymeanpsi_B-psi
  myvarpsi_B<-sum((mydist3[,4]-mymeanpsi_B)^2*mydist3[,3])
  myMSEpsi_B<-myvarpsi_B+mybiaspsi_B^2
  mymeanp_B<-sum(mydist3[,3]*mydist3[,5],na.rm='true')
  mybiasp_B<-mymeanp_B-p
  myvarp_B<-sum((mydist3[,5]-mymeanp_B)^2*mydist3[,3])
  myMSEp_B<-myvarp_B+mybiasp_B^2
  mycovar_B<-sum((mydist3[,5]-mymeanp_B)*(mydist3[,4]-mymeanpsi_B)*mydist3[,3])
  mycritA_B<-myMSEpsi_B+myMSEp_B
  mycritD_B<-myMSEpsi_B*myMSEp_B-mycovar_B^2
  
  pempty<-100*mydist[mydist[,1]==0,3]
  if (!length(pempty)) pempty=0
  pbound<-100*sum(mydist[mydist[,4]==1,3],na.rm='true')
  if (!length(pbound)) pbound=0
  
  # compute processing time
  t2=proc.time()
  
  # print in screen results
  if (doprint) {
    cat("\n--------------------------------------------------------------------------\n",sep = "")
    cat("Evaluation of design K = ",k," S = ",s, " (TS = ",s*k,")\n",sep = "")
    cat("--------------------------------------------------------------------------\n",sep = "")
    cat("estimator performance (excl empty histories)\n",sep = "")
    cat("psi: bias = ",sprintf("%+0.4f",mybiaspsi),"   var = ",sprintf("%+0.4f",myvarpsi),"   MSE = ",sprintf("%+0.4f",myMSEpsi),"\n",sep = "")
    cat("  p: bias = ",sprintf("%+0.4f",mybiasp),"   var = ",sprintf("%+0.4f",myvarp),"   MSE = ",sprintf("%+0.4f",myMSEp),"\n",sep = "")
    cat("    covar = ",sprintf("%+0.4f",mycovar)," critA = ",sprintf("%+0.4f",mycritA)," critD = ",sprintf("%+.3e",mycritD),"\n",sep = "")
    cat("estimator performance (excl also histories leading to boundary estimates)\n",sep = "")
    cat("psi: bias = ",sprintf("%+0.4f",mybiaspsi_B),"   var = ",sprintf("%+0.4f",myvarpsi_B),"   MSE = ",sprintf("%+0.4f",myMSEpsi_B),"\n",sep = "")
    cat("  p: bias = ",sprintf("%+0.4f",mybiasp_B),"   var = ",sprintf("%+0.4f",myvarp_B),"   MSE = ",sprintf("%+0.4f",myMSEp_B),"\n",sep = "")
    cat("    covar = ",sprintf("%+0.4f",mycovar_B)," critA = ",sprintf("%+0.4f",mycritA_B)," critD = ",sprintf("%+.3e",mycritD_B),"\n",sep = "")
    cat(" empty histories = ",sprintf("%0.1f",pempty),"%\n",sep = "")
    cat(" boundary estimates = ",sprintf("%0.1f",pbound),"%\n",sep = "")
    cat("this took ", (t2-t1)[1],"seconds \n")
    cat("--------------------------------------------------------------------------\n\n",sep = "")
  }
  
  # write results 
  myres <- list(dist=mydist,biaspsi=mybiaspsi,varpsi=myvarpsi,MSEpsi=myMSEpsi,biasp=mybiasp,varp=myvarp,
                MSEp=myMSEp,covar=mycovar,critA=mycritA,critD=mycritD,biaspsi_B=mybiaspsi_B,varpsi_B=myvarpsi_B,
                MSEpsi_B=myMSEpsi_B,biasp_B=mybiasp_B,varp_B=myvarp_B,MSEp_B=myMSEp_B,covar_B=mycovar_B,
                critA_B=mycritA_B,critD_B=mycritD_B,pempty=pempty,pbound=pbound)
  
  if (doplot) plotMLEdist(myres$dist,p,psi)
  
  return(myres)
  
}

loglikf <- function(params, SD, d, s, k, lin) {
  
  if (lin){
    psi         = params[1]   
    p           = params[2]  
  } else {
    psi         = 1/(1+exp(-params[1]))  
    p           = 1/(1+exp(-params[2]))
  }
  
  loglik = -(SD*log(psi)+d*log(p)+(k*SD-d)*log(1-p)+(s-SD)*log((1-psi)+psi*(1-p)^k))
}

plotMLEdist <- function(mydist, p, psi)
{        
  mcol<-rev(rainbow(200))
  mcol<-c(mcol[52:200],mcol[1:16])
  range=cbind(1,length(mcol))	
  x <-mydist[,3]
  y <-as.integer((range[2]-range[1])*(x-min(x))/(max(x)-min(x)))+1
  plot(mydist[,5],mydist[,4],col=mcol[y],xlim=cbind(0,1),ylim=cbind(0,1),xlab=expression(hat("p")),
       ylab=expression(hat("psi")),pch=19)
  abline(v=p, col="lightgray")
  abline(h=psi, col="lightgray")
}



```


Running the simulations, where psi = psi (large-scale occupancy), p = fish eDNA detection probability, s = number of sites, k = number of replicates.

```{r Evaluating simulation trends}
#How many minimum meaningful sites (s) for a range of detection probabilities (p) if 2 replicates (k), psi = 1, empty histories = 0%, and error <0.5%

myres<-evaldesign(psi=1,p=0.9,s=2,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.8,s=3,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.7,s=4,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.6,s=5,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.5,s=6,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.4,s=8,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.3,s=16,k=2,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.25,s=50,k=2,nits=10000,doprint=1,doplot=1)


#How many minimum meaningful sites (s) for a range of detection probabilities (p) if 3 replicates (k), psi = 1, empty histories = 0%, and error <0.5%
myres<-evaldesign(psi=1,p=0.9,s=2,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.8,s=2,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.7,s=3,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.6,s=3,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.5,s=4,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.4,s=5,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.3,s=7,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.25,s=9,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.2,s=15,k=3,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.15,s=32,k=3,nits=10000,doprint=1,doplot=1)

#How many minimum meaningful sites (s) for a range of detection probabilities (p) if 4 replicates (k), psi = 1, empty histories = 0%, and error <0.5%
myres<-evaldesign(psi=1,p=0.9,s=2,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.8,s=2,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.7,s=2,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.6,s=2,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.5,s=3,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.4,s=4,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.3,s=6,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.25,s=7,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.2,s=10,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.15,s=16,k=4,nits=10000,doprint=1,doplot=1)
myres<-evaldesign(psi=1,p=0.1,s=40,k=4,nits=10000,doprint=1,doplot=1)

sim_data <- data.frame(Detect.Prob=c(0.9, 0.8, 0.7, 0.6, 0.5, 0.4, 0.3, 0.25, 0.2, 0.15, 0.1),
                  Two = c(2,3,4,5,6,8,16,50,NA,NA,NA),
                  Three = c(2,2,3,3,4,5,7,9,15,32, NA),
                  Four = c(2,2,2,2,3,4,6,7,10,16,40)) %>%
  pivot_longer(cols= Two:Four, names_to = "ReplicateN", values_to = "SiteN")

sim_data$ReplicateN <- factor(sim_data$ReplicateN, levels = c("Two", "Three", "Four"))

glimpse(sim_data)
```





```{r Simulation for worse-case fish detection probability (Lake Tomarata)}
#Probability detection for Lake Tomarata / sediment samples: p=0.21
#The number of sites (s) and replicates (k) was adjusted until psi MSE <0.05 and empty histories = 0%
myres<-evaldesign(psi=1,p=0.21,s=6,k=5,nits=10000,doprint=1,doplot=1)

#Probability detection for Lake Tomarata / water samples: p=0.07
#The number of sites (s) and replicates (k) was adjusted until psi MSE <0.05 and empty histories = 0%
myres<-evaldesign(psi=1,p=0.07,s=20,k=8,nits=10000,doprint=1,doplot=1)
```



```{r Simulation for Lake Pounui and Lake Waitawa }
#Best probability detection for perch eDNA Lake Pounui: p = 0.99 (sediment samples)
#But the simulations can only calculate p=<0.97, so we ran the simulations at 0.97
myres<-evaldesign(psi=1,p=0.97,s=2,k=2,nits=10000,doprint=1,doplot=1)
#we selected 2 sites and 2 replicates because occupancy modelling needs replication (>1 item)

#Best probability detection for perch eDNA in Lake Waitawa : p = 0.89
myres<-evaldesign(psi=1,p=0.89,s=2,k=2,nits=10000,doprint=1,doplot=1)
#Best probability detection for perch eDNA in Lake Waitawa : p = 0.93
myres<-evaldesign(psi=1,p=0.93,s=2,k=2,nits=10000,doprint=1,doplot=1)
```



```{r Plotting simulation trends and individual lakes}


ggplot()+
  geom_point(data = sim_data, aes(x = Detect.Prob, y = SiteN, color = ReplicateN, shape=ReplicateN), size = 2.5) +
  geom_line(data = sim_data, aes(x = Detect.Prob, y = SiteN, color = ReplicateN, shape=ReplicateN)) +
  #adding where Tomarata, Waitawa, and Pounui would be
  geom_star(aes(x = c(0.89, 0.93, 0.99), y = 2), starshape=1, fill= "#F6D13C", color = "#D1AA0A", size=4) +
  geom_star(aes(x = c(0.07, 0.21), y = c(20, 6)), starshape=1, fill= "#F6D13C", color = "#D1AA0A", size=4) +
  scale_color_manual(name = "Number of replicates", values = c("#92DA10", "#DA4489", "#6195ED")) +#  "#DAA944")) +
  scale_shape_manual(name = "Number of replicates",
                     #labels = c("Control, Non-F", "Control, Flwr", "Exclosure, Non-F", "Exclosure, Flwr"),
                     values = c(15,16,17)) +
  labs(x= "Fish eDNA detection probabilities (p)", y = "Minimum meaningful number of sites", legend="Number of Replicates") + 
  theme_bw() + 
  theme(text = element_text(size = 18),
        axis.title = element_text(size = 18),
        axis.text = element_text(size = 17),
        legend.text = element_text(size = 18))
  

ggsave("Figure 4 final.png", height = 8, width = 12)


```

