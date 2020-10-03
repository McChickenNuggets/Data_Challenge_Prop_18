![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Team_Outlier.png)
# Data_Challenge

This is a project focusing on the impacts of PP16 (Proposition 16). We retrieve data from Twitter using rtweet and twitteR, and use text mining through 3 differnt sentiment-analysis to distinguish the opposition and support implied by the contexts of each tweet on Proposition 16. Using the result generated by sentiment-analysis, we construct a logistic model using the filtered-dataset and investigate the effectiveness, sensitivity, specificity of our model. Besides, combining the past admission summaries based on ethnicity from University of Califonia, we generalize ideas from social medias to make assumptions about the following conseuences of the Proposition 16 on UC Davis admission based on ethnicity.

## Reseach Question
- How controversial is the pp16? Who is supporting and who is opposing pp16?
- What's the impacts on UC Davis admission if pp16 pass?

## Background Information
- mid-1960s: CA has affirmative actions (to diversity and to prefer the disadvantage group)
- 1996/11/15: pp209 passed → ban affirmative = remove minority advantage
    - Impact: Prop 209 caused large immediate changes to URG UC applicants’ likelihood of UC admission and enrollment. 
    - Each URG UC applicant became substantially less likely to earn admission at every UC campus in 1998, with average declines as high as 25 percent at UC Berkeley and down to 4 percent at UC Riverside, which admitted all UC-eligible applicants. 
    - In general, URG applicants became 8 percent less likely to earn admission at any UC campus after 1998. 
- 2020: still under pp209 = no affirmative 

