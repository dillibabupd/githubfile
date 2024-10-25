import streamlit as st
import googleapiclient.discovery
import pandas as pd
import re 
import isodate
from sqlalchemy import create_engine
from sqlalchemy.exc import IntegrityError

api_service_name = "youtube"
api_version = "v3"
api_key = "AIzaSyBcU2gEV96w-iBDGOnI8W5tUYnGpu80JlM"
youtube = googleapiclient.discovery.build(api_service_name, api_version, developerKey=api_key)

def channel_details(channel_id):
    request = youtube.channels().list(
        part="snippet,contentDetails,statistics",
        id=channel_id
    )
    response = request.execute()
    if response['items']:
        channel_info = response['items'][0]
        details = {
            'Channel_Name': channel_info['snippet']['title'],
            'Channel_Description': channel_info['snippet']['description'],
            'Channel_Video': channel_info['statistics'].get('videoCount', 'N/A'),
            'Channel_Views': channel_info['statistics'].get('viewCount', 'N/A'),
            'Subscription_Count': channel_info['statistics'].get('subscriberCount', 'N/A'),
            'Playlist_Id': channel_info['contentDetails']['relatedPlaylists']['uploads'],
        }
        return details
    else:
        return {"Error": "Channel not found or invalid channel ID"}

def get_channel_details(channel_id):
    request = youtube.channels().list(
        part="snippet,contentDetails,statistics,status",
        id=channel_id
    )
    response = request.execute()
    if 'items' in response and response['items']:
        channel_info = response['items'][0]
        details = {
            'Channel_ID': channel_id,
            'Channel_Name': channel_info['snippet']['title'],
            'Channel_Description': channel_info['snippet']['description'][:150],
            'Channel_Type': channel_info['snippet'].get('customUrl', 'N/A'),
            'Channel_Views': channel_info['statistics']['viewCount'],
            'Channel_Status': channel_info['status']['privacyStatus'],
            'Playlist_Id': channel_info['contentDetails']['relatedPlaylists']['uploads']
        }
        return details
    else: 
        return {"Error": "Channel not found or invalid channel ID"}

def get_playlist_details(playlist_id):
    request = youtube.playlists().list(
        part="snippet",
        id=playlist_id
    )
    response = request.execute()
    if 'items'in response and response ['items']:
        playlist_info = response['items'][0]
        details = {
            'Playlist_ID':playlist_id,
            'Channel_ID': playlist_info['snippet']['channelId'],
            'Playlist_Name': playlist_info['snippet']['title'],
        }
        return details
    else:
        return {"Error": "Playlist not found or invalid playlist ID"}

def get_video_ids(youtube,playlist_id):
    request = youtube.playlistItems().list(
        part="contentDetails",
        playlistId=playlist_id,
        maxResults=50
    )
    response = request.execute()

    video_ids=[]
    for i in range(len(response['items'])):
        video_ids.append(response['items'][i]['contentDetails']['videoId'])
        
    next_page_token=response.get('nextPageToken')
    more_pages=True

    while more_pages:
        if next_page_token is None:
            more_pages=False
        else:
            request = youtube.playlistItems().list(
                part="contentDetails",
                playlistId=playlist_id,
                maxResults=50,
                pageToken=next_page_token
            )
            response = request.execute()
            for i in range(len(response['items'])):
                video_ids.append(response['items'][i]['contentDetails']['videoId'])
                
            next_page_token=response.get('nextPageToken')
    return video_ids 

def get_comments(video_ids):
    comments = []  
    for video_id in video_ids:
        try:
            request = youtube.commentThreads().list(
                part='snippet',
                videoId=video_id,
                maxResults=50
            )
            response = request.execute()
            for item in response['items']:
                comment = item['snippet']['topLevelComment']['snippet']
                comment_id = item['id']
                comment_text = comment['textOriginal'][0:200]
                comment_author = comment['authorDisplayName']
                comment_published_at = comment['publishedAt']
                
                datetime_str = comment_published_at[:-1]  
                published_date = re.sub(r'T', ' ', datetime_str)  
                
                comments.append({
                    'Comment_ID': comment_id,
                    'Video_ID': video_id,
                    'Comment_Text': comment_text,
                    'Comment_Author': comment_author,
                    'Published_Date': published_date
                })

        except Exception as e:
            print(f"Error fetching comments for video {video_id}: {e}")
            continue

    return comments

