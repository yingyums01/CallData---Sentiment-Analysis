# CallData---Sentiment-Analysis
---
title: "CallData - Sentiment Analysis"
author: 'Doris Kuo'
date: "02/06/2020"
output:
  radix::radix_article:
    toc: true
    toc_depth: 2
---


```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

# The Data


Unzip Data

```{r}
#unzip("wweCalls.zip")
```



## Step 1: CLeaning

Read all of the parsed transcripts into R.
read all files

```{r}

#get the filepath
getwd()

#get all files name in a path
file_name = list.files("C:/Users/yingy/OneDrive/Desktop/ND/2019 Fall course/3rd module/unstructured data/assignment/A1")
file_name
#file_name = file_name[5:37]
#file_name

#concat every table together
#raw <- lapply(file_name,function(x){
#  raw_s = as.data.frame(read.csv(x))
#  return(raw_s)
#}
#  )

#raw <- do.call("rbind",raw)
#write.csv(raw,"total.csv")
```

```{r}
raw<-read.csv("total.csv")
```



Perform some initial exploration of the text and perform any initial cleaning.

```{r}
colnames(raw)

sort(unique(raw$title))

library(dplyr)

# check current level for factor variable 
# preparation for cleaning NA
length(levels(raw$title))

# Get levels and add "None"
levels <- levels(raw$title)
levels[length(levels) + 1] <- "Unknown"

# refactor title to include "Unknown" as a factor level
# and replace NA with "Unknown"
raw$title <- factor(raw$title, levels = levels)
raw$title[is.na(raw$title)] <- "Unknown"
length(levels(raw$title))

#have a better understanding of NA in title: 
#refer to the name column
raw%>%
  filter(title=="Unknown")%>%
  select(title,name,firstName)%>%
  count(name)%>%
  arrange(desc(n))

#if the name is operator(the majority), change the the title from unknown to operator
raw%>%
  mutate(title=as.character(title))%>%
  mutate(title=ifelse((name=="Operator" & title=="Unknown"), "Operator",title))%>%
  filter(title=="Unknown")%>%
  select(title,name,firstName)%>%
  count(name)%>%
  arrange(desc(n))


raw<- raw%>%
  mutate(title=as.character(title))%>%
  mutate(title=ifelse((name=="operator" & title=="Unknown"), "Operator",title))

unique(raw$title)

# integrate and clean the title
raw<- raw%>%
  mutate(title = ifelse( (grepl("VP|Vice President",title) | grepl("IR|Investor Relations",title) ), "VP/IR", title ))%>%
  mutate(title = ifelse( grepl("CEO|Chief Executive Officer",title), "CEO", title ))%>%
  mutate(title = ifelse( (grepl("Planning",title) & grepl("Analysis",title) ), "Planning/Analysis", title ))%>%
  mutate(title = ifelse( grepl("Analyst",title), "Analyst", title ))%>%
  mutate(title = ifelse( (grepl("Chief",title) & grepl("Fin|Fia",title) ), "CFO", title ))%>%
  mutate(title = ifelse( grepl("CFO",title), "CFO", title ))%>%
  mutate(title = ifelse( grepl("I think",title), "Unknown", title ))

#check the title
unique(raw$title)
```



## Step 2: Sentiment Analysis

Perform sentiment analyses on the texts. 
### lexicon_nrc
```{r}
library(textdata)
library(sentimentr)
# nrc: different emotion
nrcWord <- textdata::lexicon_nrc()
head(nrcWord)

#select one emotion I want in nrc: trust 
#extract trust word:
nrcWord_trust <- nrcWord%>%
  filter(sentiment=="trust")
head(nrcWord_trust)

#get score in hash_sentiment_nrc
nrcValues <- lexicon::hash_sentiment_nrc
head(nrcValues) #the score table
trust_score <- nrcValues[nrcValues$x %in% nrcWord_trust$word,]
head(trust_score)

#get my df with title and text
df_title <- raw%>%
  select(title,text)


title_trust_Sentiment <- sentiment(get_sentences(df_title), 
          polarity_dt = trust_score) %>% 
  group_by(title) %>% 
  summarize(meanTrust = mean(sentiment))%>%
  arrange(desc(meanTrust))
head(title_trust_Sentiment)