Early signs suggest voter skepticism of permitting affirmative action once again in California: 47% of likely voters object to Prop. 16 while 31% would vote for it today, according to a [poll released Wednesday night](https://www.ppic.org/publication/ppic-statewide-survey-californians-and-their-government-september-2020/). That’s a wider gap than the one seen in 1996, when voters [approved a ban](https://ballotpedia.org/California_Proposition_209,_Affirmative_Action_Initiative_(1996)) on affirmative action in the state 54% to 46%. The Wednesday poll showed that 22% haven’t made up their minds. Liberals were more likely to support Prop. 16 than moderates and conservatives.

#  How controversial is the Proposition 16?

Collect the number of relevant tweets for each pp (15~24)
```r
tw15= length(twitteR::searchTwitter('#Prop15', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw16= length(twitteR::searchTwitter('#Prop16', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw17= length(twitteR::searchTwitter('#Prop17', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw18= length(twitteR::searchTwitter('#Prop18', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw19= length(twitteR::searchTwitter('#Prop19', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw20= length(twitteR::searchTwitter('#Prop20', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw21= length(twitteR::searchTwitter('#Prop21', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw22= length(twitteR::searchTwitter('#Prop22', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
tw23= length(twitteR::searchTwitter('#Prop23', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))    
tw24= length(twitteR::searchTwitter('#Prop24', n=1e4, since = '2020-09-01', retryOnRateLimit = 1e3))
```
Compare 16 with the other 9 pp with bar graph 
```r
ggplot(data_pie1, aes(x=Proposition, y=num, fill=Proposition)) +
  geom_bar(stat="identity")+theme_minimal()+ 
  ggtitle("Proposition Popularity Share") + 
  ylab("Number of Tweets")
```

![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Proposition_popularity_share.png)

Based on the proposition popularity share, we can see Proposition 16 is the most controversial one among all the propositions. In the past 30 days, there are nearly 3000 tweets that contain the hashtag of #Prop16. 

```r
# Retrieve System Time A week ago
toDate <- format(Sys.time() - 60 * 60 * 24 * 7, "%Y%m%d%H%M")

# Search information for #Prop 16 for past 30days in USA
rt <- search_30day("#Prop16", n = 15000,
                   env_name = "research", toDate = toDate,
                   geocode = lookup_coords("usa"))
```
 Quickly visualize frequency of tweets over time using `ts_plot()`
```r
# Time_plot to display the frequencies of post on twitter about #Prop16
rt %>%
  ts_plot("3 hours") +
  theme_minimal() +
  theme(plot.title = element_text(face = "bold")) +
  labs(
    x = NULL, y = NULL,
    title = "Frequency of #Prop16 Twitter statuses from past 30 days",
    subtitle = "Twitter status (tweet) counts aggregated using three-hour intervals",
    caption = "\nSource: Data collected from Twitter's REST API via rtweet"
  )
```
[Image Required]

## Data Processing
Filter the Dataset
```r
# Filter important attributes
rt<- rt %>% select(user_id,screen_name:source,is_quote:hashtags,lang,name:verified,-url,-account_created_at)

# Check whether each attribute is convertible
sapply(rt,class)

# Transform the attribute
rt$hashtags<-as.character(rt$hashtags)

# Write to CSV file
write.csv(rt,"C:\\Users\\Jaune\\Desktop\\pp16twitter.csv")
```
Import the Dataset and Reprocess
```r
pp16twitter <- read_csv("pp16twitter.csv", col_types = cols())

# redefine original data as new df for analysis
pp16twi <- pp16twitter %>% 
  drop_na(c(hashtags, location)) %>%
  transmute(
    support = case_when(
    str_detect(hashtags, regex("(yes)|(Opportunity4All)", ignore_case=TRUE)) ~ "yes",
    str_detect(hashtags, regex("(no)|(stop)|(unitynotdivision)", ignore_case=TRUE)) ~ "no",
    TRUE ~ "undef"
  ), 
  text, 
  inCA = ifelse(str_detect(location, regex("(CA)|(California)", ignore_case=TRUE)), "yes", "no"), 
  influential = case_when(
    followers_count > 1000 ~ "yes",
    followers_count < 1000 ~ "no"
  ),
  username = screen_name,
  bio = description
  )

# twitter dataset split into 2 parts: df1 - yes/no defined in hashtags & df2 - those undefined
df1 <- pp16twi %>% filter(support != "undef")
df2 <- pp16twi %>% filter(support == "undef")

# those undefined (df2) need to be defined based on text sentiment analysis 
textdf <- df2 %>% mutate(id = 1:nrow(df2))
```
## Text Mining (Sentiment Analysis)
#### 1st method: unigram sentiment analysis for tweets with 'bing' lexicon
```{r}
# each tweet split into single word and take out stop words
tweet_token <- textdf %>% 
  unnest_tokens(word, text) %>%
  anti_join(stop_words) %>%
  group_by(id) %>%
  count(word,sort = TRUE) %>%
  arrange(id)

# freq of each word in each tweet
tweet_propotions <- tweet_token %>% 
  group_by(id) %>% 
  mutate(propotion = n / sum(n)) %>% 
  select(-n) %>%
  arrange(id, desc(propotion))
 
 # define yes/no based on positive/negative sentiment on prop16
sentimentM1 <- tweet_token %>% 
  left_join(get_sentiments("bing")) %>% 
  group_by(id) %>% 
  summarize(  # the frequencies are ignored in this analysis
    positive = sum(sentiment == "positive", na.rm = TRUE), 
    negative = sum(sentiment == "negative", na.rm = TRUE), 
    neutral = n() - positive - negative) %>%
  mutate(
    id,
    sentiment1 = case_when(
      positive > negative ~ "positive",
      positive < negative ~ "negative",
      TRUE ~ "neutral"
    )
  ) %>% 
  left_join(select(textdf, id, username)) %>% 
  select(sentiment1, id, username)
```
#### 2nd method: bigrams sentiment analysis for tweets with 'bing' lexicon
```{r}
tweet_token2 <- textdf %>%
  unnest_tokens(bigram, text, token = "ngrams", n = 2) %>% 
  separate(bigram, c("word1", "word2"), sep = " ")

negate_words <- c("not", "without", "no", "can't", "don't", "won't")

sentimentM2 <- tweet_token2 %>% 
  group_by(id) %>% 
  count(word1, word2) %>% 
  left_join(get_sentiments("bing"), by = c("word2" = "word")) %>%
  mutate(sentiment = case_when(
    word1 %in% negate_words & sentiment == "negative" ~ "positive", 
    word1 %in% negate_words & sentiment == "positive" ~ "negative",
    TRUE ~ sentiment)) %>% 
  summarize(
    positive = sum(sentiment == "positive", na.rm = TRUE), 
    negative = sum(sentiment == "negative", na.rm = TRUE), 
    neutral = n() - positive - negative) %>%
  mutate(
    id,
    sentiment2 = case_when(
      positive > negative ~ "positive",
      positive < negative ~ "negative",
      TRUE ~ "neutral"
    )
  ) %>% 
  left_join(select(textdf, id, username)) %>% 
  select(sentiment2, id, username)
```
#### 3rd method: unigram sentiment analysis for tweets with 'afinn' lexicon
```{r}
sentimentM3 <- textdf %>% 
  unnest_tokens(word, text) %>% 
  left_join(get_sentiments("afinn")) %>% 
  replace_na(list(value=0L)) %>%
  mutate(
    id,
    sentiment3 = case_when(
      sum(value) > 0 ~ "positive",
      sum(value) < 0 ~ "negative",
      TRUE ~ "neutral"
    )
  ) %>% 
  transmute(id, sentiment3) %>%
  unique() %>%
  left_join(select(textdf, id, username)) %>% 
  select(sentiment3, id, username)
```
Combine the results of 3 different Sentiment Analysis approach
```r
# new df contain 3 sentiment result, and pick the mode
dfM123 <- sentimentM1 %>% 
  left_join(sentimentM2, by = c("id"="id")) %>% 
  left_join(sentimentM3, by = c("id"="id"))

dfM123$sentiment1 <- str_replace(dfM123$sentiment1,"positive","1")
dfM123$sentiment1 <- str_replace(dfM123$sentiment1,"negative","-1")
dfM123$sentiment1 <- str_replace(dfM123$sentiment1,"neutral","0")

dfM123$sentiment2 <- str_replace(dfM123$sentiment2,"positive","1")
dfM123$sentiment2 <- str_replace(dfM123$sentiment2,"negative","-1")
dfM123$sentiment2 <- str_replace(dfM123$sentiment2,"neutral","0")

dfM123$sentiment3 <- str_replace(dfM123$sentiment3,"positive","1")
dfM123$sentiment3 <- str_replace(dfM123$sentiment3,"negative","-1")
dfM123$sentiment3 <- str_replace(dfM123$sentiment3,"neutral","0")

dfM123$sentiment1<-as.numeric(dfM123$sentiment1)
dfM123$sentiment2<-as.numeric(dfM123$sentiment2)
dfM123$sentiment3<-as.numeric(dfM123$sentiment3)

df2_new <- dfM123 %>% 
  mutate(s = (sentiment1+sentiment2 +sentiment3)) %>%
  transmute(
    id,
    support = case_when(
      s > 0 ~ "yes",
      s < 0 ~ "no",
      TRUE ~ "neutral"
    )
  ) %>%
  right_join(textdf, by=c("id" = "id")) %>%
  transmute(support = support.x, text, inCA, influential, username = username, bio )

pp16df <- df1 %>% bind_rows(df2_new)
write.csv(pp16df,"C:\\Users\\Jaune\\Documents\\Github\\Data_Challenge\\pp16df.csv")
```
#  Who is supporting and who is opposing pp16?
Using the result derived from sentiment-analysis through the text-mining process, we obtained the proponents and opponents for pp16. Therefore, we visualize the common features of the tweets on prop16 through word cloud.

Process the data for Word Cloud
```r
pp16_no<-pp16df %>% filter(support=="no") %>% transmute(bio) %>% unique()
pp16_yes<-pp16df %>% filter(support=="yes") %>%  transmute(bio) %>% unique()
```

Word Cloud on Username by most engaged in the topic
```r
name_count<-pp16twitter %>% group_by(screen_name) %>%
  summarize(n = n()) %>% 
  arrange(desc(n)) %>% 
  na.omit()

name_count %>% with(wordcloud(
  screen_name, n, min.freq = 1, max.words = 20, random.order = FALSE,
  colors = brewer.pal(8, "Dark2")))
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/user_name.png)
Word Cloud on User bio by Yes Group
```r
word_pat <- "[\\D]+"
word_pat1 <- "\\w{3,}"

words_collec_yes_pre <- str_extract_all(pp16_yes$bio, word_pat) %>%  unlist()
words_collec_yes <- str_extract_all(words_collec_yes_pre , word_pat1)%>% unlist() 

words_collec_yes <- tolower(words_collec_yes)


words_lc_yes<-sapply(words_collec_yes, tolower)

df_words_lc_yes <- tibble(line = 1:length(words_lc_yes), text = words_lc_yes)

df_words_lc_yes <- df_words_lc_yes %>% 
  unnest_tokens(word, text)%>% 
  anti_join(stop_words)

df_words_lc_yes_count<-df_words_lc_yes%>%
  group_by(word) %>%
  summarize(n = n()) %>% 
  arrange(desc(n)) %>% 
  na.omit()

###
df_words_lc_yes_count%>% with(wordcloud(
  word, n, min.freq = 1, max.words = 15, random.order = FALSE,
  colors = brewer.pal(8, "Dark2")))
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/yes_user_bio.png)
Word Cloud on User bio by No Group
```r
word_pat <- "[\\D]+"
word_pat1 <- "\\w{3,}"

words_collec_no_pre <- str_extract_all(pp16_no$bio, word_pat) %>%  unlist()
words_collec_no <- str_extract_all(words_collec_no_pre , word_pat1)%>% unlist() 

words_collec_no <- tolower(words_collec_no)


words_lc_no<-sapply(words_collec_no, tolower)

df_words_lc_no <- tibble(line = 1:length(words_lc_no), text = words_lc_no)

df_words_lc_no <- df_words_lc_no %>% 
  unnest_tokens(word, text)%>% 
  anti_join(stop_words)

df_words_lc_no_count<-df_words_lc_no%>%
  group_by(word) %>%
  summarize(n = n()) %>% 
  arrange(desc(n)) %>% 
  na.omit()

df_words_lc_no_count%>% with(wordcloud(
  word, n, min.freq = 1, max.words = 15, random.order = FALSE,
  colors = brewer.pal(8, "Dark2")))
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/no_user_bio.png)

## Logistic Regression
Read and Process the dataset.
```r
# Read the data
pp16twitter <- read_csv("pp16df.csv", col_types = cols())