def get_video_ids(youtube, playlist_id):
    video_ids = []
    request = youtube.playlistItems().list(
        part="contentDetails",
        playlistId=playlist_id,
        maxResults=50
    )
    response = request.execute()

    for item in response['items']:
        video_ids.append(item['contentDetails']['videoId'])

    next_page_token = response.get('nextPageToken')
    while next_page_token:
        request = youtube.playlistItems().list(
            part="contentDetails",
            playlistId=playlist_id,
            maxResults=50,
            pageToken=next_page_token
        )
        response = request.execute()

        for item in response['items']:
            video_ids.append(item['contentDetails']['videoId'])

        next_page_token = response.get('nextPageToken')

    return video_ids

def get_video_details(youtube, video_id, playlist_id):
    request = youtube.videos().list(
        part="snippet,contentDetails,statistics",
        id=video_id
    )
    response = request.execute()
    if response['items']:
        video_info = response['items'][0]
        
        duration_str = video_info['contentDetails']['duration']
        duration = isodate.parse_duration(duration_str)
        duration_sec = int(duration.total_seconds())
        
        published_date = video_info['snippet']['publishedAt']
        published_date = re.sub(r'T', ' ', published_date[:-1])
        
        details = {
            'Video_ID': video_id,
            'Playlist_ID': playlist_id,
            'Channel_ID': video_info['snippet']['channelId'], 
            'Video_Name': video_info['snippet']['title'],
            'Video_Description': video_info['snippet']['description'][:150],  
            'Published_Date': published_date,
            'View_Count': video_info['statistics'].get('viewCount', 'N/A'),
            'Like_Count': video_info['statistics'].get('likeCount','N/A'),
            'Dislike_Count': video_info['statistics'].get('dislikeCount', 'N/A'),
            'Favorite_Count': video_info['statistics'].get('favoriteCount', 'N/A'),
            'Comment_Count': video_info['statistics'].get('commentCount', 'N/A'),
            'Duration': duration_sec,
            'Thumbnail': video_info['snippet']['thumbnails']['high']['url'],
            'Caption_Status': video_info['contentDetails'].get('caption', 'N/A')
        }
        
        return details
    else:
        return {"Error": "Video not found or invalid video ID"}
       
option = st.selectbox('YouTube Project', ('Scraping', 'Migrade', 'Question'))
if option == 'Scraping':
    channel_id = st.text_input('Enter the channel ID')
    
    if channel_id and st.button('Scrape'):
        channel_in= channel_details(channel_id)
        st.write(channel_in)
       
engine = create_engine('mysql+mysqlconnector://root:12345678@localhost/youtube')

if option == 'Migrade':
    channel_id = st.text_input('Enter the channel ID')
    
    if channel_id and st.button('Insert Data'):
        def insert_tables(): 
            channel_data = get_channel_details(channel_id)
            if channel_data:
                channeldf = pd.DataFrame([channel_data])
                
                try:
                    channeldf.to_sql('channel', engine, if_exists='append', index=False)
                except IntegrityError:
                    st.warning(f"Channel ID: {channel_id} is already exists. Skip insertion.")

                playlist_id = channel_data['Playlist_Id']
                playlist_data = get_playlist_details(playlist_id)
                
                if playlist_data:
                    playlistdf = pd.DataFrame([playlist_data])
                    playlistdf.to_sql('playlist', engine, if_exists='append', index=False)
                    
                    video_ids = get_video_ids(youtube, playlist_id)
                    for video_id in video_ids:
                        video_data = get_video_details(youtube, video_id, playlist_id)
                        if video_data:
                            videodf = pd.DataFrame([video_data])
                            videodf.replace('N/A', 0, inplace=True)  
                            videodf.to_sql('video', engine, if_exists='append', index=False)
                    
                    comment_data = get_comments(video_ids)
                    if comment_data:
                        commentdf = pd.DataFrame(comment_data)
                        commentdf.to_sql('comment', engine, if_exists='append', index=False)

        insert_tables()
    st.title('Tables data inserted')

    table_option = st.sidebar.selectbox('Tables', ('channel', 'playlist', 'comment', 'video'))
    def data_from_channel(): 
        query = 'select *  from channel'  
        data = pd.read_sql(query, engine)  
        return data
    if table_option == 'channel':
        st.title('Table From Channel')
        data = data_from_channel()
        st.dataframe(data)

    def data_from_playlist(): 
            query = 'select * from playlist'  
            data = pd.read_sql(query, engine)  
            return data
    if table_option == 'playlist':
        st.title('Table From Playlist')
        data = data_from_playlist()
        st.dataframe(data)
    
    def data_from_comment():  
        query = 'select * from comment'  
        data = pd.read_sql(query, engine)  
        return data 
    if table_option == 'comment':
        st.title('Table From Comment')
        data = data_from_comment()
        st.dataframe(data)      
            
    def data_from_video(): 
            query = 'select * from video'  
            data = pd.read_sql(query, engine)  
            return data 
    if table_option == 'video':
        st.title('Table From video')
        data = data_from_video()
        st.dataframe(data) 
               