```


### lexicon_nrc_vad: select dominance
positive, excitement, dominance

```{r}
library(sentimentr)
nrcDominance <- textdata::lexicon_nrc_vad()
head(nrcDominance)

#select one emotion I want : dominance
#extract
nrcDominance<- nrcDominance%>%
  select(Word,Dominance)
head(nrcDominance)

#dominance is the score already


#Just don't know why it can't work........................ 
#>> just select out the words in nrcdominance without weight

class(nrcDominance)
nrcDominance<-nrcDominance%>%
  rename(x=Word)%>%
  rename(y=Dominance)

#sentiment (popularity_dt need to be a data.table with only x, y columns)
nrcDominance <- data.table::as.data.table(nrcDominance)
nrcDominance

newshifters <- lexicon::hash_valence_shifters
newshifters <- newshifters[!(newshifters$x %in% nrcDominance$x),]

#get my df with title and text
df_title <- raw%>%
  select(title,text)

#Calculate dominant score across title
?sentiment
title_dominance_Sentiment <- sentiment(get_sentences(as.character(df_title$text)), polarity_dt = nrcDominance, valence_shifters_dt = newshifters) %>% 
  group_by(title) %>% 
  summarize(meanDominance = mean(sentiment))%>%
  arrange(desc(meanDominance))
head(title_dominance_Sentiment)
```


### loughran_mcdonald: for fanancial document

```{r}
LMValues <- lexicon::hash_sentiment_loughran_mcdonald
head(LMValues)

nrcWord <- textdata::lexicon_nrc()
nrcWord_trust <- nrcWord%>%
  filter(sentiment=="trust")


#1
#get trust score in loughran_mcdonald
nrcWord_trust
trust_score_m <- LMValues[LMValues$x %in% nrcWord_trust$word,]
head(trust_score_m)


title_trust_Sentiment_m <- sentiment(get_sentences(df_title), 
          polarity_dt = trust_score_m) %>% 
  group_by(title) %>% 
  summarize(meanTrust_m = mean(sentiment))%>%
  arrange(desc(meanTrust_m))
head(title_trust_Sentiment_m)



#2
#get dominance score in loughran_mcdonald
dominance_score_m <- LMValues[LMValues$x %in% nrcDominance$Word,]
dominance_score_m
#Calculate dominant score across title
title_dominance_Sentiment_m <- sentiment(get_sentences(df_title), 
          polarity_dt = dominance_score_m) %>% 
  group_by(title) %>% 
  summarize(meanDominance_m = mean(sentiment))%>%
  arrange(desc(meanDominance_m))
title_dominance_Sentiment_m

```


### Visualization for three scores


```{r}
title_trust_Sentiment <- as.data.frame(title_trust_Sentiment)
full <- title_trust_Sentiment%>%
  merge( . , title_dominance_Sentiment)%>%
  merge(.,title_trust_Sentiment_m)%>%
  merge(.,title_dominance_Sentiment_m)


library(tidyverse)
full<-full%>%
  gather("method","score",-title)
full

full<-full%>%
  mutate(title = as.factor(title))
full
library(ggplot2)
library(gganimate)
animatedMethod <- full%>%
  ggplot(aes(title,score,color=method))+
  geom_point(size=4) +
  theme_minimal() +
  theme(axis.text.x=element_text(angle =90))+
  transition_states(method,
                    transition_length = length(unique(full$method)),
                    state_length = 3 ) + #state_length fixed to 3
  ggtitle('Method: {closest_state}')

animate(animatedMethod, fps = 5)

```


CEO is the most dominance in two methods.



# Silver

## Step 3

Register for a free API key from <a href"https://www.alphavantage.co/documentation/">alphavantage</a>. Using the API key, get the daily time series for the given ticker and explore the 10 trading days around each call's date (i.e., the closing price for 5 days before the call, the closing price for the day of the call, and the closing price for the 5 days after the call). 

```{r}
# Your dedicated access key is: LFZAVK2SZ6DQGB0V.
library(httr)


price <- GET("https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=WWE&apikey=LFZAVK2SZ6DQGB0V&outputsize=full")

# price: status 200 means success

#price$content : raw data we can't interpret
# parse that(parse content)
parseTest <- content(price, as = "parsed")
parseTest[1]
closeValues = parseTest$`Time Series (Daily)`
# return is the list
# the $ is the name of the list