# Drop out the observations that are ambiguious
temp<-pp16twitter %>% filter(support!="neutral")

# Select Required Varaible
temp<-temp %>% select(support,inCA,influential)

# Replace variable with 1 and 0 to fit the logistic regression
temp$support=ifelse(temp$support=='yes',1,0)
temp$inCA=ifelse(temp$inCA=='yes', 1,0)
temp$influential=ifelse(temp$influential=='yes', 1,0)
```
Visualize the correlation between variables using `corrgram()`
```r
# correlation between numeric variables
library(corrgram)
corrgram(temp, order = TRUE,
         lower.panel = panel.shade, upper.panel = panel.pie,
         text.panel = panel.txt,
         main = "Correlogram of Variables")
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Correlogram.png)

Conduct the Logistic Regressopm
```r
initial<-glm(formula = support~inCA+influential+inCA*influential,data=temp,family = 'binomial')
summary(initial)

library(MASS)
# Run stepAIC to get the best fit model
stepAIC(initial,scope = list(lower=~1), direction = "backward",k=2)
```
Based on the result and determine the best model is the initial model, and we use the initial model to make predictions on the proportion and support of Proposition 16, compare it with the true result we obtained through text mining, and examine its sensitivity and speicifity.
```r
# Test the percentage of correctness given our model
log_predprob = predict(initial, temp,type='response')

# Change standard to 0.5, all of the predicted response is less than 0.5, 
# So We decide to set the standard to 0.4 to view the chanegs in result.
standard<-0.4
log_pred = log_predprob > standard
log_cm = table(true = temp$support, predicted = log_pred)
log_cm

# Calculate the percentage of correctness of our model in prediction
sum(diag(log_cm))/sum(log_cm)

#Sensitivity: proportion of predictions for Yes on prop16 out of the number of samples which actually support prop16.
sens=54/(54+246)
sens

#Specificity: proportion of predictions for No on prop16 out of the number of samples which actually object prop16.
spec=779/(779+64)
spec
```
The proportion of prediction for Yes on prop16 out of the number of samples which actually support prop16 is only 18% correct based on our model, and the propotion of predictions for No on prop16 out of the number of samples which actually object prop16 is 92.4% correct based on our model.Even though our model has a high specificity, but we have a negligible sensitivty. Therefore,  we conclude our current model is not significant given the limited explanatory variables we have after we filtered the useful information.

