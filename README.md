## Recommend me a band please!

👋 This is a short project I put together that loads in my most recent Spotify listening data, authenticates and makes a call to the Spotify API and prints out a related artist from the most recent listened to list along with their top songs. I was curious to see if the Spotify Explore functionality was really just a round up of related artists or if it was more complex than that. 

In the future I might make this a bit more extensible so other folks could load in their data, but really this was just an experiment to play with Spotify's API and see what kind of data it returned. 


### Loading in the JSON file from Spotify:

```python

# import what i need for this cell 
import pandas as pd
import json
from datetime import datetime
import random

# load in the streaming music history i requested from Spotify which is the last year (March 2023 - March 2024)
with open("StreamingHistory_music_0.json") as f:
    json_data = json.load(f)

# create a dataframe from that loaded in json object 
streaming_music_df = pd.DataFrame(json_data)

# rename the columns 
streaming_music_df.columns = ["end_time", "artist_name", "track_name", "ms_played"]

```

### Cleaning it a little so I can visualize the minutes per band. I only want to print out a related arist from someone i've listened to a fair amount of time 

```python
# i only want my most recent data for now 
streaming_music_df_2024 = streaming_music_df[
    streaming_music_df["end_time"]
    >= datetime.strptime("2024-01-01", "%Y-%m-%d").strftime("%Y-%m-%d")
]

# create a dataframe that counts the number of times an artist appears in my streaming list 
artist_counts_df = streaming_music_df["artist_name"].value_counts().reset_index()
# rename the columns 
artist_counts_df.columns = ["artist_name", "play_count"]
# i only want the list of artists that i've listened to a few times 
artist_counts_df = artist_counts_df[artist_counts_df["play_count"] >= 5]
# order the artists
artist_counts_df = artist_counts_df.sort_values(by="play_count", ascending=False)


# create a new dataframe that sums the ms_played per artist_name
artist_playtime_df = (
    streaming_music_df_2024.groupby("artist_name")["ms_played"].sum().reset_index()
)

# add a new column that converts ms to minutes & sort the values. only include artists with at least 20 minutes to weed out noise 
artist_playtime_df["minutes_played"] = round(artist_playtime_df["ms_played"] / 60000)
artist_playtime_df = artist_playtime_df.sort_values(by="minutes_played", ascending=False)
artist_playtime_df = artist_playtime_df[artist_playtime_df["minutes_played"] >= 20]

# create a list of the unique artists that i can use in other cells 

# list comprehensions - https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions
unique_artist_list = [name.lower() for name in artist_playtime_df["artist_name"]]
```


<img width="1002" alt="Screenshot 2024-04-08 at 7 40 19 AM" src="https://github.com/ohitsmekatie/give-me-a-band/assets/9855295/a7677472-16dc-4eab-ba73-3eadcb29aa62">


### Authorize & get tokens. My client id/secret is saved to the project itself as a hidden variable 

```python

# https://developer.spotify.com/documentation/web-api

import base64
from requests import post, get
import json

def get_spotify_token():
    auth_string = spotify_client_id + ":" + spotify_client_secret
    auth_bytes = auth_string.encode("utf-8")
    auth_base64 = str(base64.b64encode(auth_bytes), "utf-8")

    url = "https://accounts.spotify.com/api/token"
    headers = {
        "Authorization": "Basic " + auth_base64,
        "Content-Type": "application/x-www-form-urlencoded",
    }

    data = {"grant_type": "client_credentials"}
    result = post(url, headers=headers, data=data)
    json_result = json.loads(result.text)
    token = json_result["access_token"]
    return token

def get_auth_header(token):
    return {"Authorization": "Bearer " + token}

# get the spotify token and save as variable which is needed for each function
token = get_spotify_token()

```

### Next, search to get the artist id which is how you look up artists w/ the Spotify Web API

```python

def search_for_artist(token):
    url = "https://api.spotify.com/v1/search"
    headers = get_auth_header(token)

    artist_id_list = []

    for name in unique_artist_list:
        query = f"?q={name}&type=artist&limit=1"
        query_url = url + query
        result = get(query_url, headers=headers)
        json_result = json.loads(result.text)

        artist_id = json_result["artists"]["items"][0]["id"]
        artist_id_list.append(artist_id)

    return artist_id_list


# Call the function and store the result
artist_id_list = search_for_artist(token)

```

### Get related artists and print 

```python

related_artist_list = []
final_list = []

for artist_id in artist_id_list:

    url = f"https://api.spotify.com/v1/artists/{artist_id}/related-artists"
    headers = get_auth_header(token)
    result = get(url, headers=headers)
    #json_result = json.loads(result.text)

    json_result = json.loads(result.text)["artists"][0]
    # json_result = json_result[0]
    artist_name = json_result["name"]

    related_artist_list.append(artist_name)

    final_list.extend(related_artist_list)
    related_artist_list = []

# get 1 random value from final_list
random_artist = random.choice(final_list) # if final_list else None

print(random_artist)

```


### Get related artists top songs 

```python


url = "https://api.spotify.com/v1/search"
headers = get_auth_header(token)
query = f"?q={artist_name}&type=artist&limit=1"
query_url = url + query
result = get(query_url, headers=headers)

# the spotify search result based on the artist name. contains metadata about that artist
json_result = json.loads(result.text)["artists"]["items"]
if not json_result:
    return []  # Return an empty list if no artist is found
json_result = json_result[0]
artist_id = json_result["id"]

top_track_url = (
    f"https://api.spotify.com/v1/artists/{artist_id}/top-tracks?country=US"
)
top_track_result = get(top_track_url, headers=headers)
top_track_json_result = json.loads(top_track_result.text)["tracks"]

# Extract the list of song names from the top tracks
song_list = [track["name"] for track in top_track_json_result]
for song in song_list:
    print(song)
```


Everything can be found in [this](https://app.hex.tech/katiesipos/app/bc8af7ef-fafb-44b9-8df2-a22b066ce4f7/latest) Hex notebook. 
