---
title: Pushing Content With Python - A Long Guide to a Short Script
date: 2025-09-15 18:01
categories: [Development,Python Scripts]
tags: [scripts, programming, scraping, html, api, c#, development, python, json, web, data, asp.net core, .net]
---

## Seeding Data with an API

I wrote in a [previous post]({% post_url 2025-09-10-A-Python-Scraper %}) about using a Python scraper script to turn an existing website's news archive into a .json structured data archive. This (mercifully shorter) follow up describes another python script which I used to transform and seed that data via post requests to a Web API. 

As with the previous post, there were plenty of approaches I could take with this - such as creating an endpoint for the WebAPI which takes the entire json payload, or seeding it all internally by opening the file programmatically - but it's fun to try things out in python. These one-time-only scripts are a great vehicle to do so, and to my mind it's better for long-term maintenance not to create needless 'one time only' functions or endpoints inside a long-lived codebase.

The bit that made this fun was the image handling. This API is designed to receive and store an image filename, which in the normal course of events (i.e. when a new News post is created within the CMS) is provided to it *after* an image has been uploaded and processed into multiple web-friendly sizes of .webp. This script has the extra challenge of replicating that process. 

## The Mission
1. Transform our structured data into the correct format for the API's Create endpoint
2. Download images from urls in the data, resize and compress them, upload them to Azure Blob Storage and supply a valid filename to as part of the payload

## Recap: The Source Data & the Destination

I've got a json file which looks like this:
```json
[
  {
    "title": "A Blog About Python",
    "date": "2025-09-01",
    "image": "https://burtcher-old/image1.file",
    "body": "<p>Here's a paragraph!</p><p>Here's another!</p>",
    "contentBlocks": [
      {
        "content" : "<h2>Look At This Stuff</h2><p>Isnt it neat?<p><p>Wouldn't you say my...</p> ",
        "img-src" : "https://burtcher-old/image2.file",
        "img-pos" : "left",
        "img-size": "small",
        "order": 0
      },
      {
        "content" : "<p><em>...collection's complete?</em>",
        "img-src" : "https://burtcher-old/image3.file",
        "img-pos" : "center",
        "img-size": "right",
        "order": 1
      }
    ]
  }
  ... etc
]
```
and I want to get that data into this API endpoint:
```csharp
[HttpPost("NewsArticle")]
public async Task<IActionResult> PostNewsArticle([FromBody] NewsPageDto newsPageDto)
{
    if (newsPageDto == null)
    {
        return BadRequest("NewsArticle data is required.");
    }
    int newId = await _contentService.AddNewsArticle(newsPageDto);
    if (newId > 0)
    {
        return CreatedAtAction(nameof(GetNewsArticle), new { id = newId }, newId);
    }
    return BadRequest("Failed to create news article.");
}
```
If you're unfamiliar with ASP.NET Core's model binding process, it's not complex: this endpoint is expecting a Post request with a body that can be mapped to the  dto below by matching key/value pairs. Essentially the names and value types need to match or the NewsPageDto will not be correctly bound and a 400 error will be returned. 

```csharp
public class NewsPageDto 
{
  public DateTime Date { get; set; }
  public string Title { get; set; } 
  public string Slug { get; set; } 
  public string ImageThumbnail { get; set; } 
  public string ImageMain { get; set; } 
  public List<ContentBlockDto> Content { get; set; } = [];
}

public class ContentBlockDto
{
  public string? ImageFile { get; set; }
  public ImageSize ImageSize { get; set; }
  public ImagePosition ImagePosition { get; set; }
  public string? Content { get; set; }
  public int Priority { get; set; }
}
```
_See the previous post for more about the `List<ContentBlockDto>` property if it's unfamiliar._
{: .syntax-caption}

You'll no doubt notice that our data doesn't currently map neatly onto the expected dto. Let's get to work.

## Packages
Once again, the fun of python scripts is using packages and libraries with wild abandon. What would be challenging becomes trivial. Here's what I used for this:

```py
import json
import requests
from bs4 import BeautifulSoup
import os
import io
from dotenv import load_dotenv
from PIL import Image
from azure.storage.blob import BlobServiceClient
```
I used **json**, **requests** & **BeautifulSoup** in the [previous exercise]({% post_url 2025-09-10-A-Python-Scraper %}) - they're for encoding/decoding json, making http requests and parsing/manipulating HTML respectively.

We didn't use **os** and **io** previously: these are Python modules designed to interact with the operating system the script is running on (os) and handle file streaming, or reading and writing to files (io) respectively. The former I only use with `dotenv`, and the latter I use for creating new image files.

**dotenv** is a [great library](https://pypi.org/project/python-dotenv/) for treating a secrets file like an environment variables store. In ASP.NET Core, I work a lot with configuration files (like appsettings.json) and environment variables (paricularly in the Azure App Service / Containers) which lets me pull in secrets at runtime without commiting them to the repository. dotenv lets me do something similar here, which prevents me from directly pasting in sensitive things like my Azure Storage Connection String.
To use it, I create a file called `.env` and populate it as follows:
```
# Development settings
API_TOKEN=xyz
AZURE_STORAGE_CONNECTION_STRING=xyz
AZURE_BLOB_BASE_URL=xyz
```
Then dotenv, via the `load_dotenv()` function, lets me access these values by using the key like an environment variable (see the [Constants](#constants) section below for how to use them).

**PIL** comes from [Pillow](https://pillow.readthedocs.io/en/stable/), a fork of the now-defunct **P**ython **I**maging **L**ibrary. It does loads of stuff, and crucially for us it can resize and compress images, as well as save them to different formats.

**azure.storage.blob** is the Azure Blob Storage SDK in python form. It's near identical to the .NET version (hooray!) so hyper simple to use. We'll use this to upload our images once they've been reformatted.

## Constants & Dictionaries
There's a bit of setup in this script - I like to start with a good, clear list of constants so I can set 'em and forget 'em.

The top few are just strings. When I start scripting I don't really know how many times I'll end up referencing these, and if I end up starting with, say, a development endpoint and then switching to a production endpoint, this makes it super easy to do so: 

```py
ORIGIN = "https://burtcher-old.net"
API_BASE = "https://burtcher-new.net"
NEWS_ENDPOINT = "/api/content/news"
```

The next lot are loaded in from my `dotenv` package (and could easily be real environment variables), so I can keep the secrets out of the script itself. All I'm doing here is referring to those secrets via their key, so the value is stored in my constant.

```py
load_dotenv() # I run this first to activate the .env file
TOKEN = os.getenv("API_TOKEN")
BLOB_CONNECTION_STRING = os.getenv("AZURE_STORAGE_CONNECTION_STRING")
BLOB_BASE = os.getenv("AZURE_BLOB_BASE_URL")
```
Next up are my image size array & dictionaries. The size array is self explanatory - I want to generate versions of each extracted image for each width - but the dictionaries might need a little more unpacking if you're not super into Enums like I am.
```py
SIZES = [1920, 1600, 1280, 1024, 800, 640, 320]
SIZE_DICTIONARY = {
    "small": 0,
    "medium": 1,
    "large": 2
}
POS_DICTIONARY = {
    "left": 0,
    "right": 1,
    "center": 2,
    "top": 3,
    "bottom": 4
}
CONTENT_BLOCK_TYPE = {
    "standard": 0,
    "youtube": 1,
    "flipbook": 2
}
```
The dictionaries all map a set of strings to a number. Why? If you look back up at the `ContentBlockDto`, you'll see that the properties for ImageSize and ImagePosition aren't `string` types but `ImageSize` and `ImagePosition` *types*. In my API application, they are Enums, currently my preferred way of enforcing (and remembering) any pattern where value should be one of a limited, fixed and short list of options. They're defined like this:

```csharp
public enum ImagePosition
{
  Left,
  Right,
  Center
}
public enum ImageSize
{
  Small = 0, // These are optional. If you miss them out, the numbers are implied.
  Medium = 1,
  Large = 2,
}
```
This means that while they look like Types, they basically translate to sequential `int` values. So when we're mapping our data to an API friendly object, we don't want to send the `string` values, we want to send the corresponding `int`. These dictionaries will let us do that.

> In my opinion, defining and working with Enums is helpful in three distinct ways:
   1. Type/Value Safety. If I set a property type to be ImagePosition, my intellisense/completion options will always limit me to the only possible values.
   2. Speed. Working with ints in memory is faster than working with strings.
   3. My brain. I could just use ints, sure. But will I always struggle to remember if 0 is Large or Small? Yes, I will, so I don't.
{: .prompt-emphasis }

Lastly, we have the Blob Container object. This is all you need to do to access a specific container in Blob Storage using the SDK:

```py
blob_service = BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
container_client = blob_service.get_container_client("images")
```

## Logic

We're going to loop through our .json file and process each data point in turn, using, at most, two functions:
```py
def import_news_article(item):
def process_image(url,filename):
```
### import_news_article(item)

#### Text Properties

The mapping of text data doesn't take much ceremony. **Note** that the Date property is just submitted as text - ASP.NET Core model binding takes care of the DateTime conversion because we took the time to format the string in a way that it recognises.
```py
dto = {
  "date": item["date"],
  "title": item["title"],
  "body": item["body"],
  "additionalContent": []
}
```
I'm leaving the `additionalContent` array empty for now, because what we want to do is check for any images we might need to upload on a block by block basis. 

#### Preparing Image Filenames

First, though, the main image. If the item we're trying to submit has one, let's create a filename for it, and process it. (All the images are going to process the same way, so if that's most interesting to you, [jump to the processing section](#process_imageurlfilename) for the detail on that.

```py
if item["image"]:
  imageFilename =  f"newsimage_{item['date']}" # What, no file extension?! I'll explain on the way.
  dto["imageMain"] = imageFilename
  dto["imageThumbnail"] = imageFilename
  process_image(item["image"], imageFilename)
```
_Note the issue here: if two articles have the same date, this process will cause conflicts/overwrites later. I'm only getting away with it because I know for a fact there aren't any matching dates in my source data. The other way would be to generate a unique id for the news article by creating it, then use that to create a unique filename, then update the newsarticle. Not a bad pattern but I don't need to do it in this case!_
{: .syntax-caption}

#### Extracting Images from the Body HTML
Next, our body content. While we know from our previous exercise that the ContentBlocks specifically designated images, the main body content can nonetheless, within our lovingly prepared `<p></p>` tags, contain `<img>`s. We can't just transpose these as-is; we'll either be hotlinking from unknown sources or, more likely, linking to locally hosted images which will all disappear when we finish transitioning the website. So we need more processing logic, which can be vastly improved by bringing BeautifulSoup back into action.

```py
if item["body"]:
  imgcount = 0
  html = item["body"]
  soup = BeautifulSoup(html, "html.parser")
  for img in soup.find_all("img"):
      href = img.get("src")
      filename = f"newsimage_{item['id']}_inlineimg{imgcount}";
      process_image(ORIGIN + href, filename)
      item["body"] = item["body"].replace(href, f"{BLOB_BASE}/{filename}-640.webp")
      imgcount += 1
```
_It's possible to do this without soup by just manipulating text, but it's less fun and there are more tedious edge cases to try and figure out._
{: .syntax-caption }

What we do here is:
 - Extract the html from the body property and parse it with BeautifulSoup
 - Find all the img tags and loop through them
 - Extract the img's src attribute into the href variable
 - Create a filename based on the index number of the found image
 - Process (upload!) the image by prepending the relative href with the ORIGIN domain
 - Finally replace the href in the text with a new url reflective of the file we just uploaded

#### ContentBlocks
And we execute a similar procedure for appending ContentBlocks, making sure that images are upload en-route. We loop through the source array, setting the imageSize and imagePosition ints/enums using our dictionaries, and renaming the uploaded image as we have with the others. Then we append our `blockDto` object to the `additionalContent` array we initialised earlier.

```py
if item["contentBlocks"]:
  for block in item["contentBlocks"]:
      blockDto = {
          "content": block["content"],
          "imageSize": SIZE_DICTIONARY.get(block["img-size"], 0),
          "imagePosition": POS_DICTIONARY.get(block["img-position"], 0),
          "priority": block["order"],
      }

      # Process Image
      if block["image"]:
          blockImage = f"newsarchive_{item['id']}_block{block['order']}"
          blockDto["imageFile"] = blockImage

      dto["additionalContent"].append(blockDto)
```

#### Posting to the Endpoint
Now the data object is assembled and the images are all processed, we can push it up to the API:
```py
response = requests.post(
    f"{API_BASE}{NEWS_ENDPOINT}",
    headers={
      "Authorization": f"Bearer {TOKEN}",
      "Content-Type": "application/json"
    },
    json=dto,
    verify=False
)

if response.status_code == 201:
    newsArticleId = response.text
```
_Note the inclusion of the Bearer token for Auth!_
{: .syntax-caption }

### process_image(url,filename)

We want to process all images in the same way, producing the same set of results and filename patterns, so our front-end code can be easily maintained. To recap, that means we want to:
- Grab the image using its current url
- Save an 'original size' jpg with light compression and upload it to Blob Storage
- Produce a set of sequentially lower resolution .webp images for responsive display, and save them all the Blob Storage

The **Pillow** library and **io** make manipulating the image pretty simple:

```py
resp = requests.get(url) # Get the Image with requests
resp.raise_for_status() # Throw an error if the http response is bad
img = Image.open(io.BytesIO(resp.content)) # Create an in-memory, file-like object with io, then open it as an image with Pillow
buffer = io.BytesIO() # Open an empty file buffer with io
img.save(buffer, format="JPEG", quality=85) # Compress the image with Pillow
buffer.seek(0) # rewind the buffer's cursor to the start (housekeeping)
```
_I later learned that I really should have called `ImageOps.exif_transpose(img).convert("RGB")` before saving the image as a jpeg, as a format with an alpha channel (such as a png) would throw an error. Again, good for general but not really applicable here: I knew everything served up by the previous CMS was a jpg._
{: .syntax-caption }

Then the **Azure Storage SDK** makes the upload pretty easy too.
```py
filename = f"{filename}.jpg" # filename was passed into the function with the URL

try:
    container_client.upload_blob( # Upload the data to the container client we created earlier
        name=filename,
        data=buffer,
        overwrite=True
    )
except Exception as e:
    print(f"Failed to upload Original: {filename}: {e}")
```
Great! So that's the "original size" image uploaded as a reasonable jpg. Now we want to create our collection of reduced size images, destined to be deployed as a `srcset` in our responsive design. It's the same thing again, but with a resize step as we loop through our `SIZES` array:

```py
for width in SIZES:
  # maintain aspect ratio
  ratio = width / float(img.width)
  height = int(img.height * ratio)
  resized = img.resize((width, height), Image.Resampling.LANCZOS)

  # convert to webp in memory
  buf = io.BytesIO()
  resized.save(buf, format="WEBP", quality=85)
  buf.seek(0)

  # generate filename
  filename = f"{imageSrcRoot}-{width}.webp"

  try:
    container_client.upload_blob(
        name=filename,
        data=buf,
        overwrite=True
    )
    print(f"Uploaded {filename}")
  except Exception as e:
    print(f"Failed to upload {filename}: {e}")
```

## Putting It All Together

Once again, we can use the `if __name__ == "__main__":` syntax to make our script runnable from the command line.

The only bit left to add is a `with open()` command to open the news.json file, and a `json.load()` call to parse it into a data object.

Then - a final bit of anti-headache code - we make an initial GET request to our News endpoint to grab a list of existing News Articles. That way we can do a title comparison to make sure we're not uploading the same article multiple times for some reason. It's not foolproof but it's good enough for this script.

The finished thing therefore looks like this:

```py
if __name__ == "__main__":
  limit = 100
  processed = 0
  existing_articles = []

  # Call the api and get the existing news items
  response = requests.get(f"{API_BASE}{NEWS_ENDPOINT}", headers={
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
    }, verify=False,params={"page":1,"pageSize":1000})

  # Catch any existing article titles and put them in a collection
  if response.status_code == 200:
    existing_articles = response.json()
    existing_article_titles = [article["title"] for article in existing_articles["items"]]

    # Open the structure data file and parse it as an object:
    with open("news.json","r",encoding="utf-8",errors="replace") as f:
      news_data = json.load(f)

    # Loop through the items up to a specific limit and import each one.
    for item in news_data:
      if processed >= limit:
        print(f"Reached processing limit of {limit}, stopping.")
        break
      if item["title"] not in existing_article_titles:
        import_news_article(item)
        processed += 1
      else:
        print(f"Article already exists: {item['title']}, skipping.")
  else:
    print(f"Failed to retrieve existing articles: {response.status_code}, {response.text}")
```

## Wrapping Up
It was fun to write this script and again, it took a lot less time than writing out a little guide to it - but writing the guide is a great way to commit it the core ideas to memory.

Python has some fantastic libraries, and for this kind of 'utility' script or one-shot, it makes for an extremely versatile platform. 

There's many ways to skin a cat, as they say, and posting data to an API is one of those cats. If your native tongue is too cumbersome, or you need a cleanser, maybe consider skinning your next cat with python. Doing this kind of content porting is a bit of a chore, so livening it up with some learning of an unfamiliar syntax is a great way to make it into a positive.

Thanks for reading! Feel free to use the social links to badger me if you are a python person and spot something nightmarishly egregious in what I've written.