# What's the impacts on UC Davis admission if pp16 pass?
We obtain admission summary for [UC Davis](https://www.universityofcalifornia.edu/infocenter/admissions-residency-and-ethnicity), and we separate the admission summary into two differnt periods to see the influences of the affirmative action. The first period is the admission summary from 1994-2003 since pp209, which ban affirmative action in 1996. Using the result from 1994 to 2003, We can see the change in admission for UC Davis before and after the affirmative action being banned. Compare the admission summary from 1994 to 2003 and admission summary from 2010 to 2019, we might see a similar pattern to investigate the upcoming consequences if pp16 passes, and use the pattern to predict the influences of pp16 on UC Davis admission for the following years.

#### Change in Admission by Ethnicity from 1994-2003
```r
# Read Admission Summary for 1994 to 2003
ucdavis1994_2003<-read.csv('Freshmen_Eth_data_1994_2003.csv')
# Change column names to make it readable
colnames(ucdavis1994_2003)<-c('Category','Ethnicity','Academic_year','Count','Percentage')
# Use temporary dataframe to prevent losses
temp_Admits2<-ucdavis1994_2003 %>% filter(Ethnicity!='All',Ethnicity!='Unknown',Category=='Admits')
# # Line Plot ofr Admits by Ethnicity from 1994 to 2003
ggplot(data=temp_Admits2, aes(x=Academic_year, y=Count, group=Ethnicity)) +
  geom_line(aes(linetype=Ethnicity,color=Ethnicity))+
  geom_point(aes(shape=Ethnicity,color=Ethnicity))+
  scale_x_continuous(name = " ", breaks = c(1994,1995,1996,1997,1998,1999,2000,2001,2002,2003))+
  labs(title="Admits Change For All Ethnicities Over 1994 to 2003",y='Student_Numbers')
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Admits_Change_1994_2003.png)
```
ggplot(data=temp_Admits2, aes(fill=Ethnicity, y=Count, x=Academic_year)) + 
  geom_bar(position="fill", stat="identity")+
  scale_x_continuous(name = " ", breaks = c(1994,1995,1996,1997,1998,1999,2000,2001,2002,2003))+
  labs(title="Proportion in Admits For All Ethnicities Over 1994 to 2003",y='Student_Numbers')
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Proportion_in_Admits_1994_2003.png)