#date = names(closeValues[1])
#close = closeValues[[1]]$`4. close`

#lapply: return list and another list of dataframes
out<- lapply(1:length(closeValues),function(x){
  date = names(closeValues[x])
  close = closeValues[[x]]$`4. close`
  data.frame(date = date,
             close = close)
  })

allprice<-do.call("rbind",out)
head(allprice)

#get the unique value and build sequence 
# seq.Date(): date sequences
#lubridate package

df<-raw%>%
  separate(date,c("Day","Month","Year"),sep="-")

unique(df$Month)
unique(df$Day)
unique(df$Year)

#tranform date data
df<-df%>%
  mutate(Month = ifelse((Month=="Dec"),"12",Month))%>%
  mutate(Month = ifelse((Month=="Nov"),"11",Month))%>%
  mutate(Month = ifelse((Month=="Feb"),"02",Month))%>%
  mutate(Month = ifelse((Month=="Jun"),"06",Month))%>%
  mutate(Month = ifelse((Month=="Mar"),"03",Month))%>%
  mutate(Month = ifelse((Month=="Aug"),"08",Month))%>%
  mutate(Month = ifelse((Month=="May"),"05",Month))%>%
  mutate(Month = ifelse((Month=="Sep"),"09",Month))%>%
  mutate(Day = ifelse(Day=="1" | Day=="2" | Day=="3"| Day=="4" | Day=="5"| Day=="6"| Day=="7", paste0("0",Day),Day))%>%
  mutate(date=paste0("20",Year,"-", Month, "-", Day))%>%
  mutate(date=as.Date(date))


# nrc: different emotion
nrcWord <- textdata::lexicon_nrc()
head(nrcWord)

#select one emotion I want in nrc: trust 
#extract trust word:
nrcWord_trust <- nrcWord%>%
  filter(sentiment=="trust")
head(nrcWord_trust)

#get score in hash_sentiment_nrc
nrcValues <- lexicon::hash_sentiment_nrc
trust_score <- nrcValues[nrcValues$x %in% nrcWord_trust$word,]
head(trust_score)


#get my df with title and text
df_date <- df%>%
  select(text,date)
head(df_date)

#Calculate average dominace score groupby date
date_trust_Sentiment <- sentiment(get_sentences(df_date), 
          polarity_dt = trust_score) %>% 
  group_by(date) %>% 
  summarize(meanTrust = mean(sentiment))%>%
  arrange(desc(meanTrust))
head(date_trust_Sentiment)



date_trust_Sentiment<-date_trust_Sentiment%>%
  mutate(PriceBefore = 0)%>%
  mutate(Price = 0)%>%
  mutate(PriceAfter = 0)


for (i in 1:length(date_trust_Sentiment$date)){
  call_date=(date_trust_Sentiment$date[i])
  k = which(allprice$date==as.character(call_date)) #k=row
  #price of that date
  date_trust_Sentiment[i,4] = as.numeric(as.character(allprice[k,]$close))
  #price of 5 days before
  date_trust_Sentiment[i,3] = as.numeric(as.character(allprice[k+5,]$close))
  #price of 5 days after
  date_trust_Sentiment[i,5] = as.numeric(as.character(allprice[k-5,]$close))
}

date_trust_Sentiment%>%
  arrange(meanTrust)

head(date_trust_Sentiment)

date_trust_Sentiment<-date_trust_Sentiment%>%
  mutate(BeforeToCall=Price-PriceBefore)%>%
  mutate(CallToAfter=PriceAfter-Price)


date_trust_Sentiment%>%
  ggplot()+
  geom_line(aes(date,Price))+
  geom_line(aes(date,PriceBefore),color="#099A00")+
  geom_line(aes(date,PriceAfter),color="#C80000")+
  theme_minimal()

date_trust_Sentiment%>%
  ggplot()+
  geom_line(aes(meanTrust,Price))+
  geom_point(aes(meanTrust,PriceBefore),color="#099A00",alpha=0.5,size=3)+
  geom_point(aes(meanTrust,PriceAfter),color="#C80000",alpha=0.5,size=3)+
  
  theme_minimal()


a<-date_trust_Sentiment%>%
  ggplot(aes(date,meanTrust))+
  geom_col()+
  theme_minimal()
 

