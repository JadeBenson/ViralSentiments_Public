# Viral Sentiments 
## Teammates: Jade Benson (https://github.com/JadeBenson), Egemen Pamukcu (https://github.com/egemenpamukcu), Sabina Hartnett (https://github.com/sabinahartnett)

**_Introduction_** 

During the COVID-19 pandemic, online expression has become, arguably, more important than ever. With social and physical outlets limited, many individuals have used the internet, namely social media to express their thoughts, questions and concerns, as well as to connect with others and form identity groups. Additionally, with the uncertainty of the pandemic, concerns of media credibility and bipartisan-politically driven rhetorics, many individuals have used crowd-sourcing methods to find information. Twitter is often studied by academics for its wide range of user-interests, text-based format and API credibility. Our group is interested in observing the type of information shared on Twitter in regards to the COVID-19 pandemic. Following wide-scale vaccination in the US and the CDC announcement confirming the safety of mask-less interaction for vaccinated individuals, we are interested in conducting a brief exploratory cross-sectional study that examines current attitudes towards various aspects of the now seemingly-ending pandemic. We intend to describe current public attitudes on Twitter towards COVID, masks, and vaccination with the hope that this research could further inform continued vaccination efforts and public health interventions.    

In order to get a representative sample of the conversation happening on Twitter this project implements a number of large-scale tools for data analysis and a data sample large enough to warrant parallelized scraping and analysis.


**_Methods_** 

In order to pursue these goals, we used the Twitter API to collect Tweets which mention terms that include the following key words: ```["mask", "CDC", "vaccine", lockdown", "quarantine", "pfizer", "moderna", "covid", "stayathome", "johnson&johnson", "J&J"]```. We partitioned the keywords into three groups: 

Group 1: ```[“lockdown”, “quarantine”, “stayathome”] ``` 

Group 2: ```[“cdc”, “covid”, "mask"]  ```

Group 3: ```[“j&j”, “moderna”, “pfizer”, “vaccine”]  ```

We then use two data collection methodologies: historical tweet scraping and real-time tweet streaming. 

*Historical Scraping* 

