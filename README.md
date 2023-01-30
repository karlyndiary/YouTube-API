
# YouTube API 

## Summary

Founded in 2005, Youtube has grown to become the second largest search engine in the world (behind Google) that processes more than 3 billion searches per month. It is, however, generally a myth how the Youtube algorithm works, what makes a video get views and be recommended over another. In fact, YouTube has one of the largest scale and most sophisticated industrial recommendation systems in existence. For new content creators, it is a challenge to understand why a video gets video and others do not. There are many "myths" around the success of a Youtube video, for example if the video has more likes or comments, or if the video is of a certain duration. It is also worth experimenting and looking for "trends" in the topics that Youtube channels are covering in a certain niche.

Having recently stepping into the content creation world with a new Youtube channel on data analytics and data science, I decided to gain some insights on this topic which might be useful for other new content creators. The scope of this small project is limited to data science channels and I will not consider other niches (that might have a different characteristics and audience base). Therefore, in this project will explore the statistics of around 10 most successful data science Youtube channel.

Data analysis for the case study follows the following steps:

* [1. Prepare](#1-prepare)
* [2. Process](#2-process)
* [3. Data Cleaning](#3-data-cleaning)
* [4. Data Visualization](#4-data-visualization)

## 1. Prepare

The data is collected using the youtube api key with the help of the documentation provided with it. According to [YouTube API Guide](https://developers.google.com/youtube/v3/getting-started), the usage of Youtube API is free of charge given that your application send requests within a quota limit. "The YouTube Data API uses a quota to ensure that developers use the service as intended and do not create applications that unfairly reduce service quality or limit access for others. " The default quota allocation for each application is 10,000 units per day, and you could request additional quota by completing a form to YouTube API Services if you reach the quota limit.

Since all data requested from Youtube API is public data (which everyone on the Internet can see on Youtube), there is no particular privacy issues as far as I am concerned. In addition, the data is obtained only for research purposes in this case and not for any commercial interests.


## 2. Process

### 2.1 Installing the necessary packages needed for cleaning and analysis.
#### Run Locally: The Google APIs Client Library for Python:
```
pip install --upgrade google-api-python-client
```
```
#Google API
from googleapiclient.discovery import build
```
```
import pandas as pd
from IPython.display import JSON
from dateutil import parser
import isodate #convert youtube duration to seconds
```
```
#Data viz packages
import seaborn as sns
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker
```
```
#NLP
import nltk
from nltk.corpus import stopwords
nltk.download('stopwords')
from wordcloud import WordCloud
```

### 2.2 Data creation with YouTube API

```bash
api_key = 'GOOGLE API KEY HERE'
```

```bash
#channel ids
channel_ids = ['UChBEbMKI1eCcejTtmI32UEw']
```

```
api_service_name = "youtube"
api_version = "v3"

#Get credentials and create an API client
youtube = build(api_service_name, api_version, developerKey=api_key)
```
### Get Channel Statistics
This function will help use get information about the channel's statistics like the title, subscriber count, total no of videos published.
```
def get_channel_stats(youtube, channel_ids):
    
    all_data = []
    
    request = youtube.channels().list(
      part="snippet,contentDetails,statistics",
      id=','.join(channel_ids)
    )
    
    response = request.execute()

    # loop through items
    for item in response['items']:
        data = {'channelName': item['snippet']['title'],
                'subscribers': item['statistics']['subscriberCount'],
                'views': item['statistics']['videoCount'],
                'playlistId': item['contentDetails']['relatedPlaylists']['uploads']
               }
        
        all_data.append(data)
        
    return(pd.DataFrame(all_data))
```
### Calling the function to get the channel basic information
```
channel_stats = get_channel_stats(youtube, channel_ids)
channel_stats
```
### Defining a function to return all the video ids in the playlist.
```
playlist_id = 'UUhBEbMKI1eCcejTtmI32UEw'

def get_video_ids(youtube, playlist_id):
    
    video_ids = []
    
    request = youtube.playlistItems().list(
        part="snippet,contentDetails",
        playlistId=playlist_id,
        maxResults = 50
    )
    response = request.execute()
    
    for item in response['items']:
        video_ids.append(item['contentDetails']['videoId'])
        
    next_page_token = response.get('nextPageToken')
    while next_page_token is not None:
        request = youtube.playlistItems().list(
                    part='contentDetails',
                    playlistId = playlist_id,
                    maxResults = 50,
                    pageToken = next_page_token)
        response = request.execute()

        for item in response['items']:
            video_ids.append(item['contentDetails']['videoId'])

        next_page_token = response.get('nextPageToken')
        
    return video_ids
```
### Calling the function to return the list of video ids from the playlist.
```
video_ids = get_video_ids(youtube, playlist_id)
len(video_ids)	
```
### Defining a function to get the channels details
Details such as channel title, title of the video, description, like count, comment count, duration and more.
```
def get_video_details(youtube, video_ids):

    all_video_info = []
    
    for i in range(0, len(video_ids), 50):
        request = youtube.videos().list(
            part="snippet,contentDetails,statistics",
            id=','.join(video_ids[i:i+50])
        )
        response = request.execute() 

        for video in response['items']:
            stats_to_keep = {'snippet': ['channelTitle', 'title', 'description', 'tags', 'publishedAt'],
                             'statistics': ['viewCount', 'likeCount', 'favouriteCount', 'commentCount'],
                             'contentDetails': ['duration', 'definition', 'caption']
                            }
            video_info = {}
            video_info['video_id'] = video['id']

            for k in stats_to_keep.keys():
                for v in stats_to_keep[k]:
                    try:
                        video_info[v] = video[k][v]
                    except:
                        video_info[v] = None

            all_video_info.append(video_info)
    
    return pd.DataFrame(all_video_info)
```
### Returns the video details
```
video_df = get_video_details(youtube, video_ids)
video_df
```
### 2.3 Write video data to CSV file for future reference
```
video_df.to_csv('video_data.csv')
```
 ## 3. Data Cleaning
**Checking for any null values**
```bash
video_df.isnull().any()
```
**Check datatypes**
```
video_df.dtypes
```
**Converting to numeric**
```
numeric_cols = ['viewCount', 'likeCount', 'favouriteCount', 'commentCount']
video_df[numeric_cols] = video_df[numeric_cols].apply(pd.to_numeric, errors = 'coerce', axis = 1)
  ```
**Publish day of week**
```
video_df['publishedAt'] = video_df['publishedAt'].apply(lambda x: parser.parse(x))
video_df['publishedDayName'] = video_df['publishedAt'].apply(lambda x: x.strftime("%A"))
```
**Converting duration to seconds**
```
video_df['durationSecs'] = video_df['duration'].apply(lambda x: isodate.parse_duration(x))
video_df['durationSecs'] = video_df['durationSecs'].astype('timedelta64[s]')

video_df[['durationSecs','duration']]
```
**Add tag count**
```
video_df['tag_count'] = video_df['tags'].apply(lambda x: 0 if x is None else len(x))
video_df
```
## 4. Data Visualization
### 4.1 Best Performing video
```
ax = sns.barplot(x = 'title', y = 'viewCount', data = video_df.sort_values('viewCount', ascending=False)[0:10])
plot = ax.set_xticklabels(ax.get_xticklabels(), rotation = 90)
ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda x, pos:'{:,.0f}'.format(x/1000) + 'K'))
plt.xlabel('Video Title')
plt.ylabel('View Count')
plt.title('Best Performing Videos')
```
![download](https://user-images.githubusercontent.com/116041695/215266326-5807bcfd-c801-4546-9c63-168363e1353f.png)

From the bar chart, we can see that this channel has a series playlist called "But Better", and four videos out of the series are in the top 10 best performing videos along with other best performing videos. 

### 4.2 Worst Performing Video
```
ax = sns.barplot(x = 'title', y = 'viewCount', data = video_df.sort_values('viewCount', ascending=True)[0:10])
plot = ax.set_xticklabels(ax.get_xticklabels(), rotation = 90)
ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda x, pos:'{:,.0f}'.format(x/1000) + 'K'))
plt.xlabel('Video Title')
plt.ylabel('View Count')
plt.title('Worst Performing Videos')
```
![download (1)](https://user-images.githubusercontent.com/116041695/215266342-bc157bec-2c91-41ed-ab03-b4cc6dd1dd38.png)

We can see the top 10 worst performing videos. 

### 5.3 Video Distribution per video
```
sns.violinplot(x ='channelTitle', y ='viewCount', data = video_df)
plt.ylabel('View Count')
plt.xlabel('Channel Title')
plt.title('View Distribution')
```
![download (2)](https://user-images.githubusercontent.com/116041695/215266362-5eb9e8a1-6fd9-46ea-9a57-007b8be8682f.png)

### 4.4 Comments, Likes vs Views
```
fig, ax = plt.subplots(1,2)
sns.scatterplot(data = video_df, x = 'commentCount', y = 'viewCount', ax = ax[0], color = 'violet')
sns.scatterplot(data = video_df, x = 'likeCount', y = 'viewCount', ax = ax[1], color = 'purple')
#plt.xticks(rotation=45)
fig.suptitle('Comment and Like Count vs. Views')
ax[0].set(xlabel='Comment Count', ylabel='View Count')
ax[1].set(xlabel='Like count', ylabel='View Count')

plt.show()
```
![download (7)](https://user-images.githubusercontent.com/116041695/215266711-19d74354-ff50-4d0a-bf35-75e52904dc45.png)

From the subplot, we notice both the graphs are similar, meaning that higher the view count, higher the likes and comments.

### 4.5 Video Duration
```
p = sns.histplot(data = video_df, x = 'durationSecs', bins = 30)
plt.title('Video duration')
plt.xlabel('Duration in Seconds')

plt.show()
```
![download (3)](https://user-images.githubusercontent.com/116041695/215266380-bcec29bb-e6f3-4f56-8554-60dc57b95cb4.png)

We observe that the duration is mostly in the middle ranging between 350 to 700 seconds which is quite good.

### 4.6 WordCloud from video titles
```
stop_words = set(stopwords.words('english'))
video_df['title_no_stopwords'] = video_df['title'].apply(lambda x: [item for item in str(x).split() if item not in stop_words])

all_words = list([a for b in video_df['title_no_stopwords'].tolist() for a in b])
all_words_str = ' '.join(all_words) 

def plot_cloud(wordcloud):
    plt.figure(figsize=(30, 20))
    plt.imshow(wordcloud) 
    plt.axis("off");

wordcloud = WordCloud(width = 2000, height = 1000, random_state=1, background_color='black', 
                      colormap='viridis', collocations=False).generate(all_words_str)
plot_cloud(wordcloud)
```
![download (6)](https://user-images.githubusercontent.com/116041695/215266481-6492810d-d06f-4792-9d14-81906bb1893c.png)

From the wordcloud, we can see some common words like home, homemade, faster, easist, cheaper, guide - this could be that the channel revolves more around the easy but homemade, easy recipes.

### 4.7 Upload Schedule
```
plt.figure(figsize=(50, 50))
day_df = pd.DataFrame(video_df['publishedDayName'].value_counts())
weekdays = [ 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
day_df = day_df.reindex(weekdays)
ax = day_df.reset_index().plot.bar(x='index', y='publishedDayName', rot=45)
plt.xlabel('Days of the week')
ax.get_legend().remove()

plt.show()
```
![download (5)](https://user-images.githubusercontent.com/116041695/215266416-5230bbce-c2ef-4d4d-954d-b73cfd563e6c.png)

The upload schedule is fairly spaced out and regular. Wednesday and Sunday being the regular schedule and maybe friday could have some extra bonus videos.