b<-date_trust_Sentiment%>%
  ggplot()+
  geom_col(aes(date,BeforeToCall),fill="#099A00",alpha=0.5)+
  geom_col(aes(date,CallToAfter),fill="red",alpha=0.5)+
  theme_minimal()
library(gridExtra)
grid.arrange(a,b,nrow=2)

date_trust_Sentiment%>%
  ggplot()+
  geom_col(aes(as.factor(meanTrust),BeforeToCall),fill="#099A00",alpha=0.5)+
  geom_col(aes(as.factor(meanTrust),CallToAfter),fill="red",alpha=0.5)+
  theme_minimal()


```


# Platinum

## Step 4

There are two calls within the zip file that did not used for the previous steps -- they are not already parsed.

file1:

```{r}
d1<-read.csv(file_name[38])

d1[1:15,]

name<-c("Michael Weitz","Vince McMahon","George Barrios","Evan Wingren","Brandon Ross","Eric Katz","Dan Medina","Daniel Moore","Rob Routh","Operator")
organization<-c("World Wrestling Entertainment","World Wrestling Entertainment","World Wrestling Entertainment","Pacific Crest Partners","BTIG","Wells Fargo","Needham Investor Group","CJS Securities","FBN Securities","NA")
title<-c("VP/IR","CEO","CFO","Analyst","Analyst","Analyst","Analyst","Analyst","Analyst","Operator")
quarter = "Q3"
date = "2016-10-27"

temp<-data.frame(name,organization,title,quarter,date)
temp


df1_raw <- data.frame(matrix(ncol = 2, nrow = 0))
x <- c("name", "text")
colnames(df1_raw) <- x

j=0
for (i in d1[15:175,]){
  if (grepl("(^[A-Z][a-z]+\\s[A-Z][a-z]+$)|(^[A-Z][a-z]+$)",i)){
    j=j+1
    df1_raw[j,1] <- i 
  }
  
  else{
    my_str = df1_raw$text[j]
    df1_raw$text[j] <- paste0(my_str,i)
  } 
}
library(fuzzyjoin)
df1_raw<-stringdist_left_join(df1_raw, temp, c("name"="name"), 
                     method = "jw", distance_col = "distance", 
                     max_dist = .2)

df1_raw$text<-str_replace(df1_raw$text,"NA","")
head(df1_raw,10)

```

file2:

```{r}
file_name
d2<-read.csv(file_name[39])

d2[1:15,]

name<-c("Michael Weitz","Vince McMahon","George Barrios","Evan Wingren","Brandon Ross","Eric Katz","Laura Martin","Dan Moore","Rob Routh","Operator")
organization<-c("World Wrestling Entertainment","World Wrestling Entertainment","World Wrestling Entertainment","Pacific Crest Partners","BTIG","Wells Fargo","Needham Investor Group","CJS Securities","FBN Securities","NA")
title<-c("VP/IR","CEO","CFO","Analyst","Analyst","Analyst","Analyst","Analyst","Analyst","Operator")
quarter = "Q2"
date = "2016-07-28"

temp<-data.frame(name,organization,title,quarter,date)
temp


df2_raw <- data.frame(matrix(ncol = 2, nrow = 0))
x <- c("name", "text")
colnames(df2_raw) <- x

j=0
for (i in d2[15:137,]){
  if (grepl("(^[A-Z][a-z]+\\s[A-Z][a-z]+$)|(^[A-Z][a-z]+$)",i)){
    j=j+1
    df2_raw[j,1] <- i 
  }
  
  else{
    my_str = df2_raw$text[j]
    df2_raw$text[j] <- paste0(my_str,i)
  } 
}


df2_raw<-stringdist_left_join(df2_raw, temp, c("name"="name"), 
                     method = "jw", distance_col = "distance", 
                     max_dist = .2)

df2_raw$text<-str_replace(df2_raw$text,"NA","")
head(df2_raw,10)

```

Combine file1 and file2:

```{r}
df_combine<-rbind(df1_raw,df2_raw)
df_combine<-df_combine%>%
  select(-distance,-name.y)%>%
  rename(name=name.x)%>%
  mutate(date=as.Date(date))
df_combine
```

integrate to the original df:

```{r}
length(df)
df_new<-df%>%
  select(name,text,organization,title,quarter,date)%>%
  rbind(df_combine)
length(df_new)
```