We use each of our Twitter API accounts to query tweets within the last 7 days (June 26th, 2021 - June 3rd, 2021), using one of the groups of keywords for each of our accounts (i.e. Jade's API queried group 2 key words). We perform this scraping using Amazon Web Service's (AWS) EC2 instances. To prevent reaching our API query limits, we perform the scraping in small batches (2,000 tweets) with 15min rest intervals between. We collect information on the unique tweet identifying number, the user's profile location, the time the tweet was created, the text of the tweet, any hashtags, and using AWS' comprehend various sentiment metrics: overall sentiment (positive, negative, neutral) and sentiment scores for each of the emotive categories (positive, negative, neutral, and mixed). We additionally search the text of the tweet for each of the keywords and create an indicator for their presence in order to more easily compare attitudes towards these words. We save each of these variables to a dataframe and write it to csvs which we transfer to an S3 bucket for later analysis. We copied the code from Tweepy_historical (on github) to each of the 3 EC2 instances with each of our Twitter credentials. 

In total, we collected 187,251 historical tweets. 


*Real-Time Streaming* 

In order to get large-scale real-time information on public's sentiment towards various dimensions of the COVID-19 pandemic, we utilized Twitter API's tweet streaming endpoint to collect the tweets in English language that contain the keywords specified in the above section. We made use of several AWS resources to collect, process, and store real-time data. Later we set up four AWS EC2 t2.micro instances. One of these instances served as our producer — we connected the producer to the Twitter API to access the live stream, handle some minor reformatting and filter out the tweets that were written in foreign languages. Then we set up an AWS Kinesis Stream to distribute the streamed data to the consumer instances. The Kinesis Stream we set up has three shards and each keyword group described above is assigned to a shard. The three t2.micro consumer instances later iterated over one of these shards to access one part of the live stream. The consumer instances accessed their portion of live data, ran sentiment analysis on the text body of the tweet by calling AWS Comprehend’s sentiment detection functionality, reformatted it to a flat dictionary format (as opposed to the nested format we get from Twittter) and wrote it to a common database. Our choice to scale the process to three shards here allowed us to conduct sentiment analysis on the high throughput Twitter data without missing any information. Since we needed an easily scalable database to store relatively unstructured text data, we used the DynamoDB service provided by AWS. As we wanted to write information produced by three separate EC2 instances, we requested three write capacity units from Amazon, and simultaneously wrote the raw data as well as the sentiment scores that were calculated in real-time. At the end we ended up with still-growing NoSQL database of COVID related tweets ready for further analysis. 

We streamed a total of 140,973 tweets in real time.


*Analysis* 

Since our data was so large (a total of 328,224 tweets) we decided to implement Dask for data analysis. First, we looked at the sentiment scores for each of the tags searched in our data-scraping in order to analyze differences in sentiment surrounding different aspects of the pandemic. Then we 'zoomed in' on the 3 different vaccines currently distributed in the US to see if they recieved particularly dichotomous sentiments. Finally, we used the geo-graphic information included by some twitter users to control for 5 states to analyze the tweet sentiment for users from those states towards both masks and vaccines. These geographic and subject-based analyses can help us understand the ways information is shared on twitter and what conversations may be happening around various COVID-related themes.


**_Results_** 

   <img width="492" alt="Screen Shot 2021-06-04 at 9 51 19 PM" src="https://user-images.githubusercontent.com/18289013/120877981-147a2600-c57f-11eb-810d-f8afc11d6493.png">

    Figure 1. Sentiment Scores for each Tag

In analyzing the images described in analysis, we found that, interestingly, most tweets were not an expression of a strong sentiment. Rather, many were neutral/mixed sentiments. The 'most neutral' tag, it appears, was actually ```stayathome```, which we assume may be a tag used only by users of a certain disposition (more opinionated users may use 'lockdown' instead). The same plot revealed that Johnson & Johnson search tags returned the highest average positive sentiment, and CDC search tags returned the highest average negative sentiment. This initial pulse on the conversation happening on twitter informed our next step to investigate differences between the three vaccines. 

  <img width="492" alt="Screen Shot 2021-06-04 at 9 51 27 PM" src="https://user-images.githubusercontent.com/18289013/120877983-17751680-c57f-11eb-81cc-a280a08c2189.png">

    Figure 2. Sentiment Boxplot for each vaccine.

As explained in the video, we see here (as well as in Figure 1.) that the vaccines actually (surprisingly) did not attract very opinionated statements on twitter. Although within the 3 shown here, Pfizer has the *most negative* average score.

   <img width="737" alt="Screen Shot 2021-06-04 at 9 51 34 PM" src="https://user-images.githubusercontent.com/18289013/120877990-1e9c2480-c57f-11eb-86b2-d982c95feb05.png">

    Figure 3. Sentiment towards masks and vaccines (respectively) for users in 5 states.

The 5 states included in this geographic analysis include a range of political affiliation, covid rates and vaccine distribution rates. Masks elicited an especially negatively opinionated response from users in Texas and Florida - this aligns with some events that have been visible to the public eye. Interestingly, the states show more similar negative sentiment averages for vaccines than masks.

**_Discussion & Next Steps_** 

Our data collection and storage methods allowed us to seamlessly move data from the twitter API, conduct AWS (comprehend) analysis and store in AWS DynamoDB and s3 buckets. This was critical to our data for the format in which it was collected - the streaming data required a dynamic database to grow as more data came in and the historical data, which was collected in batches (due to limitations on our Twitter API accounts) needed to be stored as such. These formats also allowed us to load in and seamlessly run Dask analyses on the data. 

Without these large scale streaming, collection, storage and analysis methods, we would be forced to rely on a much smaller dataset to interpret a conversation that is happening at a global scale. Our methods and the scale at which we were able to conduct this research gave us a more accurate cross-sectional profile of public opinion.

This research is largely an exploratory project/proof of concept for how public health professionals can use Twitter to gauge public opinion on quickly evolving situations and tailor their efforts accordingly. However, our findings suggest that more granular data (available to professional/academic Twitter API users/subscribers), passed through a similar analysis pipeline, could glean even more interesting insights. We look forward to possibilities to continue this research and learn what others in the field are doing.

Some additional avenues of analysis that we would like to see with this data/pipeline:
  More time to collect data (longer study window)
  Compare before and after certain events 
  Could use this to determine the effectiveness of changing public opinion on vaccination in response to different public health campaigns.
  Network analysis to determine groups of users that could benefit from more direct interventions? 
  Geographic information to determine whether attitudes depend on areas with worse COVID outbreaks, greater restrictions, majority political party, etc. 

**_Limitations_**

Given our limited status on both AWS and the Twitter API, we exceeded our limits a number of times and had to deal with deleted accounts and deleted data. Moving forward, we would recommend researchers back up data or maintain an unlimited credit on their AWS account. From this experience, we learned important lessons about data management, cost awareness, budgeting, and setting up our own individual accounts so we can have more control and warning (good news for future projects). 
