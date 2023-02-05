# youtube-API-analysis-project-
## Data pipeline 
A data pipeline is a method in which raw data is ingested from various data sources and then ported to data store, like a data lake or data warehouse, for analysis. Before data flows into a data repository, it usually undergoes some data processing. This is inclusive of data transformations, such as filtering, masking, and aggregations, which ensure appropriate data integration and standardization. This is particularly important when the destination for the dataset is a relational database. This type of data repository has a defined schema which requires alignment—i.e. matching data columns and types—to update existing data with new data.
![image](https://user-images.githubusercontent.com/60258264/216823250-9d769084-1725-4753-8bf7-fbcc3c246997.png)
### steps token

 1. pulling the required data from youtube with specific channel using youtube API documentation (there is max limit per day around 10,000)
 2. selecting the required statistcs information
 3. converting the json format into pandas data frame
 4. Exploratory data analysis for the data 
 5. saving the dataframe into database 
 6. pushing the database on AWS cloud 

 ### pulling the data 
 * from your google account enable your API key after creating the project
 ![image](https://user-images.githubusercontent.com/60258264/216827186-52d5da74-2b85-4d9b-91ae-8e082c214f81.png)
 
* select the wanted channel and get the channel ID I used FIFA channel :heart_eyes: :smiling_face_with_three_hearts: you can use multiple channels too

![image](https://user-images.githubusercontent.com/60258264/216827330-2b089e78-f74d-40b4-9c25-301a6ffadb25.png)

* form youtube Data API will grab channels , playlistitems and videos 

![image](https://user-images.githubusercontent.com/60258264/216827747-00247070-59f7-42cc-9aa9-dfb3609bf31b.png)

Channel ID

```
def channel_status(youtube,channel_id):  
   # Get credentials and create an API client
    all_videos = [] 
    request = youtube.channels().list(
    part="snippet,contentDetails,statistics",
    id=channel_ID
     )
    response = request.execute()
    for item in response['items']:
        data = {'subscribes':item['statistics']['subscriberCount'],
                'views':item['statistics']['viewCount'],
                'totalvideos':item['statistics']['videoCount'],
                'playlistid':item['contentDetails']['relatedPlaylists']['uploads']} #will use playlist to get all the videos
        all_videos.append(data)
    return (pd.DataFrame(all_videos)) 
```
playlist items
```
playlist_id = "UUpcTrCXblq78GZrTUTLWeBw" #what we got from the response 
def get_video_ids(youtube,playlist_id):
    video_ids = []
    request = youtube.playlistItems().list(
        part="snippet,contentDetails",
        playlistId = playlist_id ,
        maxResults = 50 #as per documentation max results 50
    )
    response = request.execute()
    
    for item in response['items']:
        video_ids.append(item['contentDetails']['videoId'])
        
    next_page_token = response.get('nextPageToken') #to return all pages 
    while next_page_token is not None:
        request = youtube.playlistItems().list(
                    part="contentDetails",
                    playlistId = playlist_id ,
                    maxResults = 50,
                    pageToken = next_page_token)    
        response = request.execute()
        for item in response['items']:
            video_ids.append(item['contentDetails']['videoId'])
        
        next_page_token = response.get('nextPageToken')
        
    return video_ids 
```    
videos     
```   
 def videos_details(youtube,video_ids):
    
    all_videos = []
    
    for i in range (0,len(video_ids),50):
        request = youtube.videos().list(
        part="snippet,contentDetails,statistics",
        id=','.join(video_ids[i:i+50]) #max token per page
       )
        response = request.execute()
    
        for video in response['items']:
            req_stats = {'snippet':['title','description','publishedAt'],
                'statistics':['viewCount','likeCount','commentCount'],
                 'contentDetails':['duration']}
            video_info = {}
            video_info['video_id'] = video['id']
            for k in req_stats.keys():
                for v in req_stats[k]:
                    try:
                        video_info[v] = video[k][v]
                    except:
                        video_info[v] = None
            all_videos.append(video_info) 
    
    return pd.DataFrame(all_videos)   

```
### EDA
After getting the required statistics information like count of like(youtube disabled showing the dislike numbers) , comments and view we did some data preprocessing for the data 

Questions that we might take into consedration 

**most and least videos views**

**length distrbution for the videos**

**likes and comments comparing to the views**

**correlation between the data**

**most common words on titles and description**

**Does length of the videos affect on the views?**

* length of the videos distrbutaion to divide them into short and long videos 
 ![image](https://user-images.githubusercontent.com/60258264/216842007-ed85626d-327e-4b0d-a4ec-8268fa468059.png)
 
* distrbutaion of the likes comparing to the views most of the data records between million and 20 million views outliers mostly on short videos but people prefer most videos less than 270 sec which is around 4 to 5 minutes 
 
 ![image](https://user-images.githubusercontent.com/60258264/216842044-50c6d489-d4c1-4bb0-b968-6496e7e75e54.png)
 
* below image showing the correlation between the data like and comments obviousely impacting positivley high on the views however duration doesnt have any relation on the views cause most of the records between specific numbers 250 sec to 270 sec 
 
 
![image](https://user-images.githubusercontent.com/60258264/216842748-6b8f118b-1231-48c2-be76-f1b3aa634adc.png)

* the most watched video was dreamer song from the last world cup with 120 millions views 

![image](https://user-images.githubusercontent.com/60258264/216842850-e3173044-305a-4829-a8f6-eea67ce42f43.png)

* sentiment analysis for the common words on the videos title #world cup #fifaworld #match highlight are the most

![image](https://user-images.githubusercontent.com/60258264/216842996-f433bec7-fea9-4f48-997f-143c9638c58d.png)

### saving the data frame into database(mysql)

creating database tables after logging with the credentials 

```
#creating the tables into the database
create_table_command = """CREATE TABLE videos(
            video_id varchar(255) primary key,
            title varchar(255) not null,
            description text,
            publishedAt DATE not null,
            viewCount int(20) not null,
            likeCount int(20) ,
            commentCount int(20),
            duration varchar(255) not null,
            durationSecs int(20) not null,
            length enum ('short','long') not null
            )"""
cursor.execute(create_table_command)
```
inserting all the dataframe records into the database and make a func to update the record if exists for the primary key 

```
cols = "`,`".join([str(i) for i in dataset.columns.tolist()])

# Insert DataFrame recrds one by one.
try:
    for i,row in dataset.iterrows():
        sql = "INSERT INTO `videos` (`" +cols + "`) VALUES (" + "%s,"*(len(row)-1) + "%s)"
        cursor.execute(sql, tuple(row))
except:
    pass  
    # the connection is not autocommitted by default, so we must commit to save our changes
#cursor = conn.cursor(buffered=True) 
conn.commit()
```
The table and the records were updated succssfuley 

![image](https://user-images.githubusercontent.com/60258264/216844524-17c9ba8f-3515-4821-991c-3450bd066db7.png)

### pushing to the cloud

you can push the records on the cloud AWS by creating free account on the amazon RDS so you can save millions of records and better memory performance 

![image](https://user-images.githubusercontent.com/60258264/216844750-c192e5ab-307a-483f-b194-80fbf1ee5f8f.png)

### Next 

* we can on the automatically insert new records 
* pulling comments information to apply different text analysis with NLP

### Refrences 

https://developers.google.com/youtube/v3/docs

https://github.com/youtube/api-samples/tree/master/python

https://jingwen-z.github.io/how-to-get-a-youtube-video-information-with-youtube-data-api-by-python/

https://www.dataquest.io/blog/sql-insert-tutorial/

https://stackoverflow.com/questions/55165631/youtube-api-v3-nextpagetoken-repeats