if option == 'Question':
    question_option = st.sidebar.selectbox('Qustion',(1,2,3,4,5,6,7,8,9,10))
    if question_option == 1:
        def query_no_1():
            query = '''SELECT video.video_name,playlist.channel_id
                        FROM video
                        INNER JOIN playlist 
                        ON video.playlist_id = playlist.playlist_id'''
            data = pd.read_sql(query, engine)    
            return data
        st.title('What are the names of all the videos and their corresponding channels?')
        data = query_no_1()
        st.dataframe(data) 
    elif question_option == 2:
        def query_no_2():
            query = '''SELECT channel.channel_id, channel.channel_name, COUNT(video.video_id) AS video_count
                        FROM channel 
                        JOIN video  ON channel.channel_id = video.channel_id
                        GROUP BY channel.channel_id, channel.channel_name
                        ORDER BY video_count DESC'''
            data = pd.read_sql(query, engine)
            return data
        st.title('Which channels have the most number of videos, and how many videos do they have?')
        data = query_no_2()
        st.dataframe(data)
    elif question_option == 3:
        def query_no_3():
            query = '''SELECT video.video_name, channel.channel_name, video.view_count
                        FROM video
                        INNER JOIN channel 
                        ON video.channel_id = channel.channel_id
                        ORDER BY video.view_count DESC LIMIT 10'''
            data = pd.read_sql(query,engine)
            return data
        st.title('What are the top 10 most viewed videos and their respective channels?')
        data = query_no_3()
        st.dataframe(data)
    elif question_option == 4:
        def query_no_4():
            query = '''SELECT video.video_name, COUNT(comment.comment_text) AS Comment_count
                        FROM video
                        INNER JOIN comment
                        ON video.video_id = comment.video_id
                        GROUP BY video.video_name'''
            data = pd.read_sql(query,engine)
            return data
        st.title('How many comments were made on each video, and what are their corresponding video names?')
        data = query_no_4()
        st.dataframe(data)
    elif question_option == 5:
        def query_no_5():
            query='''SELECT video_name, like_count, dislike_count 
                    FROM video'''
            data = pd.read_sql(query,engine)
            return data
        st.title('What is the total number of likes and dislikes for each video, and what are their corresponding video names?')
        data = query_no_5() 
        st.data_editor(data)  
    elif question_option == 6:
        def query_no_6():
            query='''SELECT video_name, like_count, dislike_count 
                    FROM video'''
            data=pd.read_sql(query,engine)
            return data
        st.title('What is the total number of likes and dislikes for each video, and what are their corresponding video names?')
        data = query_no_6()
        st.dataframe(data)
    elif question_option == 7:
        def query_no_7():
            query='''SELECT channel.channel_name, SUM(video.view_count) as Total_view_Count
                    FROM channel
                    INNER JOIN video 
                    ON channel.channel_id = video.channel_id
                    GROUP BY channel.channel_name'''
            data = pd.read_sql(query,engine)
            return data
        st.title('What is the total number of views for each channel, and what are their corresponding channel names?')
        data =query_no_7()
        st.dataframe(data)
    elif question_option == 8:
        def query_no_8():
            query='''SELECT DISTINCT channel.channel_name
                    FROM channel
                    INNER JOIN video 
                    ON channel.channel_id = video.channel_id
                    WHERE YEAR(video.published_date) = 2024'''
            data = pd.read_sql(query,engine)
            return data
        st.title('What are the names of all the channels that have published videos in the year 2022?')
        data = query_no_8()
        st.dataframe(data)
    elif question_option == 9:
        def query_no_9():
            query='''SELECT channel.channel_name, 
                    AVG(video.duration) AS average_duration
                    FROM channel
                    INNER JOIN video 
                    ON channel.channel_id = video.channel_id
                    GROUP BY channel.channel_name'''
            data = pd.read_sql(query,engine )
            return data
        st.title('What is the average duration of all videos in each channel, and what are their corresponding channel names?')
        data = query_no_9()
        st.dataframe(data)
    elif question_option == 10:
        def query_no_10():
            query='''SELECT video.video_name, channel.channel_name, video.comment_count
                    FROM video
                    INNER JOIN channel
                    ON video.channel_id = channel.channel_id
                    ORDER BY video.comment_count DESC LIMIT 10'''
            data = pd.read_sql(query,engine)
            return data 
        st.title('Which videos have the highest number of comments, and what are their corresponding channel names?')
        data = query_no_10()
        st.dataframe(data)
        