#### Change in Admission by Ethnicity from 2010-2019
```r
# Read Admission Summary for 2010 to 2019
ucdavis2010_2019<-read.csv('Freshmen_Eth_data_2010_2019.csv')
# Change column names to make it readable
colnames(ucdavis2010_2019)<-c('Category','Ethnicity','Academic_year','Count','Percentage')
# Use temporary dataframe to prevent losses
temp_Admits<-ucdavis2010_2019 %>% filter(Ethnicity!='All',Ethnicity!='Unknown',Category=='Admits')
# Line Plot for Admits by Ethnicity from 2010-2019
ggplot(data=temp_Admits, aes(x=Academic_year, y=Count, group=Ethnicity)) +
  geom_line(aes(linetype=Ethnicity,color=Ethnicity))+
  geom_point(aes(shape=Ethnicity,color=Ethnicity))+
  scale_x_continuous(name = " ", breaks = c(2010,2011,2012,2013,2014,2015,2016,2017,2018,2019))+
  labs(title="Admits Change For All Ethnicities Over 2010 to 2019",y='Student_Numbers')
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Admits%20Change.png)
```r
# Bar Plot for Admits by Ethnicity from 2010-2019
ggplot(data=temp_Admits, aes(fill=Ethnicity, y=Count, x=Academic_year)) + 
  geom_bar(position="fill", stat="identity")+
  scale_x_continuous(name = " ", breaks = c(2010,2011,2012,2013,2014,2015,2016,2017,2018,2019))+
  labs(title="Proportion in Admits For All Ethnicities Over 2010 to 2019",y='Student_Numbers')
```
![image](https://github.com/McChickenNuggets/Data_Challenge/blob/master/img/Proportion%20in%20Admits.png)

## Conclusion
